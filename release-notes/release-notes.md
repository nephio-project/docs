# Nephio R1 Release Notes

## Overview
## Prerequisites

Refer to install guide (link here) for the prerequisites on supported environments. 


## Support Matrix
* Supported Platforms / Operating Systems
  * For sandbox installations Ubuntu 22.x running on :
    * Bare metal
    * Vsphere Version ?
    * Openstack version ?
    * Vagrant on Virtual box on Windows 10/11.
* Supported cloud environments
  * Google Cloud Platform
* **Other k8s systems?**

## Features
### API
CRDs provided for UPF, SMF and AMF 5g core services
### Web UI
Basic web UI to view and manage Package Variants and Package variant sets.
### Packages
* Kpt packages for all [free5gc](https://free5gc.org/) services
* Packages for core Nephio services
* Packages for cluster API services for cluster creation
* Packages for dependent services
### Functionalities
* Create kubernetes clusters. This functionality ia based on cluster API. At this time only KIND clusters creation is supported.
* Fully automated deployment of UPF, SMF and AMF services of [free5Gc](https://free5gc.org/) . These are deployed on multiple clusters based on user's intent expressed via CRDs.
* Inter cluster networking setup.
* Deployment of other free5gc functions. Some manual configuration such as IP addresses may be needed for these services.
* Auto-scale up of UPF, SMF and AMF services based on changes to capacity requirements expressed as user intent.

## Limitations
* In terms of infrastructure automation, only creation of KIND clusters is supported.
* Deployment of  free5gc functions other than SMF, UPF and AMF may need some manual configuration such as IP addresses.
* Inter cluster networking is not dynamic which means as more clusters are deployed some manual tweak will be needed for inter cluster communications. 
* Feedback of workload deployments from workload clusters to the management cluster is limited. You may need to directly connect to the workload cluster using kubectl to debug the deployment issues. 
* Web UI features are limited to view/edit of Package Variants and Package variant sets. More features will be added in subsequent releases. 
* When the capacity of UPF,SMF and AMF is changed, the free5gc operator on the workload cluster will instantiate a new POD with correspondingly modified resources (CPU, memory etc.) During this pods will restart. This is the limitation of free5gc.

## Known Issues and Workarounds
* In case of deploying sandbox environment on ubuntu VM running on openstack, the deployment may fail. Reinstall the packages to get around this issue. ( **More details needed here**).
* End-to-end call issues and workarounds. (**More details needed here**)
* **Others???**


