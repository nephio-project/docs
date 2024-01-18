---
title: Free5GC Testbed Deployment and E2E testing with UERANSIM
description: >

weight: 2
---

## Table of Contents

[[_TOC_]]

## Introduction

Be sure you have followed the [installation guide]({{< ref "docs/guides/install-guides ">}})
before trying these exercises.

These exercises will take you from a system with only the Nephio Management cluster setup to a deployment with:

- A Core cluster 
- A Regional cluster
- An Edge Cluster
- Repositories for each cluster, registered with Nephio, and with Config Sync set up to pull from those repositories.
- Inter-cluster networking between those clusters
- A complete OAI Core, RAN and UE deployment including:

  - NRF, AUSF, UDR, UDM, MYSQL (Database backend for UDR) running on the Core cluster and communicating via the Kubernetes default network
  - AMF, SMF running on the core cluster and attached to the secondary Multus networks as needed
  - UPF running on the Edge cluster and attached to the secondary Multus networks as needed
  - CU-CP running on the Regional cluster and attached to the secondary Multus networks as needed
  - CU-UP and DU (RF Simulated) running on the Regional cluster and attached to the secondary Multus networks as needed
  - NR-UE (RF Simulated) running on the Regional cluster and attached to the secondary Multus networks as needed

The network configuration is illustrated in the following figure:

To perform these exercises, you will need:

- Access to the installed demo VM environment and can login as the `ubuntu` user to have access to the necessary files.
- Access to the Nephio UI as described in the installation guide

Access to Gitea, used in the demo environment as the Git provider, is optional. 

## Step 1: Setup the infrastructure

Our e2e topology consists of Regional, Core and Edge cluster. First, check to make sure that both the mgmt
and mgmt-staging repositories are in the Ready state. The mgmt repository is used to manage the contents of the
Management cluster via Nephio; the mgmt-staging repository is just used internally during the cluster bootstrapping
process.

Use the session just started on the VM to run these commands:

```bash
kubectl get repositories
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
catalog-distros-sandbox     git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-infra-capi          git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-nephio-core         git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-nephio-optional     git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-free5gc   git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-oai-ran   git    Package   false        True    https://github.com/nephio-project/catalog.git
catalog-workloads-tools     git    Package   false        True    https://github.com/nephio-project/catalog.git
core                        git    Package   true         True    http://172.18.0.200:3000/nephio/core.git
edge                        git    Package   true         True    http://172.18.0.200:3000/nephio/edge.git
mgmt                        git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
oai-core-packages           git    Package   false        True    https://github.com/OPENAIRINTERFACE/oai-packages.git
regional                    git    Package   true         True    http://172.18.0.200:3000/nephio/regional.git
```
</details>

Since those are Ready, you can deploy packages from these repositories. You can use our pre-defined `PackageVariantSets` for creating workload clusters

```bash
kubectl apply -f test-infra/e2e/tests/oai/001-infra.yaml
```

<details>
<summary>The output is similar to:</summary>

```console
```
</details>

It will take around 15 mins to create the three clusters. You can check the progress by looking at commits made in gitea `mgmt` and `mgmt-staging` repository. After couple of minutes you should see three independent repositories (Core, Regional and Edge) for each workload cluster. 

You can also look at the state of `packagerevisions` for the three packages. You can use the below command

```bash
kubectl get packagerevisions | grep -E 'core|regional|edge'
```

<details>
<summary>The output is similar to:</summary>

```console
```
</details>


## Step 2: Check the status of the Workload clusters

You can check if all the clusters have been added to the management cluster:

```bash
kubectl get cl
```
or
```bash
kubectl get clusters.cluster.x-k8s.io
```
<details>
<summary>The output is similar to:</summary>

```console

```
</details>

To access the API server of that cluster, you need to retrieve the `kubeconfig` file by pulling it from the Kubernetes
Secret and decode the base64 encoding:

```bash
kubectl get secret core-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/core-kubeconfig
kubectl get secret regional-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/regional-kubeconfig
kubectl get secret edge-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/edge-kubeconfig
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/regional-kubeconfig:$HOME/.kube/core-kubeconfig:$HOME/.kube/edge-kubeconfig
```

To retain the KUBECONFIG environment variable permanently across sessions for the
user, add it to the `~/.bash_profile` and source the `~/.bash_profile` file


You can then use it to access the Workload cluster directly:

```bash
kubectl get ns --context core-admin@core
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

You should also check that the KinD clusters have come up fully with `kubectl get machinesets`. You should see READY and
AVAILABLE replicas.

```bash
kubectl get machinesets
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>


