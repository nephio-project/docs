# Tutorials

## Local Porch Deployment

### Prerequisites

*   A running Kubernetes cluster with Porch installed. (For the local deployment install docker and kind)
*   `porchctl` and `kubectl` CLIs installed and configured.
*   A Git repository registered with Porch (e.g., `porch-test`).

### First time local deployment
To fulfill the requirements you can use the provided scripts, you will have to install manually kubectl and docker.

Run these commands from the root folder of the porch repository

1. Bring up a kind cluster with custom configuration.

```bash
./scripts/setup-dev-env.sh
```

2. Build and load the images into the kind cluster (choose one)

```bash
# CR-CACHE
make run-in-kind
```

```bash
# DB-CACHE with Postgres
make run-in-kind-db-cache
```

3. Wait until the scripts end, and check the status and the resources

```bash
# Check if the pods are READY
kubectl get pods -n porch-system

# Check if the CRD packagerevisions is loaded
kubectl api-resources | grep packagerevisions
```

4. (Optional) `porchctl` will be built into the folder `.build`

```bash
# COPY porchctl into bin
sudo cp ./.build/porchctl /usr/bin/porchctl
# OR update your PATH
export PATH="$HOME/porch/.build:$PATH"
```

5. (Optional) Visit gitea UI [http://localhost:3000/](http://localhost:3000/) with username: nephio and password: secret

<!-- FIRST SECTION ENDS -->
---

### [FLOWCHART EXPLAINING FLOW E2E] init → pull → locally do changes → push → proposed → approved/rejected → if rejected changes required then re proposed → if approved → becomes published/latest → A good flowchart of the different states of packages would be good posssibly using
todo

![Flowchart](static/flowchart.drawio.svg)


Gallery to use diagrams as code as to make them more update-able (BIGGEST ISSUE WITH DIAGRAMS IN DOCS IS KEEPING THEM UPDATED) [diagrams as code could alleviate this a bit]

### [STEP 1: SETUP PORCH REPOSITORIES RESOURCE] LIKELY FIRST STEP FROM A DEPLOYMENT OF PORCH TO USE IT
todo

### [STEP 2: CREATING FIRST PACKAGE] INIT HOLLOW PKG -> PULL PKG LOCALLY FROM REPO -> MODIFY LOCALLY -> PUSH TO UPSTREAM -> PROPOSE FOR APPROVAL -> APPROVE TO UPSTREAM REPO.
todo

<!-- SECOND SECTION ENDS -->
---
### [GET/LIST EXAMPLES] [FOR EACH: PORCHCTL EXAMPLE + KUBECTL ALTERNATIVE]
todo

    - [LIST ALL PACKAGE REVISIONS]

    - [GET SPECIFIC PACKAGE REVISION]

    - [LIST WITH FILTER]

    - [LABEL SELECTOR] --selector

    - [FIELD SELECTOR] --field-selector

WE WILL NEED TEST PACKAGES FOR THESE USE CASES (possibly leverage test packages from kpt examples)


## Cloning Porch Packages

### [CLONE EXAMPLES]


[cloning an upstream package] deploy without any changes

[cloning an upstream] with changes.



## Deleting Porch Packages
### [DELETE EXAMPLES]
todo

ADD & Delete type changes at both file level(lines in a file getting added and removed) and also package level(files getting added and removed)



## Copying Porch Packages

### [COPY EXAMPLES]
todo

detailed different examples here







## Upgrading Porch Packages

The package upgrade feature in Porch is a powerful mechanism for keeping deployed packages (downstream) up-to-date with their source blueprints (upstream). This guide walks through the entire workflow, from creating packages to performing an upgrade, with a special focus on the different upgrade scenarios and merge strategies.

## Part A: The Complete Upgrade Workflow

This example demonstrates the end-to-end process of creating, customizing, and upgrading a package.

### Step 1: Create a Base Blueprint Package (v1)

First, we create the initial version of our blueprint. This will be the "upstream" source for our deployment package.

```bash
# Initialize a new package draft named 'blueprint' in the 'porch-test' repository
porchctl rpkg init blueprint --namespace=porch-demo --repository=porch-test --workspace=1

# Propose the draft for review
porchctl rpkg propose porch-test.blueprint.1 --namespace=porch-demo

# Approve and publish the package, making it available as v1
porchctl rpkg approve porch-test.blueprint.1 --namespace=porch-demo
```

### Step 2: Create a New Blueprint Version (v2)

Next, we'll create a new version of the blueprint to simulate an update. In this case, we add a new ConfigMap.

```bash
# Create a new draft (v2) by copying v1
porchctl rpkg copy porch-test.blueprint.1 --namespace=porch-demo --workspace=2

# Pull the contents of the new draft locally to make changes
porchctl rpkg pull porch-test.blueprint.2 --namespace=porch-demo ./tmp/blueprint-v2

# Add a new resource file to the package
kubectl create configmap test-cm --dry-run=client -o yaml > ./tmp/blueprint-v2/new-configmap.yaml

# Push the local changes back to the Porch draft
porchctl rpkg push porch-test.blueprint.2 --namespace=porch-demo ./tmp/blueprint-v2

# Propose and approve the new version
porchctl rpkg propose porch-test.blueprint.2 --namespace=porch-demo
porchctl rpkg approve porch-test.blueprint.2 --namespace=porch-demo
```
At this point, we have two published blueprint versions: `v1` (the original) and `v2` (with the new ConfigMap).

### Step 3: Clone Blueprint v1 into a Deployment Package

Now, a user clones the blueprint to create a "downstream" deployment package.

```bash
# Clone blueprint v1 to create a new deployment package
porchctl rpkg clone porch-test.blueprint.1 --namespace=porch-demo --repository=porch-test --workspace=1 deployment

# Pull the new deployment package locally to apply customizations
porchctl rpkg pull porch-test.deployment.1 --namespace=porch-demo ./tmp/deployment-v1

# Apply a local customization (e.g., add an annotation to the Kptfile)
kpt fn eval --image gcr.io/kpt-fn/set-annotations:v0.1.4 ./tmp/deployment-v1/Kptfile -- kpt.dev/annotation=true

# Push the local changes back to Porch
porchctl rpkg push porch-test.deployment.1 --namespace=porch-demo ./tmp/deployment-v1

# Propose and approve the deployment package
porchctl rpkg propose porch-test.deployment.1 --namespace=porch-demo
porchctl rpkg approve porch-test.deployment.1 --namespace=porch-demo
```

### Step 4: Discover and Perform the Upgrade

Our deployment package is based on `blueprint.1`, but we know `blueprint.2` is available. We can discover and apply this upgrade.

```bash
# Discover available upgrades for packages cloned from 'upstream' repositories
porchctl rpkg upgrade --discover=upstream
# This will list 'porch-test.deployment.1' as having an available upgrade to revision 2.

# Upgrade the deployment package to revision 2 of its upstream blueprint
# This creates a new draft package: 'porch-test.deployment.2'
porchctl rpkg upgrade porch-test.deployment.1 --namespace=porch-demo --revision=2 --workspace=2

# Propose and approve the upgraded package
porchctl rpkg propose porch-test.deployment.2 --namespace=porch-demo
porchctl rpkg approve porch-test.deployment.2 --namespace=porch-demo
```

After approval, `porch-test.deployment.2` is the new, published deployment package. It now contains:
1.  The `new-configmap.yaml` from the upstream `blueprint.2`.
2.  The local `kpt.dev/annotation=true` customization applied in Step 3.



## Part B: Upgrade Scenarios & Merge Strategies

The outcome of an upgrade depends entirely on the changes made in the upstream blueprint and the local deployment package, combined with the chosen merge strategy.

Let's define the three states in a merge:
*   **Original:** The state of the package when it was first cloned (Blueprint v1).
*   **Upstream:** The new version of the blueprint (Blueprint v2).
*   **Local:** The current state of the deployment package, including any customizations.

### Default Strategy: `resource-merge`

This is a structural 3-way merge designed for Kubernetes resources. It understands the structure of YAML files and attempts to intelligently merge changes.

**When to use `resource-merge`:** This is the **recommended default strategy** for managing Kubernetes configuration. Use it when both blueprints and deployments contain standard Kubernetes resources and you want to preserve local customizations while incorporating upstream updates.

---

### Alternative Strategies

You can specify a different strategy using the `--strategy` flag, for example: `porchctl rpkg upgrade ... --strategy=copy-merge`.

#### Strategy: `copy-merge`

**Logic:** A file-level replacement strategy.
1.  Keeps any files that exist *only* in the local package.
2.  For any file present in both local and upstream, the **upstream version is used, overwriting local changes**.
3.  Adds any files that exist *only* in the new upstream package.
4.  Deletes any files from the local package that were removed in the new upstream package.


---

#### Strategy: `force-delete-replace`

**Logic:** The most aggressive strategy. It completely discards the local package's contents and replaces them with the contents of the new upstream package.

**When to use `force-delete-replace`:**
*   To completely reset a deployment package to a new blueprint version, abandoning all previous customizations.
*   When a package has become so heavily modified that resolving conflicts is more work than starting fresh from the new blueprint.

---

#### Strategy: `fast-forward`

**Logic:** A fail-fast safety check. The upgrade only succeeds if the local package has **zero modifications** compared to the original blueprint version it was cloned from.

**When to use `fast-forward`:**
*   When you want to guarantee that you are only upgrading unmodified packages. This prevents accidental overwrites of important local customizations.

---

## Part 3: Additional Information

### Command Flags

The `porchctl rpkg upgrade` command has several key flags:

*   `--workspace=<name>`: (Mandatory) The name for the new workspace where the upgraded package draft will be created.
*   `--revision=<number>`: (Optional) The specific revision number of the upstream package to upgrade to. If not specified, Porch will automatically use the latest published revision.
*   `--strategy=<strategy>`: (Optional) The merge strategy to use. Defaults to `resource-merge`. Options are `resource-merge`, `copy-merge`, `force-delete-replace`, `fast-forward`.

For more details, run `porchctl rpkg upgrade --help`.

### Best Practices

*   **Separate Repositories:** For better organization and access control, keep blueprint packages and deployment packages in separate Git repositories.
*   **Understand Your Strategy:** Before upgrading, be certain which merge strategy fits your use case to avoid accidentally losing important local customizations. When in doubt, the default `resource-merge` is the safest and most intelligent option.

### Cleanup

To remove the packages created in this guide, you must first propose them for deletion and then perform the final deletion.

```bash
# Clean up local temporary directory
rm -rf ./tmp

# Propose all packages for deletion
porchctl rpkg propose-delete porch-test.blueprint.1 porch-test.blueprint.2 porch-test.deployment.1 porch-test.deployment.2 --namespace=porch-demo

# Delete the packages
porchctl rpkg delete porch-test.blueprint.1 porch-test.blueprint.2 porch-test.deployment.1 porch-test.deployment.2 --namespace=porch-demo
```
<!-- THIRD SECTION ENDS -->
---