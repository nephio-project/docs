---
title: "Package Orchestration Concepts"
type: docs
weight: 2
---

### Package Orchestration: Why?

The benefits of [Configuration as Data](TODO:REFER_TO_DOCUMENT_UNDER_1_overview_ONCE_WRITTEN) (CaD) are already
available in CLI form, using [kpt](https://kpt.dev) and the kpt function ecosystem, including a hosted [functions catalog](https://catalog.kpt.dev/).
[YAML](https://yaml.org/) files can be created and organised into packages using any editor with YAML support. However,
a UI experience of WYSIWYG package management is not yet available which can support broader package lifecycle management
and necessary development guardrails.

*Package Orchestration* (Porch) enables development of such a UI experience. Part of the Nephio Configuration as Data
implementation, it offers an API and CLI which provide lifecycle management of kpt packages, including package authoring
with guardrails, a proposal/approval workflow, package deployment, and more.


## Core Concepts

Some core concepts of package orchestration:

***Package***: specifically, a [kpt package](https://kpt.dev/) - a collection of related YAML files containing one or
more **[KRM resources][krm]**. **N.B.**: there is no such thing as a "Porch Package", but **kpt packages can be stored
in/managed by Porch**.

***Repository***: a version-control [repository](./concepts_elaborated.md#repositories) used to store packages. For
example, a [Git][git] or [OCI][oci] repository.

***Package Revision***: packages are sequentially ***[versioned](./concepts_elaborated.md#package-versioning)*** and
multiple versions of the same package may exist in a repository. Each successive version is considered a *package revision*.

***Lifecycle***: a package revision may be in one of several lifecycle stages:

* ***Draft*** - the package is being created or edited. The package contents can be modified but the package revision is not
  ready to be used/deployed. Previously-published package revisions, reflecting earlier states of the package files, can
  still be deployed
* ***Proposed*** - intermediate state. The package's author has proposed that a new version of the package be published
  with its files in the current state
* ***Published*** - the changes to the package have been approved and the package is ready to be used. Published packages
  may be deployed, cloned to a new package, or edited to continue development
* ***DeletionProposed*** - intermediate state. A user has proposed that this version of the package be deleted from the
  repository

***Functions***: specifically, [KRM functions][krm functions]. Functions can be added to a package's kptfile [pipeline][pipeline]
in the course of modifying a package revision in *Draft* state. Porch runs the pipeline on the package contents, mutating
or validating the KRM resource files.

***Upstream package revision***: a package revision (a specific version of a package) may be cloned, producing a new,
***downstream package*** and package revision. The downstream package maintains a link (URL) to the upstream from which
it was cloned. ([more details](./concepts_elaborated.md#package-relationships---upstream-and-downstream))

***Package Variant*** and ***Package Variant Set***: higher levels of package revision automation. Package variants can
be used to automatically track an upstream package (at a specific revision) and manage cloning it to one or several
downstream packages, as well as preparing new downstream package revisions when a new revision of the upstream package
is published. Package variant sets enable the same behaviour for package variants themselves. Use of package variants
involves advanced concepts worthy of their own separate document: [Package Variants](../5_architecture_&_components/controllers/pkg-variant-controllers.md)

***Deployment repository***: a repository can be designated as a deployment repository. Package revisions in *Published*
state in a deployment repository are considered [deployment-ready](./concepts_elaborated.md#deployment).

***Package revision workspace***, or `workspaceName`: a user-defined string and element of package revision names automatically
assembled by Porch. Used to uniquely identify a package revision while in *Draft* state, especially to distinguish between
multiple drafts undergoing concurrent development. **N.B.**: a package revision workspace does not refer to any distinct
"folder" or "space", but only to the in-development draft. The same workspace name may be assigned to multiple package
revisions **of different packages** and **does not of itself indicate any connection between the packages**.

Some of these concepts bear examination in more detail - see [Package Orchestration Concepts Elaborated](./concepts_elaborated.md)


## Core Components of Configuration as Data Implementation

The CaD implementation consists of a set of components and APIs enabling the following broad use cases:

* Register repositories (Git, OCI) containing kpt packages
* Automatically discover existing packages in registered repositories
* Manage package revision lifecycle, including:
  * authoring and versioning of a package through creation, mutation, and deletion of package revision drafts
  * a 2-step approval process where a draft package revision is first proposed for publishing, and only published on a
    second (approval) operation
* Manage package lifecycle - operations such as:
  * package upgrade - assisted or automated rollout of a downstream (cloned) package when a new version of the upstream
    package revision is published
  * package rollback to a previous package revision
* Deploy packages from deployment repositories and observe their deployment status
* Role-based access control to Porch APIs via Kubernetes standard roles

### High-Level Architecture

At the high level, the CaD functionality comprises:

* a generic (i.e. not task-specific) package orchestration service implementing
  * package revision authoring and lifecycle management
  * package repository management

* [porchctl](../7_cli_api/porchctl.md) - a Git-native, schema-aware, extensible client-side tool for managing
  KRM packages in Porch
* a GitOps-based deployment mechanism (for example [Config Sync][Config Sync]), which distributes and deploys configuration,
  and provides observability of the status of deployed resources
* a task-specific UI supporting repository management, package discovery, authoring, and lifecycle

![CaD Core Architecture](/static/images/porch/CaD-Core-Architecture.svg)

## Package Orchestration - Porch

Having established the context of the CaD components and the overall architecture, the remainder of the document
will focus on **Porch** - the Package Orchestration service.

To reiterate the role of Porch among the CaD components, it covers:

* [Repository Management](#repository-management)
* [Package Discovery](#package-discovery)
* [Package Authoring](#package-authoring) and Lifecycle

The following section expands more on each of these areas. The term *client* used in these sections can be either a person
interacting with the API (e.g., through a web application or a command-line tool), or an automated agent or process.

### Repository Management

Porch's repository management functionality enables the client to manage Porch repositories:

* Register (create) and unregister (delete) repositories
  * a repository may be registered as a *deployment* repository to indicate that it contains deployment-ready packages
* Discover (read) and update registered repositories
  * since Porch repositories are Kubernetes objects, the update operation may be used to add arbitrary metadata, in the
    form of annotations or labels, for the benefit of applications or customers

Git repository integration is available, with limited experimental support for OCI.

### Package Discovery

Porch's package discovery functionality enables the client to:

* List package revisions in registered repositories
  * and sort/filter based on package metadata (labels) or a selection of field values
  * package revisions are automatically discovered and cached in Porch upon repository registration. Porch then polls the
    repository at a user-customisable interval to keep the cache up to date.
* Retrieve details of an individual package revision
* Discover upstream packages with new latest revisions to which their downstream packages can be upgraded
* Identify deployment-ready packages that are available to be deployed by the chosen deployment software

### Package Authoring

Porch's package lifecycle management enables the client to:

* Create a *draft* package revision in any of the following ways:
  * create an empty draft 'from scratch' (`porchctl rpkg init`)
  * clone an upstream package (`porchctl rpkg clone`) from either a registered upstream repository or from an unregistered
    repository accessible by URL
  * edit an existing package (`porchctl rpkg edit`)

* Retrieve the contents of a package's files for local review or editing (`porchctl rpkg pull`)

* Manage approval status of a package revision:
  * Propose a *draft* package for publication, moving it to *proposed* status
  * Reject a *proposed* package, setting it back to *draft* status
  * Approve a *proposed* package, releasing it to *published* status.

* Update the package contents of a draft package revision by pushing an edited local copy to the draft (`porchctl rpkg push`).
  Example edits:
  * add/change/delete resources in the package
  * add/change/delete the KRM functions in the pipeline in the package's `kptfile`
    * e.g.: mutator functions to transform the KRM resources in the package contents; validator functions to enforce validation
  * add/change/delete sub-package

* Guard against pushing invalid package changes:
  * as part of the `porchctl rpkg push` operation, Porch renders the kpt package, running the pipeline
  * if the pipeline encounters a failure, error, or validation violation, Porch refuses to update the package contents

* Perform bulk operations using package variants, such as:
  * Assisted/automated update (upgrade, rollback) of groups of packages matching specific criteria (i.e. base package
    has new version or specific base package version has a vulnerability and should be rolled back)
  * Proposed change validation (pre-validating change that adds a validator function to a base package)


* Delete an existing package or package revision

#### Authoring & Latency

An important goal of the Package Orchestration service is to support building of task-specific UIs. In order for Porch to
sustain a quick turnaround of operations, package authors must ensure their packages allow the innermost authoring loop
(depicted below) to execute quickly in the following areas:
* low-latency execution of mutations and transformations on the package contents
* low-latency rendering of the package's [KRM function][krm functions] pipeline

![Inner Loop](/static/images/porch/Porch-Inner-Loop.svg)

#### Authoring & Access Control

Using Kubernetes Roles and RoleBindings, a user can apply role-based access control to limit the operations an actor (other
user, service account) can perform. For example, access can be segregated to restrict who can:

* register and unregister repositories
* create a new draft package revision and propose it for publication
* approve (or reject) the a proposed package revision
* clone packages from a specific upstream repository
* (with package variants, scripts, user-developed client etc.) perform bulk operations such as rolling out upgrade of
  downstream packages, including rollouts across multiple downstream repositories

### Porch Architecture

Porch consists of several microservices, designed to be hosted in a [Kubernetes](https://kubernetes.io/) cluster.

The overall architecture is shown below, including additional components external to Porch (k8s API server and deployment mechanism).
![Porch Architecture](/static/images/porch/Porch-Architecture.drawio.svg)

In addition to satisfying requirements highlighted above, the focus of the architecture was to:

* establish clear components and interfaces
* support low latency in package authoring operations

The primary Porch components are:

#### Porch Server

The Porch server is implemented as a [Kubernetes extension API server][apiserver]. It serves the primary Kubernetes
resources required for basic package authoring and lifeycle management, including:

* For each package revision (see [Package Versioning](./concepts_elaborated.md#package-versioning)):
  * `PackageRevision` - represents the *metadata* of the package revision stored in a repository.
  * `PackageRevisionResources` - represents the *file contents* of the package revision
  * Note that each package revision is represented by a *pair* of resources, each presenting a different view (or
    [representation][representation]) of the same underlying package revision.
* a `Repository` [custom resource][crds], which supports repository registration

#### Function Runner

The **Function Runner** is a separate microservice responsible for evaluating [kpt functions][functions]. It exposes
a [GRPC](https://grpc.io/) endpoint which enables evaluating a specified kpt function on a provided configuration package.

GRPC was chosen for the function runner service because the [benefits of an API server](#grpc-api) that prompted its use
for the Porch server do not apply. The function runner is an internal microservice, an implementation detail not exposed
to external callers. This makes GRPC perfectly suitable.

The function runner maintains a cache of functions to support low-latency function evaluation. It achieves this through
two mechanisms available to it for evaluation of a function:

* The **Executable Evaluation** mechanism, directly using function binaries baked into the function-runner image at compile-time
* The **Pod Evaluation** mechanism, spawning a separate function pod, based on the image of the invoked function, to run
  the function on the package contents

#### CaD Library

The [kpt](https://kpt.dev/) CLI already implements the fundamental package manipulation algorithms in order to provide its
command line user experience:

* [kpt pkg init](https://kpt.dev/reference/cli/pkg/init/) - create a bare-bones, valid, KRM package
* [kpt pkg get](https://kpt.dev/reference/cli/pkg/get/) - create a downstream package by cloning an upstream package;
  set up the upstream reference of the downstream package
* [kpt pkg update](https://kpt.dev/reference/cli/pkg/update/) - update the downstream package with changes from new
  version of upstream, 3-way merge
* [kpt fn eval](https://kpt.dev/reference/cli/fn/eval/) - evaluate a kpt function on a package
* [kpt fn render](https://kpt.dev/reference/cli/fn/render/) - render the package by executing the function pipeline of
  the package and its nested packages
* [kpt fn source](https://kpt.dev/reference/cli/fn/source/) and [kpt fn sink](https://kpt.dev/reference/cli/fn/sink/) -
  read package from local disk as a `ResourceList` and write package represented as `ResourcesList` into local disk

Porch contains a fork of the kpt code in order to reuse this set of primitive operations and combine them into higher-level
operations (for example, Porch renders packages automatically on changes; future versions will support bulk operations
such as upgrade of multiple packages, etc.).

A longer-term goal is to refactor kpt and Porch to extract the package manipulation operations into a reusable CaD Library,
consumed by both the kpt CLI and the Package Orchestration service to maintain functional parity between kpt and Porch.

## Open Issues/Questions

### Deployment Rollouts & Orchestration

**Not Yet Resolved**

Cross-cluster rollouts and orchestration of deployment activity. For example, a package deployed by Config Sync in cluster
A, and only on success, the same (or a different) package deployed by Config Sync in cluster B.

## Alternatives Considered

### GRPC API

We considered the use of [GRPC](https://grpc.io/) for the Porch API. The primary advantages of implementing Porch as an extension
Kubernetes apiserver are:

* customers won't have to open another port to their Kubernetes cluster and can reuse their existing infrastructure
* customers can likewise reuse existing, familiar, Kubernetes tooling ecosystem

<!-- Reference links -->
[apiserver]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/
[Config Sync]: https://cloud.google.com/anthos-config-management/docs/config-sync-overview
[crds]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
[functions]: https://kpt.dev/book/02-concepts/#functions
[git]: https://git-scm.org/
[krm]: https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md
[krm functions]: https://github.com/kubernetes-sigs/kustomize/blob/master/cmd/config/docs/api-conventions/functions-spec.md
[oci]: https://github.com/opencontainers/image-spec/blob/main/spec.md
[optimistic-concurrency]: https://en.wikipedia.org/wiki/Optimistic_concurrency_control
[pipeline]: https://kpt.dev/book/04-using-functions/#declarative-function-execution
[representation]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#differing-representations