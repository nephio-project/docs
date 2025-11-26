---
title: "Inspecting Packages"
type: docs
weight: 3
description: "A guide to viewing, querying, and inspecting packages in Porch"
---

## Prerequisites

- Porch deployed on a Kubernetes cluster [Setup Porch Guide]({{% relref "/docs/neo-porch/3_getting_started/installing-porch.md" %}}).
- **Porchctl** CLI tool installed [Setup Porchctl Guide]({{% relref "/docs/neo-porch/3_getting_started/installing-porch.md" %}}).
- A Git repository registered with Porch [Setup Repositories Guide]({{% relref "/docs/neo-porch/4_tutorials_and_how-tos/setting-up-repositories.md" %}}).
- **Kubectl** configured to access your cluster.

---

## Basic Operations

These operations cover the fundamental commands for viewing and inspecting packages in Porch.

### List All Packages

View all packages across all repositories in a namespace:

```bash
porchctl rpkg get --namespace default
```

**What this does:**

- Queries Porch for all package revisions in the specified namespace
- Displays a summary table with key information
- Shows packages from all registered repositories

{{% alert title="Note" color="primary" %}}
`porchctl rpkg list` is an alias for `porchctl rpkg get` and can be used interchangeably: 

```bash
porchctl rpkg list --namespace default
```

{{% /alert %}}

**Example output:**

```bash
NAME                             PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.my-app.v1             my-app             v1              1          true     Published   porch-test
porch-test.my-app.v2             my-app             v2              0          false    Draft       porch-test
blueprints.nginx.main            nginx              main            5          true     Published   blueprints
blueprints.postgres.v1           postgres           v1              0          false    Proposed    blueprints
```

**Understanding the output:**

- **NAME**: Full package revision identifier following the pattern `repository.([pathnode.]*)package.workspace`
  - Format: `<repository>.[<path-nodes>.]<package>.<workspace>`
  - Example: `porch-test.basedir.subdir.edge-function.v1`
    - Repository: `porch-test`
    - Path: `basedir/subdir` (directory structure)
    - Package: `edge-function`
    - Workspace: `v1`
  - Simple example: `blueprints.nginx.main` (no path nodes)
    - Repository: `blueprints`
    - Package: `nginx`
    - Workspace: `main`

- **PACKAGE**: Package name with directory path if not in repository root
  - Example: `basedir/subdir/network-function` shows location in repository

- **WORKSPACENAME**: User-selected identifier for this package revision
  - Scoped to the package - `v1` in package A is independent from `v1` in package B
  - Maps to Git branch or tag name

- **REVISION**: Version number indicating publication status
  - `1+`: Published revisions (increments with each publish: 1, 2, 3...)
  - `0`: Unpublished revisions (Draft or Proposed)
  - `-1`: Placeholder pointing to Git branch/tag head

- **LATEST**: Whether this is the latest published revision
  - Only one revision per package marked as latest
  - Based on highest revision number

- **LIFECYCLE**: Current state of the package revision
  - `Draft`: Work-in-progress, freely editable, visible to authors
  - `Proposed`: Read-only, awaiting approval, can be approved or rejected
  - `Published`: Immutable, production-ready, assigned revision numbers
  - `DeletionProposed`: Marked for removal, awaiting deletion approval

- **REPOSITORY**: Source repository name

---

### Get Detailed Package Information

View complete details about a specific package:

```bash
porchctl rpkg get porch-test.my-app.v1 --namespace default -o yaml
```

**What this does:**

- Retrieves the full PackageRevision resource
- Shows all metadata, spec, and status fields
- Displays in YAML format for easy reading

**Example output:**

```yaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevision
metadata:
  creationTimestamp: "2025-11-24T13:00:14Z"
  labels:
    kpt.dev/latest-revision: "true"
  name: porch-test.my-first-package.v1
  namespace: default
  resourceVersion: 5778e0e3e9a92d248fec770cef5baf142958aa54
  uid: f9f6507d-20fc-5319-97b2-6b8050c4f9cc
spec:
  lifecycle: Published
  packageName: my-first-package
  repository: porch-test
  revision: 1
  tasks:
  - init:
      description: My first Porch package
    type: init
  workspaceName: v1
status:
  publishTimestamp: "2025-11-24T16:38:41Z"
  upstreamLock: {}
```

