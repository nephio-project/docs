---
title: "Managing Porch Repositories"
type: docs
weight: 2
description: "A how-to guide on registering Git repositories with Porch"
---

# Setting Up Repositories

Before Porch can manage packages, you must register the repositories where those packages are stored. This tells Porch where to find package blueprints and where to store deployment packages. Porch supports both Git repositories and OCI registries (OCI is not fully supported).

This guide covers the primary method for registering a repository with Porch.

## Using `porchctl`

The `porchctl` command-line tool provides a straightforward way to register a repository.

### Command Syntax

The basic command for registering a repository is:

```bash
porchctl repo register REPOSITORY [flags]
```

For more information about the `repo` command and available flags, see the porchctl CLI guide [Repository registration]({{% relref "/docs/neo-porch/7_cli_api/relevant_old_docs/porchctl-cli-guide.md#repository-registration" %}}).

### Example

This example registers a private Git repository hosted on Gitea and configures it as a deployment repository.

```bash
# Register a Git repository with Porch
porchctl repo register http://gitea.gitea:3000/nephio/porch-test.git \
  --name=porch-test \
  --description="Test repository for Porch packages" \
  --branch=main \
  --deployment=true \
  --repo-basic-username=nephio \
  --repo-basic-password=secret
```

## Creating Additional Gitea Repositories

You can create repositories **manually** using the Gitea web UI by following the displayed steps, or you can **automate** their creation using the Gitea API or CLI. The Porch project includes an example automated setup script that demonstrates how to create repositories and initialize branches:

[install-dev-gitea-setup.sh](https://github.com/nephio-project/porch/blob/23da894a8ef61fea4a4843294f249c3e1817a104/scripts/install-dev-gitea-setup.sh#L82-L100)

You can customize the `$git_repo_name` variable for the custom repository you wish to create.

Below is a high-level explanation of the steps performed by that script when initializing a new Gitea repository for use with Porch.

1. Create the repository via the Gitea REST API
2. Clone the (now-empty) repository locally
3. Ensure a main branch exists and push an initial commit
4. Clean up temporary files
5. Register the new repository with Porch using the command above

# See Also

In this example we demonstrate a simple HTTP Basic auth setup using a Kubernetes `Secret`. For production environments, prefer secret management solutions (external secret stores, sealed-secrets, or platform secrets) and avoid embedding plaintext credentials in scripts.

[Authenticating to Remote Git Repositories]({{% relref "/docs/neo-porch/6_configuration_and_deployments/configurations/private-registries.md" %}})
