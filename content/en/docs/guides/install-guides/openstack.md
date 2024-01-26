---
title: OpenStack deployment management
description: >
  OpenStack Multi-Cluster Deployment Management with Nephio
weight: 6
---

## Prerequisites

* Openstack Cluster Management (master)
  * 4 VCPU 4 NODES
  * 8GB RAM
  * Kubernetes version 1.24+
* Openstack Cluster Edge n
  * 2 VCPU 1 NODE
  * 4GB RAM
  * Kubernetes version 1.24+
* KPT beta [releases](https://github.com/kptdev/kpt/releases)

## Automatic Installation of the management cluster
* Change ansible variables to reflect your cluster and run the installation script
  
  1. Add the following to *test-infra\e2e\provision\playbooks\roles\bootstrap\tasks\prep-gitea.yml*
```yaml
- name: Create PersistentVolume
kubernetes.core.k8s:
context: "{{ k8s.context }}"
state: present
definition:
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: data-gitea-postgresql-0
    spec:
    capacity:
        storage: 10Gi
    volumeMode: Filesystem
    accessModes:
        - ReadWriteOnce
    storageClassName: standard
    hostPath:
        path: /tmp/
namespace: "{{ gitea.k8s.namespace }}"

- name: Create PersistentVolumeClaim
kubernetes.core.k8s:
context: "{{ k8s.context }}"
state: present
definition:
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: data-gitea-postgresql-0
    spec:
    storageClassName: standard
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
        storage: 10Gi
namespace: "{{ gitea.k8s.namespace }}"

- name: Create PersistentVolume
kubernetes.core.k8s:
context: "{{ k8s.context }}"
state: present
definition:
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: data-gitea-0
    spec:
    capacity:
        storage: 10Gi
    volumeMode: Filesystem
    accessModes:
        - ReadWriteOnce
    storageClassName: standard
    hostPath:
        path: /tmp/
namespace: "{{ gitea.k8s.namespace }}"

- name: Create PersistentVolumeClaim
kubernetes.core.k8s:
context: "{{ k8s.context }}"
state: present
definition:
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: data-gitea-0
    spec:
    storageClassName: standard
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
        storage: 10Gi
namespace: "{{ gitea.k8s.namespace }}"
```
  2. Override the default Ansible values and run the installation script in
  *test-infra\e2e\provision\install_sandbox.sh* by
```bash
ANSIBLE_CMD_EXTRA_VAR_LIST="k8s.context='kubernetes-admin@cluster.local' kind.enable=false host_min_vcpu=4 host_min_cpu_ram=8" ./install_sandbox.sh
```

## Manual Installation of the management cluster using kpt
TDB (manual install of kpt, porch, configsync, nephio-webui, capi, metallb)

## Manual Installation of the Edge cluster using kpt

```bash
kpt pkg get https://github.com/nephio-project/nephio-packages.git/nephio-configsync@v1.0.1
```

Change *nephio-configsync/rootsync.yaml* and point spec.git.repo to the edge git repository
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
kpt live init nephio-configsync
kpt live apply nephio-configsync --reconcile-timeout=5m
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
Using the web-ui add a new deployment with destination the edge cluster.

