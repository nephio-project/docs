---
title: Installing base Nephio components
description: >
  After installing the environment-specific dependencies, you can install the base Nephio components. There are two
  essential components: Porch, and Nephio Controllers.

weight: 2
---

{{% pageinfo %}}
This page is draft and the separation of the content to different categories is not clearly done. 
{{% /pageinfo %}}


{{% alert title="Note" color="primary" %}}

If you want to use a version other than that at the tip of Nephio `catalog` repo, then replace the `@main` suffix on the package URLs on the `kpt pkg get` commands below with the tag of the version you wish to use.

{{% /alert %}}

## Porch

This "Package Orchestration" component provides the Kubernetes APIs for
Repositories, PackageRevisions, PackageRevisionResources, PackageVariants, and
PackageVariantSets. Nephio relies on it to inventory, clone, and mutate
packages. It also provides the API layer that shields the Nephio components
from direct interaction with the Git (or OCI) storage layer.

Fetch the package using `kpt`, and run any `kpt` functions, and then apply the
package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog/nephio/core/porch@main
kpt fn render porch
kpt live init porch
kpt live apply porch --reconcile-timeout=15m --output=table
```

## Nephio Operators

The Nephio Operators provide implementations of the Nephio-specific APIs. This
includes the code that implements the various package specialization features -
such as integration with IPAM and VLAN allocation, and NAD generation - as well
as operators that can provision repositories and bootstrap new clusters into
Nephio.


To install the Nephio Operators, repeat the `kpt` steps, but for that package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/nephio-operator@main
```

The Nephio Operator package by default uses the Gitea instance at `172.18.0.200:3000` as 
the git repository. Change it to point to your git instance in  
`nephio-operator/app/controller/deployment-token-controller.yaml` and 
`nephio-operator/app/controller/deployment-controller.yaml`

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

A GitOps tool (ConfigSync) is installed to allow
GitOps-based application of packages on the management cluster itself. It is
not needed if you only want to provision network functions, but it
is used extensively in the cluster provisioning workflows.

Different GitOps tools may be used, but these instructions only cover ConfigSync.
To install it on the management cluster, we again follow the same process.
Later, we will configure it to point to the `mgmt` repository:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@main
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

## Nephio Stock Repositories

The repositories with the Nephio packages used in the exercises are available to
be installed via a package for convenience. This will install repository
resources pointing directly to the GitHub repositories, with read-only access.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/stock-repos@main
kpt fn render stock-repos
kpt live init stock-repos
kpt live apply stock-repos --reconcile-timeout=15m --output=table
```

## WebUI (Optional)

Nephio WebUI is optional and to install it you can follow this [document](/content/en/docs/guides/install-guides/webui.md)  