---
title: About Nephio
linkTitle: Docs
menu: {main: {weight: 20}}
weight: 1
---

Our mission is "to deliver carrier-grade, simple, open, Kubernetes-based
cloud-native intent automation and common automation templates that materially
simplify the deployment and management of multivendor cloud infrastructure and
network functions across large-scale edge deployments." But what does that mean?
With this release and the accompanying documentation, we hope to make that
clear.

Our mission outlines the basic goals. The [About Nephio
page](https://nephio.org/about/) describes the high-level architecture of
Nephio. It is important to understand that Nephio is about managing complex,
interrelated workloads *at scale*. If we simply want to deploy a network
function, then existing methods, such as Helm charts and scripts, are
sufficient. Similarly, if we want to deploy some infrastructure, then using
existing Infrastructure-as-Code tools can accomplish that. Configuring running
network functions can already be done today with element managers.

Why do we need Nephio? The problems Nephio aims to solve start only
once we try to operate at scale. "Scale", in this case, does not simply mean
"a large number of sites". "Scale" can encompass many different areas: the
number of sites, services, and workloads, the size of each individual workload,
the number of machines needed to operate the workloads, the complexity of the
organization running the workloads, and other factors. The fact that our
infrastructure, workloads, and workload configurations are all interconnected
greatly increases the difficulty in managing these architectures at scale.

To address these challenges, Nephio follows a [few basic
principles](https://cloud.google.com/blog/topics/telecommunications/network-automation-csps-linus-nephio-cloud-native)
that experience has shown to enable higher scaling with fewer management
overheads. These principles are as follows:
- It is *Intent-driven* to enable the user to specify what they want and
  let the system figure out how to make that happen.
- It has *Distributed actuation* to increase reliability across widely
  distributed fleets.
- It has *Uniformity in systems* to reduce redundant tooling and processes,
  and simplify operations.

Nephio also leverages the "configuration as data" principle. This
methodology means that the "intent" is captured in a machine-manageable format
that we can treat as data, rather than code. In Nephio, we use the Kubernetes
Resource Model (KRM) to capture the intent. As Kubernetes itself is already an
intent-driven system, this model is well suited to our needs.

To understand why we need to treat configuration as data, let us consider an
example. In a given instance, a network function may have, for example, 100
parameters that need to be decided upon. 100 such network functions, across
10,000 clusters, results in 100,000,000 inputs that need to be defined and
managed. Handling such a large number of values, with their interdependencies
and a need for consistency management between them, requires *data management*
techniques, rather than *code* management techniques. This is why existing
methodologies begin to break down at scale, particular at the edge-level scale.

It should also be considered that no individual human will be able to understand
all of these values. These values relate not only to the workloads, but also to
the infrastructure that is required to support the workloads. They require
different areas of expertise and different organizational boundaries of control.
For example, you will need input from network planning (IP address, VLAN tags,
ASNs, and so on), input from the compute infrastructure teams (types of hardware,
or available VMs or OSs), the Kubernetes platform teams, the security teams, the
network function experts, and many other individuals and teams. Each of these
teams will have their own systems for tracking the values they control, as well
as processes for allocating and distributing those values. This coordination
between teams is a fundamental *organizational* problem with operating at scale.
The existing tools and methods do not even attempt to address these parts of the
problem. They *start* once all of the "input" decisions are made.

The Nephio project believes that the organizational challenge of figuring out
these values is one of the primary limiting factors to achieving the efficient
management of large, complex systems at scale. This challenge becomes even
greater when we understand the need to manage the changes to these values over
time, and how changes to some values implies the need to change other values.
This challenge is currently left to ad-hoc processes that differ across
organizations. Nephio is working on how to structure the intent to make it
manageable using data management techniques.

This Nephio release focuses on the following:
- Demonstrating the core Nephio principles, such as Configuration-as-Data and
  leveraging the intent-driven, active-reconciliation nature of Kubernetes.
- Infrastructure orchestration/automation using controllers based on
  the Cluster API. Currently, only Kubernetes in Docker (KIND) cluster creation
  is supported.
- Orchestration/automation of the deployment and management of 5G Core and RAN
  networks. 

While the current release uses Cluster API, KIND, and free5gc/OAI for
demonstration purposes, the same principles and code can be used for managing
other infrastructure and network functions. The *uniformity in systems*
principle means that as long as something is manageable via the Kubernetes
Resource Model, it is manageable via Nephio.

