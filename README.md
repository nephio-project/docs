# Nephio R1

## Introduction

Welcome to the R1 release of Nephio.  Nephio’s mission is "to deliver carrier-grade, simple, open, Kubernetes-based cloud native intent automation and common automation templates that materially simplify the deployment and management of multi-vendor cloud infrastructure and network functions across large scale edge deployments." But what does that mean? With this release and the accompanying documentation, we hope to make that clear.

To do that, let's step back a little and consider the problem Nephio is trying to solve for a communications service provider (CSP). 

**** Grab the blog here ****


Nephio is about managing complex, inter-related workloads at scale. That *scale* can be across many different dimensions: number of sites, number of developers, number of workloads, size of the individual workloads, complexity of the organization, and other factors.

To manage these challenges, Nephio follows a few basic principles
Nephio is a Kubernetes-based intent-driven automation of network functions and the underlying infrastructure that supports those functions. It allows users to express high-level intent, and provides intelligent, declarative automation that can set up the cloud and edge infrastructure, render initial configurations for the network functions, and then deliver those configurations to the right clusters to get the network up and running.

### Why Now?

Technologies like distributed cloud enable on-demand, API-driven access to the edge. Unfortunately, existing brittle, imperative, fire-and-forget orchestration methods struggle to take full advantage of the dynamic capabilities of these new infrastructure platforms. To succeed at this, Nephio uses new approaches that can handle the complexity of provisioning and managing a multi-vendor, multi-site deployment of interconnected network functions across on-demand distributed cloud.

The solution is intended to address the initial provisioning of the network functions and the underlying cloud infrastructure, and also provide Kubernetes-enabled reconciliation to ensure the network stays up through failures, scaling events, and changes to the distributed cloud. 

Nephio leverages "configuration as data" principle and Kubernetes declarative, actively-reconciled methodology along with machine-manipulable configuration to tame the complexity of Network Functions deployment and life-cycle management..

This release of Nephio focuses on:
* Exhibiting the core Nephio principles such as Configuration as data and leveraging the intent driver, actively reconciled nature of kubernetes.
*  Infrastructure orchestration/automation using controllers based on  cluster API. At this time only KIND cluster creation is supported.
* Orchestration/automation of 5G core network functions deployment and management. This release focuses on network functions from free5gc. 

## How

From the very high-level, an intent-based system only does two things:
- Enables the user to specify their intent ("I want...")
- Ensures that the intent is realized at all times ("Make it so, and keep it that way")

To address "specifying intent", we need a language to unambigously describe our intent. In Nephio, we have chosen the Kubernetes Resource Model (KRM) as our basic language for specifying intent. KRM models everything as a "resource", and every resource has some basic properties like a name and user-defined labels. Additionally, most resources include two fields specifically to help with managing intent: `Spec` and `Status`. The spec (short for "specification") describes the intent for that resource, whereas the status is the last known state of the resource. It is the job of the system to reconcile the difference between the two ("make it so").

## User Documentation
* [Core Concepts](https://github.com/nephio-project/docs/blob/main/concepts.md)
* [Demo Sandbox Environment Installation](https://github.com/nephio-project/docs/blob/main/install-guide/README.md)
* [Quick Start Exercises](https://github.com/nephio-project/docs/blob/main/user-guide/README.md)
* [User Guide](https://github.com/nephio-project/docs/blob/main/user-guide/README.md)

## Other Documentation

* [Developer Documentation](https://github.com/nephio-project/nephio)
* [Project Resources](https://github.com/nephio-project/docs/blob/main/resources.md)
