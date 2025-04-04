---
title: Installing base Nephio components
description: >
  After installing the environment-specific dependencies, you can install the base Nephio components. There are two
  essential components: Porch, and Nephio Controllers.

weight: 2
---

{{% pageinfo %}}
This page is draft and the separation of the content to different categories is not done. 
{{% /pageinfo %}}


{{% alert title="Note" color="primary" %}}

If you want to use a version other than that of v3.0.0 of Nephio *catalog* repository, then replace the *@origin/v3.0.0*
suffix on the package URLs on the `kpt pkg get` commands below with the tag/branch of the version you wish to use.

While using kpt you can [either pull a branch or a tag](https://kpt.dev/book/03-packages/01-getting-a-package) from a
git repository. By default it pulls the tag. In case, you have branch with the same name as a tag then to:

```bash
#pull a branch 
kpt pkg get --for-deployment <git-repository>@origin/v3.0.0
#pull a tag
kpt pkg get --for-deployment <git-repository>@v3.0.0
```

{{% /alert %}}

## Porch

This "Package Orchestration" component provides the Kubernetes APIs for Repositories, PackageRevisions,
PackageRevisionResources, PackageVariants, and PackageVariantSets. Nephio relies on it to inventory, clone, and mutate
packages. It also provides the API layer that shields the Nephio components from direct interaction with the Git
(or OCI) storage layer.

Fetch the package using `kpt`, and run any `kpt` functions, and then apply the package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog/nephio/core/porch@origin/v3.0.0
kpt fn render porch
kpt live init porch
kpt live apply porch --reconcile-timeout=15m --output=table
```

## Nephio Operators

The Nephio Operators provide implementations of the Nephio-specific APIs. This includes the code that implements the
various package specialization features - such as integration with IPAM and VLAN allocation, and NAD generation - as
well as operators that can provision repositories and bootstrap new clusters into Nephio.

To install the Nephio Operators, repeat the `kpt` steps, but for that package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/nephio-operator@origin/v3.0.0
```

The Nephio Operator package by default uses the Gitea instance at *172.18.0.200:3000* as 
the git repository. Change it to point to your git instance in  
*nephio-operator/app/controller/deployment-token-controller.yaml* and 
*nephio-operator/app/controller/deployment-controller.yaml*.

You also need to create a secret with your Git instance credentials: 

```bash
kubectl apply -f  - <<EOF
apiVersion: v1
kind: Secret
metadata:
    name: git-user-secret
    namespace: nephio-system
type: kubernetes.io/basic-auth
stringData:
    username: <GIT_USER_NAME>
    password: <GIT_PASSWORD>
EOF
```

Now you can continue the installation process:

```bash
kpt fn render nephio-operator
kpt live init nephio-operator
kpt live apply nephio-operator --reconcile-timeout=15m --output=table
```

## Management Cluster GitOps Tool

A GitOps tool (configsync) is installed to allow GitOps-based application of packages on the management cluster itself.
It is not needed if you only want to provision network functions, but it is used extensively in the cluster provisioning
workflows.

Different GitOps tools may be used, but these instructions only cover configsync.
To install it on the management cluster, we again follow the same process.
Later, we will configure it to point to the *mgmt* repository:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@origin/v3.0.0
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

## Nephio Stock Repositories

The repositories with the Nephio packages used in the exercises are available to be installed via a package for
convenience. This will install repository resources pointing directly to the GitHub repositories, with read-only access.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/stock-repos@origin/v3.0.0
kpt fn render stock-repos
kpt live init stock-repos
kpt live apply stock-repos --reconcile-timeout=15m --output=table
```
