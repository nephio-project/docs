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

> **_NOTE:_** If you want to use a version other than that at the tip of Nephio `catalog` repo, then replace the `@main` suffix on the package URLs on the `kpt pkg get` commands below with the tag of the version you wish to use.


### Porch

This "Package Orchestration" component provides the Kubernetes APIs for
Repositories, PackageRevisions, PackageRevisionResources, PackageVariants, and
PackageVariantSets. Nephio relies on it to inventory, clone, and mutate
packages. It also provides the API layer that shields the Nephio components
from direct interaction with the Git (or OCI) storage layer.

Fetch the package using `kpt`, and run any `kpt` functions, and then apply the
package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog/nephio/core/proch@main
kpt fn render porch
kpt live init porch
kpt live apply porch --reconcile-timeout=15m --output=table
```

### Nephio Operators

The Nephio Operators provide implementations of the Nephio-specific APIs. This
includes the code that implements the various package specialization features -
such as integration with IPAM and VLAN allocation, and NAD generation - as well
as operators that can provision repositories and bootstrap new clusters into
Nephio.

To install the Nephio Operators, repeat the `kpt` steps, but for that package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/nephio-operator@main
kpt fn render nephio-operator
kpt live init nephio-operator
kpt live apply nephio-operator --reconcile-timeout=15m --output=table
```

### Management Cluster GitOps Tool

In the R1 demo environment, a GitOps tool (ConfigSync) is installed to allow
GitOps-based application of packages to the management cluster itself. This is
not strictly needed if all you want to do is provision network functions, but it
is used extensively in the cluster provisioning workflows.

Different GitOps tools may be used, but these instructions only cover ConfigSync.
To install it on the management cluster, we again follow the same process.
Later, we will configure it to point to the `mgmt` repository:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog/nephio/core/configsync@main
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

### Nephio Stock Repositories

The repositories with the Nephio packages used in the exercises are available to
be installed via a package for convenience. This will install Repository
resources pointing directly to the GitHub repositories, with read-only access.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog/nephio/optional/stock-repos@main
kpt fn render stock-repos
kpt live init stock-repos
kpt live apply stock-repos --reconcile-timeout=15m --output=table
```
