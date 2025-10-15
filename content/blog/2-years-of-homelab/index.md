---
title: '2 Years of Homelab'
date: '2025-10-15'
draft: true
comments: true
tags:
  - homelab
---

It's been almost 2 years since I started my Homelab project, and it's been a great journey!

My setup changed a lot over the time, but recently it has stabilized quite a bit and I rarely change it in any significant way. This current iteration of my Homelab fulfills the following properties:

- GitOps: everything is deployed and reproducible with FluxCD
- Secrets are encrypted and stored in the git repository
- All services have valid SSL certificates issued by Let's Encrypt
- Hardware acceleration enabled for video transcoding
- Observability for both the services and the node itself
- It's fun :)

In this post, I'll describe the current state of my setup and highlight a few key learnings along the way. I plan on doing some big changes in the near future, so this post will also serve as a way to reflect about what I like about this setup and what could be improved.

![Glance homepage for my Homelab](glance-screenshot.png)

My Homelab runs as single node Kubernetes cluster (I'm using K3s) that hosts all my services. The Kubernetes deployments and configuration are entirely managed in a GitOps workflow, using FluxCD. The OS running in the node is Ubuntu Server 22.04. Besides the main K3s process, the node also runs Tailscale to setup a peer-to-peer VPN from my client devices to my server. All of this runs in a used Mini PC that sits quietly under my desk.

## Why I started a homelab

Mostly to learn new things

## What I'm currently running

My Homelab runs a variety of services for different purposes. This is a non exhaustive list of reasons that I think of for running a service. When I decide whether to deploy a new service or keep an existing one, I usually ask the following questions:

- Will I learn something new by deploying and maintaining it?
- Will it serve some purpose to me or my family in my day to day life?
- Is it something that I'm curious about and want to try out?

If the answer for any of those questions is "yes" for a given service, then it earns a spot in the cluster.

I currently have about 47 pods with 120 containers running in my cluster. Most of these are Kubernetes services themselves, like CoreDNS, FluxCD components, custom operators that I've installed, etc. Which means that although there are dozens of pods in this cluster, I don't have that many user facing services.

Here's a non exhaustive list of the services that I'm currently running:

- Jellyfin
- Arr stack
- Mealie
- Headlamp
- Grafana
- Prometheus
- Loki
- [zettelkasten-exporter](https://github.com/luissimas/zettelkasten-exporter) (check it out, I've heard it's great if you happen to be an observability and note taking nerd)

## The hardware

On the hardware side, I started like many do: with an old laptop. It had pretty modest specs: an AMD dual core CPU, 4GB of RAM and 500GB of hard drive storage. It was perfectly fine for most of the tasks I threw at it.

Sadly, at one point I tried to watch an HEVC encoded movie in Jellyfin and noticed that it was practically impossible to watch due to the low framerate. This experience thought me that hardware accelerated transcoding is needed for some types of video encoding, and that the integrated graphics card of my old laptop didn't have support for that.

This event motivated me to buy the first (and so far the only) dedicated hardware to my homelab. The main thing I was looking for was an integrated GPU with QuickSync to enable hardware acceleration to transcode video files. I did some research and was convinced (and still am) that (used) Mini PCs are the best hardware for a small homelab. So, in February of 2024, I bought an used HP Elitedesk 800 G3 Mini, and its been serving me very well so far.

> TODO: picture of the server

The specs are still not high end by any standards, but it is plenty for my needs. It has an Intel i5-7500T, 16GB of RAM and 1TB of SSD storage. The main highlight is that the integrated GPU (Intel HD Graphics 630) has QuickSync support, so it's able to perform hardware accelerated transcoding. Besides that, it also has some room for memory and storage upgrades, but I just haven't had the need yet.

To give you a better idea of resource consumption on this setup. This is the memory and CPU usage graph for the past week with everyday usage (which translates to mostly being idle during the day and streaming something from Jellyfin in the evenings).

![Resource usage on the server for the past week](resource-usage.png)

## Networking

I don't have anything fancy in the network side of things. I just use my ISP router and plug the server directly to it via an Ethernet cable.

The only extra thing I have setup here is Tailscale. It runs in the server node and in all client devices that I want to grant access to the server. Tailscale provides a peer to peer VPN service. It's not really 100% peer to peer because it requires the devices to talk to a proprietary Tailscale coordinator service for the control plane, but I'm not really worried about that currently.

The end result is that I can access my server from anywhere in the world, not only in my LAN network. I have full control of which devices are connected to the VPN and can easily add new devices when I need to.

If you are curious, check out [how Tailscale works](https://tailscale.com/blog/how-tailscale-works), I think it's pretty great. There are some alternatives that solve the "proprietary coordinator service" problem, such as NetBird. I just never felt the need to explore other solutions.

## The good parts

### Learning Kubernetes

What operators I'm using

- cert-manager + duckdns
- sealed secrets
- cloudnativepg
- kube-prometheus-stack

### Enabling new projects

One of the most interesting benefits of having a Homelab is that is enables other projects. Having your own infrastructure to run applications without any additional cost opens up a lot of possibilities. I saw myself creating tiny services such as [zettelkasten exporter](https://github.com/luissimas/zettelkasten-exporter).

## The bad parts

### Learning Kubernetes

### Hardware transcoding

### Random hardware failures

## What's next

- SSO for my services
- K3s running in Incus container
- Hardware upgrades
