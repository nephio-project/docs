---
title: "Troubleshooting Guide"
description: "Solutions to common problems and known issues when installing and using Nephio."
weight: 100
---

This guide provides solutions to common issues encountered during the installation and operation of Nephio.

## Package Management & Deployment Issues

### Packages Fail to Become Approved

**Symptom:** After deploying a package, it remains in a *Proposed* state for a long time and is not automatically approved by the controller.

**Solution:** This can sometimes happen if the controllers become unresponsive. You can restart the Porch and Nephio controllers to resolve the issue. This is a safe operation and will cause the controllers to re-evaluate the current state.

```bash
kubectl -n porch-system rollout restart deploy porch-server
kubectl -n nephio-system rollout restart deploy nephio-controller
```

### `porchctl rpkg copy` Command Fails

**Symptom:** Running the `porchctl rpkg copy` command occasionally fails with an error message similar to: `Error: Internal error occurred: error applying patch: conflict: fragment line does not match src line`.

**Solution:** This is often a transient timing issue within the Porch server. Waiting a few moments and retrying the command usually resolves the error. If the problem persists, restarting the Porch server (as shown in the previous solution) can also help.

### Duplicate `parameterRef` Extensions During Specialization

**Symptom:** During the package specialization process, duplicate *parameterRef* extensions are created, leading to incorrect configurations and failed deployments.

**Solution:** This issue can be caused by the NFDeploy reconciliation not being idempotent. The current workaround is to redeploy the package. This will typically clear the invalid state and allow the specialization to proceed correctly.

### free5GC Operator Creates Duplicate Entries

**Symptom:** The free5GC operator may create duplicate entries in the SMF (Session Management Function) configuration.

**Solution:** This is a known issue within the operator itself. While it creates redundant configuration entries, it does not typically cause harm to the functionality of the network function. This can be safely ignored, though it may be addressed in future versions of the operator.

## Installation & Environment Issues

### Sandbox Deployment Fails on OpenStack

**Symptom:** When deploying the Nephio sandbox environment on an Ubuntu VM running on OpenStack, the installation may fail intermittently.

**Solution:** This has been observed as an occasional environmental issue. Re-running the installation script or reinstalling the packages will typically resolve the problem and lead to a successful deployment.
