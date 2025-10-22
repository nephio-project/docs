---
title: O-RAN O2 IMS Operator Deployment
description: >
  A step by step guide to deploying a workload cluster, 
  using the provisioningrequests.o2ims.provisioning.oran.org CRD.
weight: 2
---

## Prerequisites

- A Nephio Management cluster: 
  - [installation guides]({{ relref "/docs/guides/install-guides/_index.md" }}) for detailed environment options.
- The following *optional* operator package deployed:
  - [o2ims operator]({{ relref "/docs/guides/install-guides/optional-components.md#o2ims-operator" }})

{{% alert title="Note" color="primary" %}}

If using a [sandbox demo environment]({{ relref "/docs/guides/install-guides/_index.md#kicking-off-an-installation-on-a-virtual-machine" }}), 
most of the above prerequisites are already satisfied.

{{% /alert %}}

This exercise will take us from a system with only the Nephio Management cluster setup, to a deployment with:

- A [workload cluster](https://github.com/nephio-project/catalog/tree/main/infra/capi/nephio-workload-cluster) specified as a template when instantiating an instance of the [ProvisioningRequest CRD](https://github.com/nephio-project/api/blob/main/config/crd/bases/o2ims.provisioning.oran.org_provisioningrequests.yaml).
- A repository for said cluster, registered with Nephio Porch.

To perform these exercises, we will need:

- Access to the installed demo VM environment as the ubuntu user.
- Access to the Nephio WebUI as described in the installation guide (optional).
- Access to Gitea, used in the demo environment as the Git provider (optional).


### Step 1: Deploy the `provisioningrequests.o2ims.provisioning.oran.org` CRD

{{% alert title="Note" color="primary" %}}

After a fresh docker install, verify the docker supplementary group is loaded by executing `id | grep docker`.
If not, logout and login to the VM or execute `newgrp docker` to ensure the docker supplementary group is loaded.

{{% /alert %}}

First, verify that the [catalog blueprint repositories](https://github.com/nephio-project/catalog.git) are registered 
and `Ready`:
```bash
kubectl get repository
```
Sample output:
```bash
NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
catalog-distros-sandbox     git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-infra-capi          git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-nephio-core         git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-nephio-optional     git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-free5gc   git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-oai-ran   git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-tools     git    Package   false        True    https://github.com/nephio-project/catalog.git
mgmt                        git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
oai-core-packages           git    Package   false        True    https://github.com/OPENAIRINTERFACE/oai-packages.git
```

Deploy the CRD
```bash
kubectl create -f https://raw.githubusercontent.com/nephio-project/api/refs/heads/main/config/crd/bases/o2ims.provisioning.oran.org_provisioningrequests.yaml
```
Verify that the CRD is created by executing
```bash
kubectl get crd | grep provisioningrequests
```
Output
```bash
NAME                                                         CREATED AT
provisioningrequests.o2ims.provisioning.oran.org             2025-02-18T20:20:06Z
```

## Step 2: Create a new `ProvisioningRequest` CR that will create a Workload Cluster:
```bash
cat << EOF | kubectl apply -f
apiVersion: o2ims.provisioning.oran.org/v1alpha1
kind: ProvisioningRequest
metadata:
  name: edge-cluster
spec:
  name: sample-edge
  description: "Provisioning request for setting up a sample edge kind cluster."
  templateName: nephio-workload-cluster
  templateVersion: v3.0.0
  templateParameters:
    clusterName: edge
    labels:
      nephio.org/site-type: edge
      nephio.org/region: europe-paris-west
      nephio.org/owner: nephio-o2ims
EOF
```

Verify if the *provisioningrequest* CR is created
```bash
kubectl get provisioningrequests edge-cluster
```
Output
```bash
NAME           AGE
edge-cluster   7h31m
```

Verify if *packagevariants* are created for the edge-cluster
```bash
kubectl get packagevariants
```
Output
```bashNAME                          AGE
edge-cluster                  4d12h
edge-configsync               4d12h
edge-crds                     4d12h
edge-kindnet                  4d12h
edge-local-path-provisioner   4d12h
edge-metallb                  4d12h
edge-multus                   4d12h
edge-repo                     4d12h
edge-rootsync                 4d12h
edge-vlanindex                4d12h

```

Examine the details of the *provisioningrequest* CR created
```bash
kubectl get provisioningrequests edge-cluster -o yaml
```
Output
```bash
apiVersion: o2ims.provisioning.oran.org/v1alpha1
kind: ProvisioningRequest
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"o2ims.provisioning.oran.org/v1alpha1","kind":"ProvisioningRequest","metadata":{"annotations":{},"name":"edge-cluster"},"spec":{"description":"Provisioning request for setting up a sample edge kind cluster.","name":"sample-edge","templateName":"nephio-workload-cluster","templateParameters":{"clusterName":"edge","labels":{"nephio.org/owner":"nephio-o2ims","nephio.org/region":"europe-paris-west","nephio.org/site-type":"edge"}},"templateVersion":"v3.0.0"}}
    provisioningrequests.o2ims.provisioning.oran.org/kopf-managed: "yes"
    provisioningrequests.o2ims.provisioning.oran.org/last-ha-a.A3qw: |
      {"spec":{"description":"Provisioning request for setting up a sample edge kind cluster.","name":"sample-edge","templateName":"nephio-workload-cluster","templateParameters":{"clusterName":"edge","labels":{"nephio.org/owner":"nephio-o2ims","nephio.org/region":"europe-paris-west","nephio.org/site-type":"edge"}},"templateVersion":"v3.0.0"}}
    provisioningrequests.o2ims.provisioning.oran.org/last-handled-configuration: |
      {"spec":{"description":"Provisioning request for setting up a sample edge kind cluster.","name":"sample-edge","templateName":"nephio-workload-cluster","templateParameters":{"clusterName":"edge","labels":{"nephio.org/owner":"nephio-o2ims","nephio.org/region":"europe-paris-west","nephio.org/site-type":"edge"}},"templateVersion":"v3.0.0"}}
  creationTimestamp: "2025-02-18T20:30:15Z"
  generation: 1
  name: edge-cluster
  resourceVersion: "2377164"
  uid: 8b17865d-60d4-47d4-b1b7-db7098340a22
spec:
  description: Provisioning request for setting up a sample edge kind cluster.
  name: sample-edge
  templateName: nephio-workload-cluster
  templateParameters:
    clusterName: edge
    labels:
      nephio.org/owner: nephio-o2ims
      nephio.org/region: europe-paris-west
      nephio.org/site-type: edge
  templateVersion: v3.0.0
status:
  provisionedResourceSet:
    oCloudInfrastructureResourceIds:
    - 5bcaf4ed-b467-4aa0-9dec-fb1517adb3f2
    oCloudNodeClusterId: 6f8759ea-f3ff-4bb5-ab8d-d0c35978f606
  provisioningStatus:
    provisioningMessage: Cluster resource created
    provisioningState: fulfilled
    provisioningUpdateTime: "2025-02-18T20:30:33Z"
```

## Step 3: Check the cluster installation

You can check if the cluster has been added to the management cluster:

```bash
kubectl get cl
```
or
```bash
kubectl get clusters.cluster.x-k8s.io
```
Sample output:
```
NAME       CLUSTERCLASS   PHASE         AGE    VERSION
edge       docker         Provisioned   179m   v1.31.0
```

You should also check that the KinD cluster has come up fully by checking the `machinesets`. 
You should see READY and AVAILABLE replicas.

```bash
kubectl get machinesets
```
Sample output:
```
NAME                        CLUSTER    REPLICAS   READY   AVAILABLE   AGE    VERSION
edge-md-0-lmsqz-7nzzc       edge       1          1       1           3h1m   v1.31.0
```
