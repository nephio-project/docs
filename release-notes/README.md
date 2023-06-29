# Nephio R1 Release Notes

## Overview

## Prerequisites

Refer to [install
guide](https://github.com/nephio-project/docs/blob/main/install-guide/README.md)
for the prerequisites on supported environments.

## Support Matrix

The sandbox environment requires a physical or virtual machine with:
- Linux Flavour: Ubuntu-20.04-focal
- 8 cores
- 32 GB memory
- 200 GB disk size
- default user with sudo passwordless permissions

This install has been verified on VMs running on Google Cloud, OpenStack, AWS,
vSphere, and Azure. It has been verified on Vagrant VMs running on Windows and
Linux.

For non-sandbox installations, any conforming Kubernetes cluster is sufficient
for the management cluster.

## Features

### API

CRDs provided for UPF, SMF and AMF 5g core services

### Web UI

Basic web UI to view and manage packages and the resources within them.

### Packages

* Kpt packages for all [free5gc](https://free5gc.org/) services
* Packages for core Nephio services
* Packages for cluster API services for cluster creation
* Packages for dependent services

### Functionalities

* Create Kubernetes clusters. This functionality is based on cluster API. At
  this time only KIND clusters creation is supported.
* Fully automated deployment of UPF, SMF and AMF services of
  [free5Gc](https://free5gc.org/) . These are deployed on multiple clusters
  based on user's intent expressed via CRDs.
* Deployment of other free5gc functions.
* Auto-scale up of UPF, SMF and AMF services based on changes to capacity
  requirements expressed as user intent.

## Limitations

* In terms of infrastructure automation, only creation of KIND clusters is
  supported.
* Inter cluster networking is not dynamic which means as more clusters are
  deployed some manual tweak will be needed for inter cluster communications.
* Provisioning of VLAN interfaces on nodes is manual at this time.
* Feedback of workload deployments from workload clusters to the management
  cluster is limited. You may need to directly connect to the workload cluster
  using kubectl to debug the deployment issues.
* Web UI features are limited to view/edit of packages and resources in those
  packages, and the deployment of those packages. More features will be added
  in subsequent releases.
* When the capacity of UPF,SMF and AMF is changed, the free5gc operator on the
  workload cluster will instantiate a new POD with correspondingly modified
  resources (CPU, memory etc.) During this pods will restart. This is the
  limitation of free5gc.
* Only Gitea works with automated cluster provisioning to create new
  repositories and join them to Nephio. To use a different Git provider, you
  must manually provision cluster repositories, register them to the Nephio
  management server, and set up Config Sync on the workload cluster.
* The WebUI does not require authentication in the current demo configuration.
  Testing of the WebUI with authentication configured has not been done at this
  time.
* The WebUI only shows resources in the default namespace.
* While many types of Git authentication are supported, the testing was only
  done with token-based Git authentication in Gitea.

## Known Issues and Workarounds

* In case of deploying sandbox environment on ubuntu VM running on openstack,
  the deployment may fail. Reinstall the packages to get around this issue.
* Occasionally packages may take a long time to be approved by the auto-approval
  controller.
* Occasionally calls to `kpt alpha rpkg copy` may fail with a message like
  `Error: Internal error occurred: error applying patch: conflict: fragment line
  does not match src line`. Try again in a little while, this may clear up on
  its own.
