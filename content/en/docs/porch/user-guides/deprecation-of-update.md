---
title: "Deprecation of update"
type: docs
weight: 4
description: ""
---

## Motivation

The PackageRevision API object was meant to represent only metadata related to a package revision, while the contents of the package revision (i.e. the YAML files) are meant to be exposed via a companion PackageRevisionResources object. However PackageRevision's spec.tasks field contains all changes applied to the contents of the package revision in the form of patches, thus the contents of the package are leaking into the object that supposed to represent only the metadata. This implies that the PackageRevision can quickly grow bigger in size, than the contents of the package it represents.
For more details, see: https://github.com/nephio-project/nephio/issues/892

## Solution

We have introduced a new first task type, called upgrade. When there is a need to update a downstream package revision to a more up to date revision of its upstream package, do not store unnecessarily the diff's between the package revisions. Instead, now we use a 3-way-merge operation, where the old upstream, new upstream and the local revision changes are merged together. The introduced 3 way merge implementation is based on the kyaml's 3-way-merge solution.
With this approach, we can reduce the task list to only one element, also we can deprecate the Patch/Eval/Update task types, since there will be no need for these. The remaining Init/Edit/Clone/Upgrade task can clearly identify the origin of a PackageRevision.
For more details, see: https://github.com/nephio-project/porch/pull/252

### Example
    porchctl rpkg upgrade repository.package.1 --namespace=porch-demo --revision=2 --workspace=2
This command upgrades the package `repository.package.1` to the second version (revision=2) of its parent package.
It then creates a new package `repository.package.2` with the workspace name specified in the command (workspace=2).

### Migration guide from `update` to `upgrade`
The `upgrade` command now internally handles the functionality previously provided by:
    porchctl rpkg copy --replay-strategy=true
This eliminates the need for users to manually copy a cloned package. Additionally, the `upgrade` command operates on
approved packages.

#### Previous workflow:
    porchctl rpkg copy repository.package-copy.2 --namespace=porch-demo --workspace=3 --replay-strategy=true
    porchctl rpkg update --discover=upstream
    porchctl rpkg update porch-test.subpackage-copy.3 --namespace=porch-demo --revision=2
#### New workflow:
    porchctl rpkg upgrade --discover=upstream
    porchctl rpkg upgrade porch-test.subpackage-copy.2 --namespace=porch-demo --revision=2 --workspace=3
