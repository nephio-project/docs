---
title: Centralized FluxCD Workload Cluster Deployment
description: >
  A step by step guide to deploy a sample workload to a workload cluster, 
  using a centralized gitops approach.
weight: 2
---

## Prerequisites

- A Nephio Management cluster. See the [installation guides]({{< relref "/docs/guides/install-guides/_index.md"> }}) 
for detailed environment options.
- The following *optional* pkg deployed:
  - [FluxCD controllers]({{< relref "/docs/guides/install-guides/optional-components.md#fluxcd-controllers"> }})

{{% alert title="Note" color="primary" %}}

If using a [sandbox demo environment]({{< relref "/docs/guides/install-guides/_index.md#kicking-off-an-installation-on-a-virtual-machine"> }}), 
most of the above prerequisites are already satisfied.

{{% /alert %}}

This exercise will take us from a system with only the Nephio Management cluster setup, to a deployment with:

- A [FluxCD specific workload cluster](https://github.com/nephio-project/catalog/tree/main/infra/capi/nephio-workload-cluster-flux).
- A repository for said cluster, registered with Nephio Porch.
- A sample workload deployed to said cluster.

To perform these exercises, we will need:

- Access to the installed demo VM environment as the ubuntu user.
- Access to the Nephio WebUI as described in the installation guide.

Access to Gitea, used in the demo environment as the Git provider, is optional.

### Step 1: Deploy the FluxCD Workload Cluster

{{% alert title="Note" color="primary" %}}

After fresh docker install, verify docker supplementary group is loaded by executing `id | grep docker`.
If not, logout and login to the VM or execute the `newgrp docker` to ensure the docker supplementary group is loaded.

{{% /alert %}}

First, verify that the [catalog blueprint repositories](https://github.com/nephio-project/catalog.git) are registered 
and *Ready*:
```bash
kubectl get repository
```
Sample output:
```bash
NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
catalog-distros-sandbox     git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-infra-capi          git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-nephio-core         git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-nephio-optional     git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-free5gc   git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-oai-ran   git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-tools     git    Package   false        True    https://github.com/nephio-project/catalog.git
mgmt                        git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
oai-core-packages           git    Package   false        True    https://github.com/OPENAIRINTERFACE/oai-packages.git
```

Once *Ready*, we can utilize blueprint packages from these upstream repositories.

In this example, we will use the [Porch package variant controller]({{< relref "/docs/porch/package-variant.md#core-concepts"> }}) 
to deploy the new Workload Cluster.

This fully automates the onboarding process, including the auto approval and publishing of the new package.

{{% alert title="Note" color="primary" %}}

An [alternate (manual) method](https://github.com/nephio-project/test-infra/blob/main/e2e/tests/flux/001.sh) 
of deploying the pkg can also be utilized.

{{% /alert %}}

Create a new *PackageVariant* CR for the Workload Cluster:

```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: flux-regional-cluster
spec:
  upstream:
    repo: catalog-infra-capi
    package: nephio-workload-cluster-flux
    workspaceName: main
  downstream:
    repo: mgmt
    package: regional
  annotations:
    approval.nephio.org/policy: initial
  pipeline:
    mutators:
    - image: gcr.io/kpt-fn/set-labels:v0.2.0
      configMap:
        nephio.org/site-type: regional
        nephio.org/region: us-west1
EOF
```
Sample output:
```
packagevariant.config.porch.kpt.dev/flux-regional-cluster created
```

configsync running in the Management cluster will create all the resources necessary to
provision a KinD cluster, and register it with Nephio. This can take some time.

## Step 2: Check the cluster installation

You can check if the cluster has been added to the management cluster:

```bash
kubectl get cl
```
or
```bash
kubectl get clusters.cluster.x-k8s.io
```
Sample output:
```
NAME       CLUSTERCLASS   PHASE         AGE    VERSION
regional   docker         Provisioned   179m   v1.31.0
```

To access the API server of that cluster, you can retrieve the *kubeconfig* file by pulling it from the Kubernetes
Secret and decode the base64 encoding:

```bash
kubectl get secret regional-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/regional-kubeconfig
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/regional-kubeconfig
```

You can then use it to access the Workload cluster directly:

```bash
kubectl get ns --context regional-admin@regional
```
Sample output:
```
NAME                 STATUS   AGE
default              Active   3h
kube-node-lease      Active   3h
kube-public          Active   3h
kube-system          Active   3h
local-path-storage   Active   179m
metallb-system       Active   179m
```

You should also check that the KinD cluster has come up fully by checking the *machinesets*. 
You should see READY and AVAILABLE replicas.

```bash
kubectl get machinesets
```
Sample output:
```
NAME                        CLUSTER    REPLICAS   READY   AVAILABLE   AGE    VERSION
regional-md-0-lmsqz-7nzzc   regional   1          1       1           3h1m   v1.31.0
```

## Step 3: Investigate the FluxCD specific CRs

Verify that the regional-flux-gitrepo-kustomize *PackageRevision* has been created for the FluxCD specific CRs. 
We want the *Published* v1 revision. 
```bash
kubectl get packagerevision -o custom-columns="NAME:.metadata.name,PACKAGE_NAME:.spec.packageName,REVISION:.spec.revision" | grep "regional-flux-gitrepo-kustomize" | grep "v1"```
```
Sample output:
```
mgmt-b14f9b729988f6260dd816782f2039b5f31f4bfa      regional-flux-gitrepo-kustomize      v1
```
Once verified:

Check the *Ready* state of the [Flux GitRepository Source](https://fluxcd.io/flux/components/source/gitrepositories/) CR:
```bash
kubectl get gitrepo regional
```
Sample output:
```
NAME       URL                                            AGE     READY   STATUS
regional   http://172.18.0.200:3000/nephio/regional.git   3h35m   True    stored artifact for revision 'main@sha1:cbaa1fa49bc711c4a179acc060c333f23d05f89c'
```
Now, we can get more details of the GitRepository CR:
```bash
kubectl get gitrepo regional -o jsonpath='{.spec}'
```
Sample output:
```json
{
   "interval":"2m",
   "ref":{
      "branch":"main"
   },
   "secretRef":{
      "name":"regional-access-token-porch"
   },
   "timeout":"60s",
   "url":"http://172.18.0.200:3000/nephio/regional.git"
} 
```
The default target *url* points to the internal Gitea repository created as part of the Workload Cluster installation, 
along with an associated *secretRef* to access it. It also defaults to reference the **main** `branch` of the repository.


Next check the *Ready* state of the associated [Flux Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/) CR:
```bash
kubectl get ks regional
```
Sample output:
```
NAME       AGE     READY   STATUS
regional   3h35m   True    Applied revision: main@sha1:cbaa1fa49bc711c4a179acc060c333f23d05f89c
```
Now, we can get more details of the Kustomization CR:
```bash
kubectl get ks regional -o jsonpath='{.spec}'
```
Sample output:
```json
{
   "force":false,
   "interval":"60s",
   "kubeConfig":{
      "secretRef":{
         "name":"regional-kubeconfig"
      }
   },
   "path":"./",
   "prune":true,
   "sourceRef":{
      "kind":"GitRepository",
      "name":"regional"
   }
}
```
As you can see, it *sourceRef* references the *regional* GitRepository CR and defaults to the root *path* of the repository.
It also holds a reference to the *kubeconfig* Secret used to access the Workload Cluster, which was created 
during the cluster API instantiation.


## Step 4: Deploy a sample workload to the Cluster

Create a new *PackageVariant* CR for the sample workload:
For demo purposes, we are deploying the 
[Free5GC Control Plane](https://github.com/nephio-project/catalog/tree/main/workloads/free5gc/free5gc-cp) blueprint pkg, 
again, using a Porch *PackageVariant*.

```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: free5gc-control-plane
spec:
  upstream:
    repo: catalog-workloads-free5gc
    package: free5gc-cp
    workspaceName: main
  downstream:
    repo: regional
    package: free5gc-cp
  annotations:
    approval.nephio.org/policy: initial
  pipeline:
    mutators:
    - image: docker.io/nephio/gen-kustomize-res:latest
EOF
```
Sample output:
```
packagevariant.config.porch.kpt.dev/free5gc-control-plane created
```
{{% alert title="Note" color="primary" %}}

Due to the extensive use of *local-config* in the current[Nephio Workload packages](https://github.com/nephio-project/catalog/tree/main/workloads), 
plus the target *spec.path* of **root** defined in the Flux Kustomization CR used, 
we require a workaround to generate a *kustomization.yaml* at the root of each kpt pkg deployed to the same downstream Repository.
In the example above, this is done via a *PackageVariant.spec.pipeline.mutators* [kpt function](https://github.com/nephio-project/nephio/blob/main/krm-functions/gen-kustomize-res/README.md),
which gets added to the mutation pipeline of the downstream pkg.
This will need to addressed in the next iteration.

{{% /alert %}}

Once the Porch and Nephio controllers have completed their tasks, the mutated *spec.upstream.package* resources 
are pushed to the *spec.downstream.repo* git repository.

We can then check the state of the Workload Cluster deployment:
```bash
kubectl get po -n free5gc-cp --context regional-admin@regional
```
Sample output:
```
NAME                            READY   STATUS    RESTARTS   AGE
free5gc-ausf-5c44c54dbc-cr4n6   1/1     Running   0          11m
free5gc-nrf-b65d7f6-mn2qt       1/1     Running   0          11m
free5gc-nssf-857dd8d4b5-pqln2   1/1     Running   0          11m
free5gc-pcf-85c58bc6-5kcsj      1/1     Running   0          11m
free5gc-udm-784cc65c78-pwlcm    1/1     Running   0          11m
free5gc-udr-57549dcdf8-hl9cp    1/1     Running   0          11m
free5gc-webui-f54cb7d6f-89xcl   1/1     Running   0          11m
mongodb-0                       1/1     Running   0          11m
```

We can also describe the Flux Kustomize CR reconciliation state:
```bash
kubectl describe ks regional
```
Sample output:
```
Name:         regional
Namespace:    default
Labels:       app.kubernetes.io/managed-by=configmanagement.gke.io
              configsync.gke.io/declared-version=v1
Annotations:  config.k8s.io/owning-inventory: config-management-system_mgmt
              configmanagement.gke.io/managed: enabled
              configmanagement.gke.io/source-path: regional-flux-gitrepo-kustomize/flux-wc-kustomization.yaml
              configmanagement.gke.io/token: 1ddb6395a183f5190c63fc94cfab9e7bd4ee478f
              configsync.gke.io/git-context: {"repo":"http://172.18.0.200:3000/nephio/mgmt.git","branch":"main","rev":"HEAD"}
              configsync.gke.io/manager: :root_mgmt
              configsync.gke.io/resource-id: kustomize.toolkit.fluxcd.io_kustomization_default_regional
              internal.kpt.dev/upstream-identifier: kustomize.toolkit.fluxcd.io|Kustomization|default|example-cluster-name
              nephio.org/cluster-name: regional
API Version:  kustomize.toolkit.fluxcd.io/v1
Kind:         Kustomization
Metadata:
  Creation Timestamp:  2025-02-07T10:25:13Z
  Finalizers:
    finalizers.fluxcd.io
  Generation:        1
  Resource Version:  86871
  UID:               4193eb4f-ff19-4cb2-9a86-30d92fee99e9
Spec:
  Force:     false
  Interval:  60s
  Kube Config:
    Secret Ref:
      Name:  regional-kubeconfig
  Path:      ./
  Prune:     true
  Source Ref:
    Kind:  GitRepository
    Name:  regional
Status:
  Conditions:
    Last Transition Time:  2025-02-07T14:27:56Z
    Message:               Applied revision: main@sha1:5d1e1fb504749e6d00d3058ea297858544de34dc
    Observed Generation:   1
    Reason:                ReconciliationSucceeded
    Status:                True
    Type:                  Ready
  Inventory:
    Entries:
      Id:                   _free5gc-cp__Namespace
      V:                    v1
      Id:                   free5gc-cp_mongodb__ServiceAccount
      V:                    v1
      Id:                   free5gc-cp_ausf-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_nrf-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_nssf-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_pcf-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_udm-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_udr-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_webui-configmap__ConfigMap
      V:                    v1
      Id:                   free5gc-cp_ausf-nausf__Service
      V:                    v1
      Id:                   free5gc-cp_mongodb__Service
      V:                    v1
      Id:                   free5gc-cp_nrf-nnrf__Service
      V:                    v1
      Id:                   free5gc-cp_nssf-nnssf__Service
      V:                    v1
      Id:                   free5gc-cp_pcf-npcf__Service
      V:                    v1
      Id:                   free5gc-cp_udm-nudm__Service
      V:                    v1
      Id:                   free5gc-cp_udr-nudr__Service
      V:                    v1
      Id:                   free5gc-cp_webui-service__Service
      V:                    v1
      Id:                   free5gc-cp_free5gc-ausf_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_free5gc-nrf_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_free5gc-nssf_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_free5gc-pcf_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_free5gc-udm_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_free5gc-udr_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_free5gc-webui_apps_Deployment
      V:                    v1
      Id:                   free5gc-cp_mongodb_apps_StatefulSet
      V:                    v1
  Last Applied Revision:    main@sha1:5d1e1fb504749e6d00d3058ea297858544de34dc
  Last Attempted Revision:  main@sha1:5d1e1fb504749e6d00d3058ea297858544de34dc
  Observed Generation:      1
Events:
  Type    Reason                   Age                      From                  Message
  ----    ------                   ----                     ----                  -------
  Normal  ReconciliationSucceeded  2m45s (x234 over 3h54m)  kustomize-controller  (combined from similar events): Reconciliation finished in 1.716413245s, next run in 1m0s
```
