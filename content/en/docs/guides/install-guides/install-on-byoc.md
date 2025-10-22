---
title: Installation on BYOC
description: >
  Step by step guide to install Nephio on any cluster
weight: 3
---

## Introduction

There are many ways to assemble a Nephio installation. This Installation Guide describes the common pieces across
environments, and describes the choices that need to be made to create a "Bring Your Own Cluster" Nephio installation.
Because there are so many combinations, a comprehensive guide is not practical. Instead, several guides showing
opinionated installations are available.

## Prerequisites

Regardless of the specific choices you make, you will need the following
prerequisites. This is in addition to any prerequisites that are specific to
your environment and choices.
 - a Linux workstation with internet access
 - *kubectl* [installed ](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)on your workstation
 - *kpt* [installed](https://kpt.dev/installation/kpt-cli) on your workstation
   (version v1.0.0-beta.43 or later)
 - *porchctl* [installed]({{ relref "/docs/porch/user-guides/porchctl-cli-guide.md" }}) on your workstation
 - Sudo-less *docker*, *Podman*, or *nerdctl*. If using *Podman* or *nerdctl*,
   you must set the
[`KPT_FN_RUNTIME`](https://kpt.dev/reference/cli/fn/render/?id=environment-variables)
environment variable.

As part of all installations, you will create or utilize an existing Kubernetes
management cluster. The management cluster must have internet access, and must
be a non-EOL Kubernetes version. Additionally:
 - Your default *kubectl* context should point to the cluster
 - You will need cluster administrator privileges (in particular you will need
   to be able to create namespaces and other cluster-scoped resources).

- Your default `kubectl` context should point to the cluster
- You will need cluster administrator privileges (in particular you will need to be able to create namespaces and other
  cluster-scoped resources).

You will use `kpt` for most of the installation packages in these instructions. Alternatively, you could also use `kubectl`
directly to apply the resources, once they are configured.

After installing the prerequisites, create a local directory on your workstation to hold the local package instances for
installing the various components:

```bash
mkdir nephio-install
cd nephio-install
```

The instructions for setting up the opinionated installations will assume you
have installed the prerequisites and created the *nephio-install* directory.

## Opinionated Installations

Instructions are provided for several different opinionated installations in the table below. Following this section are
descriptions of the various options, if you wish to assemble your own set of components.

| Environment | Description                                                |
| ----------- | ---------------------------------------------------------- |
| [Single VM]({{ relref "/docs/guides/install-guides/install-on-single-vm.md" }}) | The single VM demo environment, set up "the hard way" - without using the included provisioning script. This creates a complete Nephio-in-a-VM, just like the R1 demo environment. These instructions cover both Ubuntu and Fedora. |
| [Multiple VM]({{ relref "/docs/guides/install-guides/install-on-multiple-vm.md" }}) | The multiple VM environment, set up Nephio on multiple VMs. These instructions cover both Ubuntu and Fedora. |
| [Google Cloud Platform]({{ relref "/docs/guides/install-guides/install-on-gcp.md" }}) | Nephio running in GCP. A GKE cluster is used as the management cluster, with Anthos Config Controller for GCP infrastructure provisioning, Gitea as the Git provider, and WebUI authentication and authorization via Google OAuth 2.0 |
| [OpenShift]({{ relref "/docs/guides/install-guides/install-on-openshift.md" }}) | Nephio running in OpenShift, with Cluster API as the cluster provisioner, Gitea as the Git provider and WebUI authentication backed by OpenShift OIDC. |

## À La Carte Installation

If you wish to create a completely "à la carte" installation rather than using a documented opinionated environment,
this section will help you understand the choices you need to make among various dependencies and components.

### Git Providers

Nephio can support multiple Git providers for the repositories that contain packages. In R1, R2 and R3 only Gitea
repositories can be provisioned directly by Nephio; other Git providers will require manual provisioning of new
repositories. But most Git providers can be supported (via standard Git protocols) as repositories for packages for read
and write. It is also perfectly fine to use multiple providers; in the R1 demo environment, GitHub is used for upstream
external repositories while Gitea is used for the workload cluster repositories.

A non-exhaustive list of options:

| Provider                                                        | Workloads | Provisioning  |
| --------------------------------------------------------------- | --------- | ------------- |
| [GitHub](https://github.com)                                    | Yes       | No            |
| [Gitea](https://about.gitea.com/)                               | Yes       | Yes           |
| [GitLab](https://about.gitlab.com/)                             | Yes       | No            |
| [Google CSR](https://cloud.google.com/source-repositories/docs) | Yes       | Yes, with KCC |

See the [Porch user guide](https://kpt.dev/guides/porch-user-guide?id=repository-registration) to see how to register
repositories in Nephio.

In R1, we must install Gitea, even if you are using another provider. However, there are slight differences per
environment, so that installation will be documented in the specific environment instructions.

### GitOps Tool

As configured in the R1, R2 and R3 reference implementation, Nephio relies on ConfigSync. However, it is possible to
configure it to use a different GitOps tool, such as [FluxCD]({{ relref "/docs/guides/install-guides/optional-components.md#fluxcd-controllers" }})
or ArgoCD to apply packages to the clusters.

### Cluster Provisioner

R1 uses Cluster API, but other options may be used such as Crossplane, Google KCC, or AWS Controllers for Kubernetes.
You can provision more than one.

| Provider    | Notes                                                                                   |
| ----------- | --------------------------------------------------------------------------------------- |
| [Cluster API](https://cluster-api.sigs.k8s.io/) | Kubernetes project cluster provisioner for a variety of cluster providers.              |
| [KCC](https://cloud.google.com/config-connector/docs/overview)         | Google's Kubernetes Config Connector for GKE clusters and other GCP resources.          |
| [Crossplane](https://docs.crossplane.io/latest/getting-started/introduction/)  | API composition framework with cluster and other infrastructure providers.              |

### Load Balancer

The R1, R2 and R3 demo environments use [MetalLB](https://metallb.universe.tf/), but if you are running in a cloud, you
probably do not need anything special here. However, depending on your choice of GitOps tool and Git provider, some of
the packages may need customization to provision or use a well-known load balancer IP or DNS name.

### Gateway or Ingress

If you wish to avoid running `kubectl port-forward`, the use of Kubernetes Ingress or Gateway is recommended.

### Nephio WebUI Authentication and Authorization

In the default configuration, the Nephio WebUI **is wide open with no
authentication**. The WebUI itself authenticates to the cluster using a static
service account, which is bound to the cluster admin role. Any user accessing
the WebUI is **acting as a cluster admin**.

This configuration is designed for **testing and development only**. You must not
use this configuration in any other situation, and even for testing and
development it must not be exposed on the internet (for example, via a
LoadBalancer service, Ingress, or Route).

The WebUI currently supports the following options:

- [Google OAuth or OIDC]({{ relref "/docs/guides/install-guides/web-ui/webui-auth-gcp.md" }})
- [OIDC with Okta]({{ relref "/docs/guides/install-guides/web-ui/webui-auth-okta.md" }})

### Nephio Stock Repositories

It is recommended that you create a repository specific to your installation environment. The packages in this
repository can be derivatives of the various Nephio packages that are part of the demonstration environment. This allows
exiting PackageVariant and PackageVariantSet resources to work as expected, simply by changing the Git repository
pointed to by the Repository resource.

You may want to create a package containing those Repository resources, much as is done for the sandbox environment.
