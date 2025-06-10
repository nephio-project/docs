---
title: "Using Conditions and ReadinessGates to manage PackageRevision lifecycle"
type: docs
weight: 4
description: ""
---

A PackageRevision can be configured with readiness information (Conditions and ReadinessGates) to mark it as "ready" to change from one lifecycle status to another. The readiness gates available in Porch are similar in concept to the Kubernetes-native readiness gates used to mark a Pod as ready for use (see [Kubernetes documentation on "Pod conditions" and "Pod readiness"](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions)). However, readiness gates in Porch are a distinct implementation exclusive to Porch, and evaluation of PackageRevision readiness based on readiness gates is performed on a per-operation basis in Porch's code.


## 1. Structure and attributes

A readiness gate applied to a PackageRevision appears in the PackageRevision's YAML manifest as shown below - the relevant elements are:
- `status.conditions`
  - Each Condition in this list represents a detail about the current condition of the PackageRevision
  - Attributes:
    - `message` (string) - human-readable message indicating details about the Condition's current state
    - `reason` (string) - unique, one-word, CamelCase reason for the Condition's current state
    - `status` (string) - current status of the Condition. Valid values are `"True"`, `"False"`, or `"Unknown"`
    - `type` (string) - unique type of the Condition
- `spec.readinessGates`
  - Attributes:
    - `conditionType` (string) refers to a Condition in the `status.conditions` list with matching `type` attribute

As a general rule, for each ReadinessGate in a PackageRevision's `readinessGates` list, if either:
- no Condition in the `conditions` list has `type` matching the ReadinessGate's `conditionType`
- or such a Condition exists with `status` **not** set to `"True"`

then the PackageRevision is counted as "unready". If an attempt is made to propose or approve an unready PackageRevision, the Porch API server will detect this and reject the operation with an error.

```yaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevision
metadata:
  creationTimestamp: "2025-02-05T12:34:29Z"
  name: ...
  ...
spec:
  ...
  readinessGates:
  - conditionType: PackagePipelinePassed
  ...
  tasks:
  - init:
      description: sample description
    type: init
  - eval:
      config: null
      image: render
      match: {}
    type: eval
  ...
status:
  conditions:
  - message: package pipeline completed successfully
    reason: PipelinePassed
    status: "True"
    type: PackagePipelinePassed
  ...
...
```
| |
| :---: |
| *Table 1: ReadinessGate and Condition structure in PackageRevision manifest* |

## 2. Default readiness gates managed by Porch API Server

### 2.1 `PackagePipelinePassed`

By default, every PackageRevision is created with the `PackagePipelinePassed` ReadinessGate and Condition. The `PackagePipelinePassed` Condition monitors the rendering status of the PackageRevision's underlying Kpt pipeline, and is managed internally by the API server as follows:
- the Condition is set to `"False"` before rendering the Kpt pipeline
- the Condition is set to `"True"` only if the pipeline passes (completes with a successful result), as determined by the render operation not having returned any error output

The Kpt pipeline is rendered, with attendant update(s) to the `PackagePipelinePassed` Condition, on initial creation of the PackageRevision (whether through an init, clone, or copy operation) and on any update to the PackageRevision or its contents. In effect, the `PackagePipelinePassed` ReadinessGate and Condition prevent the PackageRevision from being proposed or approved if the pipeline has not passed.

The states of the `PackagePipelinePassed` readiness gate through the lifecycle flow of a PackageRevision is illustrated in *Figure 1* below:

<!--
  Source for Figure 1:
      source diagram file: /static/images/porch/Lifecycle & readiness-gate flows.drawio
      page: "Figure 1: PackageRevision lifecycle and readiness gate flow"
-->
| ![Figure 1: PackageRevision lifecycle and readiness gate flow](/static/images/porch/packagerevision-lifecycle-and-readiness.png) |
| :---: |
| *Figure 1: PackageRevision lifecycle and readiness gate flow* |


## 3. Default readiness gates managed by Porch package variant controller

### 3.1 `PVOperationsComplete`

For a PackageRevision managed by a PackageVariant, the PackageVariant may need to perform multiple operations, potentially triggering the Kpt pipeline several times, before the PackageRevision can be considered "ready" from the PackageVariant's perspective. For example, on the PackageVariant's initial creation of a PackageRevision, the package variant controller performs up to 2 operations:
- an upstream PackageRevision is cloned to create the new, downstream PackageRevision. This triggers a render of the cloned PackageRevision's pipeline
- if the PackageVariant includes additional changes - for example, replacing the Kpt pipeline or injecting configuration - the new PackageRevision's resources are updated. This triggers another render of the pipeline based on the updated resources

To prevent erroneous propose or approve operations from "slipping through the gap" when the default `PackagePipelinePassed` Condition is `"True"` between the clone and resource-update operations, the package variant controller sets a separate ReadinessGate and Condition on the PackageRevision, of type `PVOperationsComplete`. The PackageVariant manages the `PVOperationsComplete` Condition as follows:
- the package variant controller creates the cloned downstream PackageRevision with the `PVOperationsComplete` Condition already in place and set to `"False"`
- the controller sets the Condition `"True"` only if the update operation to add the PackageVariant's additional changes succeeds without error
- for further updates performed according to the PackageVariant, the controller:
  - updates the PackageRevision beforehand to set the `PVOperationsComplete` Condition to `"False"`
  - performs the required update
  - if the update succeeded, updates the PackageRevision again to set the `PVOperationsComplete` Condition to `"True"`
This allows the PackageVariant to ensure that all its operations can be performed in the desired order, without any risk of the PackageRevision being erroneously proposed or approved.


The states of the `PVOperationsComplete` readiness gate, as managed by a PackageVariant in a downstream PackageRevision, is illustrated in *Figure 2* below:

<!--
  Source for Figure 2:
      source diagram file: /static/images/porch/Lifecycle & readiness-gate flows.drawio
      page: "Figure 2: PackageVariant readiness gate flow"
-->
| ![Figure 2: PackageVariant readiness gate flow](/static/images/porch/packagevariant-readiness-flow.png) |
| :---: |
| *Figure 2: PackageVariant readiness gate flow* |


## 4. Additional readiness gates and conditions

To add an additional readiness gate, append a Condition entry to a PackageRevision's `status.conditions` list and a ReadinessGate to the `spec.readinessGates` list, including the attributes listed in [1. Structure and attributes](#1-structure-and-attributes), and using the structure in Table 1 above as an example. This will take effect both at creation-time and in a subsequent edit when applied either via `kubectl` or through the Porch API server's endpoints.
{{% alert title="Note" color="primary" %}}
Any additional readiness gates with `status` other than `"True"` will prevent propose and approve operations from proceeding, in the same manner as the default readiness gates. However, Porch will not otherwise monitor or manage additional gates' status - this must be done by the user or by a controller external to Porch.
{{% /alert %}}