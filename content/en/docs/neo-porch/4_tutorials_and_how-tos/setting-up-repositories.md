---
title: "Managing Porch Repositories"
type: docs
weight: 2
description: "Tutorial on setting up porch repositories and using them. Before Porch can manage packages, you must register the Git repositories where those packages are stored. This tells Porch where to find package blueprints and where to store deployment packages."
---

If you don't have a Git repository already created and initialized, follow the steps below to create and use your own Git repository.

## Creating and initializing a Git Repository

1. **Create a new repository** in your Git hosting service (e.g., GitHub, GitLab, Gitea, Bitbucket). Navigate to your user or organization page and create a new repository. Provide a name (e.g., `porch-repo`), description, and set visibility as needed.
2. **Initialize the repository** for the main/master branch to exist (typically done by adding a README.md file in the UI) or by cloning it locally and add initial content.

For detailed instructions on repository creation, refer to your Git hosting service documentation. [For example on GitHub](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories).

## Register a Git-Repository as a Porch-Repository

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

{{% alert title="Note" color="primary" %}}
Replace the Git URL, repository name, username, and password with your values.
{{% /alert %}}

# See Also

In this example we demonstrate a simple HTTP Basic auth setup using a Kubernetes `Secret`. For production environments, prefer secret management solutions (external secret stores, sealed-secrets, or platform secrets) and avoid embedding plaintext credentials in scripts.

[Authenticating to Remote Git Repositories]({{% relref "/docs/neo-porch/6_configuration_and_deployments/relevant_old_docs/git-authentication-config.md" %}})
