---
title: Nephio manual deployment on multiple VMs
description: >
  Nephio manual deployment on different VMs
weight: 7
---

## Prerequisites

* OpenStack Cluster Management (master)
  * 4 vCPU
  * 8 GB RAM
  * Kubernetes version 1.26+

* OpenStack Cluster Edge n
  * 2 vCPU 1 NODE
  * 4 GB RAM
  * Kubernetes version 1.26+

* KPT beta [releases](https://github.com/kptdev/kpt/releases)


## Installation of the management cluster
### Automatic installation of the management cluster using the sandbox Ansible script
Override the default Ansible values and run the installation script in 
*test-infra\e2e\provision\init.sh* by changing the k8s.context with your

```kubectl config get-contexts```

then run:
```bash
export NEPHIO_USER=$USER
export ANSIBLE_CMD_EXTRA_VAR_LIST="k8s.context=kubernetes-admin@cluster.local kind.enable=false host_min_vcpu=4 host_min_cpu_ram=8"

curl -fsSL https://raw.githubusercontent.com/nephio-project/test-infra/main/e2e/provision/init.sh | sudo -E bash
```
- - - -

> **_NOTE:_** The Ansible script will try to install some utilities, if for example the script fails to uninstall them first, you can manually delete them and re-run the script.
```find /usr/lib/python* -type f -name "PyYAML*"``` and ```sudo rm /usr/lib/python3/dist-packages/PyYAML-5.3.1.egg-info```

### Manual Installation of the management cluster using kpt

- [Common Components](/content/en/docs/guides/install-guides/common-components.md)
- [Common Dependencies](/content/en/docs/guides/install-guides/common-dependencies.md)

## Manual Installation of the Edge cluster using kpt
Install config-sync using:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@main
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```
Get the Roosync kpt package and edit it:
```bash
kpt pkg get https://github.com/nephio-project/catalog.git/nephio/optional/rootsync@main
```

Change *./rootsync/rootsync.yaml* and point spec.git.repo to the edge git repository
```yaml
 spec:
   sourceFormat: unstructured
   git:
    repo: <http url of your edge repo>
     branch: main
     auth: none
```
Deploy the modified configsync
```bash
kpt live init rootsync
kpt live apply rootsync --reconcile-timeout=15m --output=table
```

## Configure Management Cluster to manage Edge Cluster
Get a [GitHub token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) if your repository is private, or allow Porch to make modifications.

Register the edge repository using kpt cli or nephio web-ui.
```bash
GITHUB_USERNAME=<Github Username>
GITHUB_TOKEN=<GitHub Token>

kpt alpha repo register \
  --namespace default \
  --repo-basic-username=${GITHUB_USERNAME} \
  --repo-basic-password=${GITHUB_TOKEN} \
  --create-branch=true \
  --deployment=true \
  <http url of your edge repo>
```

## Deploy packages to the edge clusters
Using the web-ui, add a new deployment to the edge workload cluster.

