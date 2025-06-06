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