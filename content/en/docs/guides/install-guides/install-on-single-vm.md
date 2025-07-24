---
title: Installation on a single VM
description: >
  Nephio Installation in a sandbox VM
weight: 4
---

{{% pageinfo %}}
This page is draft and the separation of the content to different categories is not done. 
{{% /pageinfo %}}

In this guide, you will set up Nephio running in a single VM with:

- **Management Cluster**: [kind](https://kind.sigs.k8s.io/)
- **Cluster Provisioner**: [Cluster API](https://cluster-api.sigs.k8s.io/)
- **Workload Clusters**: kind
- **GitOps Tool**: ConfigSync
- **Git Provider**: Gitea running in the Nephio management cluster will be the
  git provider for cluster deployment repositories. Some external repositories
  will be on GitHub.
- **Web UI Auth**: None
- **Ingress/Load Balancer**: [MetalLB](https://metallb.universe.tf/), but only internally to the VM.

## Provisioning Your Sandbox VM

In addition to the general prerequisites, you will need:

* Access to a Virtual Machine provided by an hypervisor ([VirtualBox](https://www.virtualbox.org/),
  [Libvirt](https://libvirt.org/)) and running an OS supported by Nephio (Ubuntu 22.04, Fedora 34) with a minimum
  of 16 vCPUs and 32 GB in RAM.
* [Kubernetes IN Docker](https://kind.sigs.k8s.io/) (*kind*) installed and set up your workstation.

## Provisioning Your Management Cluster

The Cluster API services require communication with the Docker socket for creation of workload clusters. The command
below creates an All-in-One Nephio management cluster through the KinD tool, mapping the */var/run/docker.sock* socket
file for Cluster API communication.

```bash
cat << EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.27.1
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
EOF
```

## Gitea Installation

While you may use other Git providers as well, Gitea is required in the R2 setup. To install Gitea, use `kpt`. From your
*nephio-install* directory, run:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/distros/sandbox/gitea@origin/main
kpt fn render gitea
kpt live init gitea
kpt live apply gitea --reconcile-timeout 15m --output=table
```

## Common Dependencies

There are a few dependencies that are common across most installations, and do not require any installation-specific
setup. You should install these next, as described in the
[common dependencies documentation](/content/en/docs/guides/install-guides/common-dependencies.md).

## Common Components

With the necessary dependencies now installed, you can now install the essential Nephio components. This is documented
in the [common components documentation](/content/en/docs/guides/install-guides/common-components.md).

## Provisioning Cluster API

For managing the Kubernetes cluster infrastructure, it is necessary to install
[Cluster API project](https://cluster-api.sigs.k8s.io/). This package depends on
[cert-manager project](https://cert-manager.io/) to generate certificates.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/distros/sandbox/cert-manager@origin/main
kpt fn render cert-manager
kpt live init cert-manager
kpt live apply cert-manager --reconcile-timeout 15m --output=table
```

Once *cert-manager* is installed, you can proceed with the installation of Cluster API components

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/infra/capi/cluster-capi@origin/main
kpt fn render cluster-capi
kpt live init cluster-capi
kpt live apply cluster-capi --reconcile-timeout 15m --output=table
```

Cluster API uses infrastructure providers to provision cloud resources required by the clusters. You can manage local
resources with the Docker provider, it can be installed with the followed package.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/infra/capi/cluster-capi-infrastructure-docker@origin/main
kpt fn render cluster-capi-infrastructure-docker
kpt live init cluster-capi-infrastructure-docker
kpt live apply cluster-capi-infrastructure-docker --reconcile-timeout 15m --output=table
```

The last step is required for defining cluster, machine and kubeadm templates for controller and worker docker
machines. These templates define the kubelet arguments, etcd and coreDNS configuration and image repository as other
things.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/infra/capi/cluster-capi-kind-docker-templates@origin/main
kpt fn render cluster-capi-kind-docker-templates
kpt live init cluster-capi-kind-docker-templates
kpt live apply cluster-capi-kind-docker-templates --reconcile-timeout 15m --output=table
```

## Installing Packages

The management or workload cluster both need *config-sync*, *root-sync* and a cluster git repository to manage packages. 

### Install Config-sync

Install *config-sync* using:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@origin/main
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

### Create Git Repositories

For the management cluster, you need to create two repositories named *mgmt* and *mgmt-staging*. Repository mgmt will be mapped to management cluster via config-sync to inject nephio workloads while mgmt-staging repository is used internally during cluster bootstrapping.

If you are using Gitea then you can use the following steps:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/distros/sandbox/repository@origin/main mgmt
kpt fn render mgmt
kpt live init mgmt
kpt live apply mgmt --reconcile-timeout=15m --output=table
```

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/distros/sandbox/repository@origin/main mgmt-staging
kpt fn render mgmt-staging
kpt live init mgmt-staging
kpt live apply mgmt-staging --reconcile-timeout=15m --output=table
```

{{% alert title="Note" color="primary" %}}

* In the *repository* package the default Gitea address is *172.18.0.200:3000*. 
In *repository/set-values.yaml* change this to your Gitea address.
* *repository/token-configsync.yaml* and *repository/token-porch.yaml* are 
responsible for creating secrets with the help of the Nephio token controller 
for accessing the git instance for root-sync. 
You would need the name of the config-sync token to provide it to root-sync.

{{% /alert %}}

### Install Root-sync

Get the *root-sync* kpt package and edit it:

```bash
kpt pkg get https://github.com/nephio-project/catalog.git/nephio/optional/rootsync@origin/main
```

Open *./rootsync/rootsync.yaml* file and update the *spec.git.secretRef.name* to name of config-sync token created in previous step and *metadata.name* to *mgmt*. Also verify that spec.git.repo has correct URL of mgmt git repository

```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata: # kpt-merge: config-management-system/example-cluster-name
  name: mgmt
  namespace: config-management-system
  annotations:
    internal.kpt.dev/upstream-identifier: configsync.gke.io|RootSync|config-management-system|example-cluster-name
spec:
  sourceFormat: unstructured
  git:
    repo: http://<GIT_URL>/nephio/mgmt.git
    branch: main
    auth: token
    secretRef:
      name: mgmt-access-token-configsync
    period: 15s
```

Deploy the modified root-sync:

```bash
kpt live init rootsync
kpt live apply rootsync --reconcile-timeout=15m --output=table
```

If the output of `kubectl get rootsyncs.configsync.gke.io -A` 
is similar to the following, then root-sync is properly configured. 

```console
kubectl get rootsyncs.configsync.gke.io -A
NAMESPACE                  NAME   RENDERINGCOMMIT                            RENDERINGERRORCOUNT   SOURCECOMMIT                               SOURCEERRORCOUNT   SYNCCOMMIT                                 SYNCERRORCOUNT
config-management-system   mgmt   ddc9676c997696d4a102a5cf2c67d0a0c459ceb3                         ddc9676c997696d4a102a5cf2c67d0a0c459ceb3                      ddc9676c997696d4a102a5cf2c67d0a0c459ceb3   
```

## Managing Clusters via Nephio 

You can use the [pre-configured workload cluster package](https://github.com/nephio-project/catalog/tree/main/infra/capi/nephio-workload-cluster).  
