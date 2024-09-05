# Nephio documentation

This repository contains the source files for the documentation for the [Nephio](https://nephio.org/) project. 
The documentation is served from [docs.nephio.org](https://docs.nephio.org/) and the status of our Netlify build is

[![Netlify Status](https://api.netlify.com/api/v1/badges/9a7a49cd-9710-49c7-bd97-1dfb2272717c/deploy-status)](https://app.netlify.com/sites/nephio/deploys)

## Status of the documentation

In R2 release, a Hugo / Docsy based documentation site was introduced for the Nephio documentation. Hugo / Docsy uses the Markdown files hosted in the Github repo to generate the documentation website. We are still working on the restructuring the content and finalizing the look and feel of the website.

## How to contribute to the documentation

### Setting up the environment

1. The site is using Hugo as the documentation generating engine and Hugo depends on Go in mudule handling you need to
   have Go installed on your computer. Most Linux distributions have Go preinstalled, but some of them have a too old
   version for Hugo. You should have at least version 1.18.0. You can install version 1.18 of go on an Ubuntu using the
   following commands:

    1. `sudo add-apt-repository ppa:longsleep/golang-backports`
    1. `sudo apt update`
    1. `sudo apt install golang-1.18`

1. The site is using Hugo as the documentation generating engine, therefore you need to install Hugo. As the Docsy
  template that we use requires transforming Sass to CSS, you will need to install the *extended* version of Hugo.
  Link to installation instructions is [here](https://gohugo.io/installation/linux/). To ensure that you have the
  *extended* version of Hugo, run `hugo version`. The version string should have the word extended in it (Or `hugo
  version | grep extended` should not be an empty line).
2. Some functions of the theme generation are using NPM packages, therefore NodeJS and NPM will be needed. For
  compatibility reasons a Node version of at least v16.20.2 is needed. To install this version of NodeJS, follow the
  instructions from [deb.nodesource.com](http://deb.nodesource.com/) and set the `NODE_MAJOR=20`.
3. Install the npm dependencies with `npm install`

### Build the docs locally

To build and see the documentation locally run `hugo serve`. To double-check if the site will build on Netifly run `hugo
--gc --minify`.

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

The releases of Nephio to date focus on:
- Demonstrating the core Nephio principles such as Configuration-as-Data and
  leveraging the intent-driven, active-reconciliation nature of Kubernetes.
- Infrastructure orchestration/automation using controllers based on
  the Cluster API. At this time only KIND cluster creation is supported.
- Orchestration/automation of 5G core network functions deployment and
  management. This release focuses on network functions from
  [free5gc](https://free5gc.org/) and [OAI](https://openairinterface.org/).

While the releases to date use Cluster API, KIND, free5gc and OAI for demonstration
purposes, the exact same principles and even code can be used for managing other
infrastructure and network functions. The *uniformity in systems* principle
means that as long as something is manageable via the Kubernetes Resource Model,
it is manageable via Nephio.

Please use the documentation links below to learn more about Nephio, or check out our [Learning Nephio](https://wiki.nephio.org/display/HOME/Learning+with+Nephio+R1) series. The video series includes a [demo video](https://youtu.be/mFl71sy2Pdc) and short articles about different aspects of Nephio.

## User Documentation

* [Release Notes for each Nephio release](https://docs.nephio.org/docs/release-notes/)
* [Demo Sandbox Environment Installation](https://docs.nephio.org/docs/guides/install-guides/)
* [User Guide](https://docs.nephio.org/docs/guides/user-guides/)

## Other Documentation

* [Developer Documentation](https://github.com/nephio-project/nephio)
* [Developer Guide](https://docs.nephio.org/docs/guides/contributor-guides/)
* [Project Resources](https://github.com/nephio-project/docs/blob/main/resources.md)


