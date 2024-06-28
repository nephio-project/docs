---
title: O-RAN Integration
description: >
  Using Nephio enablers to realize O-RAN functionality
weight: 5
---

## Overview

The O-RAN Software Community (OSC) and Open Air Interface (OAI) and Nephio open source communities are working together to provide a reference implementation of the Open RAN (O-RAN) Alliance’s cloud-centric specifications using the Nephio enablers and capabilities in order to deploy and manage O-RAN NFs and xApps.
The focus of the O-RAN Integration within Nephio focuses on the Federated O-Cloud Orchestration and Management (FOCOM), Network Function Orchestration (NFO), Infrastructure Management (IMS) and Deployment Management (DMS) O-RAN services as depicted in the following figure:

![deployment-architecture.png](/static/images/network-architecture/o-ran/deployment-architecture.png)

These services can be categorized into:
-Infrastructure services for the lifecycle management of Cloud Resources (FOCOM, IMS): These services are deployed and utilizes the Nephio capabilities provided by Nephio Management Clusters. In some deployed architectures aspects of the IMS utilizes Nephio capabilities provided by Nephio Workload Clusters.
- Lifecycle management of deployments for O-RAN NFs (NFO, DMS): The NFO service is deployed and utilizes the Nephio capabilities provided by Nephio Management Clusters. The DMS is deployed and utilizes the Nephio capabilities provided by Nephio Workload Clusters.

## Infrastructure services for the lifecycle management of Cloud Resources

### Introduction

The primary role of the FOCOM and IMS services is to provide for the lifecycle management of the resources exposed by an O-Cloud. These services have identified the following O-Cloud capabilities that are planned to be developed using Nephio capabilities:

- O-Cloud registration
- O-Cloud Cluster Lifecycle Management
- O-Cloud Resource Inventory

{{% alert title="Note" color="primary" %}}

In R3 Nephio use cases and component architectures have been defined for O-Cloud Cluster Lifecycle Management
As stated, the role of the FOCOM function is to provide federated orchestration and management across multiple O-Clouds using the O2ims interface between the O-Clouds as shown in the figure below:

![focom-ims.png](/static/images/network-architecture/o-ran/focom-ims.png)

Due to the standardized O2ims O-RAN interface, the functions that implement the FOCOM service can interact with O-Clouds IMSs that utilize the capabilities provided by Nephio or O-Clouds that use other non-Nephio capabilities and vice-versa.

{{% /alert %}}

### O-Cloud Cluster Lifecycle Management

#### Introduction

O-Cloud Clusters are the primary method of allocating cloud resources (e.g., O-Cloud Nodes, Networks) to be used during deployments of O-RAN NFs. O-Cloud Cluster Lifecycle Management services implemented using Nephio capabilities are described in the following use cases:

- Create O-Cloud K8s Cluster
- Delete O-Cloud K8s Cluster
- Upgrade O-Cloud K8s Cluster

{{% alert title="Note" color="primary" %}}

In R3 Nephio use cases and component architectures have been defined for the Create O-Cloud K8s Cluster use case.

{{% /alert %}}

The O-RAN WG6 O2ims Provisioning working group has now introduced the concept of an “O-Cloud cluster template” in O-RAN.WG6.O2-GA&P-R003-v06.00.docx. The intention is to abstract the O-Cloud implementation specific artifacts and HW configuration from the SMO/FOCOM layer and just expose a subset of the O-Cloud cluster template, describing high-level workload cluster characteristics, capacity, and additional metadata, that is sufficient for the SMO to be able to decide which O-Cloud cluster template that shall be used based on the CNF workload characteristics and capacity requirements. The O-Cloud cluster template also contain HW Compute Profile/Resource Type characteristics requirements that must be fulfilled by the O-Cloud Site Resources and the SMO will use the information in the cluster template in order to match towards available HW inventory resources exposed over the O2ims inventory API so that it can deduct whether there is available HW resource capacity to create a new cluster based on a certain O-Cloud cluster template in a specific target O-Cloud Site.

Within FOCOM a corresponding SMO-level cluster template record is kept. The role of the FOCOM is to support integration towards multiple O-Clouds, each with its own catalog of supported O-Cloud cluster templates that contain a reference to the supporting O-Cloud IMS end point, the unique id of the O-Cloud cluster template on the O-Cloud side as well as the characteristics/capacity and metadata information about the cluster template.

The expectation is that each O-RAN CNF vendor jointly defines O-Cloud cluster template blueprint for each O-Cloud infrastructure vendor that they plan to support, due to the need for specific tailoring of the cluster worker nodes to match the HW configuration requirements (bios settings, huge pages, SRIOV NIC configuration etc.) from specific O-RAN CNFs. These O-Cloud cluster template blueprints act as a reference design to be used by each operator running their specific O-Cloud and are expected to be tailored further for each operator. The resulting O-Cloud cluster templates are then discovered by the SMO/FOCOM component as part of the O-Cloud registration process.

