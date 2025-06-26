---
title: Installing optional Nephio components
description: >
  After installing the environment-specific dependencies, and base components, the following optional Nephio 
  components can be installed. 

weight: 2
---

{{% alert title="Note" color="primary" %}}

If you want to use a version other than that of main of Nephio *catalog* repo, then replace the *@origin/main*
suffix on the package URLs on the `kpt pkg get` commands below with the tag/branch of the version you wish to use.

While using KPT you can [either pull a branch or a tag](https://kpt.dev/book/03-packages/01-getting-a-package) from a
git repository. By default, it pulls the tag. In case, you have branch with the same name as a tag then to:

```bash
#pull a branch 
kpt pkg get --for-deployment <git-repository>@origin/main
#pull a tag
kpt pkg get --for-deployment <git-repository>@v4.0.0
```

{{% /alert %}}

## Nephio WebUI

The Nephio WebUI can be installed using the following
[document](/content/en/docs/guides/install-guides/web-ui/_index.md)


## FluxCD Controllers

As an alternative Git-ops tool running on the Nephio management cluster, 
the following [Flux](https://fluxcd.io/flux/) controllers can be installed.
* [Source Controller](https://fluxcd.io/flux/components/source/)
* [Kustomize Controller](https://fluxcd.io/flux/components/kustomize/)
* [Helm Controller](https://fluxcd.io/flux/components/helm/)
* [Notification Controller](https://fluxcd.io/flux/components/notification/)

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/fluxcd@origin/main
kpt fn render fluxcd
kpt live init fluxcd
kpt live apply fluxcd --reconcile-timeout=15m --output=table
```

The controllers are deployed to the *flux-system* namespace by default.

```bash
kubectl get po -n flux-system
```
Output:
```
NAME                                      READY   STATUS    RESTARTS   AGE
helm-controller-69c875c978-85tpx          1/1     Running   0          103s
kustomize-controller-596578b94c-gt999     1/1     Running   0          103s
notification-controller-684c9f69c-hpkkq   1/1     Running   0          103s
source-controller-849cd7dbc6-58ghr        1/1     Running   0          103s
```

## O2IMS Operator

Install the operator using the below commands
```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/o2ims@origin/main /tmp/o2ims
kpt fn render /tmp/o2ims
kpt live init /tmp/o2ims
kpt live apply /tmp/o2ims --reconcile-timeout=15m --output=table
```

The operator is deployed to the *o2ims* namespace by default.

```bash
kubectl get pods -n o2ims
```
Output:
```
NAME                              READY   STATUS    RESTARTS   AGE
o2ims-operator-5595cd78b7-thggl   1/1     Running   0          5h27m
```

## Focom Operator

Install the operator using the below commands
```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/focom-operator@origin/main /tmp/focom
kpt fn render /tmp/focom
kpt live init /tmp/focom
kpt live apply /tmp/focom --reconcile-timeout=15m --output=table
```

The operator is deployed to the *focom-operator-system* namespace by default.

```bash
kubectl get pods -n focom-operator-system
```
Output:
```
NAME                                                READY   STATUS    RESTARTS   AGE
focom-operator-controller-manager-d8f4d5cb6-dqqk8   1/1     Running   0          31s
```