Once the all the clusters are ready, it is necessary to connect them. For now you are using the
[containerlab tool](https://containerlab.dev/). Eventually, the inter-cluster networking will be automated as well.

```bash
./test-infra/e2e/provision/hacks/inter-connect_workers.sh
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

You will also need to configure the nodes for the VLANs. Again, this will be automated in a future release that
addresses node setup and inter-cluster networking. For now, you must run a script that creates them in each of the
worker nodes.

```bash
./test-infra/e2e/provision/hacks/vlan-interfaces.sh
```

<details>
<summary>The output is similar to:</summary>

```console
```
</details>


Finally, you want to configure the resource backend to be aware of these clusters. The resource backend is an IP address
and VLAN index management system. It is included for demonstration purposes to show how Nephio package specialization
can interact with external systems to fully configure packages. But it needs to be configured to match our topology.

First, you will apply a package to define the high-level networks for attaching our workloads. The Nephio package
specialization pipeline will determine the exact VLAN tags and IP addresses for those attachments based on the specific
clusters. There is a predefined PackageVariant in the tests directory for this:

```bash
kubectl apply -f test-infra/e2e/tests/oai/001-network.yaml
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

Then you will create appropriate `Secret` to make sure that Nephio can authenticate to the external backend.

```bash
kubectl apply -f test-infra/e2e/tests/oai/001-secret.yaml
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

The predefined PackageVariant package defines certain resources that exist for the entire topology. However, you also
need to configure the resource backend for our particular topology. This will likely be automated in the future, but for
now you can just directly apply the configuration you have created that matches this test topology. Within this step
also the credentials and information is provided to configure the network device, that aligns with the topology.

```bash
./test-infra/e2e/provision/hacks/network-topo.sh
  
kubectl apply -f test-infra/e2e/tests/oai/003-network-topo.yaml
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

To list the networks you can use the below command 

```bash
kubectl get networks.infra.nephio.org
```

<details>
<summary>The output is similar to:</summary>

```console
NAME           READY
vpc-cu-e1      True
vpc-cudu-f1    True
vpc-internal   True
vpc-internet   True
vpc-ran        True
```
</details>


## Step 4: Deploy the MySQL database required by OAI UDR


## Step 5: Deploy OAI Core and RAN Operator in the Workload clusters

Now you will need to deploy the OAI Core and RAN operators across the Workload clusters. To do this,
you use `PackageVariant` and `PackageVariantSet`. Later uses an objectSelector to select the WorkloadCluster resources previously added to the Management cluster when you had deployed the nephio-workload-cluster packages (manually as well as via
PackageVariantSet).

```bash
kubectl apply -f test-infra/e2e/tests/oai/002-oai-operators.yaml
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

## Step 6: Check Operator Deployment

Within five minutes of applying the RAN and Core Operator YAML file, you should see `` namespaces on core, :

```bash
kubectl get ns --context edge01-admin@edge01
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                           STATUS   AGE
config-management-monitoring   Active   3h46m
config-management-system       Active   3h46m
default                        Active   3h47m
free5gc                        Active   159m
kube-node-lease                Active   3h47m
kube-public                    Active   3h47m
kube-system                    Active   3h47m
resource-group-system          Active   3h45m
```
</details>

```bash
kubectl -n free5gc get all --context edge01-admin@edge01
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

## Step 7: Deploy the Core Network Functions

Finally, you can deploy the individual network functions which the operator will instantiate. For now, you will use
individual PackageVariants targeting the Regional cluster for each of the AMF and SMF NFs and a PackageVariantSet
targeting the Edge clusters for the UPF NFs. In the future, you could put all of these resources into
yet-another-package - a "topology" package - and deploy them all as a unit. Or you can use a topology controller to
create them. But for now, let's do each manually.

```bash
kubectl apply -f test-infra/e2e/tests/005-edge-free5gc-upf.yaml
kubectl apply -f test-infra/e2e/tests/006-regional-free5gc-amf.yaml
kubectl apply -f test-infra/e2e/tests/006-regional-free5gc-smf.yaml
```

Free5gc requires that the SMF and AMF NFs be explicitly configured with information about each UPF. Therefore, the AMF
and SMF packages will remain in an "unready" state until the UPF packages have all been published.

### Check UPF deployment

You can check the UPF logs in edge cluster:

```bash
UPF1_POD=$(kubectl get pods -n free5gc-upf -l name=upf-edge01 --context edge01-admin@edge01 -o jsonpath='{.items[0].metadata.name}')
kubectl -n free5gc-upf logs $UPF1_POD --context edge01-admin@edge01
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

### Check AMF deployment

You can check the AMF logs:

```bash
AMF_POD=$(kubectl get pods -n free5gc-cp -l name=amf-regional --context regional-admin@regional -o jsonpath='{.items[0].metadata.name}')
kubectl -n free5gc-cp logs $AMF_POD --context regional-admin@regional
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

### Check SMF deployment

You can check the SMF logs:

```bash
SMF_POD=$(kubectl get pods -n free5gc-cp -l name=smf-regional --context regional-admin@regional -o jsonpath='{.items[0].metadata.name}')
kubectl -n free5gc-cp logs $SMF_POD --context regional-admin@regional
```

<details>
<summary>The output is similar to:</summary>

```console

```
</details>

## Step 8: Deploy RAN Network Functions


## Step 9: Deploy UE


## Step 10: Test the End to End Connectivity
