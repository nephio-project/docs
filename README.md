# Nephio R1

## Introduction

Welcome to the R1 release of Nephio.  Nephio’s mission is "to deliver carrier
-grade, simple, open, Kubernetes-based cloud native intent automation and
common automation templates that materially simplify the deployment and
management of multi-vendor cloud infrastructure and network functions across
large scale edge deployments." But what does that mean? With this release and
the accompanying documentation, we hope to make that clear.

The mission outlines the basic goals, and the [About Nephio page](https://nephio.org/about/)
describes the high-level architecture of Nephio. It is important to understand
in all of this that Nephio is about managing complex, inter-related workloads
*at scale*. If we simply want to deploy a network function, existing methods
like Helm charts and scripts are sufficient. Similarly, if we want to deploy
some infrastructure, then using existing Infrastructure-as-Code tools can
accomplish that. Configuring running network functions can already be done today
with element managers.

So, why do we need Nephio? The problems Nephio wants to solve solve start only
once we try to operate at scale. "Scale" here does not simply mean "large number
of sites". It can be across many different dimensions: number of sites, number
of services, number of workloads, size of the individual workloads, number of
machines needed to operate the workloads, complexity of the organization running
the workloads, and other factors. The fact that our infrastructure, workloads,
and the workload configurations are all interconnected dramatically increases
the difficulty in managing these architectures at scale.

To address these challenges, Nephio follows a [few basic
principles](https://cloud.google.com/blog/topics/telecommunications/network-automation-csps-linus-nephio-cloud-native)
that experience has shown enable higher scaling with less management overhead:
- *Intent-driven* to enable the user to specify "what they want" and let the
  system figure out "how to make that happen".
- *Distributed actuation* to increase reliability across widely distributed
  fleets.
- *Uniformity in systems* to reduce redundant tooling and processes, and
  simplify operations.

Additionally, Nephio leverages the "configuration as data" principle. This
methodology means that the "intent" is captured in a machine-manageable format
that we can treat as data, rather than code. In Nephio, we use the Kubernetes
Resource Model (KRM) to capture intent. As Kubernetes itself is already an
intent-driven system, this model is well suited to our needs.

To understand why we need to treat configuration as data, let's consider an
example. In a given instance, a network function may have, say, 100 parameters
that need to be decided upon. When we have 100 such network functions, across
10,000 clusters, this results in 100,000,000 inputs we need to define and
manage. Handling that sheer number of values, with interdependencies and a need
for consistency management between them, requires *data management* techniques,
not *code* management techniques. This is why existing methodologies begin to
break down at scale, particular edge-level scale.

Consider as well that no single human will understand all of those values. Those
values relate not only to workloads, but to the infrastructure we need to
support those workloads. These are different areas of expertise, and different
organizational boundaries of control. For example, you will need input from
network planning (IP address, VLAN tags, ASNs, etc.), you will need input from
compute infrastructure teams (types of hardware or VMs available, OS available),
Kubernetes platform teams, security teams, network function experts, and many,
many other individuals and teams. Each of those teams will have their own
systems for tracking the values they control, and processes for allocating and
distributing those values. This coordination between teams is a fundamental
*organizational* problem with operating at scale. The existing tools and methods
do not even attempt to address these parts of the problem; they *start* once all
of the "input" decisions are made.

The Nephio project believes the organizational challenge around figuring out
these values is actually one of the primary limiting factors to achieving
efficient management of large, complex systems at scale. This gets even harder
when we realize we need to manage changes to these values over time, and
understand how changes to some values implies the need to change other values.
This challenge is currently left to ad hoc processes that differ across
organizations. Nephio is working on how to structure the intent to make it
manageable using data management techniques.

This release of Nephio focuses:
- Demonstrating the core Nephio principles such as Configuration-as-Data and
  leveraging the intent-driven, actively-reconciled nature of Kubernetes.
- Infrastructure orchestration/automation using controllers based on
  Cluster API. At this time only KIND cluster creation is supported.
- Orchestration/automation of 5G core network functions deployment and
  management. This release focuses on network functions from
  [free5gc](https://free5gc.org/).

While the current release uses Cluster API, KIND, and free5gc for demonstration
purposes, the exact same principles and even code can be used for managing other
infrastructure and network functions. The *uniformity in systems* principle
means that as long as something is managable via the Kubernetes Resource Model,
it is manageable via Nephio.

## User Documentation
* [Demo Sandbox Environment Installation](https://github.com/nephio-project/docs/blob/main/install-guide/README.md)
* [Quick Start Exercises](https://github.com/nephio-project/docs/blob/main/user-guide/exercises.md)
* [User Guide](https://github.com/nephio-project/docs/blob/main/user-guide/README.md)

## Other Documentation

* [Developer Documentation](https://github.com/nephio-project/nephio)
* [Project Resources](https://github.com/nephio-project/docs/blob/main/resources.md)