**Key fields to inspect:**

- **spec.lifecycle**: Current package state
- **spec.tasks**: History of operations performed
- **status.publishTimestamp**: When it was published

{{% alert title="Tip" color="primary" %}}
Use `jq` to extract specific fields: `porchctl rpkg get <name> -n default -o json | jq '.metadata'`
{{% /alert %}}

---

### View Package Resources

Inspect the actual contents of a package:

```bash
porchctl rpkg pull porch-test.my-first-package.v1 --namespace default
```

**What this does:**

- Fetches package resources and outputs to stdout
- Shows all YAML files in ResourceList format
- Displays the complete package contents

**Example output:**

```yaml
apiVersion: config.kubernetes.io/v1
kind: ResourceList
items:
- apiVersion: ""
  kind: KptRevisionMetadata
  metadata:
    name: porch-test.my-first-package.v1
    namespace: default
    creationTimestamp: "2025-11-24T13:00:14Z"
    resourceVersion: 5778e0e3e9a92d248fec770cef5baf142958aa54
    uid: f9f6507d-20fc-5319-97b2-6b8050c4f9cc
    annotations:
      config.kubernetes.io/path: '.KptRevisionMetadata'
- apiVersion: kpt.dev/v1
  kind: Kptfile
  metadata:
    name: my-first-package
    annotations:
      config.kubernetes.io/local-config: "true"
      config.kubernetes.io/path: 'Kptfile'
  info:
    description: My first Porch package
  pipeline:
    mutators:
    - image: gcr.io/kpt-fn/set-namespace:v0.4.1
      configMap:
        namespace: production
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: kptfile.kpt.dev
    annotations:
      config.kubernetes.io/local-config: "true"
      config.kubernetes.io/path: 'package-context.yaml'
  data:
    name: example
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: test-config
    namespace: production
    annotations:
      config.kubernetes.io/path: 'test-config.yaml'
  data:
    Key: "Value"
```

**Filter specific resources:**

```bash
porchctl rpkg pull porch-test.my-first-package.v1 --namespace default | grep -A 15 "kind: Kptfile"
```

**Example output:**

```yaml
  kind: Kptfile
  metadata:
    name: my-first-package
    annotations:
      config.kubernetes.io/local-config: "true"
      config.kubernetes.io/path: Kptfile
  info:
    description: My first Porch package
  pipeline:
    mutators:
    - image: gcr.io/kpt-fn/set-namespace:v0.4.1
      configMap:
        namespace: production
```

---

## Advanced Filtering

Porch provides multiple ways to filter package revisions beyond basic listing. You can use porchctl's built-in flags, Kubernetes label selectors, or field selectors depending on your needs.

### Using Porchctl Flags

Filter package revisions using built-in porchctl flags:

**Filter by package name (substring match):**

```bash
porchctl rpkg get --namespace default --name my-app
```

**Filter by revision number (exact match):**

```bash
porchctl rpkg get --namespace default --revision 1
```

**Filter by workspace name:**

```bash
porchctl rpkg get --namespace default --workspace v1
```

**Example output:**

```bash
$ porchctl rpkg get --namespace default --name network-function
NAME                                    PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.network-function.v1          network-function   v1              1          false    Published   porch-test
porch-test.network-function.v2          network-function   v2              2          true     Published   porch-test
porch-test.network-function.main        network-function   main            0          false    Draft       porch-test
```

---

### Using Kubectl Label Selectors

