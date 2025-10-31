---
title: "Managing Porch Repositories"
type: docs
weight: 2
description: "A how-to guide on registering Git repositories with Porch"
---

# Setting Up Repositories

Before Porch can manage packages, you must register the Git repositories where those packages are stored. This tells Porch where to find package blueprints and where to store deployment packages.

If you don't have an Git repository already created and initialized then follow the steps below to use Gitea (already provided by the developer environment installation), or create and use your own Git repository.

## Creating Additional Gitea Repositories (Optional)

You can create repositories in two ways:

- **Manual — Gitea web UI:** Open your Gitea instance, sign in, and click the **New Repository** (or **+**) button in the top-right of the UI. Follow the form to set the repository name, description, visibility, and initialize the repository. This is the simplest option for one-off repositories.
- **Automated — API / CLI / script:** Automate repository creation via the Gitea REST API, the `gitea` CLI, or an automation script. The Porch project includes an example automated setup script that demonstrates creating repositories and initializing branches:

[install-dev-gitea-setup.sh](https://github.com/nephio-project/porch/blob/23da894a8ef61fea4a4843294f249c3e1817a104/scripts/install-dev-gitea-setup.sh#L82-L100)

You can customize the `$git_repo_name` variable for the custom repository you wish to create.

Below is a high-level explanation of the steps performed by the automated script when initializing a new Gitea repository for use with Porch. These numbered steps reflect the scripted (API/CLI) approach — you may copy the equivalent commands to run them manually if you prefer.

1. Create the repository via the Gitea REST API
2. Clone the (now-empty) repository locally
3. Ensure a main branch exists and push an initial commit
4. Clean up temporary files
5. Register the new repository with Porch using the command shown in the `porchctl` section above

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

# See Also

In this example we demonstrate a simple HTTP Basic auth setup using a Kubernetes `Secret`. For production environments, prefer secret management solutions (external secret stores, sealed-secrets, or platform secrets) and avoid embedding plaintext credentials in scripts.

[Authenticating to Remote Git Repositories]({{% relref "/docs/neo-porch/6_configuration_and_deployments/relevant_old_docs/git-authentication-config.md" %}})
