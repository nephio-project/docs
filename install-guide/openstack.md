# OpenStack Multi-Cluster Deployment Management with Nephio


## Prerequisites

* Openstack Cluster Management (master)
  * 4 VCPU 4 NODES
  * 8GB RAM
  * Kubernetes version 1.24+
* Openstack Cluster Edge n
  * 2 VCPU 1 NODE
  * 4GB RAM
  * Kubernetes version 1.24+
* KPT beta [realeses](https://github.com/kptdev/kpt/releases)

## Automatic Installation of the managment cluster
* Change ansible variables to reflect your cluster and run the installation script
  
  1. Add the following to *test-infra\e2e\provision\playbooks\roles\bootstrap\tasks\prep-gitea.yml*
```- name: Create PersistentVolume
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
  2. Override these parameters:
     1. Disable **kind** installation: kind.enabled=false
     2. Override **context** value into your kubernetes context: `kubectl config get-contexts`
     3. Override the CPU and RAM **minimum requirements** variables
   
  3. With all the modified parameters run the installation script in
  *test-infra\e2e\provision\install_sandbox.sh* as follows:
```
NEPHIO_HOST_MIN_VCPU=4 \
NEPHIO_HOST_MIN_CPU_RAM=8 \
ANSIBLE_CMD_EXTRA_VAR_LIST="k8s.context=kubernetes-admin@cluster.local,kind.enabled=false" \
./install_sandbox.sh
```


## Manual Installation of the Edge cluster using kpt

``` 
kpt pkg get https://github.com/nephio-project/nephio-packages.git/nephio-configsync@v1.0.1
```

Change *nephio-configsync/rootsync.yaml* and point spec.git.repo to the edge git repository
```
 spec:
   sourceFormat: unstructured
   git:
    repo: <http url of your edge repo>
     branch: main
     auth: none
```
Deploy the modified configsync
```
kpt live init nephio-configsync
kpt live apply nephio-configsync --reconcile-timeout=5m
```

## Configure Managment Cluster to manage Edge Cluster
Get a [github token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) if your repository is private, or allow Porch to make modifications.

Register the edge repository using kpt cli or nephio web-ui.
```
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

