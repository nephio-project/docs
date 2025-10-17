---
title: "Porch Concepts Elaborated"
type: docs
weight: 3
---

## Porch Concepts Elaborated

Let us elaborate in more detail on some of the concepts briefly introduced in [Package Orchestration Concepts](./porch_concepts.md)

### Repositories

{{% alert title="Note" color="primary" %}}

Currently, Porch primarily integrates with [Git][git] repositories as the priority target repository type, so for ease of
writing, this document will refer only to Git repositories. OCI support is available, but it is experimental and possibly unstable.
Support for additional repository types may be added in the future as required.

{{% /alert %}}

A *Porch repository* represents Porch's connection to a Git repository containing kpt packages. It allows a Porch user to
read existing packages from Git, and to author new or existing packages with Porch's create, clone, and modify operations.
Once a repository is *registered* (created in Porch), Porch performs an initial read operation against it, scanning its
file structure to discover kpt packages, and building a local cache to improve the performance of subsequent operations.

Any repositories added must be capable of storing the following minimum data and metadata:
* kpt packages' file contents.
* Package versions.
* Sufficient metadata associated with the package to capture:
  * Package dependency relationships (upstream - downstream).
  * Package lifecycle state (draft, proposed, published).
  * Package purpose (base package).
  * Customer-defined attributes (optionally).

At repository registration, customers must be able to specify details needed to store packages in an appropriate location
in the repository. For example, registration of a Git repository must accept a URL or directory path to locate the repository,
a branch and a directory to narrow down the location of packages, and any credentials needed to read from and/or write to
the repository

A successful repository registration results in the creation of a Repository custom resource, a *Repository object*. This
is not to be confused with, for example, the remote Git repository - the Porch repository only stores the details Porch
uses to interact with the Git repository.

{{% alert title="Note" color="primary" %}}

A user role with sufficient permissions can register a repository at practically any URL, including repositories containing
packages authored by third parties. Since the contents of the registered repositories become discoverable, a customer
registering a third-part repository must be aware of the implications and trust the contents thereof.

{{% /alert %}}

### Package Revisions

In a manner similar to Git commits, Porch allows the user to modify packages (including creating or cloning new ones) on
a basis of incremental releases. A new version of a package is not released immediately, but starts out as a draft, allowing
the user to develop it in safety before it is proposed and published as a standalone version of the package. Porch enables
this by modelling each successive version (whether published or still under development) as a *package revision*.

Package revisions are sequentially versioned using a simple integer sequence. This enables the following important capabilities:

* Compare any two versions of a package to establish "newer than", equal, or "older than" relationships.
* Automatically assign new version numbers on publication.
* [Optimistic concurrency][optimistic-concurrency] of package revision development, by comparing version numbers.
* Identify the latest (most recently published) package revision.
* A simple versioning model which easily supports automation.

Porch's get/list operations provide these versions to the user in a package revision's `revision` field.

#### Latest Package Revision

The "latest" package revision is the one most recently published, corresponding to the numerically-greatest revision number.
For additional ease of use, the PackageRevision resource type applies a Kubernetes label to the latest package revision
when read using the `porchctl` or `kubectl` CLI: `kpt.dev/latest-revision: "true"`

### Package Relationships - Upstream and Downstream

kpt packages support the concept of ***upstream*** and ***downstream*** relationships. When a package is cloned from another,
the new package (the downstream package) maintains an upstream link to the specific package revision from which it was cloned.
If a new revision of the upstream package is published, the upstream link can be used to update the downstream package.

### Deployment

The deployment mechanism is responsible for deploying packages from a repository and affecting the live state. The "default"
deployment mechanism tested with the CaD implementation is [Config Sync][Config Sync], but since the configuration is stored
in repositories of standard types, the exact software used for deployment is less of a concern.

Here we highlight some key attributes of the deployment mechanism and its integration within the CaD paradigm:

* _Published_ packages in a deployment repository are considered ready to be deployed.
* _Draft_ packages need to be identified in such a way that Config Sync can easily avoid deploying them.
* Config Sync supports deploying individual packages and whole repositories. For Git specifically, this translates to a
  requirement to be able to specify repository, branch/tag/ref, and directory when instructing Config Sync to deploy a
  package.
* Config Sync needs to be able to pin to specific versions of deployable packages in order to orchestrate rollouts and
  rollbacks. This means it must be possible to *get* a specific package revision.
* Config Sync needs to be able to discover when new package versions are available for deployment.

<!-- Reference links -->
[Config Sync]: https://cloud.google.com/anthos-config-management/docs/config-sync-overview
[git]: https://git-scm.org/
[optimistic-concurrency]: https://en.wikipedia.org/wiki/Optimistic_concurrency_control