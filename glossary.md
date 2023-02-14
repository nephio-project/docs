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

## CRD

## dehydration

See [hydration](#hydration).

## DRY

This is a common software engineering term that stands for [Don't repeat
yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), and attempts
to reduce repetition in software development. In the Kubernetes configuration
management context, a good example is a Helm chart, which attempts to abstract
the particular manifests for a given workload. A kpt package that is not yet
ready to deploy is also an example of a DRY artifact. In general, any sort of
"template" or "blueprint" is usually an attempt to capture some repeatable
pattern, following this principle.

*See also*: [hydration](#hydration), [WET](#wet)

## fanout

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

With out-of-place hydration, the author of the template has to figure out,
upfront, all the possible outcomes of the hydration process. Then, they have to
make available inputs to the pipeline in order to make all of those different
outcomes achievable. This leads to "over-parameterization" - where effectively
every option possible in the outputs becomes an option in the input. At that
point, you have mostly *moved* complexity rather than *reduced* complexity.
In-place hydration can help with the over-parameterization, as values that are
rarely changed by users can simply be edited in-place.

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

## mutation
The act of changing the configuration. There are different processes that can be
usd for mutation, including controllers, injectors, KRM functions, web hooks,
and manual in-place edits.

*See also*: [validation](#validation)

## operator

## package

## resource

## validation
The act of verifying that the configuration is syntactical correct, and that it
matches a set of rules (or policies). Those rules or policies may be for
internal consistency (e.g., matching Deployment and Service label selectors),
or they may be organizationally related (e.g., all Deployments must contain a
label indicating cost allocation center).

## value propagation
The same value in a configuration is often used in more than one place. *Value
propagation* is the technique of setting or generating the value once, and then
copying (or propagating) it to different places in the configuration. For
example, setting a Helm value in the values.yaml file, and then having it used
in multiple places across different resources.

## variant
A *variant* is an modified version of a package. Sometimes it is the output of
the hydration process, particularly when using out-of-place hydration. For
example, if you use the same Helm chart with different inputs to create
per-cluster workloads, you are generating variants.

In Nephio, we use kpt packages to help keep an association between a package and
the variants of that package. When you clone a kpt package, an association is
maintained with the upstream package. Every deployable variant of a package is a
clone of the original, upstream package. This assists greatly in Day 2
operations; when you update the original package, you can identify all variants
and merge the updates from the upstream into the downstream. This behavior is
automated via the PackageVariant controller.

## WET
This term, which we use as an acryonym for "Write Every Time", comes from [software
engineering](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself#WET), and is a somewhat pejorative term in contrast to [DRY](#dry). However, in the context of *configuration-as-data*, rather than *code*, the idea of storing the configuration as fully-formed data enables automation and the use of data-management techniques to manage the configuration at scale.

*See also*: [DRY](#dry), [hydration](#hydration)

## workload

A workload is any application running on Kubernetes, including network
functions.
