---
title: "Overview"
type: docs
weight: -1
description: Overview of Porch
---

## What is Porch

Porch is a specialization and orchestration tool for managing distributed systems. It helps GitOps engineers,
developers, integrators and telecom operators to manage complex systems in a cloud native environment.  Porch runs
[kpt](https://kpt.dev/) at a scale for package specialization. It provides collaboration and governance enablers and
integration with GitOps for the packages.

## Introduction to Packages in Porch

A Porch package refers to a [kpt package](https://kpt.dev/book/02-concepts/#packages).

As a matter of fact there is no such thing as a "Porch" package but rather Porch utilizes kpt packages which are a
structured collection of Kubernetes [YAML](https://en.wikipedia.org/wiki/YAML) resources along with a Kptfile which is
managed via Porch's package orchestration mechanisms.

### What is kpt?

[Kpt is an open source tool](https://kpt.dev/book/02-concepts/#what-is-kpt) started by Google to manage Kubernetes
[“Configuration as Data”]({{% relref "/docs/porch/config-as-data.md" %}}) to simplify management of
[Kubernetes Resource Model](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md)
(KRM)-driven infrastructure at scale using standard YAML and [Git](https://git-scm.com/). 

### What is a kpt package?

A kpt package is a bundle of configuration data. More about the concept in the
[kpt documentation](https://kpt.dev/book/02-concepts/#what-is-kpt).


### Relationship with KRM

[WHAT IS THE RELATIONSHIP WITH KRM]

### Porch pkg untainted by telco

[WE CAN ADD SOME EXAMPLE KPT PKG UNTAINTED BY TELCO].

<https://www.est.tech/news/kptreboot/>
