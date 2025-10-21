---
title: "Upgrading Packages"
type: docs
weight: 6
description: "A guide to upgrade package revisions using Porch and porchctl"
---

# Upgrading Porch Packages

The package upgrade feature in Porch is a powerful mechanism for keeping deployed packages (downstream) up-to-date with their source blueprints (upstream). This guide walks through the entire workflow, from creating packages to performing an upgrade, with a special focus on the different upgrade scenarios and merge strategies.

## Table of Contents

- [Key Concepts](#key-concepts)
- [End-to-End Upgrade Example](#end-to-end-upgrade-example)
- [Understanding Merge Strategies](#understanding-merge-strategies)
- [Reference](#reference)

## Key Concepts

To understand the upgrade process, it's essential to be familiar with the three states of a package during a merge operation:

*   **Original:** The state of the package when it was first cloned from the blueprint (e.g., `Blueprint v1`). This serves as the common ancestor for the merge.
*   **Upstream:** The new, updated version of the source blueprint (e.g., `Blueprint v2`). This contains the changes you want to incorporate.
*   **Local:** The current state of your deployment package, including any customizations you have applied since it was cloned.

The upgrade process combines changes from the **Upstream** blueprint with your **Local** customizations, using the **Original** version as a base to resolve differences.

## End-to-End Upgrade Example

This example demonstrates the complete process of creating, customizing, and upgrading a package.

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

## Understanding Merge Strategies

The outcome of an upgrade depends on the changes made in the upstream blueprint and the local deployment package, combined with the chosen merge strategy. You can specify a strategy using the `--strategy` flag (e.g., `porchctl rpkg upgrade ... --strategy=copy-merge`).

### Merge Strategy Comparison

| Scenario                               | `resource-merge` (Default)                               | `copy-merge`                                             | `force-delete-replace`                                   | `fast-forward`                                               |
| -------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| **File added in Upstream**             | File is added to Local.                                  | File is added to Local.                                  | File is added to Local.                                  | Fails (Local must be unchanged).                             |
| **File modified in Upstream only**     | Changes are applied to Local.                            | Upstream file overwrites Local file.                     | Upstream file overwrites Local file.                     | Fails (Local must be unchanged).                             |
| **File modified in Local only**        | Local changes are kept.                                  | Local changes are kept.                                  | Local changes are discarded; Upstream version is used.   | Fails (Local must be unchanged).                             |
| **File modified in both** (no conflict) | Both changes are merged.                                 | Upstream file overwrites Local file.                     | Upstream file overwrites Local file.                     | Fails (Local must be unchanged).                             |
| **File modified in both** (conflict)   | Merge autoconflic resolution: always choose the new upstream version.                 | Upstream file overwrites Local file.                     | Upstream file overwrites Local file.                     | Fails (Local must be unchanged).                             |
| **File deleted in Upstream**           | File is deleted from Local.                              | File is deleted from Local.                              | File is deleted from Local.                              | Fails (Local must be unchanged).                             |
| **Local package is unmodified**        | Upgrade succeeds.                                        | Upgrade succeeds.                                        | Upgrade succeeds.                                        | Upgrade succeeds.                                            |

### Detailed Strategy Explanations

#### `resource-merge` (Default)
This is a structural 3-way merge designed for Kubernetes resources. It understands the structure of YAML files and attempts to intelligently merge changes from the upstream and local packages.

*   **When to use:** This is the **recommended default strategy** for managing Kubernetes configuration. Use it when you want to preserve local customizations while incorporating upstream updates.

#### `copy-merge`
A file-level replacement strategy. For any file present in both local and upstream, the upstream version is used, overwriting local changes. Files that only exist locally are kept.

*   **When to use:** When you trust the upstream source more than local changes or when Porch cannot parse the file contents (e.g., non-KRM files).

#### `force-delete-replace`
The most aggressive strategy. It completely discards the local package's contents and replaces them with the contents of the new upstream package.

*   **When to use:** To completely reset a deployment package to a new blueprint version, abandoning all previous customizations.

#### `fast-forward`
A fail-fast safety check. The upgrade only succeeds if the local package has **zero modifications** compared to the original blueprint version it was cloned from.

*   **When to use:** To guarantee that you are only upgrading unmodified packages, preventing accidental overwrites of important local customizations.

## Reference

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
# Clean up local temporary directory used in these examples
rm -rf ./tmp

# Propose all packages for deletion
porchctl rpkg propose-delete porch-test.blueprint.1 porch-test.blueprint.2 porch-test.deployment.1 porch-test.deployment.2 --namespace=porch-demo

# Delete the packages
porchctl rpkg delete porch-test.blueprint.1 porch-test.blueprint.2 porch-test.deployment.1 porch-test.deployment.2 --namespace=porch-demo
```
