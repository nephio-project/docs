---
title: "Creating your first package"
type: docs
weight: 3
description: "A tutorial designed to guide a first time user's to the package creation process in Porch" 
---

## Prerequisites

- Porch deployed on a Kubernetes cluster [Setup Porch Guide]({{% relref "/docs/neo-porch/3_getting_started/installing-porch.md" %}}).
- **Porchctl** CLI tool installed [Setup Porchctl Guide]({{% relref "/docs/neo-porch/3_getting_started/installing-porch.md" %}}).
- A Git repository registered with Porch [Setup Repositories Guide]({{% relref "/docs/neo-porch/4_tutorials_and_how-tos/setting-up-repositories.md" %}}).
- **Kubectl** configured to access your cluster.

## Tutorial Overview

You will learn how to:

1. Initialize a new package
2. Pull the package locally
3. Modify the package (update Kptfile)
4. Push changes back to Porch
5. Propose the package for review
6. Approve or reject the package

---

## Step 1: Initialize Your First Package

Create a new package in Porch using the `init` command:

```bash
porchctl rpkg init my-first-package \
  --namespace=default \
  --repository=porch-test \
  --workspace=v1 \
  --description="My first Porch package"
```

**What this does:**

- Creates a new package named `my-first-package`
- Places it in the `porch-test` repository
- Uses `v1` as the workspace name (must be unique)
- Starts in `Draft` lifecycle state

![Diagram](/static/images/porch/guides/init-workflow.drawio.svg)

**Verify the package was created:**

```bash
porchctl rpkg get --namespace default
```

You should see your package listed with lifecycle `Draft`:

```bash
NAME                             PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.my-first-package.v1   my-first-package   v1              0          false    Draft       porch-test
```

---

## Step 2: Pull the Package Locally

Download the package contents to your local filesystem:

```bash
porchctl rpkg pull porch-test.my-first-package.v1 ./my-first-package --namespace default
```

**What this does:**

- Fetches all package resources from Porch
- Saves them to the `./my-first-package` directory
- Includes the Kptfile and any other resources

![Diagram](/static/images/porch/guides/pull-workflow.drawio.svg)

**Explore the package:**

```bash
ls -al ./my-first-package
```

You'll see:

- The `Kptfile` - Package metadata and pipeline configuration
- Other YAML files (if any were created)

```bash
total 24
drwxr-x--- 2 user user 4096 Nov 24 13:27 .
drwxr-xr-x 4 user user 4096 Nov 24 13:27 ..
-rw-r--r-- 1 user user  259 Nov 24 13:27 .KptRevisionMetadata
-rw-r--r-- 1 user user  177 Nov 24 13:27 Kptfile
-rw-r--r-- 1 user user  488 Nov 24 13:27 README.md
-rw-r--r-- 1 user user  148 Nov 24 13:27 package-context.yaml
```

**Alternatively:**

If you have the tree command installed on your system you can use it to view the hierarchy of the package

```bash
tree ./my-first-package/
```

Should return the following output on your package:

```bash
my-first-package/
├── Kptfile
├── README.md
└── package-context.yaml

1 directory, 3 files
```

---

## Step 3: Modify the Package

Let's add a simple KRM function to the pipeline.

Open the `Kptfile` in your editor of choice:

```bash
vim ./my-first-package/Kptfile
```

Add a mutator function to the pipeline section so that your `Kptfile` looks like so:

```yaml
apiVersion: kpt.dev/v1
kind: Kptfile
metadata:
  name: my-first-package
  annotations:
    config.kubernetes.io/local-config: "true"
info:
  description: My first Porch package
pipeline:
  mutators:
    - image: gcr.io/kpt-fn/set-namespace:v0.4.1
      configMap:
        namespace: production
```

**What this does:**

- Adds a `set-namespace` function to the pipeline
- This function will set the namespace to `production` for all resources in the package
- Functions run automatically when the package is rendered

**Add new resource:**

Create a new configmap inside the package:

```bash
vim ./my-first-package/test-config.yaml
```

