---
title: 'Kubernetes Backups Velero'
date: '2025-11-17'
draft: true
comments: true
tags:
  - homelab
  - kubernetes
---

Recently, I started a project to migrate my current Homelab infrastructure to a new cluster. My goal was to perform some hardware upgrades, reinstall my server with a new infrastructure setup (Incus + Terraform) and get my home services up and running in the new cluster as soon as possible.

The main problem I faced was: how to migrate the data for my stateful Kubernetes applications to the new cluster?

I run several stateful applications that store their data locally in persistent volumes. This data ranges from simple config files to actual user data. I wanted a way to backup this data and then restore it in my new cluster. The goal was to not loose any data during the migration.

To solve this, I used Velero to backup the existing data to an external Object Storage (Azure Blob Storage). Then, I deployed my new Kubernetes cluster, installed Velero on it, and restored the backup to download all the data from the Object Storage.

It worked really well! But due to some specifics of my Homelab use case, it was a bit harder to setup than the usual "getting started". Given that, I decided to write about the whole process.

In this post, I'll walk you through the process of backing up stateful Kubernetes applications and restoring them in a new cluster using Velero and FluxCD. Some of the details here will only apply to my kind of setup (a on-prem Kubernetes cluster with `local-path-provisioner`), but the general idea and usage of Velero will stay the same regardless of your storage setup.

## What is Velero?

Velero is a set of tools to **back up and restore** Kubernetes clusters. It can take back ups of the cluster resources, including the state of persistent volumes, and then upload them to an external Object Storage such as S3, Azure Blob Storage and many others.

It can take backups of the manifests for resources, but we're not interested in that feature because we use GitOps. So we'll focus on the persistent volume backup capabilities.

Velero provide several ways of creating backups of the persistent volumes. If you're running on the cloud, you can use the cloud provider's api to take snapshots of the volumes. If your CSI driver supports snapshotting, it can also do that. Since I'm using Racher's local-path-provisioner, none of those options fit my use case. Luckily, Velero provides a file system backup feature, which acts directly at the filesystem level.

Velero works by providing several CRDs to represent backups and their operations. We can then interact with Velero by creating and manipulating those resources in our Kubernetes cluster.

A key thing is that Velero uses the Object Storage as the source of truth, not the cluster resources. This means that if there's a backup in the Object Storage but no custom `Backup` resource in the cluster to represent it, Velero will create it. This is perfect for migration use cases, as we can simply install Velero in our new cluster, point it to the Object Storage with the existing backups and let the controller create the `Backup` resources automatically.

## Setting up the infrastructure

We'll use Azure Blob Storage in this post, but you can use any of the [providers supported by Velero](https://velero.io/docs/v1.17/supported-providers/).

We'll keep things simple and use storage account access keys for authorization, but you can (and should!) setup an Entra ID service principal to have a more fine grained control over the access Velero has to your storage account.

Here's the bicep template that I used to provision all required resources for this migration. We'll create a storage account and a container (aka: bucket) that will store the backup data.

```bicep {filename=main.bicep}
@description('The location in which to deploy the resources.')
param location string = resourceGroup().location

@description('The name of the storage container.')
param storageContainerName string = 'velero'

@description('The name of the storage account.')
param storageAccountName string = 'homelab${uniqueString(resourceGroup().id)}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2025-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_ZRS'
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}

resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2025-01-01' = {
  name: 'default'
  parent: storageAccount
}

resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2025-01-01' = {
  name: storageContainerName
  parent: blobServices
  properties: {
    publicAccess: 'None'
  }
}

output storageAccountName string = storageAccount.name
```

To create the resources, we just need to create a resource group and apply the deployment defined in our Bicep template. In my case I'll use the `chilecentral` region, but you should adjust this based on your requirements.

```shell
$ az group create --name homelab --location chilecentral
$ az deployment group create -g homelab -f main.bicep
```

