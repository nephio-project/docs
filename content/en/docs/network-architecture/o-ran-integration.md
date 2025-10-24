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
- Infrastructure services for the lifecycle management of Cloud Resources (FOCOM, IMS): These services are deployed and utilizes the Nephio capabilities provided by Nephio Management Clusters. In some deployed architectures aspects of the IMS utilizes Nephio capabilities provided by Nephio Workload Clusters.
- Lifecycle management of deployments for O-RAN NFs (NFO, DMS): The NFO service is deployed and utilizes the Nephio capabilities provided by Nephio Management Clusters. The DMS is deployed and utilizes the Nephio capabilities provided by Nephio Workload Clusters.

## Infrastructure services for the lifecycle management of Cloud Resources

### Introduction

The primary role of the FOCOM and IMS services is to provide for the lifecycle management of the resources exposed by an O-Cloud. These services have identified the following O-Cloud capabilities that are planned to be developed using Nephio capabilities:

- O-Cloud registration
- O-Cloud Cluster Lifecycle Management
- O-Cloud Resource Inventory

{{% alert title="Note" color="primary" %}}

In R4 the Nephio implementation supports the O-Cloud Node Cluster creation as part of the-Cloud Cluster Lifecycle Management service.

{{% /alert %}}

As stated, the role of the FOCOM service is to provide federated orchestration and management across multiple O-Clouds using the O2ims interface between the O-Clouds as shown in the figure below:

![focom-ims.png](/static/images/network-architecture/o-ran/focom-ims.png)

Due to the standardized O2ims O-RAN interface, the functions that implement the FOCOM service can interact with O-Clouds IMSs that utilize the capabilities provided by Nephio or O-Clouds that use other non-Nephio capabilities and vice-versa.

### O-Cloud Cluster Lifecycle Management

O-Cloud Clusters are the primary method of allocating cloud resources (e.g., O-Cloud Nodes, Networks) to be used during deployments of O-RAN NFs. O-Cloud Cluster Lifecycle Management services implemented using Nephio capabilities are described in the following use cases:

- Create O-Cloud K8s Cluster
- Delete O-Cloud K8s Cluster
- Upgrade O-Cloud K8s Cluster

{{% alert title="Note" color="primary" %}}

In R4 the Nephio implementation supports the O-Cloud Node Cluster creation use case.

{{% /alert %}}

The O-RAN WG6 O2ims Provisioning working group has now introduced the concept of a template for O-Cloud Node Cluster and Infrastructure deployment in O-RAN.WG6.O2-GA&P-R003-v06.00.docx. The intention is to abstract the O-Cloud implementation specific artifacts and HW configuration from the SMO/FOCOM layer and just expose a subset of the O-Cloud Template, describing high-level workload cluster characteristics, capacity, and additional metadata, that is sufficient for the SMO to be able to decide which O-Cloud Template that shall be used based on the CNF workload characteristics and capacity requirements. The O-Cloud Template also contain HW Compute Profile/Resource Type characteristics requirements that must be fulfilled by the O-Cloud Site Resources and the SMO will use the information in the O-Cloud Template in order to match towards available HW inventory resources exposed over the O2ims inventory API so that it can deduct whether there is available HW resource capacity to create a new cluster based on a certain O-Cloud Template in a specific target O-Cloud Site.

Within FOCOM a corresponding SMO-level O-Cloud Template information record is kept. The role of the FOCOM is to support integration towards multiple O-Clouds, each with its own catalog of supported O-Cloud Templates. The FOCOM O-Cloud Template information record contains a reference to the supporting O-Cloud IMS end point, the name and version of the O-Cloud Template on the O-Cloud side, a schema for the instance specific input data accepted by the O-Cloud Template, as well as the characteristics/capacity and metadata information about the O-Cloud Template.

The expectation is that each O-RAN CNF vendor defines O-Cloud Templates for each O-Cloud infrastructure vendor that they plan to support, due to the need for specific tailoring of the cluster worker nodes to match the HW configuration requirements (bios settings, huge pages, SR-IOV NIC configuration etc.) from specific O-RAN CNFs. These O-Cloud Templates act as a reference design to be used by each operator running their specific O-Cloud and are expected to be tailored further for each operator. The exposed information about the O-Cloud Templates are then discovered by the SMO/FOCOM component as part of the O-Cloud registration process.

### High-Level design of SMO FOCOM using Nephio in R4

A FOCOM service implementation based on Nephio enablers uses a Kpt-based cluster package management solution where:

- The stored O-Cloud Template information in FOCOM is realized by a Git based cluster template repository where each O-Cloud Template information record is realized with a Kpt package blueprint
- Each FOCOM O-Cloud Template Kpt package blueprint refer one-to-one to a corresponding IMS-side O-Cloud Template in a specific O-Cloud IMS

The Nephio based FOCOM implementation will use the Porch NBI for deployment of O-Cloud Node Clusters based on the O-Cloud Template. To trigger creation of a new Node Cluster, the applicable O-Cloud Template Kpt package blueprint will be cloned into a new draft for a Node Cluster Provisioning Request instance package. Every O-Cloud Template Kpt package blueprint will contain two manifest files, one with the O-Cloud Template information and one for a FOCOM Provisioning Request that will carry the instance specific input and be used by FOCOM to generate the O2ims Provisioning Request. Once the draft Provisioning Request instance package has been created the client shall update the FOCOM Provisioning Request manifest file inside the package to add the instance specific input. Finally the client will propose and approve the Provisioning Request instance package and this will trigger FOCOM to start the reconciliation process for the O2ims Provisioning Request towards the O-Cloud IMS.

Each O-Cloud Template Kpt package blueprint will contain a reference to the id of the O-Cloud where this O-Cloud template is supported. Each O-Cloud has been previously registered in FOCOM, in the Nephio R4 implementation this is supported with a separate O-Cloud Registration CR that is manually created in the FOCOM Nephio management cluster. In coming Nephio releases this will be done as part of the O-Cloud registration user story.

![focom1.png](/static/images/network-architecture/o-ran/focom1.png)

The FOCOM Provisioning Request manifest file is created as a CR in the FOCOM Nephio management cluster and this will trigger a FOCOM O2ims operator to start the reconciliation towards the O2ims interface exposed by the IMS. The FOCOM O2ims operator will lookup the applicable O-Cloud registration CR to get the end point and credentials information for connection to the IMS. Then the input in the FOCOM Provisioning Request CR will be used to generate the O2ims Provisioning Request and submit it to the IMS.

![focom2.png](/static/images/network-architecture/o-ran/focom2.png)

### High level design of O-Cloud IMS using Nephio in R4

The support for the O2ims Provisioning interface is realized by an IMS using Nephio enablers through a Provisioning Request CRD over the Kubernetes API.
For Nephio R4 the existing “nephio-workload-cluster” blueprint is used as the O-Cloud Template so the O2ims Provisioning Request sent from FOCOM shall include a reference to the nephio-workload-cluster as the template. 

The Nephio based IMS implementation supports an IMS-side O2ims operator that will be triggered by the O2ims Provisioning Request CR instance and use the information provided in order to trigger the deployment of the nephio-workload-cluster blueprint through the Nephio Porch interface.

![ims-provisioning-create-cluster.png](/static/images/network-architecture/o-ran/ims-provisioning-create-cluster.png)

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

#### Demo User Guide

For a detailed demo user guide, see 
[O-RAN O-Cloud K8s Cluster deployment]({{< relref "/docs/guides/user-guides/usecase-user-guides/exercise-4-ocloud-cluster-prov.md" >}})