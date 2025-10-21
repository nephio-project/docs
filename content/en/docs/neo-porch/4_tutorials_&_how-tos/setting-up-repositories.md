---
title: "Setting Up Repositories"
type: docs
weight: 2
description: "A guide on how to register Git and OCI repositories with Porch using both `porchctl` and `kubectl`."
---

# Setting Up Repositories

Before Porch can manage packages, you must register the repositories where those packages are stored. This tells Porch where to find package blueprints and where to store deployment packages. Porch supports both Git repositories and OCI registries.

This guide covers the two primary methods for registering a repository with Porch.

## Table of Contents

- [Method 1: Using `porchctl`](#method-1-using-porchctl)
- [Method 2: Using `kubectl`](#method-2-using-kubectl)

## Method 1: Using `porchctl`

The `porchctl` command-line tool provides a straightforward way to register a repository.

### Command Syntax

The basic command for registering a repository is:

```bash
porchctl repo register REPOSITORY [flags]
```

*   **`REPOSITORY`**: The URI for the repository.
    *   For a Git repository, use the standard Git URL (e.g., `http://my-gitea.com/user/repo.git`).
    *   For an OCI registry, the URI must have the `oci://` prefix (e.g., `oci://us-central1-docker.pkg.dev/my-project/my-repo`).

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

## Method 2: Using `kubectl`

Alternatively, you can register a repository by directly applying a `Repository` Custom Resource (CR) to your Kubernetes cluster. This method is ideal for declarative, GitOps-driven workflows.

### Example

This example accomplishes the same goal as the `porchctl` command above. It creates a `Secret` for authentication and a `Repository` CR that references it.

```yaml
# Apply the YAML directly to the cluster
kubectl apply -f - <<EOF
# A dedicated namespace for our Porch resources
apiVersion: v1
kind: Namespace
metadata:
  name: porch-demo
---
# A secret to store the Git repository credentials
apiVersion: v1
kind: Secret
metadata:
  name: gitea
  namespace: porch-demo
type: kubernetes.io/basic-auth
data:
  # The username and password must be base64-encoded
  # username: nephio -> bmVwaGlv
  # password: secret -> c2VjcmV0
  username: bmVwaGlv
  password: c2VjcmV0
---
# The Repository Custom Resource definition
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: porch-test
  namespace: porch-demo
spec:
  description: "Test repository for Porch packages"
  # The type of content stored in the repository
  content: Package
  # Marks this as a deployment repository
  deployment: true
  # The repository type (git or oci)
  type: git
  git:
    # The URL of the Git repository
    repo: http://gitea.gitea:3000/nephio/porch-test.git
    # The directory where packages are stored
    directory: /
    # The branch for published packages
    branch: main
    # Whether Porch should create the branch if it doesn't exist
    createBranch: true
    # Reference to the secret containing authentication credentials
    secretRef:
      name: gitea
EOF
```

### Test Environment Git: Gitea

To create extra repositories via CLI, in this example we create "blueprints" and "deployment" repositories and we gister them with porch.

```bash
GIT_USER="nephio"
GIT_PASS="secret"
GIT_HOST="172.18.255.200:30000" # "localhost:3000" for docker-desktop users in WSL
API_URL="http://$GIT_USER:$GIT_PASS@$GIT_HOST/api/v1/user/repos"

# Define repository names
REPOS=(blueprints deployment)
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


  kubectl apply -f - <<EOF
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository

metadata:
  name: $REPO_NAME
  namespace: porch-demo

spec:
  description: $REPO_NAME packages repo
  content: Package
  deployment: true
  type: git
  git:
    repo: http://gitea.gitea:3000/nephio/$REPO_NAME.git
    directory: /
    branch: main
    createBranch: true
    secretRef:
      name: gitea
EOF
done
```
