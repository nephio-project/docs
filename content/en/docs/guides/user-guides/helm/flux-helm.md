---
title: Flux CD
description: >
  Deploying helm charts in Nephio using Flux CD
weight: 1
---


[Flux CD](https://fluxcd.io/flux/use-cases/helm/) provides a set of Kubernetes controllers to enable a GitOps driven
deployment of helm charts.

In this example, we deploy the flux [helm](https://fluxcd.io/flux/components/helm/) and
[source](https://fluxcd.io/flux/components/source/) controllers via a kpt package to the target workload cluster.

Then, we can utilize the flux Custom Resources defined in another test kpt package to deploy an example helm chart.

##  Prerequisites:

* [Nephio R1 sandbox]/docs/guides/install-guides/install-guides/_index.md): Set up the Nephio sandbox environment.
* [Access to the Nephio Web UI]({{ relref "/docs/guides/install-guides/_index.md#access-to-the-user-interfaces" }})
* [Nephio R1 sandbox workload clusters]({{ relref "/docs/guides/user-guides/usecase-user-guides/exercise-1-free5gc.md" }}):
  Create/Deploy the predefined set of workload clusters by completing the Free5GC Core quick start exercises up to and including
  [Step 3]({{ relref "/docs/guides/user-guides/usecase-user-guides/exercise-1-free5gc.md#step-3-deploy-two-edge-clusters" }}).

### Deploying the flux-helm-controllers pkg

Access the Nephio WebUI and do following steps:

We will deploy the *flux-helm-controllers* pkg from the *nephio-example-packages* repository to the *edge02* workload
cluster.

* **Step 1**

![Install flux controllers - Step 1](/static/images/user-guides/nephio-ui-edge02-deployment.png)

* **Step 2**

![Install flux controllers - Step 2](/static/images/user-guides/add-deployment-selection.png)

* **Step 3**

![Install flux controllers - Step 3](/static/images/user-guides/flux-controller-selection.png)

Click through the ***Next** button until you are through all the steps, leaving all options as default, then click
**Create Deployment**.

* **Step 4**

![Install flux controllers - Step 4](/static/images/user-guides/select-create-deployment.png)

At this point, we can take a closer look at the contents of the kpt package which contains the relevant Kubernetes
resources to deploy the controllers.

{{% alert title="Note" color="primary" %}}

We are deploying into the *flux-system* namespace by default.

{{% /alert %}}

Finally, we need to *propose* and then *approve* the pkg to initialize the deployment.

* **Step 5**

![Install flux controllers - Step 5](/static/images/user-guides/propose-selection.png)

* **Step 6**

![Install flux controllers - Step 6](/static/images/user-guides/approve-selection.png)

Shortly thereafter, you should see flux helm and source controllers in the flux-system namespace:

```bash
kubectl get po --context edge02-admin@edge02 -n flux-system
```

The output is similar to:

```console
NAME                                 READY   STATUS    RESTARTS   AGE
helm-controller-cccc87cc-zqnd6       1/1     Running   0          6m20s
source-controller-5756bf7d48-hprkn   1/1     Running   0          6m20s
```



### Deploying the onlineboutique-flux pkg

To make the demo kpt packages available in Nephio, we need to register a new *External Blueprints* repository.  We can
do this via *kubectl* towards the management cluster.

```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: nordix-nephio-packages
  namespace: default
  labels:
    kpt.dev/repository-access: read-only
    kpt.dev/repository-content: external-blueprints
spec:
  content: Package
  deployment: false
  git:
    branch: master
    directory: /packages
    repo: https://github.com/Nordix/nordix-nephio-packages.git
  type: git

EOF
```
The new repository should now have been added to the **External Blueprints** section of the UI.

![External Blueprints UI](/static/images/user-guides/external-bp-repos.png)

From here, we can see the onlineboutique-flux pkg to be deployed.

![online boutique pkg](/static/images/user-guides/nephio-pkgs-onlineboutique-show.png)

The HelmRepository Custom Resource within the kpt pkg refers to the official 
[online boutique helm charts repository.](https://github.com/GoogleCloudPlatform/microservices-demo/tree/main/helm-chart)

![Helm repository online boutique ref](/static/images/user-guides/helmrepo-onlineboutique-ref.png)

To deploy the pkg, repeat/follow **Steps 1 - 6** from above, replacing **Step 3** with the following. Take note of the
source repository and the package to be deployed.

![Add ACM pkg](/static/images/user-guides/add-deploy-onlinebout-select.png)

{{% alert title="Note" color="primary" %}}

The overrides online-boutique-values ConfigMap in the package refers to the default *values.yaml* for the
chart and can be customized prior to pkg approval.

{{% /alert %}}

Shortly thereafter, you should see the online boutique components deployed in the online-boutique namespace:

```bash
kubectl get po --context edge02-admin@edge02 -n online-boutique
```

The output is similar to:

```console
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-5464cc8db4-p9sm2               1/1     Running   0          37s
cartservice-6458db7c7c-4scnh             1/1     Running   0          37s
checkoutservice-55b497bfb8-4x8jj         1/1     Running   0          37s
currencyservice-6f868d85d8-fgq6t         1/1     Running   0          37s
emailservice-5cf5fc5898-wzmz8            1/1     Running   0          37s
frontend-56bd99cb9b-thps4                1/1     Running   0          37s
loadgenerator-796b7d99dd-894gx           1/1     Running   0          37s
paymentservice-5ff68d9c7d-74q7c          1/1     Running   0          37s
productcatalogservice-6d9568bddb-8z66q   1/1     Running   0          37s
recommendationservice-c58857d6-qwrkd     1/1     Running   0          37s
redis-cart-7495b4ff99-gbq4m              1/1     Running   0          37s
shippingservice-6f65f85b8b-j5c28         1/1     Running   0          37s
```