Once the deployment is complete, we can fetch the storage account ID and one of its keys. We'll put the key value in a credentials file that we'll then use to create a secret.

```shell
$ AZURE_STORAGE_ACCOUNT_ID=homelabn644edmi6woce
$ AZURE_STORAGE_ACCOUNT_ACCESS_KEY=$(az storage account keys list --account-name $AZURE_STORAGE_ACCOUNT_ID --query "[?keyName == 'key1'].value" -o tsv)
$ cat << EOF  > ./credentials-velero
AZURE_STORAGE_ACCOUNT_ACCESS_KEY=${AZURE_STORAGE_ACCOUNT_ACCESS_KEY}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF
```

## Installing Velero

The next step is to install the Velero components in our existing cluster. The Velero CLI provides a `velero install` command that conveniently generates all manifests and applies them to the cluster. But since I'm using Flux, I opted to stay within the GitOps flow and install Velero declaratively by creating a `HelmRelease` resource.

- Create env file and create the secret
- Create the helm release manifest
- Create the kustomization

At the end, we should be able to verify that we have a `BackupStorageLocation` custom resource in the `Available` state in our cluster.

```shell {hl_Lines=9}
$ kubectl describe backupstoragelocations.velero.io -n velero default
Name:         default
Namespace:    velero
...
Kind:         BackupStorageLocation
...
Status:
...
  Phase:                 Available
```

## File System Volume Backups

I'm using Rancher's `local-path-provisioner` as my storage class, and it does not support snapshotting of persistent volumes. Because of this, we'll have to resort to Velero's File System Volume Backups feature, which creates a backup of the filesystem contents of the PV, as opposed to a block-level snapshot.

By default, Velero does not take file system volume backups. We need to explicitly opt-in for the feature for each volume we want to backup using this feature. We do that by annotating all pods with the mounted volume with the annotation `backup.velero.io/backup-volumes` and setting its value to a comma-separated list of mounted volumes that we want to backup. For example, here's my mealie deployment manifest:

```yaml {hl_Lines=[18,31]}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie
  namespace: mealie
  labels:
    app.kubernetes.io/name: mealie
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: mealie
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mealie
      annotations:
        backup.velero.io/backup-volumes: data
    spec:
      containers:
        - name: mealie
          image: "ghcr.io/mealie-recipes/mealie:latest"
          ports:
            - name: http
              containerPort: 9000
              protocol: TCP
          volumeMounts:
            - mountPath: /app/data
              name: data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mealie-data
```

## Creating the backup

Once all deployments are annotated for using file system volume backup, we can create the actual `Backup` resource with Velero. After all the setup we've done so far, this is actually the simplest step. We simply run `velero backup create <name>` and specify which namespaces should be included in the backup. In my case, I wanted to backup the applications from the `media` and `app` namespaces.

```shell
$ velero backup create apps --include-namespaces media,mealie
```

After creating the backup, we should be able to see that a new `Backup` custom resource is present in the `velero` namespace. We can use `velero backup describe <name>` to see details about the backup and check its state. After a while, it should have the `Completed` state.

```shell {hl_Lines=13}
$ kubectl get backups.velero.io -n velero
NAME   AGE
apps   7d
$ velero backup describe apps
velero backup describe apps
Name:         apps
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  velero.io/resource-timeout=10m0s
              velero.io/source-cluster-k8s-gitversion=v1.29.3+k3s1
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=29
Phase:  Completed
Namespaces:
  Included:  media, mealie
  Excluded:  <none>
...
```

As a last check, we can also verify that the blobs were created in out Azure storage container:

```shell
$ az storage blob list --account-name $AZURE_STORAGE_ACCOUNT_ID --container velero -o table
```

Notice that we have a `velero` and a `kopia` directory. This is because Velero relies on Kopia to perform the file system backups.

With this, we have successfully created a backup of our persistent volumes. Now, we are free to tear down our existing Kubernetes cluster and deploy a new one.

## Restoring the backups
