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

To create additional repositories you can use the Gitea web UI or the Gitea API/CLI. The Porch project includes an example automated setup script that demonstrates creating repositories and initializing branches: [install-dev-gitea-setup.sh](https://github.com/nephio-project/porch/blob/main/scripts/install-dev-gitea-setup.sh#L82). Afterwards, register your new repositories directly with Porch using the `porchctl repo register` command shown above.

# See Also

In this example we demonstrate a simple HTTP Basic auth setup using a Kubernetes `Secret`. For production environments, prefer secret management solutions (external secret stores, sealed-secrets, or platform secrets) and avoid embedding plaintext credentials in scripts.

[Authenticating to Remote Git Repositories]({{% relref "/docs/neo-porch/6_configuration_and_deployments/configurations/private-registries.md" %}})
