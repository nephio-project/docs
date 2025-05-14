---
title: "Using the Porch CLI tool"
type: docs
weight: 2
description: 
---

## Setting up the porchctl CLI

When Porch was ported to Nephio, the `kpt alpha rpkg` commands in kpt were moved into a new command called `porchctl`. 

To use it locally, [download](https://github.com/nephio-project/porch/releases/download/dev/porchctl.tgz), unpack and add it to your PATH.

{{% alert title="Note" color="primary" %}}

Installation of Porch, including its prerequisites, is covered in a [dedicated document](install-and-using-porch.md).

{{% /alert %}}

*Optional*: Generate the autocompletion script for the specified shell to add to your sh profile.

```
porchctl completion bash
```

The `porchtcl` command is an administration command for acting on Porch *Repository* (repo) and *PackageRevision* (rpkg)
CRs.

The commands for administering repositories are:

| Command               | Description                    |
| --------------------- | ------------------------------ |
| `porchctl repo get`   | List registered repositories.  |
| `porchctl repo reg`   | Register a package repository. |
| `porchctl repo unreg` | Unregister a repository.       |

The commands for administering package revisions are:

| Command                        | Description                                                                             |
| ------------------------------ | --------------------------------------------------------------------------------------- |
| `porchctl rpkg approve`        | Approve a proposal to publish a package revision.                                       |
| `porchctl rpkg clone`          | Create a clone of an existing package revision.                                         |
| `porchctl rpkg copy`           | Create a new package revision from an existing one.                                     |
| `porchctl rpkg del`            | Delete a package revision.                                                              |
| `porchctl rpkg get`            | List package revisions in registered repositories.                                      |
| `porchctl rpkg init`           | Initializes a new package in a repository.                                              |
| `porchctl rpkg propose`        | Propose that a package revision should be published.                                    |
| `porchctl rpkg propose-delete` | Propose deletion of a published package revision.                                       |
| `porchctl rpkg pull`           | Pull the content of the package revision.                                               |
| `porchctl rpkg push`           | Push resources to a package revision.                                                   |
| `porchctl rpkg reject`         | Reject a proposal to publish or delete a package revision.                              |
| `porchctl rpkg update`         | Update a downstream package revision to a more recent revision of its upstream package. |

## Using the porchctl CLI

### Guide prerequisites
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)

Make sure that your `kubectl` context is set up for `kubectl` to interact with the correct Kubernetes instance (see
[installation instructions](install-and-using-porch.md) or the [running-locally](../running-porch/running-locally.md)
guide for details).

To check whether `kubectl` is configured with your Porch cluster (or local instance), run:

```bash
kubectl api-resources | grep porch
```

You should see the following API resources listed:

```bash
repositories                  config.porch.kpt.dev/v1alpha1          true         Repository
packagerevisionresources      porch.kpt.dev/v1alpha1                 true         PackageRevisionResources
packagerevisions              porch.kpt.dev/v1alpha1                 true         PackageRevision
```

## Porch Resources

Porch server manages the following resources:

1. `repositories`: a repository (Git or OCI) can be registered with Porch to support discovery or management of KRM
   configuration packages in those repositories.
2. `packagerevisions`: a specific revision of a KRM configuration package managed by Porch in one of the registered
   repositories. This resource represents a _metadata view_ of the KRM configuration package.
3. `packagerevisionresources`: this resource represents the contents of the configuration package (KRM resources
   contained in the package)

{{% alert title="Note" color="primary" %}}

`packagerevisions` and `packagerevisionresources` represent different _views_ of the same underlying KRM
configuration package. `packagerevisions` represents the package metadata, and `packagerevisionresources` represents the
package content. The matching resources share the same `name` (as well as API group and version:
`porch.kpt.dev/v1alpha1`) and differ in resource kind (`PackageRevision` and `PackageRevisionResources` respectively).

{{% /alert %}}


## Repository Registration

To use Porch with a Git repository, you will need:

