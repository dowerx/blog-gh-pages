---
title: What to look out for when setting up a K3S cluster on PIs
date:
  created: 2025-01-07
tags:
  - k8s
  - k3s
  - pi
---

<!-- more -->

## Configure the nodes
- static IP
- DNS: disable systemd-resolved, or it will conflict with the embeded DNS server of K3S
- storage:
    - nfs-kernel-sever, nfs-common
    - ZFS

## Install K3S
1st master:
`curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server --cluster-init`

Other masters:
`curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server --server https://<ip or hostname of server1>:6443`

Workers:
`curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - agent --server https://<ip or hostname of server>:6443`

## Access
The kubectl config is located at `/etc/rancher/k3s/k3s.yaml`.

## Install basic services
- keepalived
- storage
    - [NFS](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
    - [ZFS](https://github.com/ccremer/kubernetes-zfs-provisioner)
- cert-manager
  ```bash
  helm repo add jetstack https://charts.jetstack.io
  helm repo update
  
  helm install \
  cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set installCRDs=true
  ```

- DNS: coreDNS
- configure traefik
  ```yaml
  # /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
  apiVersion: helm.cattle.io/v1
  kind: HelmChartConfig
  metadata:
    name: traefik
    namespace: kube-system
  spec:
    valuesContent: |-
      additionalArguments:
      - "--entryPoints.dnsudp.address=:53/udp"
      - "--entryPoints.dnstcp.address=:53/tcp"
      ...
      ports:
        dnsudp:
          port: 53
          exposedPort: 53
          expose:
            default: true
          protocol: UDP
        dnstcp:
          port: 53
          exposedPort: 53
          expose:
            default: true
          protocol: TCP
        ...
  ```
- install registry
- confgure registry
  ```yaml
  # /etc/rancher/k3s/registries.yaml
  mirrors:
  docker.io:
    endpoint:
      - https://registry-mirror.example.org/v2
  configs:
    registry.example.org:
      auth:
        username: username
        password: password
    registry-mirror.example.org:
      auth:
        username: username
        password: password
  ```
