---
title: "Porch Contributor Guide"
type: docs
weight: 7
description:
---

## Changing Porch API

If you change the API resources, in `api/porch/.../*.go`, update the generated code by running:

```sh
make generate
```

## Components

Porch comprises of several software components:

* [api](https://github.com/nephio-project/porch/tree/main/api): Definition of the KRM API supported by the Porch
  extension apiserver
* [porchctl](https://github.com/nephio-project/porch/tree/main/cmd/porchctl): CLI command tool for administration of
  Porch `Repository` and `PackageRevision` custom resources.
* [apiserver](https://github.com/nephio-project/porch/tree/main/pkg/apiserver): The Porch apiserver implementation, REST
  handlers, Porch `main` function
* [engine](https://github.com/nephio-project/porch/tree/main/pkg/engine): Core logic of Package Orchestration -
  operations on package contents
* [func](https://github.com/nephio-project/porch/tree/main/func): KRM function evaluator microservice; exposes GRPC API
* [repository](https://github.com/nephio-project/porch/blob/main/pkg/repository): Repository integration package
* [git](https://github.com/nephio-project/porch/tree/main/pkg/externalrepo/git): Integration with Git repository.
* [oci](https://github.com/nephio-project/porch/tree/main/pkg/externalrepo/oci): Integration with OCI repository.
* [cache](https://github.com/nephio-project/porch/tree/main/pkg/cache): Package caching.
* [controllers](https://github.com/nephio-project/porch/tree/main/controllers): `Repository` CRD. No controller;
  Porch apiserver watches these resources for changes as repositories are (un-)registered.
* [test](https://github.com/nephio-project/porch/tree/main/test): Test Git Server for Porch e2e testing, and
  [e2e](https://github.com/nephio-project/porch/tree/main/test/e2e) tests.

## Running Porch

See dedicated documentation on running Porch:

* [locally]({{< relref "/docs/porch/contributors-guide/environment-setup.md" >}})
* [on GKE]({{< relref "/docs/porch/running-porch/running-on-GKE.md" >}})

## Build the Container Images

Build Docker images of Porch components:

```sh
# Build Images
make build-images

# Push Images to Docker Registry
make push-images

# Supported make variables:
# IMAGE_TAG      - image tag, i.e. 'latest' (defaults to 'latest')
# GCP_PROJECT_ID - GCP project hosting gcr.io repository (will translate to gcr.io/${GCP_PROJECT_ID})
# IMAGE_REPO     - overwrites the default image repository

# Example:
IMAGE_TAG=$(git rev-parse --short HEAD) make push-images
```

## Debugging

To debug Porch, run Porch locally [running-locally.md]({{< relref "/docs/porch/contributors-guide/environment-setup.md" >}}), exit porch server running
in the shell, and launch Porch under the debugger. VS Code debug session is pre-configured in
[launch.json](https://github.com/nephio-project/porch/blob/main/.vscode/launch.json).

Update the launch arguments to your needs.

## Code Pointers

Some useful code pointers:

* Porch REST API handlers in [registry/porch](https://github.com/nephio-project/porch/tree/main/pkg/registry/porch),
  for example [packagerevision.go](https://github.com/nephio-project/porch/tree/main/pkg/registry/porch/packagerevision.go)
* Background task handling cache updates in [background.go](https://github.com/nephio-project/porch/tree/main/pkg/registry/porch/background.go)
* Git repository integration in [pkg/git](https://github.com/nephio-project/porch/tree/main/pkg/externalrepo/git)
* OCI repository integration in [pkg/oci](https://github.com/nephio-project/porch/tree/main/pkg/externalrepo/oci)
* CaD Engine in [engine](https://github.com/nephio-project/porch/tree/main/pkg/engine)
* e2e tests in [e2e](https://github.com/nephio-project/porch/tree/main/test/e2e). See below more on testing.

## Running Tests

All tests can be run using `make test`. Individual tests can be run using `go test`.
End-to-End tests assume that Porch instance is running and `KUBECONFIG` is configured
with the instance. The tests will automatically detect whether they are running against
Porch running on local machine or k8s cluster and will start Git server appropriately,
then run test suite against the Porch instance.

## Makefile Targets

* `make generate`: generate code based on Porch API definitions (runs k8s code generators)
* `make tidy`: tidies all Porch modules
* `make fmt`: formats golang sources
* `make build-images`: builds Porch Docker images
* `make push-images`: builds and pushes Porch Docker images
* `make deployment-config`: customizes configuration which installs Porch
   in k8s cluster with correct image names, annotations, service accounts.
   The deployment-ready configuration is copied into `./.build/deploy`
* `make deploy`: deploys Porch in the k8s cluster configured with current kubectl context
* `make push-and-deploy`: builds, pushes Porch Docker images, creates deployment configuration, and deploys Porch
* `make` or `make all`: builds and runs Porch [locally]({{< relref "/docs/porch/contributors-guide/environment-setup.md" >}})
* `make test`: runs tests

## VS Code

[VS Code](https://code.visualstudio.com/) works really well for editing and debugging.
Just open VS Code from the root folder of the Porch repository and it will work fine. The folder contains the needed
configuration to Launch different functions of Porch.
