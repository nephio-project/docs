---
title: Nephio manual deployment on multiple VMs
description: >
  Nephio manual deployment on different VMs
weight: 7
---

## Prerequisites
* Cluster Management (master)
  * 4 vCPU
  * 8 GB RAM
  * Kubernetes version 1.26+
  * `kubectl` [installed ](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
  * **Ingress/Load Balancer**: [MetalLB](https://metallb.universe.tf/), but only internally to the VM
* Cluster Edge
  * 2 vCPU 1 NODE
  * 4 GB RAM
  * Kubernetes version 1.26+
  * `kubectl` [installed ](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
* `kpt` [installed](https://kpt.dev/installation/kpt-cli) (version v1.0.0-beta.43 or later)
* `porchctl` [installed](/content/en/docs/porch/using-porch/porchctl-cli-guide.md) on your workstation

## Installation of the management cluster

### Manual Installation of the management cluster using kpt

- [Common Dependencies](/content/en/docs/guides/install-guides/common-dependencies.md)
- [Common Components](/content/en/docs/guides/install-guides/common-components.md)

## Manual Installation of the Edge cluster using kpt

All the workload clusters need config-sync, root-sync 
and a cluster git repository to manage packages. 
The below steps have to be repeated for each workload cluster:

### Install Config-sync

Install config-sync using:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@@origin/v3.0.0
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

### Create Git Repository

Create a repository for your cluster either in your git provider or in gitea. 

If you want to use GitHub or GitLab then follow below steps

Get a [GitHub token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) if your repository is private,
to allow Porch to make modifications.

Register the edge repository using kpt cli or nephio web-ui.

```bash
GITHUB_USERNAME=<Github Username>
GITHUB_TOKEN=<GitHub Token>

porchctl repo register \
  --namespace default \
  --repo-basic-username=${GITHUB_USERNAME} \
  --repo-basic-password=${GITHUB_TOKEN} \
  --create-branch=true \
  --deployment=true \
  <http url of your edge repo>
```


In case, you are using Gitea then you can use the following steps

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/distros/sandbox/repository@@origin/v3.0.0 <cluster-name>
kpt fn render <cluster-name>
kpt live init <cluster-name>
kpt live apply <cluster-name> --reconcile-timeout=15m --output=table
```

{{% alert title="Note" color="primary" %}} 

* For management cluster you have to name the repository as `mgmt`.
* In the `repository` package by default gitea address is `172.18.0.200:3000` in `repository/set-values.yaml` 
change this to your git address.
* `repository/token-configsync.yaml` and `repository/token-porch.yaml` are responsible for creating secrets with the help of Nephio token controller for accessing git instance for root-sync. You would need the name of config-sync token to provide it to root-sync.

{{% /alert %}}

### Install Root-sync

Get the Root-sync kpt package and edit it:

```bash
kpt pkg get https://github.com/nephio-project/catalog.git/nephio/optional/rootsync@@origin/v3.0.0
```

Change `./rootsync/rootsync.yaml` and point `spec.git.repo` to the edge git repository and the  

```yaml
 spec:
   sourceFormat: unstructured
   git:
    repo: <http url of your edge repo>
     branch: main
     auth: none
```

If need credentials to access repository your repository then 
copy the token name from previous section and provide it in 
`./rootsync/rootsync.yaml`

```yaml
spec:
  sourceFormat: unstructured
  git:
    repo: <http url of your edge repo>
    branch: main
    auth: token
    secretRef:
      name: <TOKEN-NAME>
```

Deploy the modified root-sync

```bash
kpt live init rootsync
kpt live apply rootsync --reconcile-timeout=15m --output=table
```


If the output of `kubectl get rootsyncs.configsync.gke.io -A` 
is similar as below then root-sync is properly configured. 

```console
kubectl get rootsyncs.configsync.gke.io -A
NAMESPACE                  NAME   RENDERINGCOMMIT                            RENDERINGERRORCOUNT   SOURCECOMMIT                               SOURCEERRORCOUNT   SYNCCOMMIT                                 SYNCERRORCOUNT
config-management-system   mgmt   ddc9676c997696d4a102a5cf2c67d0a0c459ceb3                         ddc9676c997696d4a102a5cf2c67d0a0c459ceb3                      ddc9676c997696d4a102a5cf2c67d0a0c459ceb3   
```

### Add Workload CRDs in Edge workload cluster

Workload CRDs are required to manage network functions. 

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/workload-crds@@origin/v3.0.0
kpt live init workload-crds
kpt live apply workload-crds --reconcile-timeout=15m --output=table
```

## Deploy packages to the edge clusters

Using web-ui or command line add a new deployment to the edge workload cluster. 
