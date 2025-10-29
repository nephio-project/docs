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

*   **`REPOSITORY`**: The URI for the repository.
    *   For a Git repository, use the standard Git URL (e.g., `http://my-gitea.com/user/repo.git`).

### Command Flags

The following flags can be used to configure the repository registration:

| Flag                    | Description                                                                                                |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| `--name`                | A unique name for the repository. If not specified, it defaults to the last segment of the repository URL.   |
| `--branch`              | The branch where finalized packages are committed (Git-only). Defaults to `main`.                            |
| `--directory`           | The directory within the repository where packages are located. Defaults to the repository root (`/`).       |
| `--deployment`          | If set to `true`, marks the repository as a "deployment" repository, where ready-to-deploy packages reside. |
| `--description`         | A human-readable description of the repository.                                                            |
| `--repo-basic-username` | The username for basic authentication if the repository is private.                                        |
| `--repo-basic-password` | The password for basic authentication if the repository is private.                                        |

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


### Test Environment Git: Gitea

To create extra repositories via CLI, in this example we create "blueprints" and "deployment" repositories and we register them with porch.

```bash
# Define repository names to be created
REPOS=(blueprints deployment)
GIT_HOST="172.18.255.200:30000" # "localhost:3000" for docker-desktop users in WSL
GIT_USER="nephio"
GIT_PASS="secret"
API_URL="http://$GIT_USER:$GIT_PASS@$GIT_HOST/api/v1/user/repos"
NAMESPACE="porch-demo"
GIT_ROOT=$(pwd)
# Loop through each repo
for REPO_NAME in "${REPOS[@]}"; do
  echo "Creating repo: $REPO_NAME"

  # Create the repository via API
  curl -v -k -H "Content-Type: application/json" "$API_URL" --data "{\"name\":\"$REPO_NAME\"}"

  # Create temp directory and clone repo
  TMP_DIR=$(mktemp -d)
  cd "$TMP_DIR"
  git clone "http://$GIT_USER:$GIT_PASS@$GIT_HOST/$GIT_USER/$REPO_NAME.git"
  cd "$REPO_NAME"

  # Check if 'main' branch exists
  if ! git rev-parse -q --verify refs/remotes/origin/main >/dev/null; then
    echo "Add main branch to git repo: $REPO_NAME"
    git switch -c main
    touch README.md
    git add README.md
    git config user.name "$GIT_USER"
    git commit -m "first commit"
    git push -u origin main
  else
    echo "main branch already exists in git repo: $REPO_NAME"
  fi

  # Cleanup
  cd "$GIT_ROOT"
  rm -rf "$TMP_DIR"


  porchctl repo register http://gitea.gitea:3000/nephio/$REPO_NAME.git \
    --name=$REPO_NAME \
    --namespace=$NAMESPACE \
    --description="$REPO_NAME repository for Porch packages" \
    --branch=main \
    --deployment=true \
    --repo-basic-username=nephio \
    --repo-basic-password=secret

done
```

# See Also

In this example we demonstrate a simple HTTP Basic auth setup using a Kubernetes `Secret`. For production environments, prefer secret management solutions (external secret stores, sealed-secrets, or platform secrets) and avoid embedding plaintext credentials in scripts.

[Authenticating to Remote Git Repositories]({{% relref "/docs/neo-porch/6_configuration_and_deployments/configurations/private-registries.md" %}})
