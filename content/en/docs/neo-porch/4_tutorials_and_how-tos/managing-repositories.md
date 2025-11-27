---
title: "Managing Repositories"
type: docs
weight: 3
description: "A guide to registering, viewing, and managing repositories in Porch"
---

## Prerequisites

- Porch deployed on a Kubernetes cluster [Setup Porch Guide]({{% relref "/docs/neo-porch/3_getting_started/installing-porch.md" %}}).
- **Porchctl** CLI tool installed [Setup Porchctl Guide]({{% relref "/docs/neo-porch/3_getting_started/installing-porchctl.md" %}}).
- **Kubectl** configured to access your cluster.
- A Git repository to register with Porch. If you need to create one, see [GitHub's Repository Guide](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories).

---

## Understanding Repositories

Before Porch can manage packages, you must register repositories where those packages are stored. Repositories tell Porch:

- Where to find package blueprints
- Where to store deployment packages
- How to authenticate with the repository

Porch primarily supports **Git repositories** from providers like GitHub, GitLab, Gitea, Bitbucket, and other Git-compatible services.

**Repository Types by Purpose:**

- **Blueprint Repositories**: Contain upstream package templates and blueprints that can be cloned and customized. These are typically read-only sources of reusable configurations.
- **Deployment Repositories**: Store deployment-ready packages that are actively managed and deployed to clusters. Mark repositories as deployment repositories using the `--deployment` flag during registration.

---

## Repository Types

Porch primarily supports Git repositories for storing and managing packages. Git is the recommended and production-ready storage backend.

### Git Repositories

Git repositories are the primary and recommended type for use with Porch.

**Requirements:**

- Git repository with an initial commit (to establish main branch)
- For private repos: Personal Access Token or Basic Auth credentials

**Supported Git hosting services:**

- GitHub
- GitLab
- Gitea
- Bitbucket
- Any Git-compatible service

---

### OCI Repositories (Experimental)

{{% alert title="Warning" color="warning" %}}
OCI repository support is **experimental** and not actively maintained. Use at your own risk. This feature may have limitations, bugs, or breaking changes. For production deployments, use Git repositories.
{{% /alert %}}

Porch has experimental support for OCI (Open Container Initiative) repositories that store packages as container images. This feature is not recommended for production use.

---

## Registering Repositories

Registering a repository connects Porch to your Git storage backend, allowing it to discover and manage packages. You can register repositories with various authentication methods and configuration options.

### Register a Git Repository

Register a Git repository with Porch:

```bash
porchctl repo register https://github.com/example/blueprints.git \
  --namespace default \
  --name=blueprints \
  --description="Blueprint packages" \
  --branch=main
```

**What this does:**

- Registers the Git repository with Porch
- Creates a Repository resource in Kubernetes
- Begins synchronizing packages from the repository

**Example output:**

```bash
blueprints created
```

**Verify registration:**

Check that the repository was registered successfully:

```bash
porchctl repo get blueprints --namespace default
```

Look for `READY: True` in the output to confirm the repository is accessible and synchronized.

```bash
NAME         TYPE   CONTENT   SYNC SCHEDULE   DEPLOYMENT   READY   ADDRESS
blueprints   git    Package                                True    https://github.com/example-repo/blueprints.git
```

**If READY shows False:**

Inspect the detailed status to see the error message:

```bash
porchctl repo get blueprints --namespace default -o yaml
```

Check the `status.conditions` section for the error details:

```yaml
status:
  conditions:
  - type: Ready
    status: "False"
    reason: Error
    message: 'failed to list remote refs: repository not found: Repository not found.'
    lastTransitionTime: "2025-11-27T14:32:20Z"
```

**Common error messages:**

- `failed to list remote refs: repository not found` - Repository URL is incorrect or repository doesn't exist
- `failed to resolve credentials: cannot resolve credentials in a secret <namespace>/<secret-name>: secrets "<secret-name>" not found` - Authentication secret doesn't exist or name is misspelled
- `failed to resolve credentials: resolved credentials are invalid` - Credentials in the secret are invalid or malformed
- `branch "<branch-name>" not found in repository` - Specified branch doesn't exist in the repository
- `repository URL is empty` - Repository URL not specified in configuration
- `target branch is empty` - Branch name not specified in configuration

---

### Register with Authentication

For private repositories, provide authentication credentials:

**Using Basic Auth:**

```bash
porchctl repo register https://github.com/example/private-repo.git \
  --namespace default \
  --name=private-repo \
  --repo-basic-username=myusername \
  --repo-basic-password=mytoken
```

**Using Workload Identity (GCP):**

```bash
porchctl repo register https://github.com/example/private-repo.git \
  --namespace default \
  --name=private-repo \
  --repo-workload-identity
```

{{% alert title="Note" color="primary" %}}
For production environments, use secret management solutions (external secret stores, sealed-secrets) rather than embedding credentials in commands.

See [Authenticating to Remote Git Repositories]({{% relref "/docs/neo-porch/6_configuration_and_deployments/relevant_old_docs/git-authentication-config.md" %}}) for detailed authentication configuration.
{{% /alert %}}

---

### Register with Advanced Options

Configure additional repository settings:

```bash
porchctl repo register https://github.com/nephio-project/catalog \
  --namespace default \
  --name=infra \
  --directory=infra \
  --deployment=true \
  --sync-schedule="*/10 * * * *" \
  --description="Infrastructure packages"
```

**Common flags:**

- `--name`: Repository name in Kubernetes (defaults to last segment of URL)
- `--description`: Brief description of the repository
- `--branch`: Git branch to use (defaults to `main`)
- `--directory`: Subdirectory within repository containing packages. Use `/` for repository root, or specify a path like `/blueprints` or `infra/packages`. Leading slash is optional.
- `--deployment`: Mark as deployment repository (packages are deployment-ready)
- `--sync-schedule`: Cron expression for periodic sync (e.g., `*/10 * * * *` for every 10 minutes).
- `--repo-basic-username`: Username for basic authentication
- `--repo-basic-password`: Password/token for basic authentication
- `--repo-workload-identity`: Use workload identity for authentication

{{% alert title="Note" color="primary" %}}
For complete command syntax and all available flags, see the [Porchctl CLI Guide]({{% relref "/docs/neo-porch/7_cli_api/relevant_old_docs/porchctl-cli-guide.md" %}}).
{{% /alert %}}

---

## Basic Operations

These operations cover the fundamental commands for viewing and managing registered repositories.

### List Registered Repositories

View all repositories registered with Porch:

```bash
porchctl repo get --namespace default
```

**What this does:**

- Queries Porch for all registered repositories in the specified namespace
- Displays repository type, content, sync schedule, and status
- Shows the repository address

{{% alert title="Note" color="primary" %}}
`porchctl repo list` is an alias for `porchctl repo get` and can be used interchangeably:

```bash
porchctl repo list --namespace default
```

{{% /alert %}}

**Using kubectl:**

You can also use kubectl to list repositories:

```bash
kubectl get repositories -n default
```

List repositories across all namespaces:

```bash
kubectl get repositories --all-namespaces
```

**Example output:**

```bash
NAME         TYPE   CONTENT   SYNC SCHEDULE   DEPLOYMENT   READY   ADDRESS
porch-test   git    Package                                True    https://github.com/example-org/test-packages.git
blueprints   git    Package   */10 * * * *                 True    https://github.com/example/blueprints.git
infra        git    Package   */10 * * * *    true         True    https://github.com/nephio-project/catalog
```

**Understanding the output:**

- **NAME**: Repository name in Kubernetes
- **TYPE**: Repository type (`git` or `oci`)
- **CONTENT**: Content type (typically `Package`)
- **SYNC SCHEDULE**: Cron expression for periodic synchronization (if configured).
- **DEPLOYMENT**: Whether this is a deployment repository
- **READY**: Repository health status
- **ADDRESS**: Repository URL

---

### Get Detailed Repository Information

View complete details about a specific repository:

```bash
porchctl repo get porch-test --namespace default -o yaml
```

**What this does:**

- Retrieves the full Repository resource
- Shows configuration, authentication, and status information
- Displays in YAML format for easy reading

**Example output:**

```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: porch-test
  namespace: default
  creationTimestamp: "2025-11-21T16:27:27Z"
spec:
  content: Package
  type: git
  git:
    repo: https://github.com/example-org/test-packages.git
    branch: main
    directory: /
    secretRef:
      name: porch-test-auth
status:
  conditions:
  - type: Ready
    status: "True"
    reason: Ready
    message: 'Repository Ready (next sync scheduled at: 2025-11-26T09:48:03Z)'
    lastTransitionTime: "2025-11-26T09:45:03Z"
```

**Key fields to inspect:**

- **spec.type**: Repository type (typically `git`)
- **spec.git**: Git-specific configuration (repo URL, branch, directory, credentials)
- **spec.content**: Content type stored in repository
- **status.conditions**: Repository health and sync status
  - **status**: `"True"` (healthy) or `"False"` (error)
  - **reason**: `Ready` or `Error`
  - **message**: Detailed error message when status is False

---

## Repository Synchronization

Porch periodically synchronizes with registered repositories to discover new packages and updates. You can also trigger manual synchronization when you need immediate updates.

### Trigger Manual Sync

Force an immediate synchronization of a repository:

```bash
porchctl repo sync porch-test --namespace default
```

**What this does:**

- Schedules a one-time sync (minimum 1-minute delay)
- Updates packages from the repository
- Independent of periodic sync schedule

**Example output:**

```bash
Repository porch-test sync scheduled
```

---

### Sync Multiple Repositories

Sync several repositories at once:

```bash
porchctl repo sync repo1 repo2 repo3 --namespace default
```

---

### Sync All Repositories

Sync all repositories in a namespace:

```bash
porchctl repo sync --all --namespace default
```

Sync across all namespaces:

```bash
porchctl repo sync --all --all-namespaces
```

---

### Schedule Delayed Sync

Schedule sync with custom delay:

```bash
# Sync in 5 minutes
porchctl repo sync porch-test --namespace default --run-once 5m

# Sync in 2 hours 30 minutes
porchctl repo sync porch-test --namespace default --run-once 2h30m

# Sync at specific time
porchctl repo sync porch-test --namespace default --run-once "2024-01-15T14:30:00Z"
```

{{% alert title="Note" color="primary" %}}
**Sync behavior:**

- Minimum delay is 1 minute from command execution
- Updates `spec.sync.runOnceAt` field in Repository CR
- Independent of existing periodic sync schedule
- Past timestamps are automatically adjusted to minimum delay
{{% /alert %}}

---

## Unregistering Repositories

When you no longer need Porch to manage packages from a repository, you can unregister it. This removes Porch's connection to the repository without affecting the underlying Git storage.

### Unregister a Repository

Remove a repository from Porch:

```bash
porchctl repo unregister porch-test --namespace default
```

**What this does:**

- Removes the Repository resource from Kubernetes
- Stops synchronizing packages from the repository
- Removes Porch's cached metadata for the repository
- Does not delete the underlying Git repository or its contents

{{% alert title="Warning" color="warning" %}}
Unregistering a repository does not delete the underlying Git repository or its contents. It only removes Porch's connection to it.
{{% /alert %}}

**Example output:**

```bash
porch-test unregistered
```

**What happens to packages:**

- **Published packages in Git**: Remain in the Git repository and are preserved. If you re-register the same repository later, these packages will reappear when Porch synchronizes.
- **Draft/Proposed packages pushed to Git**: Also remain in Git and will reappear upon re-registration.
- **Unpushed work-in-progress packages**: Cached packages that were never pushed to Git (draft packages being edited) are removed and cannot be recovered.

---

## Troubleshooting

Common issues when working with repositories and their solutions:

**Repository shows READY: False?**

- Check repository URL is accessible
- Verify authentication credentials are correct
- Inspect repository conditions: `porchctl repo get <name> -n <namespace> -o yaml`
- Check Porch server logs for detailed errors

**Packages not appearing after registration?**

- Ensure repository has been synchronized (check SYNC SCHEDULE or trigger manual sync)
- Verify packages have valid Kptfile in repository
- Check repository directory configuration matches package location
- If re-registering a previously unregistered repository, packages in Git will reappear after sync

**Authentication failures?**

- For GitHub: Ensure Personal Access Token has `repo` scope
- For private repos: Verify credentials are correctly configured
- Check secret exists: `kubectl get secret <secret-name> -n <namespace>`

**Sync not working?**

- Verify cron expression syntax is correct
- Check minimum 1-minute delay for manual syncs
- Inspect repository status for sync errors

---

## Key Concepts

Important terms and concepts for working with Porch repositories:

- **Repository**: A Git repository registered with Porch for package management. Repositories are namespace-scoped Kubernetes resources.
- **Blueprint Repository**: Contains upstream package templates that can be cloned and customized. Typically used as read-only sources.
- **Deployment Repository**: Repository marked with `--deployment` flag containing deployment-ready packages that are actively managed.
- **Sync Schedule**: Cron expression defining periodic repository synchronization (e.g., `*/10 * * * *` for every 10 minutes).
- **Content Type**: Defines what the repository stores. `Package` is the standard type for KRM configuration packages. Other types like `Function` exist for storing KRM functions.
- **Branch**: Git branch Porch monitors for packages (defaults to `main`). Each repository tracks a single branch.
- **Directory**: Subdirectory within repository where packages are located. Use `/` for root or specify a path like `/blueprints`.
- **Namespace Scope**: Repositories exist within a Kubernetes namespace. Repository names must be unique per namespace, and packages inherit the repository's namespace. The same Git repository can be registered in multiple namespaces with different names, creating isolated package views per namespace.

---