#### High-Level design of FOCOM using Nephio

A FOCOM function that uses Nephio enablers uses a KPT-based cluster package management solution where:

- The “Cluster template list” on the SMO is realized by a Git based cluster template repository where the KPT cluster packages are onboarded
- The SMO O-Cloud cluster templates refer one-to-one to a corresponding IMS-side O-Cloud cluster template in a specific O-Cloud IMS

FOCOM uses a concept of provider plugins to be used when communicating with an O-Cloud IMS. One identified option is for FOCOM to use the Cluster API (CAPI) framework to develop an O2ims provider plugin. An alternative option is to develop a native FOCOM O2ims specific operator without the use fo the CAPI framework. The O2ims CAPI manifest files will be part of the SMO cluster template package.

The figure below depicts one implementation for the high-level design of the FOCOM:

![focom.png](/static/images/network-architecture/o-ran/focom.png)

#### Create O-Cloud K8s Cluster

The creation of an O-Cloud K8s Cluster is defined by an O2ims specific “Cluster Request” CR that is integrated with the Nephio “ClusterClaim” CR as depicted in the figure below:

![ims-provisioning-create-cluster.png](/static/images/network-architecture/o-ran/ims-provisioning-create-cluster.png)

The Nephio ClusterClaim CR:

- Models the request to create a cluster with a certain configuration while the current WorkloadCluster CR is used to model successfully deployed clusters and used as a target for NF Deployment
- Has a parameterRef to a Nephio-defined O2imsClusterParameters CR
- The o2imsClusterParameter CR has configRefs to both the O2ims standard input data as well as O-Cloud cluster template specific configuration data.

The O-Cloud Cluster Template:

- Supports installation of add-on features such as Multus networking that will require specific configuration handled through the configRefs CRDs. A configRef CR can contain both configuration that is fixed for the O-Cloud Cluster template as well as instance specific configuration that must be provided as user input.
- Is realized with a KPT package that contains the ClusterClaim CR manifest as well as the referred O2imsClusterParameters CR manifest and additional configuration data manifests

As of this release, the O-RAN Alliance has not specified O2ims provisioning interface, as such this pre-standardization version of the O2ims provisioning interface is KRM/CRD based where the:

- Kubernetes API server is used to implement the O2ims provisioning interface.
- Cluster deployment operation is performed by applying an O2ims “ClusterRequest” CR in the IMS management cluster
- O2ims ClusterRequest CR contains the reference to 

  - the O-Cloud cluster template
  - target O-Cloud Site
  - instance specific input data (will be further defined, for R3 there isn’t any)

- IMS management cluster has an “O2ims operator” that picks up the ClusterRequest CR and triggers Porch cloning of the referred O-cloud cluster template/Nephio workload cluster package as a new “Cluster1” workload cluster package deployment in the Cluster management repository.

## Lifecycle management of deployments for O-RAN NFs

### Introduction

The primary role of the NFO and DMS services is to provide for the lifecycle management of the cloud deployments for O-RAN NFs. These services have identified the following O-Cloud capabilities that are planned to be developed using Nephio capabilities:

- NF Deployment Lifecycle Management

#### NF Deployment Lifecycle Management

##### Introduction

In O-RAN, a cloudified O-RAN NF (e.g., O-DU) is deployed and managed in an O-Cloud using one or more NF Deployments. While the NFO’s focus is the lifecycle management of the deployed O-RAN NF, the DMS’s focus is the NF Deployment which is also the capabilities provided by Nephio. 
NF Deployment Lifecycle Management services implemented using Nephio capabilities are described in the following use cases:

- Deploy O-RAN NF
- Terminate O-RAN NF Deployments

{{% alert title="Note" color="primary" %}}

In R3 Nephio use cases and component architectures have been defined for the Deploy O-RAN NF use case.

{{% /alert %}}

##### Deploy O-RAN NF

The role of the NFO within the SMO is to handle functionality related to the lifecycle management for deployment of O-RAN Cloudified NFs. The deployment of the O-RAN NF includes software components realizing all or part of an NF which could be an O-DU, O-CU-CP, O-CU-UP, xApps and the Near-RT RIC components in the O-Cloud sites.
This use case describes the high-level steps and requirements for deployment of O-RAN NF that is composed as discrete NF Deployments across the O2dms using the K8s profile.

{{% alert title="Note" color="primary" %}}

In R3 the O-RAN Cloudified NFs consist of a single NF Deployment.

{{% /alert %}}

##### Sequence Chart

The following sequence chart depicts the flow between the NFO and Workload Cluster’s DMS, highlighting the use of the Nephio enablers to deploy the NF Deployment from the NFO to the DMS.

![nf-orch-smo.png](/static/images/network-architecture/o-ran/nf-orch-smo.png)