* A Git repository for your blueprints.
* A [Personal Access Token](https://github.com/settings/tokens) (when using GitHub repository) for Porch to authenticate
  with the repository. Porch requires the 'repo' scope.
* Or Basic Auth credentials for Porch to authenticate with the repository.

To use Porch with an OCI repository ([Artifact Registry](https://console.cloud.google.com/artifacts) or
[Google Container Registry](https://cloud.google.com/container-registry)), first make sure to:

* Enable [workload identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) for Porch
* Assign appropriate roles to the Porch workload identity service account
  (`iam.gke.io/gcp-service-account=porch-server@$(GCP_PROJECT_ID).iam.gserviceaccount.com`)
  to have appropriate level of access to your OCI repository.

Use the `porchctl repo register` command to register your repository with Porch:

```bash

GITHUB_USERNAME=<your github username>
GITHUB_TOKEN=<GitHub Personal Access Token>

$ porchctl repo register \
  --namespace default \
  --repo-basic-username=${GITHUB_USERNAME} \
  --repo-basic-password=${GITHUB_TOKEN} \
  https://github.com/${GITHUB_USERNAME}/blueprints.git
```

All command line flags supported:

* `--directory` - Directory within the repository where to look for packages.
* `--branch` - Branch in the repository where finalized packages are committed (defaults to `main`).
* `--name` - Name of the package repository Kubernetes resource. If unspecified, will default to the name portion (last
  segment) of the repository URL (`blueprint` in the example above)
* `--description` - Brief description of the package repository.
* `--deployment` - Boolean value; If specified, repository is a deployment repository; published packages in a
  deployment repository are considered deployment-ready.
* `--repo-basic-username` - Username for repository authentication using basic auth.
* `--repo-basic-password` - Password for repository authentication using basic auth.

Additionally, common `kubectl` command line flags for controlling aspects of
interaction with the Kubernetes apiserver, logging, and more (this is true for
all `porchctl` CLI commands which interact with Porch).

Use the `porchctl repo get` command to query registered repositories:

```bash
$ porchctl repo get

NAME         TYPE  CONTENT  DEPLOYMENT  READY  ADDRESS
blueprints   git   Package              True   https://github.com/platkrm/blueprints.git
deployments  git   Package  true        True   https://github.com/platkrm/deployments.git
```

The `porchctl <group> get` commands support common `kubectl`
[flags](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#formatting-output) to format output, for example
`porchctl repo get --output=yaml`.

The command `porchctl repo unregister` can be used to unregister a repository:

```bash
$ porchctl repo unregister deployments --namespace default
```

## Package Discovery And Introspection

The `porchctl rpkg` command group contains commands for interacting with packages managed by the Package Orchestration
service. the `r` prefix used in the command group name stands for 'remote'.

The `porchctl rpkg get` command list the packages in registered repositories:

```bash
$ porchctl rpkg get

NAME                                                 PACKAGE  WORKSPACENAME  REVISION  LATEST  LIFECYCLE  REPOSITORY
blueprints-0349d71330b89ee48ac85167598ef23021fd0484  basens   main           main      false   Published  blueprints
blueprints-2e47615fda05664491f72c58b8ab658683afa036  basens   v1             v1        true    Published  blueprints
blueprints-7e2fe44bfdbb744d49bdaaaeac596200102c5f7c  istions  main           main      false   Published  blueprints
blueprints-ac6e872be4a4a3476922deca58cca3183b16a5f7  istions  v1             v1        false   Published  blueprints
blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3  istions  v2             v2        true    Published  blueprints
...
```

The `LATEST` column indicates whether the package revision is the latest among the revisions of the same package. In the
output above, `v2` is the latest revision of `istions` package and `v1` is the latest revision of `basens` package.

The `LIFECYCLE` column indicates the lifecycle stage of the package revision, one of: `Draft`, `Proposed` or `Published`.

The `REVISION` column indicates the revision of the package. Revisions are assigned when a package is `Published` and
starts at `v1`.

The `WORKSPACENAME` column indicates the workspace name of the package. The workspace name is assigned when a draft
revision is created and is used as the branch name for proposed and draft package revisions. The workspace name 
must be unique among package revisions in the same package.

{{% alert title="Note" color="primary" %}}

Packages exist in a hierarchical directory structure maintained by the underlying repository such as git, or in a
filesystem bundle of OCI images. The hierarchical, filesystem-compatible names of packages do not satisfy the
Kubernetes naming [constraints](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names).
Therefore, the names of the Kubernetes resources representing package revisions are computed as a hash.

{{% /alert %}}


Simple filtering of package revisions by name (substring) and revision (exact match) is supported by the CLI using
`--name` and `--revision` flags:

```bash
$ porchctl rpkg get --name istio --revision=v2

NAME                                                 PACKAGE  WORKSPACENAME  REVISION  LATEST  LIFECYCLE  REPOSITORY
blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3  istions  v2             v2        true    Published  blueprints
```

The common `kubectl` flags that control output format are available as well:

```bash
$ porchctl rpkg get blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3 -ndefault -oyaml

apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevision
metadata:
  labels:
    kpt.dev/latest-revision: "true"
  name: blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3
  namespace: default
spec:
  lifecycle: Published
  packageName: istions
  repository: blueprints
  revision: v2
  workspaceName: v2
...
```

The `porchctl rpkg pull` command can be used to read the package resources.

The command can be used to print the package revision resources as `ResourceList` to `stdout`, which enables
[chaining](https://kpt.dev/book/04-using-functions/02-imperative-function-execution?id=chaining-functions-using-the-unix-pipe)
evaluation of functions on the package revision pulled from the Package Orchestration server.

```bash
$ porchctl rpkg pull blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3 -ndefault

apiVersion: config.kubernetes.io/v1
kind: ResourceList
items:
- apiVersion: kpt.dev/v1
  kind: Kptfile
  metadata:
    name: istions
...
```

Or, the package contents can be saved on local disk for direct introspection or editing:

```bash
$ porchctl rpkg pull blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3 ./istions -ndefault

$ find istions

istions
istions/istions.yaml
istions/README.md
istions/Kptfile
istions/package-context.yaml
...
```

## Authoring Packages

Several commands in the `porchctl rpkg` group support package authoring:

* `init` - Initializes a new package revision in the target repository.
* `clone` - Creates a clone of a source package in the target repository.
* `copy` - Creates a new package revision from an existing one.
* `push` - Pushes package resources into a remote package.
* `del` - Deletes one or more packages in registered repositories.

The `porchctl rpkg init` command can be used to initialize a new package revision. Porch server will create and
initialize a new package (as a draft) and save it in the specified repository.

```bash
$ porchctl rpkg init new-package --repository=deployments --workspace=v1 -ndefault

deployments-c32b851b591b860efda29ba0e006725c8c1f7764 created

$ porchctl rpkg get

NAME                                                  PACKAGE      WORKSPACENAME  REVISION  LATEST  LIFECYCLE  REPOSITORY
deployments-c32b851b591b860efda29ba0e006725c8c1f7764  new-package  v1                       false   Draft      deployments
...
```

The new package is created in the `Draft` lifecycle stage. This is true also for all commands that create new package
revision (`init`, `clone` and `copy`).

Additional flags supported by the `porchctl rpkg init` command are:

* `--repository` - Repository in which the package will be created.
* `--workspace` - Workspace of the new package.
* `--description` -  Short description of the package.
* `--keywords` - List of keywords for the package.
* `--site` - Link to page with information about the package.


Use `porchctl rpkg clone` command to create a _downstream_ package by cloning an _upstream_ package:

```bash
$ porchctl rpkg clone blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3 istions-clone \
  --repository=deployments -ndefault
deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82 created

# Confirm the package revision was created
porchctl rpkg get deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82 -ndefault
NAME                                                   PACKAGE         WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82   istions-clone   v1                         false    Draft       deployments
```

`porchctl rpkg clone` can also be used to clone packages that are in repositories not registered with Porch, for
example:

```bash
$ porchctl rpkg clone \
  https://github.com/GoogleCloudPlatform/blueprints.git cloned-bucket \
  --directory=catalog/bucket \
  --ref=main \
  --repository=deployments \
  --namespace=default
deployments-e06c2f6ec1afdd8c7d977fcf204e4d543778ddac created

# Confirm the package revision was created
porchctl rpkg get deployments-e06c2f6ec1afdd8c7d977fcf204e4d543778ddac -ndefault
NAME                                                   PACKAGE         WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
deployments-e06c2f6ec1afdd8c7d977fcf204e4d543778ddac   cloned-bucket   v1                         false    Draft       deployments
```

The flags supported by the `porchctl rpkg clone` command are:

* `--directory` - Directory within the upstream repository where the upstream
  package is located.
* `--ref` - Ref in the upstream repository where the upstream package is
  located. This can be a branch, tag, or SHA.
* `--repository` - Repository to which package will be cloned (downstream
  repository).
* `--workspace` - Workspace to assign to the downstream package.
* `--strategy` - Update strategy that should be used when updating this package;
  one of: `resource-merge`, `fast-forward`, `force-delete-replace`.


The `porchctl rpkg copy` command can be used to create a new revision of an existing package. It is a means to
modifying an already published package revision.

```bash
$ porchctl rpkg copy \
  blueprints-421a5b5e43b03bc697d96f471929efc6ba3f54b3 \
  --workspace=v3 -ndefault

# Confirm the package revision was created
$ porchctl rpkg get blueprints-bf11228f80de09f1a5dd9374dc92ebde3b503689 -ndefault
NAME                                                  PACKAGE   WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
blueprints-bf11228f80de09f1a5dd9374dc92ebde3b503689   istions   v3                         false    Draft       blueprints
```

The `porchctl rpkg push` command can be used to update the resources (package contents) of a package _draft_:

```bash
$ porchctl rpkg pull \
  deployments-c32b851b591b860efda29ba0e006725c8c1f7764 ./new-package -ndefault

# Make edits using your favorite YAML editor, for example adding a new resource
$ cat <<EOF > ./new-package/config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config-map
data:
  color: orange
EOF

# Push the updated contents to the Package Orchestration server, updating the
# package contents.
$ porchctl rpkg push \
  deployments-c32b851b591b860efda29ba0e006725c8c1f7764 ./new-package -ndefault

# Confirm that the remote package now includes the new ConfigMap resource
$ porchctl rpkg pull deployments-c32b851b591b860efda29ba0e006725c8c1f7764 -ndefault

apiVersion: config.kubernetes.io/v1
kind: ResourceList
items:
...
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-config-map
  data:
    color: orange
...
```

Package revision can be deleted using `porchctl rpkg del` command:

```bash
# Delete package revision
$ porchctl rpkg del blueprints-bf11228f80de09f1a5dd9374dc92ebde3b503689 -ndefault

blueprints-bf11228f80de09f1a5dd9374dc92ebde3b503689 deleted
```

## Package Lifecycle and Approval Flow

Authoring is performed on the package revisions in the _Draft_ lifecycle stage. Before a package can be deployed or
cloned, it must be _Published_. The approval flow is the process by which the package is advanced from _Draft_ state
through _Proposed_ state and finally to _Published_ lifecycle stage.

The commands used to manage package lifecycle stages include:

* `propose` - Proposes to finalize a package revision draft
* `approve` - Approves a proposal to finalize a package revision.
* `reject`  - Rejects a proposal to finalize a package revision

In the [Authoring Packages](#authoring-packages) section above we created several _draft_ packages and in this section
we will create proposals for publishing some of them.

```bash
# List package revisions to identify relevant drafts:
$ porchctl rpkg get
NAME                                                   PACKAGE         WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
deployments-e06c2f6ec1afdd8c7d977fcf204e4d543778ddac   cloned-bucket   v1                         false    Draft       deployments
deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82   istions-clone   v1                         false    Draft       deployments
deployments-c32b851b591b860efda29ba0e006725c8c1f7764   new-package     v1                         false    Draft       deployments

# Propose two package revisions to be be published
$ porchctl rpkg propose \
  deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82 \
  deployments-c32b851b591b860efda29ba0e006725c8c1f7764 \
  -ndefault

deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82 proposed
deployments-c32b851b591b860efda29ba0e006725c8c1f7764 proposed

# Confirm the package revisions are now Proposed
$ porchctl rpkg get
NAME                                                   PACKAGE         WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
deployments-e06c2f6ec1afdd8c7d977fcf204e4d543778ddac   cloned-bucket   v1                         false    Draft       deployments
deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82   istions-clone   v1                         false    Proposed    deployments
deployments-c32b851b591b860efda29ba0e006725c8c1f7764   new-package     v1                         false    Proposed    deployments
```

At this point, a person in _platform administrator_ role, or even an automated process, will review and either approve
or reject the proposals. To aid with the decision, the platform administrator may inspect the package contents using the
commands above, such as `porchctl rpkg pull`.

```bash
# Approve a proposal to publish a package revision
$ porchctl rpkg approve deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82 -ndefault
deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82 approved

# Reject a proposal to publish a package revision
$ porchctl rpkg reject deployments-c32b851b591b860efda29ba0e006725c8c1f7764 -ndefault
deployments-c32b851b591b860efda29ba0e006725c8c1f7764 rejected
```

Now the user can confirm lifecycle stages of the package revisions:

```bash
# Confirm package revision lifecycle stages after approvals:
$ porchctl rpkg get
NAME                                                   PACKAGE         WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
deployments-e06c2f6ec1afdd8c7d977fcf204e4d543778ddac   cloned-bucket   v1                         false    Draft       deployments
deployments-11ca1db650fa4bfa33deeb7f488fbdc50cdb3b82   istions-clone   v1              v1         true     Published   deployments
deployments-c32b851b591b860efda29ba0e006725c8c1f7764   new-package     v1                         false    Draft       deployments
```

Observe that the rejected proposal returned the package revision back to _Draft_ lifecycle stage. The package whose
proposal was approved is now in _Published_ state.