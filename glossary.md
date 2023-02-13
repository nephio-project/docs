# Nephio Glossary

We use many terms in our Nephio discussions, coming from different domains
including telco, Kubernetes, configuration management, and our own
Nephio-specific terms. This glossary is intended to help clarify our usage of
these terms.

## config

See [configuration](#configuration).

## configuration
In Nephio, this *usually* refers to the Kubernetes resources used to provision
and manage network functions, their underlying infrastructure, and their
internal operation. Unfortunately this is a very general term and often is
overloaded with multiple meanings.

Sometimes, folks will say *network config* or *workload config* to refer to the
internal configuration of the network functions. Consider that most network
functions today cannot be directly configured via Kubernetes resources. Instead,
they are configurd via a proprietary configuration file, netconf, or even an
API. In that case, those terms usually refer to this proprietary configuration
language rather than Kubernetes resources. It is a goal for Nephio to help
vendors enable KRM-based management of this internal configuration, to allow
leveraging all the techniques we are building for KRM-based configuration (this
is part of the "Kubernetes Everywhere" principle).

As a community, we should try to use a common set of terminology for different
types of configuration. See
[docs#4](https://github.com/nephio-project/docs/issues/4).

## config injection

See [injector](#injector).

## controller

This term comes from Kubernetes where [controller](https://kubernetes.io/docs/reference/glossary/?fundamental=true#term-controller) is defined as a control loop that watches the intended and actual state of the cluster, and attempts to make changes as needed to make the actual state match the intended state. More specifically, this typically refers to software that processes Kubernetes Resources residing in the Kubernetes API server, and either transforms them into new resources, or calls to other APIs that change the state of some entity external to the API server. For example, `kubelet` itself is a controller that processes Pod resources to create and manage containers on a Node.

*See also*: [operator](#operator), [injector](#injector), [KRM
function](#krm-function)

## dehydration

See [hydration](#hydration).

## DRY

This is a common software engineering term that stands for [Don't repeat
yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), and attempts
to reduce repitition in software development. In the Kubernetes configuration
management context, a good example is a Helm chart, which attempts to abstract
the particular manifests for a given workload. A kpt package that is not yet
ready to deploy is also an example of a DRY artifact. In general, any sort of
"template" or "blueprint" is usually an attempt to capture some repeatable
pattern, following this principle.

*See also*: [hydration](#hydration), [WET](#wet)

## hydration
A play on [DRY](#dry) and [WET](#wet), this is the process by which a DRY
artifact becomes ready for deployment. A familiar example is rendering a Helm
chart. A lot of the effort in the configration management aspects of Nephio are
spent on making the hydration process scalable, collaborative, and managable in
Day 2 and beyond, all of which are challenges with current techniques.

Hydration may be *out-of-place*, where the source material (e.g., the Helm
chart), is separate from the output of the hydration process (the manifests).
This is probably the most familiar type of hydration, used by Helm and
kustomize, for example. Think of it as a pipeline with an input artifact, input
values, and output artifacts.

Hydration may also be *in-place*, where modifications are directly written to
the manifests in question. There is no separate input artifact and output
artifact. Rather, you may have a starting artifact, some operations you perform
on that artifact to achieve your goal, but you store the results of those
operations directly in the same artifact. Utilization of a version control
system such as Git is critical in this case. This is the kind of hydration we
typically use when operating on kpt packages.

While related, *DRY* and *WET* are not exactly the same concepts as *in-place* and
*out-of-place* hydration. The former two refer to principles, whereas the latter
two are more about the operational pipeline.

Note that occasionally people say "dehydration" when they mean "hydration",
likely due to the fact that "dehydration" is a more familiar word in common
language. Please offer folks some leeway in this, especially since we have many
non-native English speakers.

*See also*: [DRY](#dry), [WET](#wet)

## injection

See [injector](#injector).

## injector
We introduced this term during the Nephio [ONE Summit 2022
Workshop](https://github.com/nephio-project/one-summit-22-workshop#packages).
This refers to a software component that runs in the Nephio management cluster,
and could be considered a type of [controller](#controller). However, it
specifically watches for `PackageRevision` resources in a Draft state, and
checks for the [conditions](#conditions) on those resources. When it finds
unsastisfied conditions of the type it handles, the injector will
[mutate](#mutation) (modify) the Draft package by adding (or *injecting*) new or
changed resources.

For example, the IPAM injector monitors package revision drafts for unresolved
IP address requests. When it sees one, it takes information from the request and
uses it to allocate an IP address from the IP address management system. It
writes the result back into the draft package, where a KRM function can process
the result and copy ([propagate](#value-propagation)) it to the correct
resources in the package.

An *injector* need not be an entirely separate process. The PackageDeployment
controller will also perform injection after cloning an upstream package to a
downstream repository. In this case, it looks for resources in the upstream
package that are annotated with a `automation.nephio.org/config-injection:
"true"` annotation, and uses that to find resources in the management cluster,
and copy the `spec` of those resources into the downstream package. This allows
us to combine upstream ([DRY](#dry)) configuration with cluster-specific
configuration based upon the target cluster.

## kpt

## kpt function

See [KRM function](#krm-function).

## KRM

See [Kubernetes Resource Model](#kubernetes-resource-model).

## KRM function

## Kubernetes Resource Model

## manifest

## operator

## package

## resource

## value propagation

## variant

## WET
This term, which we use as an acryonym for "Write Every Time", comes from [software
engineering](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself#WET), and is a somewhat pejorative term in contrast to [DRY](#dry). However, in the context of *configuration-as-data*, rather than *code*, the idea of storing the configuration as fully-formed data enables automation and the use of data-management techniques to manage the configuration at scale.

*See also*: [DRY](#dry), [hydration](#hydration)

## workload

A workload is any application running on Kubernetes, including network
functions.
