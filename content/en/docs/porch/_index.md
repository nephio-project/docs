---
title: "Porch documentation"
type: docs
weight: 6
description: Documentation of Porch
---

## Overview

 Porch is “kpt-as-a-service”, providing opinionated package management, manipulation, and lifecycle operations in a
 Kubernetes-based API. This allows automation of these operations using standard Kubernetes controller techniques.

Short for Package Orchestration.

## Porch in the Nephio architecture, history and outlook

Porch is a key component of the Nephio architecture and was originally developed in the
[kpt](https://github.com/kptdev/kpt) project. When kpt was donated to the [CNCF](https://www.cncf.io/projects/kpt/) it
was decided that Porch would not be part of the kpt project and the code was donated to Nephio.

Porch is now maintained by the Nephio community and it is a stable part of the Nephio R3 architecture. However there is
an active discucssion about the future of the project. It is possible that the current Porch component will be replaced
in the Nephio architecture with a different codebase implementing the same concepts but not in a backward compatible
way. Potential candidates such as [pkgserver](https://docs.pkgserver.dev/) are being discussed in the Nephio community. 