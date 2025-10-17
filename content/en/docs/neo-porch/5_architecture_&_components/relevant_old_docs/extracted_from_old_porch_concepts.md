#### Porch Server

The Porch server is implemented as a [Kubernetes extension API server][apiserver] which works with the Kubernetes API
aggregation layer. The benefits of this approach are:

* seamless integration with the well-defined Kubernetes API style
* availability of generated clients for use in code
* integration with existing Kubernetes ecosystem and tools such as  `kubectl` CLI,
  [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* avoids requirement to open another network port to access a separate endpoint running inside k8s cluster
  * this is a distinct advantage over GRPC which was initially considered as an alternative approach

Resources implemented by the Porch server include:

* For each package revision (see [Package Versioning](../../2_concepts/concepts_elaborated.md#package-versioning)):
  * `PackageRevision` - represents the *metadata* of the package revision stored in a repository.
  * `PackageRevisionResources` - represents the *file contents* of the package revision.
    {{% alert color="primary" %}}
  Note that each package revision is represented by a *pair* of resources, each presenting a different view (or [representation](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#differing-representations)) of the same underlying package revision.
    {{% /alert %}}
* A `Repository` [custom resource][crds], which supports repository registration.

The **Porch server** itself includes the following key components:

* The *aggregated API server*, which implements the integration into the main Kubernetes API server and 
  serves API requests for the `PackageRevision` and `PackageRevisionResources` resources.
* Package orchestration *engine*, which implements the package lifecycle operations and package mutation workflows.
* *CaD Library*, which implements specific package manipulation algorithms such as package rendering (evaluation of
  package's function *pipeline*), initialization of a new package, etc. The CaD Library is a fork of `kpt` to allow Porch
  to reuse the `kpt` algorithms and fulfil its overarching use case to be "kpt as a service".
* *Package cache*, which enables:
  * local caching to allow package lifecycle and content manipulation operations to be executed within the Porch server
    with minimal latency.
  * abstracting package operations upward so they can be used without having to take account of the underlying storage
    repository software mechanism (Git or OCI).
* *Repository adapters* for Git and OCI, which implement the specific logic of interacting with each repository type.
* *Function Runner runtime*, which evaluates individual [kpt functions][functions], incorporating a multi-tier cache of
  functions to support low-latency evaluation.

#### Function Runner

The **Function Runner** is a separate microservice responsible for evaluating [kpt functions][functions]. It exposes
a [GRPC](https://grpc.io/) endpoint which enables evaluating a specified kpt function on a provided configuration package.

GRPC was chosen for the function runner service because the [benefits of an API server](../../2_concepts/porch_concepts.md#grpc-api) that prompted its use
for the Porch server do not apply. The function runner is an internal microservice, an implementation detail not exposed
to external callers. This makes GRPC perfectly suitable.

The function runner maintains a cache of functions to support low-latency function evaluation. It achieves this through
two mechanisms available to it for evaluation of a function:

* The **Executable Evaluation** mechanism executes the function inside the `function-runner` pod through shell-based
  invocation of a function's binary executable. This applies only to a selected subset of popular functions, whose binaries
  are packaged inside the `function-runner` image itself in a sort of pre-cache.
* The **Pod Evaluation** mechanism is the fallback when the invoked function is not one of those packaged in the `function-runner`
  image for the Executable Evaluation approach. The `function-runner` pod spawns a separate *function pod*, based on the
  image of the invoked function, along with a corresponding front-end service. Once the pod and service are ready, the
  exposed GRPC endpoint is invoked to evaluate the function with the package as input. Once a function pod completes
  evaluation and returns the result to the `function-runner` pod, the function pod is kept in existence temporarily so
  it can be re-used quickly as a cache hit. After a pre-configured period of disuse (default 30 minutes), the function
  runner terminates the function pod and its service, to recreate them from the start on the next invocation of that function.

#### CaD Library

The [kpt](https://kpt.dev/) CLI already implements the fundamental package manipulation algorithms in order to provide its
command line user experience:

* [kpt pkg init](https://kpt.dev/reference/cli/pkg/init/) - create a bare-bones, valid, KRM package.
* [kpt pkg get](https://kpt.dev/reference/cli/pkg/get/) - create a downstream package by cloning an upstream package;
  set up the upstream reference of the downstream package.
* [kpt pkg update](https://kpt.dev/reference/cli/pkg/update/) - update the downstream package with changes from new
  version of upstream, 3-way merge.
* [kpt fn eval](https://kpt.dev/reference/cli/fn/eval/) - evaluate a kpt function on a package.
* [kpt fn render](https://kpt.dev/reference/cli/fn/render/) - render the package by executing the function pipeline of
  the package and its nested packages.
* [kpt fn source](https://kpt.dev/reference/cli/fn/source/) and [kpt fn sink](https://kpt.dev/reference/cli/fn/sink/) -
  read package from local disk as a `ResourceList` and write package represented as `ResourcesList` into local disk.

The same set of primitive operations form the foundational building blocks of the package orchestration service. Further,
Porch combines these blocks into higher-level operations (for example, Porch renders packages automatically on changes;
future versions will support bulk operations such as upgrade of multiple packages, etc.).

The implementation of the package manipulation primitives in kpt was refactored in order to:

* create a reusable CaD library, usable by both the kpt CLI and the Package Orchestration service.
* create abstractions for dependencies which differ between CLI and Porch (most notably the dependency on Docker for
  function evaluation and on the local file system for package rendering).

Over time, the CaD Library will provide the package manipulation primitives:

* create a valid empty package (init).
* clone a package and add upstream pointers (get).
* perform 3-way merge (upgrade).
* render - core package rendering algorithm using a pluggable function evaluator to support:
  * function evaluation via Docker (as used by kpt CLI).
  * function evaluation via an RPC to a service or appropriate function sandbox.
  * high-performance evaluation of trusted, built-in, functions without sandbox.
* heal configuration (restore comments after lossy transformation).

and both kpt CLI and Porch will consume the library. This approach will allow leveraging the investment already made into
the high quality package manipulation primitives, and enable functional parity between the kpt CLI and Porch.


<!-- Reference links -->
[crds]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/