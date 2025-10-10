---
title: "Configuration as Data (CaD)"
type: docs
weight: 1
description: 
---

This document provides the background context for Package Orchestration, which is further
elaborated in a dedicated [document](package-orchestration.md).

## Configuration as data (CaD)

CaD is an approach to the management of configuration. It includes the configuration of
infrastructure, policy, services, applications, and so on. CaD performs the following actions:

* Making configuration data the source of truth, stored separately from the live state.
* Using a uniform, serializable data model to represent the configuration.
* Separating the code that acts on the configuration from the data and from packages/bundles of
  data.
* Abstracting the configuration file structure and storage from the operations that act on the
  configuration data. Clients manipulating the configuration data do not need to interact directly
  with the storage (such as git, container images, and so on).

![CaD Overview](/static/images/porch/CaD-Overview.svg)

## Key principles

A system based on CaD should observe the following key principles:

* Separate handling of secrets in secret storage, in a secret-focused storage system, such as
  ([example](https://cert-manager.io/)).
* Storage of a versioned history of configuration changes by change sets to bundles of related
  configuration data.
* Reliance on the uniformity and consistency of the configuration format, including type metadata,
  to enable pattern-based operations on the configuration data, along the lines of duck typing.
* Separation of the configuration data from its schemas, and reliance on the schema information for
  strongly typed operations and disambiguation of data structures and other variations within the
  model.
* Decoupling of abstractions of configuration from collections of configuration data.
* Representation of abstractions of configuration generators as data with schemas, as with other
  configuration data.
* Finding, filtering, querying, selecting, and/or validating of configuration data that can be
  operated on by given code (functions).
* Finding and/or filtering, querying, and selecting of code (functions) that can operate on
  resource types contained within a body of configuration data.
* Actuation (reconciliation of configuration data with live state) that is separate from the
  transformation of the configuration data, and is driven by the declarative data model.
* Transformations. Transformations, particularly value propagation, are preferable to wholesale
  configuration generation, except when the expansion is dramatic (for example, >10x).
* Transformation input generation: this should usually be decoupled from propagation.
* Deployment context inputs: these should be taken from well-defined “provider context” objects.
* Identifiers and references: these should be declarative.
* Live state: this should be linked back to sources of truth (configuration).

## Kubernetes Resouce Model configuration as data (KRM CaD)

Our implementation of the Configuration as Data approach (
[kpt](https://kpt.dev),
[Config Sync](https://cloud.google.com/anthos-config-management/docs/config-sync-overview),
and [Package Orchestration](https://github.com/nephio-project/porch))
is built on the foundation of the
[Kubernetes Resource Model](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md)
(KRM).

{{% alert title="Note" color="primary" %}}

Even though KRM is not a requirement of CaD (just as Python or Go templates, or Jinja, are not
specifically requirements for [IaC](https://en.wikipedia.org/wiki/Infrastructure_as_code)), the
choice of another foundational configuration representation format would necessitate the
implementation of adapters for all types of infrastructure and applications configured, including
Kubernetes, CRDs, GCP resources, and more. Likewise, choosing another configuration format would
require the redesign of several of the configuration management mechanisms that have already been
designed for KRM, such as three-way merge, structural merge patch, schema descriptions, resource
metadata, references, status conventions, and so on.

{{% /alert %}}


**KRM CaD** is, therefore, a specific approach to implementing *Configuration as Data* which uses
the following:

* [KRM](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md)
  as the configuration serialization data model.
* [Kptfile](https://kpt.dev/reference/schema/kptfile/) to store package metadata.
* [ResourceList](https://kpt.dev/reference/schema/resource-list/) as a serialized package wire
  format.
* A function `ResourceList → ResultList` (*kpt* function) as the foundational, composable unit of
  package manipulation code.
  
  {{% alert title="Note" color="primary" %}}

  Other forms of code can also manipulate packages, such as UIs and custom algorithms not
  necessarily packaged and used as kpt functions.

  {{% /alert %}}


**KRM CaD** provides the following basic functionalities:

* Loading a serialized package from a repository (as a ResourceList). Examples of a repository may
  be one or more of the following:
  * Local HDD
  * Git repository
  * OCI
  * Cloud storage
* Saving a serialized package (as a ResourceList) to a package repository.
* Evaluating a function on a serialized package (ResourceList).
* [Rendering](https://kpt.dev/book/04-using-functions/#declarative-function-execution) a package
  (evaluating the functions declared within the package itself).
* Creating a new (empty) package.
* Forking (or cloning) an existing package from one package repository (called upstream) to another
  (called downstream).
* Deleting a package from a repository.
* Associating a version with the package and guaranteeing the immutability of packages with an
  assigned version.
* Incorporating changes from the new version of an upstream package into a new version of a
  downstream package (three-way merge).
* Reverting to a prior version of a package.

## Configuration values

The configuration as data approach enables some key values which are available in other
configuration management approaches to a lesser extent or not at all.

The values enabled by the configuration as data approach are as follows:

* Simplified authoring of the configuration using a variety of methods and sources.
* What-you-see-is-what-you-get (WYSIWYG) interaction with the configuration using a simple data
  serialization formation, rather than a code-like format.
* Layering of interoperable interface surfaces (notably GUIs) over declarative configuration
  mechanisms, rather than forcing choices between exclusive alternatives (exclusively, UI/CLI or
  IaC initially, followed by exclusively UI/CLI or exclusively IaC).
* The ability to apply UX techniques to simplify configuration authoring and viewing.
* Compared to imperative tools, such as UI and CLI, that directly modify the live state via APIs,
  CaD enables versioning, undo, audits of configuration history, review/approval, predeployment
  preview, validation, safety checks, constraint-based policy enforcement, and disaster recovery.
* Bulk changes to configuration data in their sources of truth.
* Injection of configuration to address horizontal concerns.
* Merging of multiple sources of truth.
* State export to reusable blueprints without manual templatization.
* Cooperative editing of configurations by humans and automation, such as for security remediation,
  which is usually implemented against live-state APIs.
* Reusability of the configuration transformation code across multiple bodies of configuration data
  containing the same resource types, amortizing the effort of writing, testing, and documenting
  the code.
* A combination of independent configuration transformations.
* Implementation of configuration transformations using the languages of choice, including both
  programming and scripting approaches.
* Reducing the frequency of changes to the existing transformation code.
* Separation of roles between developer and non-developer configuration users.
* Defragmenting the configuration transformation ecosystem.
* Admission control and invariant enforcement on sources of truth.
* Maintaining variants of configuration blueprints without one-size-fits-all full
  struct-constructor-style parameterization and without manually constructing and maintaining
  patches.
* Drift detection and remediation for most of the desired state via continuous reconciliation,
  using apply and/or for specific attributes via a targeted mutation of the sources of truth.

## Related articles

For more information about configuration as data and the Kubernetes Resource Model, visit the
following links:

* [Rationale for kpt](https://kpt.dev/guides/rationale)
* [Understanding Configuration as Data](https://cloud.google.com/blog/products/containers-kubernetes/understanding-configuration-as-data-in-kubernetes)
  blog post
* [Kubernetes Resource Model](https://cloud.google.com/blog/topics/developers-practitioners/build-platform-krm-part-1-whats-platform)
  blog post series