Now add the following content to this new configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  Key: "Value"
```

**Save and close the file.**

{{% alert title="Note" color="primary" %}}
Changes are LOCAL ONLY (Porch doesn't know about them yet) at this stage
{{% /alert %}}

---

## Step 4: Push Changes Back to Porch

Upload your modified package back to Porch:

```bash
porchctl rpkg push porch-test.my-first-package.v1 ./my-first-package --namespace default
```

**What this does:**

- Updates the package revision in Porch
- Triggers rendering (executes pipeline functions)
- Package remains in `Draft` state

![Diagram](/static/images/porch/guides/push-workflow.drawio.svg)

**Successful output:**

This describes how the KRM function was run by porch and has updated the namespace in our new configmap.

```bash
[RUNNING] "gcr.io/kpt-fn/set-namespace:v0.4.1"
[PASS] "gcr.io/kpt-fn/set-namespace:v0.4.1"
  Results:
    [info]: namespace "" updated to "production", 1 value(s) changed
porch-test.my-first-package.v1 pushed
```

---

## Step 5: Propose the Package

Move the package to `Proposed` state for review:

```bash
porchctl rpkg propose porch-test.my-first-package.v1 --namespace default
```

**What this does:**

- Changes lifecycle from `Draft` to `Proposed`
- Signals the package is ready for review
- Package can still be modified if needed

![Diagram](/static/images/porch/guides/propose-workflow.drawio.svg)

{{% alert title="Note" color="primary" %}}
Note a state change from `Draft` to `Proposed` means that in Git the package has moved from the `draft` branch to the `proposed` branch
{{% /alert %}}

**Verify the state change:**

```bash
porchctl rpkg get porch-test.my-first-package.v1 --namespace default
```

The lifecycle should now show `Proposed`.

```bash
NAME                             PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.my-first-package.v1   my-first-package   v1              0          false    Proposed    porch-test
```

---

## Step 6a: Approve the Package

If the package looks good, approve it to publish:

```bash
porchctl rpkg approve porch-test.my-first-package.v1 --namespace default
```

**What this does:**

- Changes lifecycle from `Proposed` to `Published`
- Package becomes **immutable** (content cannot be changed)
- Records who approved and when
- Package is now available for cloning/deployment

![Diagram](/static/images/porch/guides/approve-workflow.drawio.svg)

**Verify publication:**

```bash
porchctl rpkg get porch-test.my-first-package.v1 --namespace default -o yaml | grep -E "lifecycle|publishedBy|publishTimestamp"
```

**Verify the state change:**

```bash
porchctl rpkg get porch-test.my-first-package.v1 --namespace default
```

The lifecycle should now show `Published`.

```bash
NAME                             PACKAGE            WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
porch-test.my-first-package.v1   my-first-package   v1              1          true     Published   porch-test
```

---

## Step 6b: Reject the Package (Alternative)

If the package needs more work, reject it to return to `Draft`:

```bash
porchctl rpkg reject porch-test.my-first-package.v1 --namespace default
```

**What this does:**

- Changes lifecycle from `Proposed` back to `Draft`
- Allows further modifications
- You can then make changes and re-propose

![Diagram](/static/images/porch/guides/reject-workflow.drawio.svg)

Once rejected the process repeats from Stage 2 to [Pull -> Modify -> Propose and Approve] users are satisfied with its state.

---

## Troubleshooting

**Package stuck in Draft?**

- Check readiness conditions: `porchctl rpkg get <PACKAGE> -o yaml | grep -A 5 conditions`

**Push fails with conflict?**

- Pull the latest version first: `porchctl rpkg pull <PACKAGE> ./dir`
- The package may have been modified by someone else

**Cannot modify Published package?**

- Published packages are immutable
- Create a new revision using `porchctl rpkg copy`

---

## Understanding Package Names

Porch generates package names automatically:

- Format: `{repoName}-{packageName}-{workspaceName}`
- Example: `porch-test.my-first-package.v1`
- The workspace name must be unique per package

---

## Key Concepts

- **Draft**: Work in progress, fully editable
- **Proposed**: Ready for review, still editable
- **Published**: Approved and immutable
- **Workspace**: Unique identifier for a package revision
- **Repository**: Git repo where packages are stored
- **Pipeline**: KRM functions that transform package resources

---
