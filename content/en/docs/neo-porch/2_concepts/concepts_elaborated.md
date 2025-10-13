---
title: "Package Orchestration Concepts Elaborated"
type: docs
weight: 3
---

## CaD Concepts Elaborated

Let us elaborate in more detail on some of the concepts briefly introduced in [Package Orchestration Concepts](./package_orchestration_concepts.md)

### Repositories

Porch currently integrates with [Git][git] repositories as their priority target repository type. OCI support is available,
but experimental and possibly unstable. Support for additional repository types may be added in the future as required.

Any repositories added must include the ability to store the following minimum data and metadata:
* kpt packages' file contents
* their versions
* sufficient metadata associated with the package to capture:
  * package dependency relationships (upstream - downstream)
  * package lifecycle state (draft, proposed, published)
  * package purpose (base package)
  * (optionally) customer-defined attributes

At repository registration, customers must be able to specify details needed to store packages in an appropriate location
in the repository. For example, registration of a Git repository must accept a branch and a directory.

A successful repository registration results in the creation of a Repository custom resource, a *Repository object* or *Porch
repository*. This is not to be confused with, for example, the remote Git repository - the Porch repository only stores
the details Porch uses to interact with the Git repository.

{{% alert title="Note" color="primary" %}}

A user role with sufficient permissions can register a repository at practically any URL, including repositories containing
packages authored by third parties. Since the contents of the registered repositories become discoverable, a customer
registering a third-part repository must be aware of the implications and trust the contents thereof.

{{% /alert %}}

### Package Versioning

Package revisions are sequentially versioned using a simple integer sequence. This fulfils the following important requirements:

* ability to compare any two versions of a package to establish "newer than", equal, or "older than" relationships
* ability to support automatic assignment of versions
* ability to support [optimistic concurrency][optimistic-concurrency] of package revisions by comparing version numbers
* ability to identify the latest (most recently published) package revision
* a simple versioning model which easily supports automation

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

* _Published_ packages in a deployment repository are considered ready to be deployed
* _Draft_ packages need to be identified in such a way that Config Sync can easily avoid deploying them.
* Config Sync supports deploying individual packages and whole repositories. For Git specifically, this translates to a
  requirement to be able to specify repository, branch/tag/ref, and directory when instructing Config Sync to deploy a
  package.
* Config Sync needs to be able to pin to specific versions of deployable packages in order to orchestrate rollouts and
  rollbacks. This means it must be possible to *get* a specific package revision.
* Config Sync needs to be able to discover when new package versions are available for deployment.