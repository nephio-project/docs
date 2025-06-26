---
title: Centralized ArgoCD Workload Cluster Deployment
description: >
  A step by step guide to deploy a sample workload to a workload cluster, 
  using a centralized GitOps approach with ArgoCD.
weight: 2
---

## Prerequisites

- A Nephio Management cluster. See the [installation guides](/content/en/docs/guides/install-guides/_index.md) 
for detailed environment options.
- An ArgoCD installation, such as the [ArgoCD Full](https://github.com/nephio-project/catalog/tree/main/nephio/optional/argo-cd-full) package.

{{% alert title="Note" color="primary" %}}

If using a [sandbox demo environment](/content/en/docs/guides/install-guides/_index.md#kicking-off-an-installation-on-a-virtual-machine), 
most of the above prerequisites are already satisfied.

{{% /alert %}}

This exercise will take us from a system with only the Nephio Management cluster setup, to a deployment with:

- A [ArgoCD workload cluster](https://github.com/nephio-project/catalog/tree/main/infra/capi/nephio-workload-cluster-argo).
- A repository for said cluster, registered with Nephio Porch.
- A sample workload deployed to said cluster via an ArgoCD Config Management Plugin (CMP).

To perform these exercises, we will need:

- Access to the installed demo VM environment as the ubuntu user.
- Access to the Nephio WebUI as described in the installation guide.

Access to Gitea, used in the demo environment as the Git provider, is optional.

Access to the ArgoCD UI is optional, but highly recommended.

### Step 1: Deploy the ArgoCD Workload Cluster

{{% alert title="Note" color="primary" %}}

After fresh docker install, verify the docker supplementary group is loaded by executing `id | grep docker`.
If not, logout and login to the VM or execute the `newgrp docker` to ensure the docker supplementary group is loaded.

{{% /alert %}}

First, assuming the [ArgoCD Full](https://github.com/nephio-project/catalog/tree/main/nephio/optional/argo-cd-full) package is installed, apply our [ArgoCD KPT CMP Patch](https://github.com/nephio-project/nephio/tree/main/gitops-tools/kpt-argocd-cmp) to create the KPT Repo and KPT Render Config Management Plugins (CMPs). For convenience, there is an example patch wrapped in a shell script at the previous link. Both options are documented here:

Either apply the patch directly, changing the version of the plugins:
```json
kubectl patch deployment argocd-repo-server -n argocd --type json -p='[
   {
    "op": "add",
    "path": "/spec/template/spec/containers/-",
    "value": {
      "name": "kpt-repo-argo-cmp",
      "image": "docker.io/nephio/kpt-repo-argo-cmp:VERSION",
      "command": ["/var/run/argocd/argocd-cmp-server"],
      "securityContext": {
          "runAsNonRoot": true,
          "runAsUser": 999
      },
      "volumeMounts": [
        {
           "name": "var-files",
           "mountPath": "/var/run/argocd"
        },
        {
           "name": "cmp-tmp",
           "mountPath": "/tmp"
        },
        {
           "name": "plugins",
           "mountPath": "/home/argocd/cmp-server/plugins"
        }
      ]
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/-",
    "value": {
      "name": "kpt-render-argo-cmp",
      "image": "docker.io/nephio/kpt-render-argo-cmp:VERSION",
      "command": ["/var/run/argocd/argocd-cmp-server"],
      "securityContext": {
          "runAsNonRoot": true,
          "runAsUser": 999
      },
      "volumeMounts": [
        {
           "name": "var-files",
           "mountPath": "/var/run/argocd"
        },
        {
           "name": "cmp-tmp",
           "mountPath": "/tmp"
        },
        {
           "name": "plugins",
           "mountPath": "/home/argocd/cmp-server/plugins"
        }
      ]
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
       "name": "cmp-tmp",
       "emptyDir": {}
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
       "name": "var-run-argocd",
       "emptyDir": {}
    }
  }
]'
```

or, apply the patch via our shell script (which defaults to the `latest` tag):
```bash
curl -s https://raw.githubusercontent.com/nephio-project/nephio/refs/heads/main/gitops-tools/kpt-argocd-cmp/patch.sh > /tmp/patch.sh
/bin/bash /tmp/patch.sh
```

Now, verify that the patch has been applied, and the two CMPs are added to the argocd-repo-server (creating 3 containers):
```bash
kubectl get pods -n argocd | grep argocd-repo-server
```
Sample output:
```bash
argocd-repo-server-6d887d9cfc-9jvds                3/3     Running   0          3h29m
```

Next, verify that the [catalog blueprint repositories](https://github.com/nephio-project/catalog.git) are registered 
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

In this example, we will use the [Porch package variant controller](/content/en/docs/porch/package-variant.md#core-concepts) 
to deploy the new Workload Cluster.

This fully automates the onboarding process, including the auto approval and publishing of the new package.

{{% alert title="Note" color="primary" %}}

{{% /alert %}}

Create a new *PackageVariant* CR for the Workload Cluster:

```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: argo-regional-cluster
spec:
  upstream:
    repo: catalog-infra-capi
    package: nephio-workload-cluster-argo
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
packagevariant.config.porch.kpt.dev/argo-regional-cluster created
```

Config Sync running in the Management cluster will create all the resources necessary to
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

## Step 3: Investigate the ArgoCD specific CRs

Verify that the regional-flux-gitrepo-kustomize *PackageRevision* has been created for the ArgoCD specific CRs. 
We want the *Published* v1 revision. 
```bash
kubectl get packagerevision | grep "regional-argo-gitrepo"
```
Sample output:
```
mgmt.regional-argo-gitrepo.main                                 regional-argo-gitrepo                main               -1         false    Published   mgmt
mgmt.regional-argo-gitrepo.packagevariant-1                     regional-argo-gitrepo                packagevariant-1   1          true     Published   mgmt
```
Once verified:

Check the *Ready* state of the regional Git Repository:
```bash
kubectl get repository regional
```
Sample output:
```
NAME       TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
regional   git    Package   true         True    http://172.18.0.200:3000/nephio/regional.git
```

Now, we can get more details of the GitRepository CR:
```bash
kubectl get repository regional -o jsonpath='{.spec}'
```
Sample output:
```json
{
  "content":"Package",
  "deployment":true,
  "git":
  {
    "branch":"main",
    "directory":"/",
    "repo":"http://172.18.0.200:3000/nephio/regional.git",
    "secretRef":
    {
      "name":"regional-access-token-porch"
    }
  },
  "type":"git"
}
```

The default git *repo* points to the internal Gitea repository created as part of the Workload Cluster installation, 
along with an associated *secretRef* to access it. It also defaults to reference the **main** `branch` of the repository and the `root` path.

Check that the jobs used to create the ArgoCD repo and cluster secrets are complete, and that the secrets were created in the `argocd` namespace:
```bash
kubectl get jobs
kubectl get secrets -n argocd | grep regional
```
Sample output:
```bash
NAME                                             STATUS     COMPLETIONS   DURATION   AGE
regional-create-argocd-cluster-from-kubeconfig   Complete   1/1           4s         8s
regional-create-argocd-repo-secret-from-porch    Complete   1/1           3s         73s
cluster-regional              Opaque   3      4h22m
regional-repo                 Opaque   4      4h22m
```

Next check the *Ready* state of the associated ArgoCD App, ensuring that it is Synced and Healthy. This App will point at the `kpt-repo` CMP, acting as an app-of-apps for the regional cluster. The apps created subsequently through the workloads will target the `kpt-render` CMP:
```bash
kubectl get apps -n argocd
```
Sample output:
```bash
NAME                    SYNC STATUS   HEALTH STATUS
regional                Synced        Healthy
```

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

We can also check the ArgoCD apps to see our free5gc sub-app created:
```bash
kubectl get apps -n argocd
```
Sample output:
```bash
NAME                  SYNC STATUS   HEALTH STATUS
regional              Synced        Healthy
regional-free5gc-cp   Synced        Healthy
```

# Step 5: Remove Kubernetes Jobs (optional)

Because ConfigSync is always trying to reconcile the manifests into the cluster, and because the pods started by our Kubernetes Jobs to create ArgoCD specific Secrets eventually end, they will run indefinitely. To remove these restarts from your cluster, you must inform ConfigSync that it should no longer manage the Jobs by using the `http://configmanagement.gke.io/managed` annotation. These steps are not necessary for functionality, but they do unclutter the Jobs, Pods, and presumably reduce resource use.

*NOTE: This can be updated with porchctl steps, once defined*

To do this, clone the mgmt repo, add the annotation to the Jobs, and then push them to the mgmt repo (authentication can be through the Git repo tokens or the default Gitea credentials, depending on installation). This will prevent ConfigSync from reconciling these manifests going forward:
```bash
git clone http://172.18.0.200:3000/nephio/mgmt.git
cd mgmt/regional-argo-gitrepo/
kpt fn eval --image "gcr.io/kpt-fn/set-annotations:v0.1.4" create-argocd-kubeconfig-secret-job.yaml -- "configmanagement.gke.io/managed=disabled"
kpt fn eval --image "gcr.io/kpt-fn/set-annotations:v0.1.4" create-argocd-repo-secret-job.yaml -- "configmanagement.gke.io/managed=disabled"
git add create-argocd-kubeconfig-secret-job.yaml create-argocd-repo-secret-job.yaml && git commit -m "Unmanaged Jobs" && git push
```

Sample output:
```bash
[RUNNING] "gcr.io/kpt-fn/set-annotations:v0.1.4"
[PASS] "gcr.io/kpt-fn/set-annotations:v0.1.4" in 1.4s
[RUNNING] "gcr.io/kpt-fn/set-annotations:v0.1.4"
[PASS] "gcr.io/kpt-fn/set-annotations:v0.1.4" in 1.3s
[main e5f5c72] Unmanaged Jobs
 2 files changed, 4 insertions(+)
...
   b110ac9..e5f5c72  main -> main

```

After the Jobs and Pods expire (this may take a few minutes), you should no longer see Jobs being recreated:
```bash
kubectl get jobs
```

Sample output:
```bash
No resources found in default namespace.
```