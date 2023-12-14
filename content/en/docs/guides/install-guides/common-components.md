---
title: Installing base Nephio components
description: >
  After installing the environment-specific dependencies, you can install the base Nephio components. There are two
  essential components: Porch, and Nephio Controllers.

weight: 2
---

*Work-in-Progress*

### Porch

This "Package Orchestration" component provides the Kubernetes APIs for
Repositories, PackageRevisions, PackageRevisionResources, PackageVariants, and
PackageVariantSets. Nephio relies on it to inventory, clone, and mutate
packages. It also provides the API layer that shields the Nephio components
from direct interaction with the Git (or OCI) storage layer.

Fetch the package using `kpt`, and run any `kpt` functions, and then apply the
package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/porch-dev@v1.0.1
kpt fn render porch-dev
kpt live init porch-dev
kpt live apply porch-dev --reconcile-timeout=15m --output=table
```

### Nephio Controllers

The Nephio Controllers provide implementations of the Nephio-specific APIs. This
includes the code that implements the various package specialization features -
such as integration with IPAM and VLAN allocation, and NAD generation - as well
as controllers that can provision repositories and bootstrap new clusters into
Nephio.

To install the Nephio Controllers, repeat the `kpt` steps, but for that package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/nephio-controllers@v1.0.1
kpt fn render nephio-controllers
kpt live init nephio-controllers
kpt live apply nephio-controllers --reconcile-timeout=15m --output=table
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
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/configsync@v1.0.1
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

### Nephio Stock Repositories

The repositories with the Nephio packages used in the exercises are available to
be installed via a package for convenience. This will install Repository
resources pointing directly to the GitHub repositories, with read-only access.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/nephio-stock-repos@v1.0.1
kpt fn render nephio-stock-repos
kpt live init nephio-stock-repos
kpt live apply nephio-stock-repos --reconcile-timeout=15m --output=table
```
