---
title: "Using the Porch CLI tool"
type: docs
weight: 2
description: 
---

## Setting up the porchctl CLI

The Porch CLI uses the `porchctl` command.
To use it locally, [download](https://github.com/nephio-project/porch/releases/tag/dev), unpack and add it to your PATH.

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
* If the repository requires authentication you will require either
  - A [Personal Access Token](https://github.com/settings/tokens) (when using GitHub repository) for Porch to authenticate
    with the repository if the repository. Porch requires the 'repo' scope.
  - Basic Auth credentials for Porch to authenticate with the repository.

To use Porch with an OCI repository ([Artifact Registry](https://console.cloud.google.com/artifacts) or
[Google Container Registry](https://cloud.google.com/container-registry)), first make sure to:

* Enable [workload identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) for Porch
* Assign appropriate roles to the Porch workload identity service account
  (`iam.gke.io/gcp-service-account=porch-server@$(GCP_PROJECT_ID).iam.gserviceaccount.com`)
  to have appropriate level of access to your OCI repository.

Use the `porchctl repo register` command to register your repository with Porch.

```bash
# Unauthenticated Repositories
porchctl repo register --namespace default https://github.com/platkrm/test-blueprints.git
porchctl repo register --namespace default https://github.com/nephio-project/catalog --name=oai --directory=workloads/oai
porchctl repo register --namespace default https://github.com/nephio-project/catalog --name=infra --directory=infra
```

```bash
# Authenticated Repositories
GITHUB_USERNAME=<your github username>
GITHUB_TOKEN=<GitHub Personal Access Token>

$ porchctl repo register \
  --namespace default \
  --repo-basic-username=${GITHUB_USERNAME} \
  --repo-basic-password=${GITHUB_TOKEN} \
  https://github.com/${GITHUB_USERNAME}/blueprints.git
```

For more details on configuring authenticated repositories see [Authenticating to Remote Git Repositories](git-authentication-config.md).

The command line flags supported by `porchctl repo register` are:

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
$ porchctl repo get -A
NAMESPACE    NAME              TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
default      oai               git    Package                True    https://github.com/nephio-project/catalog
default      test-blueprints   git    Package                True    https://github.com/platkrm/test-blueprints.git
porch-demo   porch-test        git    Package   true                 http://localhost:3000/nephio/porch-test.git
```

The `porchctl <group> get` commands support common `kubectl`
[flags](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#formatting-output) to format output, for example
`porchctl repo get --output=yaml`.

The command `porchctl repo unregister` can be used to unregister a repository:

```bash
$ porchctl repo unregister test-blueprints --namespace default
```

## Package Discovery And Introspection

The `porchctl rpkg` command group contains commands for interacting with packages managed by the Package Orchestration
service. the `r` prefix used in the command group name stands for 'remote'.

The `porchctl rpkg get` command list the packages in registered repositories:

```bash
$ porchctl rpkg get -A
NAMESPACE    NAME                                                           PACKAGE                                     WORKSPACENAME   REVISION   LATEST   LIFECYCLE          REPOSITORY
default      infra.infra.baremetal.bmh-template.main                        infra/baremetal/bmh-template                main            -1         false    Published          infra
default      infra.infra.capi.cluster-capi.main                             infra/capi/cluster-capi                     main            -1         false    Published          infra
default      infra.infra.capi.cluster-capi.v2.0.0                           infra/capi/cluster-capi                     v2.0.0          -1         false    Published          infra
default      infra.infra.capi.cluster-capi.v3.0.0                           infra/capi/cluster-capi                     v3.0.0          -1         false    Published          infra
default      infra.infra.capi.vlanindex.main                                infra/capi/vlanindex                        main            -1         false    Published          infra
default      infra.infra.capi.vlanindex.v2.0.0                              infra/capi/vlanindex                        v2.0.0          -1         false    Published          infra
default      infra.infra.capi.vlanindex.v3.0.0                              infra/capi/vlanindex                        v3.0.0          -1         false    Published          infra
default      infra.infra.gcp.nephio-blueprint-repo.main                     infra/gcp/nephio-blueprint-repo             main            -1         false    Published          infra
default      infra.infra.gcp.nephio-blueprint-repo.v1                       infra/gcp/nephio-blueprint-repo             v1              1          true     Published          infra
default      infra.infra.gcp.nephio-blueprint-repo.v2.0.0                   infra/gcp/nephio-blueprint-repo             v2.0.0          -1         false    Published          infra
default      infra.infra.gcp.nephio-blueprint-repo.v3.0.0                   infra/gcp/nephio-blueprint-repo             v3.0.0          -1         false    Published          infra
default      oai.workloads.oai.oai-ran-operator.main                        workloads/oai/oai-ran-operator              main            -1         false    Published          oai
default      oai.workloads.oai.oai-ran-operator.v1                          workloads/oai/oai-ran-operator              v1              1          true     Published          oai
default      oai.workloads.oai.oai-ran-operator.v2.0.0                      workloads/oai/oai-ran-operator              v2.0.0          -1         false    Published          oai
default      oai.workloads.oai.oai-ran-operator.v3.0.0                      workloads/oai/oai-ran-operator              v3.0.0          -1         false    Published          oai
default      oai.workloads.oai.pkg-example-cucp-bp.main                     workloads/oai/pkg-example-cucp-bp           main            -1         false    Published          oai
default      oai.workloads.oai.pkg-example-cucp-bp.v1                       workloads/oai/pkg-example-cucp-bp           v1              1          true     Published          oai
default      oai.workloads.oai.pkg-example-cucp-bp.v2.0.0                   workloads/oai/pkg-example-cucp-bp           v2.0.0          -1         false    Published          oai
default      oai.workloads.oai.pkg-example-cucp-bp.v3.0.0                   workloads/oai/pkg-example-cucp-bp           v3.0.0          -1         false    Published          oai
default      oai.workloads.oai.pkg-example-cuup-bp.main                     workloads/oai/pkg-example-cuup-bp           main            -1         false    Published          oai
default      test-blueprints.basens.main                                    basens                                      main            -1         false    Published          test-blueprints
default      test-blueprints.basens.v1                                      basens                                      v1              1          false    Published          test-blueprints
default      test-blueprints.basens.v2                                      basens                                      v2              2          false    Published          test-blueprints
default      test-blueprints.basens.v3                                      basens                                      v3              3          true     Published          test-blueprints
default      test-blueprints.empty.main                                     empty                                       main            -1         false    Published          test-blueprints
default      test-blueprints.empty.v1                                       empty                                       v1              1          true     Published          test-blueprints
porch-demo   porch-test.basedir.subdir.subsubdir.edge-function.inadir       basedir/subdir/subsubdir/edge-function      inadir          0          false    Draft              porch-test
porch-demo   porch-test.basedir.subdir.subsubdir.network-function.dirdemo   basedir/subdir/subsubdir/network-function   dirdemo         0          false    Draft              porch-test
porch-demo   porch-test.network-function.innerhome                          network-function                            innerhome       2          true     Published          porch-test
porch-demo   porch-test.network-function.innerhome3                         network-function                            innerhome3      0          false    Proposed           porch-test
porch-demo   porch-test.network-function.innerhome4                         network-function                            innerhome4      0          false    Draft              porch-test
porch-demo   porch-test.network-function.main                               network-function                            main            -1         false    Published          porch-test
porch-demo   porch-test.network-function.outerspace                         network-function                            outerspace      1          false    DeletionProposed   porch-test
```

The `NAME` column gives the kubernetes name of the package revision resource. Names are of the form:

**repository.([pathnode.]*)package.workspace**

1. The first part up to the first dot is the **repository** that the package revision is in.
1. The second (optional) part is zero or more **pathnode** nodes, identifying the path of the package.
1. The second last part between the second last and last dots is the **package** that the package revision is in.
1. The last part after the last dot is the **workspace** of the package revision, which uniquely identifies the package revision in the package.

From the listing above, the package revision with the name `test-blueprints.basens.v3`, is in a repository called `test-blueprints`. It is in the root of that
repository because there are no **pathnode** entries in its name. It is in a package called `basens` and its workspace name is `v3`.

The package revision with the name `porch-test.basedir.subdir.subsubdir.edge-function.inadir` is in the repo `porch-test`. It has a path of
`basedir/subdir/subsubdir`. The package name is `edge-function` and its workspace name is `inadir`.

The `PACKAGE` column contains the package name of a pacakge revision. Of course, all package revisions in a package have the same package name. The
package name includes the path to the directory containing the package if the package is not in the root directory of the repo. For example, in the listing above
the packages `basedir/subdir/subsubdir/edge-function` and `basedir/subdir/subsubdir/network-function` are in the directory `basedir/subdir/subsubdir`. The
`basedir/subdir/subsubdir/network-function` and `network-function` packages are different packages because they are in different directories.

The `REVISION` column indicates the revision of the package.
- Revisions of `1` or greater indicate released packages. When a package is `Published`. When a package is published, it is assigned the next
  available revision number, starting at `1`. In the listing above, the `porch-test.network-function.innerhome` revision of package `network-function`
  has a revision of `2` and is the latest revision of the package. The `porch-test.network-function.outerspace` revision of the package has a
  revision of `1`. If the `porch-test.network-function.innerhome3` revision is published, it will be assigned a revision of `3` and will become
  the latest package revision.
- Packages that are not published (packages with a lifecycle status of `Draft` or `Proposed`) have a revision number of `0`. There can be many revisions
  of a package with revision `0` as is shown with revisions `porch-test.network-function.innerhome3` and `porch-test.network-function.innerhome4`
  of package `network-function` above.
- Placeholder packages that point at the head of a git branch or tag have a revision number of `-1`

The `LATEST` column indicates whether the package revision is the latest among the revisions of the same package. In the
output above, `3` is the latest revision of `basens` package and `1` is the latest revision of `empty` package.

The `LIFECYCLE` column indicates the lifecycle stage of the package revision, one of: `Draft`, `Proposed`, `Published` or `DeletionProposed`.

The `WORKSPACENAME` column indicates the workspace name of the package. The workspace name is selected by a user when a draft
revision is created. The workspace name must be unique among package revisions in the same package. 

{{% alert title="Scope of WORKSPACENAME" color="primary" %}}
The scope of a workspace name is restricted to its package and it is merely a string that identifies a package revision within a package. A user is free to
pick any workspace name that complies with [kubernetes rules for naming objects and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/).
The workspace name `V1` on the `empty` package has no relation to the workspace name `V1` on the `basens` package listed above.
A user has simply decided to use the same workspace name on two separate packages.
{{% /alert %}}

{{% alert title="Setting WORKSPACENAME and REVISION from repositories" color="primary" %}}
When Porch connects to a repository, it scans the branches and tags of the Git repository for package revisions. It descends the directory tree of the repo
looking for files called `Kptfile`. When it finds a Kptfile in a directory, Porch knows that it has found a kpt package and it does not search any child directories
of this directory. Porch then examines all branches and tags that have references to that package and finds package revisions using the following rules:
1. Look for a commit message of the form `kpt:{"package":"network-function","workspaceName":"outerspace","revision":"1"}` at the tip of the branch/tag and
   set the workspace name and revision from the commit message, `outerspace` and `1` respectively in the case of the `porch-test.network-function.outerspace`
   package revision in the listing above.
2. If 1. fails, and if the reference is of the form `<package>.v1`, set the workspace name to `v1` and the revision to `1` as is the case for the
   `test-blueprints.basens.v1` package revision in the listing above.
3. If 2. fails, set the workspace name to the branch or tag name, and the revision to `-1`, as is the case for the `infra.infra.gcp.nephio-blueprint-repo.v3.0.0`
   package revision in the listing above. The workspace name is set to the branch name `v3.0.0`, and the revision is set to `-1`.
{{% /alert %}}

## Package Revision Filtering

Simple filtering of package revisions by name (substring) and revision (exact match) is supported by the CLI using
`--name`, `--revision` and `--workspace` flags:

```bash
$ porchctl -n porch-demo rpkg get --name network-function
NAME                                    PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.network-function.dirdemo     network-function   dirdemo         1          false    Published   porch-test
porch-test.network-function.innerhome   network-function   innerhome       2          true     Published   porch-test
porch-test.network-function.main        network-function   main            -1         false    Published   porch-test

$ porchctl -n porch-demo rpkg get --revision 1
NAME                                                         PACKAGE                                   WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.basedir.subdir.subsubdir.edge-function2.diredge   basedir/subdir/subsubdir/edge-function2   diredge         1          true     Published   porch-test
porch-test.edge-function2.diredgeab                          edge-function2                            diredgeab       1          true     Published   porch-test
porch-test.edge-function.diredge                             edge-function                             diredge         1          true     Published   porch-test
porch-test.network-function3.outerspace                      network-function3                         outerspace      1          true     Published   porch-test
porch-test.network-function.dirdemo                          network-function                          dirdemo         1          false    Published   porch-test

$ porchctl -n porch-demo rpkg get --workspace outerspace
NAME                                      PACKAGE             WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.network-function2.outerspace   network-function2   outerspace      0          false    Draft       porch-test
porch-test.network-function3.outerspace   network-function3   outerspace      1          true     Published   porch-test
```

The common `kubectl` flags that control output format are available as well:

```bash

$ porchctl rpkg get -n porch-demo porch-test.network-function.innerhome -o yaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevision
metadata:
  labels:
    kpt.dev/latest-revision: "true"
  name: porch-test.network-function.innerhome
  namespace: porch-demo
spec:
  lifecycle: Published
  packageName: network-function
  repository: porch-test
  revision: 2
  workspaceName: innerhome
...
```

The `porchctl rpkg pull` command can be used to read the package resources.

The command can be used to print the package revision resources as `ResourceList` to `stdout`, which enables
[chaining](https://kpt.dev/book/04-using-functions/02-imperative-function-execution?id=chaining-functions-using-the-unix-pipe)
evaluation of functions on the package revision pulled from the Package Orchestration server.

```bash
$ porchctl rpkg pull -n porch-demo porch-test.network-function.innerhome    
apiVersion: config.kubernetes.io/v1
kind: ResourceList
items:
- apiVersion: ""
  kind: KptRevisionMetadata
  metadata:
    name: porch-test.network-function.innerhome
    namespace: porch-demo
...
```

Or, the package contents can be saved on local disk for direct introspection or editing:

```bash
$ porchctl rpkg pull -n porch-demo porch-test.network-function.innerhome ./innerhome

$ find innerhome

./innerhome
./innerhome/.KptRevisionMetadata
./innerhome/README.md
./innerhome/Kptfile
./innerhome/package-context.yaml
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
$ porchctl rpkg init new-package --repository=porch-test --workspace=my-workspace -nporch-demo
porch-test.new-package.my-workspace created

$ porchctl rpkg get -n porch-demo porch-test.new-package.my-workspace
NAME                                  PACKAGE       WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.new-package.my-workspace   new-package   my-workspace    0          false    Draft       porch-test
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
$ porchctl rpkg clone porch-test.new-package.my-workspace new-package-clone --repository=porch-deployment -n porch-demo
porch-deployment.new-package-clone.v1 created

# Confirm the package revision was created
porchctl rpkg get porch-deployment.new-package-clone.v1 -nporch-demo
NAME                                    PACKAGE             WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-deployment.new-package-clone.v1   new-package-clone   v1              0          false    Draft       porch-deployment
```

`porchctl rpkg clone` can also be used to clone packages that are in repositories not registered with Porch, for
example:

```bash
$ porchctl rpkg clone \
  https://github.com/nephio-project/catalog.git cloned-pkg-example-ue-bp \
  --directory=workloads/oai/pkg-example-ue-bp \
  --ref=main \
  --repository=porch-deployment \
  --namespace=porch-demo
porch-deployment.cloned-pkg-example-ue-bp.v1 created

# Confirm the package revision was created
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft       porch-deployment
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
  one of: `resource-merge`, `fast-forward`, `force-delete-replace`, `copy-merge`.


The `porchctl rpkg copy` command can be used to create a new revision of an existing package. It is a means to
modifying an already published package revision.

```bash
$ porchctl rpkg copy porch-test.network-function.innerhome --workspace=great-outdoors -nporch-demo
porch-test.network-function.great-outdoors created

# Confirm the package revision was created
$ porchctl rpkg get porch-test.network-function.great-outdoors -nporch-demo
NAME                                         PACKAGE            WORKSPACENAME    REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.network-function.great-outdoors   network-function   great-outdoors   0          false    Draft       porch-test
```

The `porchctl rpkg pull` and `porchctl rpkg push` commands can be used to update the resources (package contents) of a package _draft_:

```bash
$ porchctl rpkg pull porch-test.network-function.great-outdoors ./great-outdoors -nporch-demo

# Make edits using your favorite YAML editor, for example adding a new resource
$ cat <<EOF > ./great-outdoors/config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config-map
data:
  color: green
EOF

# Push the updated contents to the Package Orchestration server, updating the
# package contents.
$ porchctl rpkg push porch-test.network-function.great-outdoors ./great-outdoors -nporch-demo

# Confirm that the remote package now includes the new ConfigMap resource
$ porchctl rpkg pull porch-test.network-function.great-outdoors -n porch-demo
apiVersion: config.kubernetes.io/v1
kind: ResourceList
items:
...
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-config-map
    annotations:
      config.kubernetes.io/index: '0'
      internal.config.kubernetes.io/index: '0'
      internal.config.kubernetes.io/path: 'config-map.yaml'
      config.kubernetes.io/path: 'config-map.yaml'
  data:
    color: green
...
```
Package revision can be deleted using `porchctl rpkg del` command:

```bash
# Delete package revision
$ porchctl rpkg del porch-test.network-function.great-outdoors -n porch-demo
porch-test.network-function.great-outdoors deleted
```

## Package Lifecycle and Approval Flow

Authoring is performed on the package revisions in the _Draft_ lifecycle stage. Before a package can be deployed, copied or
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
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft       porch-deployment
porch-deployment.new-package-clone.v1          new-package-clone          v1              0          false    Draft       porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft       porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft       porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Draft       porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft       porch-test
...

# Propose two package revisions to be be published
$ porchctl rpkg propose \
  porch-deployment.new-package-clone.v1 \
  porch-test.network-function3.innerhome6 \
  -n porch-demo

porch-deployment.new-package-clone.v1 proposed
porch-test.network-function3.innerhome6 proposed

# Confirm the package revisions are now Proposed
$ porchctl rpkg get
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft       porch-deployment
porch-deployment.new-package-clone.v1          new-package-clone          v1              0          false    Proposed    porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft       porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft       porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Proposed    porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft       porch-test
```

At this point, a person in _platform administrator_ role, or even an automated process, will review and either approve
or reject the proposals. To aid with the decision, the platform administrator may inspect the package contents using the
commands above, such as `porchctl rpkg pull`.

```bash
# Approve a proposal to publish a package revision
$ porchctl rpkg approve porch-deployment.new-package-clone.v1 -n porch-demo
porch-deployment.new-package-clone.v1 approved

# Reject a proposal to publish a package revision
$ porchctl rpkg reject porch-test.network-function3.innerhome6 -n porch-demo
porch-test.network-function3.innerhome6 no longer proposed for approval
```

Now the user can confirm lifecycle stages of the package revisions:

```bash
# Confirm package revision lifecycle stages after approvals:
$ porchctl rpkg get
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft       porch-deployment
porch-deployment.new-package-clone.v1          new-package-clone          v1              1          true     Published   porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft       porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft       porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Draft       porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft       porch-test
```

Observe that the rejected proposal returned the package revision back to _Draft_ lifecycle stage. The package whose
proposal was approved is now in _Published_ state.

An approved pacakge revision cannot be directly deleted, it must first be proposed for deletion.

```bash
porchctl rpkg propose-delete -n porch-demo porch-deployment.new-package-clone.v1

# Confirm package revision lifecycle stages after deletion proposed:
$ porchctl rpkg get
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE          REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft              porch-deployment
porch-deployment.new-package-clone.v1          new-package-clone          v1              1          true     DeletionProposed   porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft              porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft              porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Draft              porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft              porch-test
```

At this point, a person in _platform administrator_ role, or even an automated process, will review and either approve
or reject the deletion.

```bash
porchctl rpkg reject -n porch-demo porch-deployment.new-package-clone.v1

# Confirm package revision deletion has been rejected:
$ porchctl rpkg get
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE          REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft       porch-deployment
porch-deployment.new-package-clone.v1          new-package-clone          v1              1          true     Published   porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft       porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft       porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Draft       porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft       porch-test
```

The package revision can again be proposed for deletion.

```bash
porchctl rpkg propose-delete -n porch-demo porch-deployment.new-package-clone.v1

# Confirm package revision lifecycle stages after deletion proposed:
$ porchctl rpkg get
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE          REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft              porch-deployment
porch-deployment.new-package-clone.v1          new-package-clone          v1              1          true     DeletionProposed   porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft              porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft              porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Draft              porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft              porch-test
```

At this point, a person in _platform administrator_ role, or even an automated process, decides to proceed with the deletion.

```bash
porchctl rpkg delete -n porch-demo porch-deployment.new-package-clone.v1

# Confirm package revision is deleted:
$ porchctl rpkg get
NAME                                           PACKAGE                    WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
...
porch-deployment.cloned-pkg-example-ue-bp.v1   cloned-pkg-example-ue-bp   v1              0          false    Draft       porch-deployment
porch-test.network-function2.outerspace        network-function2          outerspace      0          false    Draft       porch-test
porch-test.network-function3.innerhome5        network-function3          innerhome5      0          false    Draft       porch-test
porch-test.network-function3.innerhome6        network-function3          innerhome6      0          false    Draft       porch-test
porch-test.new-package.my-workspace            new-package                my-workspace    0          false    Draft       porch-test
```

