---
title: Installation on OpenShift
description: >
  Step by step guide to instal Nephio on OpenShift
weight: 3
---

*Work in progress*


In this guide, you will set up Nephio with:

- **Management Cluster**: OpenShift with [Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management)
- **Cluster Provisioner**: Assisted Service
- **Workload Clusters**: OpenShift Cluster or Single Node OpenShift Cluster or HyperShift Cluster or Remote Worker Node
- **Gitops Tool**: [OpenShift GitOps](https://www.redhat.com/en/technologies/cloud-computing/openshift/gitops) backed by
  ArgoCD
- **Git Provider**: Gitea running in the Nephio management cluster will be the git provider for cluster deployment
  repositories. Some external repositories will be  on GitHub.
- **Web UI Auth**: OpenShift OAuth.
- **Ingress/Load Balancer**: OpenShift Ingress will be used, supplying a Route to the Nephio and Gitea Web UIs.

## Prerequisites

- A Red Hat Account and access to https://console.redhat.com/openshift/
- OpenShift cli client `oc`. [Download here](https://console.redhat.com/openshift/downloads)

## Setup the Management Cluster

If you have already access to an OpenShift cluster, make sure you have cluster-admin privilege. Then go to the
[requirements](#requirements) section.

### Create the management cluster
Two methods exists: 

 - **Self-managed OpenShift**: this assumes you have resources available in your private or public cloud environment.
   Please refer to our [installation documentation](https://docs.openshift.com/container-platform/4.13/installing/index.html).
- **Managed OpenShift**: this assumes you have access to a public cloud environment with enough permission.
  You can choose one of the two offerings
  - [Azure Red Hat OpenShift (ARO)](https://www.redhat.com/en/technologies/cloud-computing/openshift/azure)
  - [Red Hat OpenShift on AWS (ROSA)](https://www.redhat.com/en/technologies/cloud-computing/openshift/aws)

### Install the requirements
Install the two following operators:

- [Red Hat Advanced Cluster Management](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html-single/install/index#installing-from-the-operatorhub)
- [OpenShift GitOps](https://docs.openshift.com/container-platform/4.13/cicd/gitops/installing-openshift-gitops.html#installing-gitops-operator-in-web-console_installing-openshift-gitops)

Once installed, you need to prepare the management cluster for zero touch provisioning of OpenShift Cluster.

1. [Enable Central Infrastructure Management service](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html/clusters/cluster_mce_overview?extIdCarryOver=true&sc_cid=701f2000001Css5AAC#enable-cim)

## Install Nephio

### OpenShift Package Repository

A repository of OpenShift-installation specific packages must be used to deploy Nephio. This repository contains
packages derived from the standard Nephio R1 packages, but with OpenShift-specific modifications.

The blueprint package act as an
[App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) installing all the
required Nephio components on the management cluster.

You can register this package as a read-only external repository by applying the `blueprints-nephio-openshift` ArgoCD
Application:

```bash
oc apply -f https://raw.githubusercontent.com/openshift-telco/blueprints-nephio-openshift/v1.0.1/nephio-mgnt/app-of-apps.yaml
```

This will take care of applying the [common dependencies](common-dependencies.md) and the [common components](common-components.md)

### Access the Nephio

- Get the Nephio URL:
  ```
  oc get route nephio -n nephio-webui -o=jsonpath=https://'{.spec.host}'
  ```

- Login using your OpenShift login

### Access the Gitea UI

- Get the Gitea URL:
  ```
  oc get route gitea -n gitea -o=jsonpath=https://'{.spec.host}'
  ```

- Login
    - user: gitea
    - password: password

## Install edge clusters

### Bare metal Single Node OpenShift

You first need to create the OpenShift context ConfigMap entries to customize the cluster configuration. Here is an
example:

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: openshift-context
  annotations:
    config.kubernetes.io/local-config: "true"
    kpt.dev/config-injection: required
data:
  release-image-name: openshift-v4.13.12
  cluster-name: ca-montreal
  base-domain: adetalhouet.ca
  machine-network: 192.168.123.0/24
  ssh-pub-key: "YOUR_SSH_KEY"
  pull-secret: "YOUR_PULL_SECRET"
  bmc-address: "redfish-virtualmedia+http://192.168.1.170:8000/redfish/v1/Systems/c505d99e-bc2a-4690-89a5-463098de4d59"
  bmc-username: "ZXhhbXBsZQo="
  bmc-password: "ZXhhbXBsZQo="
  bmc-boot-mac-address: "02:04:00:00:01:03"
```

You can now create edge clusters by using `kubectl` to apply the following PackageVariantSet to your management cluster.
It will inject the site specific configuration supplied in the ConfigMap.

```
apiVersion: config.porch.kpt.dev/v1alpha2
kind: PackageVariantSet
metadata:
  name: edge-clusters
spec:
  upstream:
    repo: openshift-packages-main
    package: nephio-workload-cluster-sno
    revision: main
  targets:
  - repositories:
    - name: management
      packageNames:
      - ca-montreal
    template:
      annotations:
        approval.nephio.org/policy: initial
      injectors:
      - kind: ConfigMap
        name: openshift-context
      pipeline:
        mutators:
        - image: gcr.io/kpt-fn/set-labels:v0.2.0
          configMap:
            nephio.org/site-type: edge
            nephio.org/region: ca-central
```
