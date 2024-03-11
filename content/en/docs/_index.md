---
title: Documentation
linkTitle: Docs
menu: {main: {weight: 20}}
weight: 1
---

# About Nephio

Our mission is "to deliver carrier-grade, simple, open, Kubernetes-based cloud
native intent automation and common automation templates that materially
simplify the deployment and management of multi-vendor cloud infrastructure and
network functions across large scale edge deployments." But what does that mean?
With this release and the accompanying documentation, we hope to make that
clear.

The mission outlines the basic goals and the [About Nephio
page](https://nephio.org/about/) describes the high-level architecture of
Nephio. It is important to understand that Nephio is about managing complex,
inter-related workloads *at scale*. If we simply want to deploy a network
function, existing methods like Helm charts and scripts are sufficient.
Similarly, if we want to deploy some infrastructure, then using existing
Infrastructure-as-Code tools can accomplish that. Configuring running network
functions can already be done today with element managers.

So, why do we need Nephio? The problems Nephio wants to solve start only
once we try to operate at scale. "Scale" here does not simply mean "large number
of sites". It can be across many different dimensions: number of sites, number
of services, number of workloads, size of the individual workloads, number of
machines needed to operate the workloads, complexity of the organization running
the workloads, and other factors. The fact that our infrastructure, workloads,
and workload configurations are all interconnected dramatically increases
the difficulty in managing these architectures at scale.

To address these challenges, Nephio follows a [few basic
principles](https://cloud.google.com/blog/topics/telecommunications/network-automation-csps-linus-nephio-cloud-native)
that experience has shown to enable higher scaling with less management overhead:
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
manage. Handling that sheer number of values, with their interdependencies and a need
for consistency management between them, requires *data management* techniques,
not *code* management techniques. This is why existing methodologies begin to
break down at scale, particular edge-level scale.

Consider as well that no single human will understand all of those values. Those
values relate not only to workloads, but also to the infrastructure we need to
support those workloads. They require different areas of expertise and different
organizational boundaries of control. For example, you will need input from
network planning (IP address, VLAN tags, ASNs, etc.), input from
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
when we realize that we need to manage changes to these values over time, and
understand how changes to some values implies the need to change other values.
This challenge is currently left to ad hoc processes that differ across
organizations. Nephio is working on how to structure the intent to make it
manageable using data management techniques.

This release of Nephio focuses:
- Demonstrating the core Nephio principles such as Configuration-as-Data and
  leveraging the intent-driven, active-reconciliation nature of Kubernetes.
- Infrastructure orchestration/automation using controllers based on
  the Cluster API. At this time only KIND cluster creation is supported.
- Orchestration/automation of 5G Core and RAN network functions deployment and
  management. 

While the current releases uses Cluster API, KIND, and free5gc/OAI for demonstration
purposes, the exact same principles and even code can be used for managing other
infrastructure and network functions. The *uniformity in systems* principle
means that as long as something is manageable via the Kubernetes Resource Model,
it is manageable via Nephio.

# Nephio Architecture

```mermaid
---
title: Nephio R2 Architecture
config:
  theme: neutral
---

flowchart TB

subgraph R2
    direction TB
    subgraph MGNT [Management Cluster]
        subgraph WebUI
        end

        subgraph Porch
            subgraph PAPI [API]
                PackageRevision[[PackageRevision]]
                PackageVariant[[PackageVariant]]
                ..[[...]]
            end
            subgraph KptPipeline [kpt pipeline]
                subgraph SI [Specialization Functions / Controllers]
                    IPAM{{IPAM}}
                    NW{{NAD}}
                    VLAN{{VLAN}}
                    DNN{{DataNetwork}}
                    SNFDeployment{{NFDeployment}}
                    CI{{Config Injection}}
                end
            end
            PApproval{{Approval Controller}}
        end
        subgraph Nephio [Nephio]
            subgraph NAPI [API]
                NFTopology[[NFTopology]]
                NFDeployment[[NFDeployment]]
                NFConfig[[NFConfig]]
                .[[...]]
                click NAPI "https://github.com/nephio-project/api/tree/v2.0.0"
            end
            subgraph NC [Controllers]
                CNETWORK{{Nework}}
                CNETWORKCONFIG{{Nework Config}}
                CRB{{Resource Backend}}
                CTOK{{Token}}
                CBP{{Boostrap packages}}
                %% CBS{{Boostrap secret}}
                CREPO{{Repository}}
            end
        end

        subgraph CSM [ConfigSync]
            direction TB
            subgraph CSMAPI [API]
                RSM[[RootSync]]
            end
            subgraph CSMControllers [Controllers]
                CSMC{{Config Management}}
            end
        end
    end

    subgraph Git
        Blueprints[(Nephio Catalog <br/>Blueprints)]
        Deployment[(Deployment)]
    end

    subgraph WORK [Workload Cluster]
        subgraph CSW [ConfigSync]
            direction TB
            subgraph CSWAPI [API]
                RSW[[RootSync]]
            end
            subgraph CSWControllers [Controllers]
                CSWC{{Config Management}}
            end
        end
        subgraph NF [Supported Network Functions]
            subgraph 5GRAN [5G RAN]
                OAIRAN{{OAI}}
            end
            subgraph 5GCORE [5G Core]
                direction TB
                OAICORE{{OAI}}
                Free5GCO{{Free5GC}}
            end
        end
    end

    subgraph Infrastructure
        subgraph Cloud
            KIND(Kind)
            GC(Google Cloud)
            OCP(OpenShift)
        end
        subgraph Network
            SRLinux(Nokia SR)
        end
    end
end

%% link 0
WebUI --> Porch
%% link 1
%% Force mermaid to render Nephio box below Porch
KptPipeline ~~~ Nephio
%% link 2
%% Force mermaid to render ConfigSync box below Nephio
Nephio ~~~ CSM
%% link 3
PackageVariant -- Upstream --> Blueprints
%% link 4
PackageVariant -- Downstream --> Deployment
%% link 5
PackageRevision <--> KptPipeline
%% link 6
PackageRevision --> Deployment
%% link 7
Deployment -- Pull ---o WORK
%% link 8
MGNT --> Infrastructure
%% link 9
WORK --> Infrastructure
%% link 10
MGNT ~~~ Git

classDef NephioFunctions fill:#96cdff;
classDef NephioFunctionComponents fill:#bee0ff
classDef NephioBackgroundLight fill:#6fbbff
classDef WhiteBackground fill:#fffff,stroke-width:3px

classDef default fill:#48a8ff,color:white

class MGNT WhiteBackground
class WORK WhiteBackground
class Git WhiteBackground
class Infrastructure WhiteBackground

class PAPI NephioFunctionComponents
class KptPipeline NephioFunctionComponents
class SI NephioFunctionComponents
class NAPI NephioFunctionComponents
class NC NephioFunctionComponents
class CSMAPI NephioFunctionComponents
class CSMControllers NephioFunctionComponents
class Cloud NephioFunctionComponents
class Network NephioFunctionComponents
class 5GCORE NephioFunctionComponents
class 5GRAN NephioFunctionComponents
class CSWAPI NephioFunctionComponents
class CSWControllers NephioFunctionComponents

class Porch NephioFunctions
class Nephio NephioFunctions
class CSM NephioFunctions
class CSW NephioFunctions
class NF NephioFunctions
class Git NephioFunctions
class Infrastructure NephioFunctions

class R2 NephioBackground
style R2 color:white

linkStyle default interpolate linear
```

Information regarding various Nephio components can be found in the main [User Guide](/content/en/docs/guides/user-guides/_index.md).