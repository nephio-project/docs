---
title: "Overview"
type: docs
weight: -1
description: Overview of Porch
---

## Introduction to Packages in Porch

A Porch package refers to a KPT package.

As a matter of fact there is no such thing as a "Porch" package but rather Porch utilizes KPT packages which are a structured collection of Kubernetes [YAML](https://en.wikipedia.org/wiki/YAML) resources along with a Kptfile which is managed via Porch's package orchestration mechanisms.

### What is KPT?

KPT is an open-source tool developed by Google to manage Kubernetes [“Configuration as Data”](../config-as-data.md) to simplify management of [Kubernetes Resource Model](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md) (KRM)-driven infrastructure at scale using standard YAML and [Git](https://git-scm.com/). KPT originated as a Google ContainerTools project and was later contributed to the CNCF.
Think of kpt as "package management for Kubernetes configurations", similar to how [apt](https://en.wikipedia.org/wiki/APT_(software)) or [yum](https://en.wikipedia.org/wiki/Yum_(software)) works for software packages but in this case, it deals with YAML files that define Kubernetes resources.

### What is a KPT package?

| Concept           | Description                                                                                           |
| ----------------- | ----------------------------------------------------------------------------------------------------- |
| **KPT package**   | A directory of Kubernetes YAML files (manifests) with a `Kptfile` that tracks metadata and lifecycle. |
| **Kptfile**       | A metadata file that defines pipeline functions, upstream sources (like Git repos), and ownership.    |
| **Functions**     | Extensible KRM functions (as containers or binaries) that mutate or validate package content.         |

### What Does a kpt Package Look Like?

```bash
my-nginx/
├── Kptfile
├── deployment.yaml
└── service.yaml
```

`Kptfile`: Specifies metadata, upstream repo, function pipeline, etc.
`deployment.yaml` & `service.yaml`: Standard Kubernetes resource manifests.

[POINT TO KPT DOCS FOR KPT PKG]

### Example of a KPT file

[KPT FILE],

### Relationship with KRM

[WHAT IS THE RELATIONSHIP WITH KRM]

### Porch pkg untainted by telco

[WE CAN ADD SOME EXAMPLE KPT PKG UNTAINTED BY TELCO].

<https://www.est.tech/news/kptreboot/>