Filter using Kubernetes [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#list-and-watch-filtering) with the `--selector` flag:

**Find latest published revisions:**

```bash
kubectl get packagerevisions -n default --selector 'kpt.dev/latest-revision=true'
```

**Example output:**

```bash
$ kubectl get packagerevisions -n default --show-labels --selector 'kpt.dev/latest-revision=true'
NAME                        PACKAGE   WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY        LABELS
porch-test.my-app.v2        my-app    v2              2          true     Published   porch-test        kpt.dev/latest-revision=true
blueprints.nginx.main       nginx     main            5          true     Published   blueprints        kpt.dev/latest-revision=true
```

{{% alert title="Note" color="primary" %}}
PackageRevision resources have limited labels. To filter by repository, package name, or other attributes, use `--field-selector` instead (see next section).
{{% /alert %}}

---

### Using Kubectl Field Selectors

Filter using PackageRevision [fields](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/) with the `--field-selector` flag:

**Supported fields:**

- `metadata.name`
- `metadata.namespace`
- `spec.revision`
- `spec.packageName`
- `spec.repository`
- `spec.workspaceName`
- `spec.lifecycle`

**Filter by repository:**

```bash
kubectl get packagerevisions -n default --field-selector 'spec.repository==porch-test'
```

**Filter by lifecycle:**

```bash
kubectl get packagerevisions -n default --field-selector 'spec.lifecycle==Published'
```

**Filter by package name:**

```bash
kubectl get packagerevisions -n default --field-selector 'spec.packageName==my-app'
```

**Combine multiple filters:**

```bash
kubectl get packagerevisions -n default \
  --field-selector 'spec.repository==porch-test,spec.lifecycle==Published'
```

**Example output:**

```bash
$ kubectl get packagerevisions -n default --field-selector 'spec.repository==porch-test'
NAME                             PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.my-app.v1             my-app             v1              1          false    Published   porch-test
porch-test.my-app.v2             my-app             v2              2          true     Published   porch-test
porch-test.my-service.main       my-service         main            3          true     Published   porch-test
```

{{% alert title="Note" color="primary" %}}
The `--field-selector` flag supports only the `=` and `==` operators. **The `!=` operator is not supported** due to Porch's internal caching behavior.
{{% /alert %}}

---

## Additional Operations

Beyond basic listing and filtering, these operations help you monitor changes and format output.

### Watch for Package Changes

Monitor package revisions in real-time:

```bash
kubectl get packagerevisions -n default --watch
```

### Sort by Creation Time

Find recently created packages:

```bash
kubectl get packagerevisions -n default --sort-by=.metadata.creationTimestamp
```

### Output Formatting

Both `porchctl` and `kubectl` support standard Kubernetes [output formatting flags](https://kubernetes.io/docs/reference/kubectl/#output-options):

- `-o yaml` - YAML format
- `-o json` - JSON format
- `-o wide` - Additional columns
- `-o name` - Resource names only
- `-o custom-columns=...` - Custom column output

{{% alert title="Note" color="primary" %}}
For a complete reference of all available command options and flags, see the [Porch CLI Guide]({{% relref "/docs/neo-porch/7_cli_api/relevant_old_docs/porchctl-cli-guide.md" %}}).
{{% /alert %}}

---

## Troubleshooting

**No packages shown?**

- Verify repositories are registered and healthy (see [Repository Management Guide]({{% relref "/docs/neo-porch/4_tutorials_and_how-tos/setting-up-repositories.md" %}}))
- Ensure packages exist in the Git repository

**Permission denied errors?**

- Check RBAC permissions: `kubectl auth can-i get packagerevisions -n default`
- Verify service account has proper roles

**Package not found?**

- Confirm the exact package name: `porchctl rpkg get --namespace default`
- Check you're using the correct namespace
- Verify the package hasn't been deleted

---

## Key Concepts

- **PackageRevision**: A specific version of a package in Porch
- **Namespace**: Kubernetes namespace where packages are managed
- **Repository**: Git repository containing package sources
- **Lifecycle**: Current state of the package (Draft, Proposed, Published)
- **Workspace**: Unique identifier for a package revision (maps to Git branch/tag)
- **Latest**: Flag indicating the most recent published revision
- **Path nodes**: Directory structure within repository where package is located

---
