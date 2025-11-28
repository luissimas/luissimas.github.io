---
title: 'Migrating Kubernetes clusters with Velero and Flux'
date: '2025-11-17'
draft: true
comments: true
tags:
  - homelab
  - kubernetes
---

In my Homelab Kubernetes cluster, I run several stateful applications that store their data locally in persistent volumes. Recently, I wanted to migrate my old cluster to a new one, with improved hardware and better infrastructure management. I use a GitOps workflow with Flux, so I was confident that I could easily spin up my workloads in the new cluster. But I also didn't want to loose any data in the process, so I had to figure out how to backup and migrate data for stateful Kubernetes applications.

Then I found out about Velero, and used it to backup my application's data to an external object storage and then restore it to my new cluster. In the end, it worked out really well! I also had to deal with some specific quirks of my setup with K3s and Rancher `local-path-provisioner`, so I decided to write about this process.

In this post, I'll walk you through my process of backing up and restoring the data for the stateful applications in my Homelab using Velero and Flux. This isn't meant to be an exact step-by-step guide. My goal here is to share what I've learned and give you a brief introduction to Velero, and how you can leverage it within a GitOps workflow to perform cluster migrations.

## A few notes on my setup

Before we get started, I want to set the stage by describing my current Kubernetes setup. You can check everything in the [the GitHub repository](https://github.com/luissimas/homelab), but the relevant bits for this post are:

- I use K3s with the default `local-path-provisioner` storage class
- I use Flux to power my GitOps workflow, syncing the manifests from my git repository and applying them to my cluster
- I use the `sealed-secrets` controller to encrypt my secrets, allowing me to commit them to my GitOps repository

With that said, let's get into the migration process!

## What is Velero?

Velero is a set of tools to **back up and restore** Kubernetes clusters. It can take back ups of the cluster resources, including the state of persistent volumes, and then upload them to an external Object Storage such as S3, Azure Blob Storage and many others.

Velero works by providing several CRDs to represent backups and their operations. We can then interact with Velero by creating and manipulating those resources in our Kubernetes cluster.

Since I'm using Flux to manage my cluster with a GitOps workflow, I'll focus only on the persistent volume backups capabilities of Velero. All my workloads and other resources are already defined in my git repository and don't really need a backup in this case.

Velero provide several ways of creating backups of the persistent volumes. If you're running on the cloud, Velero can use the cloud provider's API to take snapshots of the volumes. If your CSI driver supports snapshotting, it can also use that capability. Since I'm using Racher's local-path-provisioner, none of those options are available to me my use case. Luckily, Velero provides a file system backup feature, which acts directly at the filesystem level.

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

Once the deployment is complete, we can fetch the storage account ID and one of its keys. We'll put the key value in a credentials file that we'll then use to create a secret. Velero will use this secret to fetch the credentials to authenticate with Azure Blob Storage.

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

We need four resources

A namespaces to install Velero into:

```yaml {filename=namespace.yaml}
apiVersion: v1
kind: Namespace
metadata:
  name: velero
```

The helm repository:

```yaml {filename=repository.yaml}
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: vmware-tanzu
  namespace: velero
spec:
  interval: 1h0m0s
  url: https://vmware-tanzu.github.io/helm-charts
```

The secret containing the credentials that allow Velero to authenticate with the chosen object storage. Here we'll use the `credentials-velero` file we created in the previous step. I'm using `sealed-secrets` to encrypt my secrets, but you should adjust this step to create the secret in the appropriate way for your setup. The only important thing is that the secret should have a single key with a value containing the entire contents of the `credentials-velero` file.

```shell
kubectl create secret generic cloud-credentials \
       --from-file=cloud=credentials-velero \
       --dry-run=client \
       -o yaml | \
       kubeseal \
       --controller-namespace sealed-secrets \
       -o yaml > secret-sealed.yaml
```

Then, we need to create the helm release itself. The important things here are

```yaml {filename=release.yaml}
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: velero
  namespace: velero
spec:
  chart:
    spec:
      chart: velero
      version: 11.1.1
      sourceRef:
        kind: HelmRepository
        name: vmware-tanzu
  interval: 1h0m0s
  values:
    snapshotsEnabled: false
    deployNodeAgent: true
    credentials:
      existingSecret: cloud-credentials
    configuration:
      backupStorageLocation:
        - name: default
          provider: azure
          bucket: <your-bucket-name>
          credential:
            name: cloud-credentials
            key: cloud
          config:
            resourceGroup: <your-resource-group>
            storageAccount: <your-storage-account-name>
            storageAccountKeyEnvVar: AZURE_STORAGE_ACCOUNT_ACCESS_KEY
            subscriptionId: <your-subscription-id>
    initContainers:
      - name: velero-plugin-for-microsoft-azure
        image: velero/velero-plugin-for-microsoft-azure:v1.13.0
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - mountPath: /target
            name: plugins
```

And finally, a Kustomization to apply the resources via Flux

```yaml {filename=kustomization.yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - secret-sealed.yaml
  - repository.yaml
  - release.yaml
```

- Create env file and create the secret
- Create the helm release manifest
- Create the kustomization

After committing and pushing those changes to our git repository, Flux will sync and apply those manifests to our cluster. At the end, we should be able to verify that we have a `BackupStorageLocation` custom resource in the `Available` state in the `velero` namespace.

```shell {hl_Lines=9}
$ kubectl get backupstoragelocations.velero.io -n velero
TODO: get output
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

Now that Velero is setup and can access our object storage, we need to actually create the backups.

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

With this, we have successfully created a backup of our persistent volumes. Now, we are free to tear down our existing Kubernetes cluster and deploy a new one. I won't get into this process because it'd go way out of the scope of this post. Once we have our new cluster deployed, we can finally restore the backup.

## Restoring the backups

We'll need to:

1. Suspend the Flux Kustomization for the apps we want to restore. This prevents Flux from creating new PVCs for our workloads. We want to be able to create our PVs and PVCs by restoring Velero's backup instead
2. Install Flux in the new cluster and let it sync the Kustomizations that were not suspended by us. This will include Velero's Kustomization
3. Restore the Velero backup to create the PVCs and its PVs with the existing data
4. Re-enable the Kustomization for the apps we suspendd in step 1. Flux will then start to sync the state for those apps, creating our deployments and resources. Since the PVCs were restored and already exist in the cluster, Flux will "adopt" those existing resources instead of creating new ones

To suspend the Kustomization:

```yaml {hl_Lines=18}
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infra-configs
  interval: 1m
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./flux/apps
  prune: true
  wait: true
  suspend: true
```

Then, to bootstrap Flux:

```shell
$ GITHUB_TOKEN=<your-token> flux bootstrap github \
                                  --token-auth \
                                  --owner=<gh-user> \
                                  --repository=<gh-repo> \
                                  --branch=main \
                                  --path=<cluster-path-in-repo>
```

Then, we wait for Flux to reconcile our resources. In the end, we should be able to verify that all our Kustomizations are ready, and the `apps` Kustomization is suspended.

```shell
$ flux get kustomizations
NAME                    REVISION                SUSPENDED       READY   MESSAGE
apps                    main@sha1:458f2c63      True           Unknown Reconciliation in progress
core-services           main@sha1:458f2c63      False           True    Applied revision: main@sha1:458f2c63
flux-system             main@sha1:458f2c63      False           True    Applied revision: main@sha1:458f2c63
infra-configs           main@sha1:458f2c63      False           True    Applied revision: main@sha1:458f2c63
infra-controllers       main@sha1:458f2c63      False           True    Applied revision: main@sha1:458f2c63
```

Now that our core infrastructure controllers (including Velero) are installed, we can check that the backup resource we took in the old cluster is present in the new cluster:

```shell
$ velero backup get
NAME   STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
apps   Completed   0        0          2025-11-15 10:10:49 -0300 -03   17d       default            <none>
```

Since Velero uses the Object Storage as the source of truth, it was able to check that a backup existed there but not in our new fresh cluster. It then created the backup resource in the cluster, and now we can interact with it.

To restore the backup, we create a new `restore` from our backup:

```shell
$ velero restore create --from-backup apps
```

Then, we can list and describe the restore resource to check it's state:

```shell
$ velero restore get
$ velero restore describe <restore-name>
```

After the restore process is complete, we should be able to verify its status as `Completed`:

```shell
$ velero restore get
NAME                  BACKUP   STATUS            STARTED                         COMPLETED                       ERRORS   WARNINGS   CREATED                         SELECTOR
apps-20251116112235   apps     Completed         2025-11-16 11:22:36 -0300 -03   2025-11-16 11:22:39 -0300 -03   0        0          2025-11-16 11:22:35 -0300 -03   <none>
```

We can then verify that all PVCs were restored and are `Bound`:

```shell
$ kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
bazarr-config        Bound    pvc-08fea9b5-dd07-42a1-a1b7-cda0c7ef3659   500Mi      RWO            local-path     <unset>                 58m
jellyfin-config      Bound    pvc-a6f6fa51-8062-4c83-9b5a-b0b24355a4d3   500Mi      RWO            local-path     <unset>                 58m
prowlarr-config      Bound    pvc-2bbaf457-9227-420a-8e19-5178c3be2acf   500Mi      RWO            local-path     <unset>                 58m
qbittorrent-config   Bound    pvc-8818b46b-e2bc-4585-9947-43f6810b6f0d   500Mi      RWO            local-path     <unset>                 58m
radarr-config        Bound    pvc-27e72001-699e-4dc3-90e8-43e4980e019f   500Mi      RWO            local-path     <unset>                 58m
readarr-config       Bound    pvc-2e9349c4-3975-4b97-b0ba-9b095f8a6145   500Mi      RWO            local-path     <unset>                 58m
sonarr-config        Bound    pvc-ef06a736-93cc-4ae1-9748-9953df3c5143   500Mi      RWO            local-path     <unset>                 58m
```

Finally, we can resume the reconciliation for the kustomization we paused in the first step. Flux will then create the new resources and adopt the existing PVCs instead of creating new ones.
