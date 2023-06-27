# Nephio R1 Release Notes

## Overview
Nephio is a Kubernetes-based intent-driven automation of network functions and the underlying infrastructure that supports those functions. It allows users to express high-level intent, and provides intelligent, declarative automation that can set up the cloud and edge infrastructure, render initial configurations for the network functions, and then deliver those configurations to the right clusters to get the network up and running.

Technologies like distributed cloud enable on-demand, API-driven access to the edge. Unfortunately, existing brittle, imperative, fire-and-forget orchestration methods struggle to take full advantage of the dynamic capabilities of these new infrastructure platforms. To succeed at this, Nephio uses new approaches that can handle the complexity of provisioning and managing a multi-vendor, multi-site deployment of interconnected network functions across on-demand distributed cloud.

The solution is intended to address the initial provisioning of the network functions and the underlying cloud infrastructure, and also provide Kubernetes-enabled reconciliation to ensure the network stays up through failures, scaling events, and changes to the distributed cloud. 

Nephio leverages "configuration as data" principle and Kubernetes declarative, actively-reconciled methodology along with machine-manipulable configuration to tame the complexity of Network Functions deployment and life-cycle management..

This release of Nephio focuses on:
* Exhibiting the core Nephio principles such as Configuration as data and leveraging the intent driver, actively reconciled nature of kubernetes.
*  Infrastructure orchestration/automation using controllers based on  cluster API. At this time only KIND cluster creation is supported.
* Orchestration/automation of 5G core network functions deployment and management. This release focuses on network functions from free5gc. 

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


