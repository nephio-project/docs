---
title: Installing common dependencies
description: >
  This guide describes how to install some required dependencies that are the same across all environments.

weight: 1
---

Some of these, like the resource-backend, will move out of the "required"
category in later releases.  Even if you do not use these directly in your
installation, the CRDs that come along with them are necessary.

> **_NOTE:_** If you want to use a version other than that at the tip of Nephio `catalog` repo, then replace the `@main` suffix on the package URLs on the `kpt pkg get` commands below with the tag of the version you wish to use.

### Network Config Operator

This component is a controller for applying configuration to routers and
switches.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/network-config@main
kpt fn render network-config
kpt live init network-config
kpt live apply network-config --reconcile-timeout=15m --output=table
```

### Resource Backend

The resource backend provides IP and VLAN allocation.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog/nephio/optional/resource-backend@main
kpt fn render resource-backend
kpt live init resource-backend
kpt live apply resource-backend --reconcile-timeout=15m --output=table
```
