# Quick Start Exercises

## Table of Contents

- [Introduction](#introduction)
- [Create the Regional cluster](#step-1-create-the-regional-cluster)
- [Check the Regional cluster installation](#step-2-check-the-regional-cluster-installation)
- [Deploy two Edge clusters](#step-3-deploy-two-edge-clusters)
- [Deploy Free5GC control plane functions](#step-4-deploy-free5Gc-control-plane-functions)
- [Deploy Free5GC Operator in the Workload clusters](#step-5-deploy-free5GC-operator-in-the-workload-clusters)
- [Check Free5GC Operator deployment](#step-6-check-free5GC-operator-deployment)
- [Deploy AMF, SMF and UPF](#step-7-deploy-the-amf-smf-and-upf-nfs)
- [Deploy UERANSIM](#step-8-deploy-UERANSIM)
- [Change the Capacities of the UPF and SMF NFs](#step-9-change-the-capacities-of-the-upf-and-smf-nfs)

## Introduction

Be sure you have followed the [installation
guide](https://github.com/nephio-project/docs/blob/main/install-guide/README.md)
before trying these exercises.

These exercises will take you from a system with only the Nephio Management
cluster setup to a deployment with:
- A Regional cluster
- Two Edge clusters
- Repositories for each cluster, registered with Nephio, and with Config Sync
  set up to pull from those repositories.
- Inter-cluster networking between those clusters
- A complete free5gc deployment including:
  - AUSF, NRF, NSSF, PCF, UDM, UDR running on the Regional cluster and
    communicating via the Kubernetes default network
  - SMF, AMF running on the Regional cluster and attached to the secondary
    Multus networks as needed
  - UPF running on the Edge clusters and attached to the secondary Multus
    networks as needed
  - The free5gc WebUI and MongoDB as supporting services
- A registered subscriber in the free5gc core
- UERANSIM running on the edge01 cluster and simulating a gNB and the subscriber's
  UE

Additionally, you can use Nephio to change the capacity requirement for the UPF and
SMF NFs and see how the free5gc operator translates that into increased memory and
CPU requirements for the underlying workload.

To perform these exercises, you will need:
- Access to the installed demo VM environment and can login as the
  `ubuntu` user to have access to the necessary files.
- Access to the Nephio UI as described in the installation guide

Access to Gitea, used in the demo environment as the Git provider, is
optional. Later in the exercises, we will also access the free5gc Web UI.

## Step 1: Create the Regional cluster

Our e2e topology consists of one Regional cluster, and two Edge clusters.
Let's start by deploying the Regional cluster. In this case, you will use
manual kpt commands to deploy a single cluster. First, check to make sure
that both the mgmt and mgmt-staging repositories are in the Ready state.
The mgmt repository is used to manage the contents of the Management
cluster via Nephio; the mgmt-staging repository is just used internally
during the cluster bootstrapping process.

Use the session just started on the VM to run these commands:

```bash
kubectl get repositories
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                      TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
free5gc-packages          git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
mgmt                      git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging              git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
nephio-example-packages   git    Package   false        True    https://github.com/nephio-project/nephio-example-packages.git
```
</details>

Since those are Ready, you can deploy a package from the
nephio-example-packages repository into the mgmt repository. To do this, you
retrieve the Package Revision name using `kpt alpha rpkg get`, clone
that specific Package Revision via the `kpt alpha rpkg clone` command, then
propose and approve the resulting package revision. You want to use the latest
revision of the nephio-workload-cluster package, which you can get with the
command below (your latest revision may be different):

```bash
kpt alpha rpkg get --name nephio-workload-cluster
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                                                               PACKAGE                   WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
nephio-example-packages-05707c7acfb59988daaefd85e3f5c299504c2da1   nephio-workload-cluster   main            main       false    Published   nephio-example-packages
nephio-example-packages-781e1c17d63eed5634db7b93307e1dad75a92bce   nephio-workload-cluster   v1              v1         false    Published   nephio-example-packages
nephio-example-packages-5929727104f2c62a2cb7ad805dabd95d92bf727e   nephio-workload-cluster   v2              v2         false    Published   nephio-example-packages
nephio-example-packages-cdc6d453ae3e1bd0b64234d51d575e4a30980a77   nephio-workload-cluster   v3              v3         false    Published   nephio-example-packages
nephio-example-packages-c78ecc6bedc8bf68185f28a998718eed8432dc3b   nephio-workload-cluster   v4              v4         false    Published   nephio-example-packages
nephio-example-packages-46b923a6bbd09c2ab7aa86c9853a96cbd38d1ed7   nephio-workload-cluster   v5              v5         false    Published   nephio-example-packages
nephio-example-packages-17bffe318ac068f5f9ef22d44f08053e948a3683   nephio-workload-cluster   v6              v6         false    Published   nephio-example-packages
nephio-example-packages-0fbaccf6c5e75a3eff7976a523bb4f42bb0118ce   nephio-workload-cluster   v7              v7         false    Published   nephio-example-packages
nephio-example-packages-7895e28d847c0296a204007ed577cd2a4222d1ea   nephio-workload-cluster   v8              v8         true     Published   nephio-example-packages
```
</details>

Then, use the NAME from that in the `clone` operation, and the resulting
PackageRevision name to perform the `propose` and `approve` operations:

```bash
kpt alpha rpkg clone -n default nephio-example-packages-7895e28d847c0296a204007ed577cd2a4222d1ea --repository mgmt regional
```

<details>
<summary>The output is similar to:</summary>

```console
mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 created
```
</details>

Next, you will want to ensure that the new Regional cluster is labeled as regional.
Since you are using the CLI, you will need to pull the package out, modify it, and
then push the updates back to the Draft revision. You will use `kpt` and the
`set-labels` function to do this.

To pull the package to a local directory, use the `rpkg pull` command:

```bash
kpt alpha rpkg pull -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
```

The package is now in the `regional` directory. So you can execute the
`set-labels` function against the package imperatively, using `kpt fn eval`:

```bash
kpt fn eval --image gcr.io/kpt-fn/set-labels:v0.2.0 regional -- "nephio.org/site-type=regional" "nephio.org/region=us-west1"
```

<details>
<summary>The output is similar to:</summary>

```console
[RUNNING] "gcr.io/kpt-fn/set-labels:v0.2.0"
[PASS] "gcr.io/kpt-fn/set-labels:v0.2.0" in 5.5s
    Results:
    [info]: set 7 labels in total
```
</details>

If you wanted to, you could have used the `--save` option to add the
`set-labels` call to the package pipeline. This would mean that function gets
called whenever the server saves the package. If you added new resources
later, they would also get labeled.

In any case, you now can push the package with the labels applied back to the
repository:

```bash
kpt alpha rpkg push -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
```

<details>
<summary>The output is similar to:</summary>

```console
[RUNNING] "gcr.io/kpt-fn/apply-replacements:v0.1.1" 
[PASS] "gcr.io/kpt-fn/apply-replacements:v0.1.1"
```
</details>

Finally, you propose and approve the package.

```bash
kpt alpha rpkg propose -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
```

<details>
<summary>The output is similar to:</summary>

```console
mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 proposed
```
</details>

```bash
kpt alpha rpkg approve -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
```

<details>
<summary>The output is similar to:</summary>

```console
mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 approved
```
</details>

ConfigSync running in the Management cluster will now pull out this new
package, create all the resources necessary to provision a KinD cluster, and
register it with Nephio. This will take about five minutes or so.

## Step 2: Check the Regional cluster installation

You can check if the cluster has been added to the management cluster:

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
NAME       PHASE         AGE     VERSION
regional   Provisioned   52m     v1.26.3
```
</details>

To access the API server of that cluster, you
need to retrieve the `kubeconfig` file by pulling it from the Kubernetes Secret and decode the base64
encoding[^getcapikubeconfig]:

```bash
kubectl get secret regional-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/regional-kubeconfig
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/regional-kubeconfig
```

You can then use it to access the Workload cluster directly:

```bash
kubectl get ns --context regional-admin@regional
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                           STATUS   AGE
config-management-monitoring   Active   3h35m
config-management-system       Active   3h35m
default                        Active   3h39m
kube-node-lease                Active   3h39m
kube-public                    Active   3h39m
kube-system                    Active   3h39m
```
</details>

You should also check that the KinD cluster has come up fully with `kubectl get
machinesets`. You should see READY and AVAILABLE replicas.

```bash
kubectl get machinesets
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                                   CLUSTER    REPLICAS   READY   AVAILABLE   AGE     VERSION
regional-md-0-zhw2j-58d497c498xkz96z   regional   1          1       1           3h58m   v1.26.3
```
</details>

## Step 3: Deploy two Edge clusters

Next, you can deploy two Edge clusters by applying the
PackageVariantSet that can be found in the `tests` directory:

```bash
kubectl apply -f test-infra/e2e/tests/002-edge-clusters.yaml
```

<details>
<summary>The output is similar to:</summary>

```console
packagevariantset.config.porch.kpt.dev/edge-clusters created
```
</details>

This is equivalent to doing the same `kpt` commands used earlier for the Regional
cluster, except that it uses the PackageVariantSet controller, which is
running in the Nephio Management cluster. It will
clone the package for each entry in the field `packageNames` in the
PackageVariantSet. You can observe the progress by looking at the UI, or by
using `kubectl` to monitor the various package variants, package revisions,
and KinD clusters.

To access the API server of these clusters, you will
need to get the `kubeconfig` file. To retrieve the file, you
pull it from the Kubernetes Secret and decode the base64 encoding:

```bash
kubectl get secret edge01-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/edge01-kubeconfig
kubectl get secret edge02-kubeconfig -o jsonpath='{.data.value}' | base64 -d > $HOME/.kube/edge02-kubeconfig
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/regional-kubeconfig:$HOME/.kube/edge01-kubeconfig:$HOME/.kube/edge02-kubeconfig
```

Once the Edge clusters are ready, it is necessary to connect them. For now we
are using the [containerlab tool](https://containerlab.dev/). Eventually, the
inter-cluster networking will be automated as well.

```bash
workers=""
for context in $(kubectl config get-contexts --no-headers --output name | sort); do
    workers+=$(kubectl get nodes -l node-role.kubernetes.io/control-plane!= -o jsonpath='{range .items[*]}"{.metadata.name}",{"\n"}{end}' --context "$context")
done
echo "{\"workers\":[${workers::-1}]}" | tee /tmp/vars.json
sudo containerlab deploy --topo test-infra/e2e/tests/002-topo.gotmpl --vars /tmp/vars.json
```

<details>
<summary>The output is similar to:</summary>

```console
{"workers":["edge01-md-0-5xpjv-d578b7b8bxwph6d-6sv2n","edge02-md-0-fvpvh-99498794cxhfzsn-q5xvl","regional-md-0-p6zbf-586d7b54d8xw6b5x-qv77v"]}
INFO[0000] Containerlab v0.41.2 started
INFO[0000] Parsing & checking topology file: 002-topo.gotmpl
INFO[0000] Could not read docker config: open /root/.docker/config.json: no such file or directory
INFO[0000] Pulling ghcr.io/nokia/srlinux:latest Docker image
INFO[0266] Done pulling ghcr.io/nokia/srlinux:latest
INFO[0266] Creating lab directory: /tmp/test-infra/e2e/clab-free5gc-net
INFO[0268] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24", IPv6Subnet="2001:172:20:20::/64", MTU="1500"
INFO[0271] Creating container: "N6"
INFO[0276] Creating virtual wire: N6:e1-1 <--> edge02-md-0-fvpvh-99498794cxhfzsn-q5xvl:eth1
INFO[0276] Creating virtual wire: N6:e1-2 <--> regional-md-0-p6zbf-586d7b54d8xw6b5x-qv77v:eth1
INFO[0276] Creating virtual wire: N6:e1-0 <--> edge01-md-0-5xpjv-d578b7b8bxwph6d-6sv2n:eth1
INFO[0277] Adding containerlab host entries to /etc/hosts file
+---+--------------------------------------------+--------------+-----------------------+---------------+---------+----------------+--------------------------+
| # |                    Name                    | Container ID |         Image         |     Kind      |  State  |  IPv4 Address  |       IPv6 Address       |
+---+--------------------------------------------+--------------+-----------------------+---------------+---------+----------------+--------------------------+
| 1 | edge01-md-0-5xpjv-d578b7b8bxwph6d-6sv2n    | 44e78769fc1e | kindest/node:v1.26.3  | ext-container | running | 172.18.0.11/16 | fc00:f853:ccd:e793::b/64 |
| 2 | edge02-md-0-fvpvh-99498794cxhfzsn-q5xvl    | 38eb76c0323b | kindest/node:v1.26.3  | ext-container | running | 172.18.0.8/16  | fc00:f853:ccd:e793::8/64 |
| 3 | regional-md-0-p6zbf-586d7b54d8xw6b5x-qv77v | 142a4f0cff7e | kindest/node:v1.26.3  | ext-container | running | 172.18.0.5/16  | fc00:f853:ccd:e793::5/64 |
| 4 | net-free5gc-net-N6                         | 1581d603e174 | ghcr.io/nokia/srlinux | srl           | running | 172.20.20.2/24 | 2001:172:20:20::2/64     |
+---+--------------------------------------------+--------------+-----------------------+---------------+---------+----------------+--------------------------+
```
</details>

We will also need to configure the nodes for the VLANs. Again, this
will be automated in a future release that addresses node setup and
inter-cluster networking. For now, you must run a script that creates them
in each of the worker nodes.

```bash
./test-infra/e2e/provision/hacks/vlan-interfaces.sh
```

<details>
<summary>The output is similar to:</summary>

```console
docker exec "edge01-md-0-znvpq-56ff577758xjgj8b-qbrzh" ip link add link eth1 name eth1.2 type vlan id 2
docker exec "edge01-md-0-znvpq-56ff577758xjgj8b-qbrzh" ip link add link eth1 name eth1.3 type vlan id 3
docker exec "edge01-md-0-znvpq-56ff577758xjgj8b-qbrzh" ip link add link eth1 name eth1.4 type vlan id 4
docker exec "edge01-md-0-znvpq-56ff577758xjgj8b-qbrzh" ip link set up eth1.2
docker exec "edge01-md-0-znvpq-56ff577758xjgj8b-qbrzh" ip link set up eth1.3
docker exec "edge01-md-0-znvpq-56ff577758xjgj8b-qbrzh" ip link set up eth1.4
docker exec "edge02-md-0-kk5rv-6d944f5f4cx8fb4n-42ttj" ip link add link eth1 name eth1.2 type vlan id 2
docker exec "edge02-md-0-kk5rv-6d944f5f4cx8fb4n-42ttj" ip link add link eth1 name eth1.3 type vlan id 3
docker exec "edge02-md-0-kk5rv-6d944f5f4cx8fb4n-42ttj" ip link add link eth1 name eth1.4 type vlan id 4
docker exec "edge02-md-0-kk5rv-6d944f5f4cx8fb4n-42ttj" ip link set up eth1.2
docker exec "edge02-md-0-kk5rv-6d944f5f4cx8fb4n-42ttj" ip link set up eth1.3
docker exec "edge02-md-0-kk5rv-6d944f5f4cx8fb4n-42ttj" ip link set up eth1.4
docker exec "regional-md-0-6hqq6-79bf858cd5xcxzl8-6x9d7" ip link add link eth1 name eth1.2 type vlan id 2
docker exec "regional-md-0-6hqq6-79bf858cd5xcxzl8-6x9d7" ip link add link eth1 name eth1.3 type vlan id 3
docker exec "regional-md-0-6hqq6-79bf858cd5xcxzl8-6x9d7" ip link add link eth1 name eth1.4 type vlan id 4
docker exec "regional-md-0-6hqq6-79bf858cd5xcxzl8-6x9d7" ip link set up eth1.2
docker exec "regional-md-0-6hqq6-79bf858cd5xcxzl8-6x9d7" ip link set up eth1.3
docker exec "regional-md-0-6hqq6-79bf858cd5xcxzl8-6x9d7" ip link set up eth1.4
```
</details>


Finally, we want to configure the resource backend to be aware of these clusters.
The resource backend is an IP address and VLAN index management system. It is
included for demonstration purposes to show how Nephio package specialization
can interact with external systems to fully configure packages. But it needs to
be configured to match our topology.

First, we will apply a package to define the high-level networks for attaching our
workloads. The Nephio package specialization pipeline will
determine the exact VLAN tags and IP addresses for those attachments based on
the specific clusters. There is a predefined PackageVariant in the tests
directory for this:

```bash
kubectl apply -f test-infra/e2e/tests/003-network.yaml
```

<details>
<summary>The output is similar to:</summary>

```console
packagevariant.config.porch.kpt.dev/network created
```
</details>

Then we will create appropriate `Secret` to make sure that Nephio can authenticate to the external backend.

```bash
kubectl apply -f test-infra/e2e/tests/003-secret.yaml
```

<details>
<summary>The output is similar to:</summary>

```console
secret/srl.nokia.com created
```
</details>

The predefined PackageVariant package defines certain resources that exist for the entire topology.
However, we also need to configure the resource backend for our particular
topology. This will likely be automated in the future, but for now you can
just directly apply the configuration we have created that matches this test
topology. Within this step also the credentials and information is provided
to configure the network device, that aligns with the topology.

```bash
./test-infra/e2e/provision/hacks/network-topo.sh

kubectl apply -f test-infra/e2e/tests/003-network-topo.yaml
```

<details>
<summary>The output is similar to:</summary>

```console
rawtopology.topo.nephio.org/nephio created
```
</details>

## Step 4: Deploy Free5GC Control Plane Functions

While the Edge clusters are deploying (which will take 5-10 minutes), you can
install the free5gc functions other than SMF, AMF, and UPF. For this,
you will use the Regional cluster. Since these are all installed with a single
package, you can use the UI to pick the `free5gc-cp` package from the
`free5gc-packages` repository and clone it to the `regional` repository (you
could have also used the CLI).

![Install free5gc - Step 1](free5gc-cp-1.png)

![Install free5gc - Step 2](free5gc-cp-2.png)

![Install free5gc - Step 3](free5gc-cp-3.png)

Click through the "Next" button until you are through all the steps, then
click "Add Deployment". On the next screen, click "Propose", and then
"Approve".

![Install free5gc - Step 4](free5gc-cp-4.png)

![Install free5gc - Step 5](free5gc-cp-5.png)

![Install free5gc - Step 6](free5gc-cp-6.png)

Shortly thereafter, you should see free5gc-cp in the cluster namespace:

```bash
kubectl get ns --context regional-admin@regional
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                           STATUS   AGE
config-management-monitoring   Active   28m
config-management-system       Active   28m
default                        Active   28m
free5gc-cp                     Active   3m16s
kube-node-lease                Active   28m
kube-public                    Active   28m
kube-system                    Active   28m
local-path-storage             Active   28m
resource-group-system          Active   27m
```
</details>

And the actual workload resources:

```bash
kubectl -n free5gc-cp get all --context regional-admin@regional
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                                 READY   STATUS    RESTARTS   AGE
pod/free5gc-ausf-7d494d668d-k55kb    1/1     Running   0          3m31s
pod/free5gc-nrf-66cc98cfc5-9mxqm     1/1     Running   0          3m31s
pod/free5gc-nssf-668db85d54-gsnqw    1/1     Running   0          3m31s
pod/free5gc-pcf-55d4bfd648-tk9fs     1/1     Running   0          3m31s
pod/free5gc-udm-845db6c9c8-54tfb     1/1     Running   0          3m31s
pod/free5gc-udr-79466f7f86-wh5bt     1/1     Running   0          3m31s
pod/free5gc-webui-84ff8c456c-g7q44   1/1     Running   0          3m31s
pod/mongodb-0                        1/1     Running   0          3m31s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/ausf-nausf      ClusterIP   10.131.151.99    <none>        80/TCP           3m32s
service/mongodb         ClusterIP   10.139.208.189   <none>        27017/TCP        3m32s
service/nrf-nnrf        ClusterIP   10.143.64.94     <none>        8000/TCP         3m32s
service/nssf-nnssf      ClusterIP   10.130.139.231   <none>        80/TCP           3m31s
service/pcf-npcf        ClusterIP   10.131.19.224    <none>        80/TCP           3m31s
service/udm-nudm        ClusterIP   10.128.13.118    <none>        80/TCP           3m31s
service/udr-nudr        ClusterIP   10.137.211.80    <none>        80/TCP           3m31s
service/webui-service   NodePort    10.140.177.70    <none>        5000:30500/TCP   3m31s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/free5gc-ausf    1/1     1            1           3m31s
deployment.apps/free5gc-nrf     1/1     1            1           3m31s
deployment.apps/free5gc-nssf    1/1     1            1           3m31s
deployment.apps/free5gc-pcf     1/1     1            1           3m31s
deployment.apps/free5gc-udm     1/1     1            1           3m31s
deployment.apps/free5gc-udr     1/1     1            1           3m31s
deployment.apps/free5gc-webui   1/1     1            1           3m31s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/free5gc-ausf-7d494d668d    1         1         1       3m31s
replicaset.apps/free5gc-nrf-66cc98cfc5     1         1         1       3m31s
replicaset.apps/free5gc-nssf-668db85d54    1         1         1       3m31s
replicaset.apps/free5gc-pcf-55d4bfd648     1         1         1       3m31s
replicaset.apps/free5gc-udm-845db6c9c8     1         1         1       3m31s
replicaset.apps/free5gc-udr-79466f7f86     1         1         1       3m31s
replicaset.apps/free5gc-webui-84ff8c456c   1         1         1       3m31s

NAME                       READY   AGE
statefulset.apps/mongodb   1/1     3m31s
```
</details>

## Step 5: Deploy Free5GC Operator in the Workload clusters

Now you will need to deploy the free5gc operator across all of the Workload
clusters (regional and edge). To do this, you use another PackageVariantSet.
This one uses an objectSelector to select the WorkloadCluster resources
previously added to the Management cluster when you had deployed the
nephio-workload-cluster packages (manually as well as via
PackageVariantSet).

```bash
kubectl apply -f test-infra/e2e/tests/004-free5gc-operator.yaml
```

<details>
<summary>The output is similar to:</summary>

```console
packagevariantset.config.porch.kpt.dev/free5gc-operator created
```
</details>

## Step 6: Check Free5GC Operator Deployment

Within five minutes of applying the free5gc Operator YAML file, you should see `free5gc` namespaces on
your regional and edge clusters:

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
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/free5gc-operator-controller-controller-58df9975f4-sglj6   2/2     Running   0          164m

NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/free5gc-operator-controller-controller   1/1     1            1           164m

NAME                                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/free5gc-operator-controller-controller-58df9975f4   1         1         1       164m
```
</details>

## Step 7: Deploy the AMF, SMF and UPF NFs

Finally, you can deploy the individual network functions which the operator will
instantiate. For now, you will use individual PackageVariants targeting the Regional
cluster for each of the AMF and SMF NFs and a PackageVariantSet targeting the
Edge clusters for the UPF NFs. In the future, you could put all of these
resources into yet-another-package - a "topology" package - and deploy them all as a
unit. Or you can use a topology controller to create them. But for now, let's do each
manually.

```bash
kubectl apply -f test-infra/e2e/tests/005-edge-free5gc-upf.yaml
kubectl apply -f test-infra/e2e/tests/006-regional-free5gc-amf.yaml
kubectl apply -f test-infra/e2e/tests/006-regional-free5gc-smf.yaml
```

Free5gc requires that the SMF and AMF NFs be explicitly configured with information
about each UPF. Therefore, the AMF and SMF packages will remain in an "unready"
state until the UPF packages have all been published.

### Check UPF deployment

You can check the UPF logs in edge01 cluster:

```bash
UPF1_POD=$(kubectl get pods -n free5gc-upf -l name=upf-edge01 --context edge01-admin@edge01 -o jsonpath='{.items[0].metadata.name}')
kubectl -n free5gc-upf logs $UPF1_POD --context edge01-admin@edge01
```

<details>
<summary>The output is similar to:</summary>

```console
2023-07-15T09:05:51Z [INFO][UPF][Main] UPF version:
	free5GC version: v3.2.1
	build time:      2023-06-09T16:41:08Z
	commit hash:     4972fffb
	commit time:     2022-06-29T05:46:33Z
	go version:      go1.20.5 linux/amd64
2023-07-15T09:05:51Z [INFO][UPF][Cfg] Read config from [/free5gc/config/upfcfg.yaml]
2023-07-15T09:05:51Z [INFO][UPF][Cfg] ==================================================
2023-07-15T09:05:51Z [INFO][UPF][Cfg] (*factory.Config)(0xc0003c25f0)({
	Version: (string) (len=5) "1.0.3",
	Description: (string) (len=17) "UPF configuration",
	Pfcp: (*factory.Pfcp)(0xc0003d2fc0)({
		Addr: (string) (len=11) "172.1.1.254",
		NodeID: (string) (len=11) "172.1.1.254",
		RetransTimeout: (time.Duration) 1s,
		MaxRetrans: (uint8) 3
	}),
	Gtpu: (*factory.Gtpu)(0xc0003d3170)({
		Forwarder: (string) (len=5) "gtp5g",
		IfList: ([]factory.IfInfo) (len=1 cap=1) {
			(factory.IfInfo) {
				Addr: (string) (len=11) "172.3.0.254",
				Type: (string) (len=2) "N3",
				Name: (string) "",
				IfName: (string) ""
			}
		}
	}),
	DnnList: ([]factory.DnnList) (len=1 cap=1) {
		(factory.DnnList) {
			Dnn: (string) (len=8) "internet",
			Cidr: (string) (len=11) "10.1.0.0/24",
			NatIfName: (string) (len=2) "n6"
		}
	},
	Logger: (*factory.Logger)(0xc000378be0)({
		Enable: (bool) true,
		Level: (string) (len=4) "info",
		ReportCaller: (bool) false
	})
})
2023-07-15T09:05:51Z [INFO][UPF][Cfg] ==================================================
2023-07-15T09:05:51Z [INFO][UPF][Main] Log level is set to [info] level
2023-07-15T09:05:51Z [INFO][UPF][Main] starting Gtpu Forwarder [gtp5g]
2023-07-15T09:05:51Z [INFO][UPF][Main] GTP Address: "172.3.0.254:2152"
2023-07-15T09:05:51Z [INFO][UPF][Buff] buff server started
2023-07-15T09:05:51Z [INFO][UPF][Pfcp][172.1.1.254:8805] starting pfcp server
2023-07-15T09:05:51Z [INFO][UPF][Pfcp][172.1.1.254:8805] pfcp server started
2023-07-15T09:05:51Z [INFO][UPF][Main] UPF started
2023-07-15T09:10:45Z [INFO][UPF][Pfcp][172.1.1.254:8805] handleAssociationSetupRequest
2023-07-15T09:10:45Z [INFO][UPF][Pfcp][172.1.1.254:8805][rNodeID:172.1.0.254] New node
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
2023-07-15T09:08:55Z [INFO][AMF][CFG] config version [1.0.3]
2023-07-15T09:08:55Z [INFO][AMF][Init] AMF Log level is set to [info] level
2023-07-15T09:08:55Z [INFO][LIB][NAS] set log level : info
2023-07-15T09:08:55Z [INFO][LIB][NAS] set report call : false
2023-07-15T09:08:55Z [INFO][LIB][NGAP] set log level : info
2023-07-15T09:08:55Z [INFO][LIB][NGAP] set report call : false
2023-07-15T09:08:55Z [INFO][LIB][FSM] set log level : info
2023-07-15T09:08:55Z [INFO][LIB][FSM] set report call : false
2023-07-15T09:08:55Z [INFO][LIB][Aper] set log level : info
2023-07-15T09:08:55Z [INFO][LIB][Aper] set report call : false
2023-07-15T09:08:55Z [INFO][AMF][App] amf
2023-07-15T09:08:55Z [INFO][AMF][App] AMF version:
	free5GC version: v3.2.1
	build time:      2023-06-09T16:40:22Z
	commit hash:     a3bd5358
	commit time:     2022-05-01T14:58:26Z
	go version:      go1.20.5 linux/amd64
2023-07-15T09:08:55Z [INFO][AMF][Init] Server started
2023-07-15T09:08:55Z [INFO][AMF][Util] amfconfig Info: Version[1.0.3] Description[AMF initial local configuration]
2023-07-15T09:08:55Z [INFO][AMF][NGAP] Listen on 172.2.2.254:38412
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
2023-07-15T09:10:45Z [INFO][SMF][CFG] SMF config version [1.0.2]
2023-07-15T09:10:45Z [INFO][SMF][CFG] UE-Routing config version [1.0.1]
2023-07-15T09:10:45Z [INFO][SMF][Init] SMF Log level is set to [debug] level
2023-07-15T09:10:45Z [INFO][LIB][NAS] set log level : info
2023-07-15T09:10:45Z [INFO][LIB][NAS] set report call : false
2023-07-15T09:10:45Z [INFO][LIB][NGAP] set log level : info
2023-07-15T09:10:45Z [INFO][LIB][NGAP] set report call : false
2023-07-15T09:10:45Z [INFO][LIB][Aper] set log level : info
2023-07-15T09:10:45Z [INFO][LIB][Aper] set report call : false
2023-07-15T09:10:45Z [INFO][LIB][PFCP] set log level : info
2023-07-15T09:10:45Z [INFO][LIB][PFCP] set report call : false
2023-07-15T09:10:45Z [INFO][SMF][App] smf
2023-07-15T09:10:45Z [INFO][SMF][App] SMF version:
	free5GC version: v3.2.1
	build time:      2023-06-09T16:40:53Z
	commit hash:     de70bf6c
	commit time:     2022-06-28T04:52:40Z
	go version:      go1.20.5 linux/amd64
2023-07-15T09:10:45Z [INFO][SMF][CTX] smfconfig Info: Version[1.0.2] Description[SMF configuration]
2023-07-15T09:10:45Z [INFO][SMF][CTX] Endpoints: [172.3.0.254]
2023-07-15T09:10:45Z [INFO][SMF][CTX] Endpoints: [172.3.1.254]
2023-07-15T09:10:45Z [INFO][SMF][Init] Server started
2023-07-15T09:10:45Z [INFO][SMF][Init] SMF Registration to NRF {7011c946-4ca4-45ff-bac6-32116bd93934 SMF REGISTERED 0 0xc0001da168 0xc0001da198 [] []   [smf-regional] [] <nil> [] [] <nil> 0 0 0 area1 <nil> <nil> <nil> <nil> 0xc0000b81c0 <nil> <nil> <nil> <nil> <nil> map[] <nil> false 0xc0001da060 false false []}
2023-07-15T09:10:45Z [INFO][SMF][PFCP] Listen on 172.1.0.254:8805
2023-07-15T09:10:45Z [INFO][SMF][App] Sending PFCP Association Request to UPF[172.1.1.254]
2023-07-15T09:10:45Z [INFO][LIB][PFCP] Remove Request Transaction [1]
2023-07-15T09:10:45Z [INFO][SMF][App] Received PFCP Association Setup Accepted Response from UPF[172.1.1.254]
2023-07-15T09:10:45Z [INFO][SMF][App] Sending PFCP Association Request to UPF[172.1.2.254]
2023-07-15T09:10:45Z [INFO][LIB][PFCP] Remove Request Transaction [2]
2023-07-15T09:10:45Z [INFO][SMF][App] Received PFCP Association Setup Accepted Response from UPF[172.1.2.254]
```
</details>

## Step 8: Deploy UERANSIM

The UERANSIM package can be deployed to the edge01 cluster, where it will
simulate a gNodeB and UE. Just like our other packages, UERANSIM needs to be
configured to attach to the correct networks and use the correct IP address.
Thus, we use our standard specialization techniques and pipeline to deploy
UERANSIM, just like the other network functions.

However, before we do that, let us register the UE with free5gc as a subscriber.
You will use the free5gc Web UI to do this. To access it, you need to open
another port forwarding session. Assuming you have the `regional-kubeconfig`
file you created earlier in your home directory, you need to establish another
ssh session from your workstation to the VM, port forwarding port 5000.

Before moving on to the new terminal, let's copy `regional-kubeconfig` to the home directory:

```bash
cp $HOME/.kube/regional-kubeconfig .
```

Then, use the same form of the command you did for other sessions. For example, for the
generic Ubuntu VM, the command would be:

```bash
ssh <user>@<vm-address> \
                -L 5000:localhost:5000 \
                kubectl --kubeconfig regional-kubeconfig \
                port-forward --namespace=free5gc-cp svc/webui-service 5000
```

You should now be able to navigate to
[http://localhost:5000/](http://localhost:5000/) and access the free5gc WebUI.
The test subscriber is the same as the standard free5gc default subscriber.
Thus, you can follow the
[instructions](https://free5gc.org/guide/New-Subscriber-via-webconsole/) on the
free5gc site, but start at Step 4.

Once the subscriber is registered, we can deploy UERANSIM:

```bash
kubectl apply -f test-infra/e2e/tests/007-edge01-ueransim.yaml
```

You can check to see if the simulated UE is up and running by checking the
UERANSIM deployment. First, you can use the `get_capi_kubeconfig` shortcut to
retrieve the kubeconfig for the edge01 cluster, and then query that cluster for
the UERANSIM pods.

```bash
get_capi_kubeconfig edge01
kubectl --kubeconfig edge01-kubeconfig -n ueransim get po
```

<details>
<summary>The output is similar to:</summary>

```console
NAME                                  READY   STATUS    RESTARTS   AGE
ueransimgnb-edge01-748b45f684-sbs8h   1/1     Running   0          81m
ueransimue-edge01-56fccbc4b6-h42k7    1/1     Running   0          81m
```

</details>

Let's see if we can simulate the UE pinging out to our DNN.

```bash
UE_POD=$(kubectl --kubeconfig edge01-kubeconfig get pods -n ueransim -l app=ueransim -l component=ue -o jsonpath='{.items[0].metadata.name}')
kubectl --kubeconfig edge01-kubeconfig -n ueransim exec -it $UE_POD -- /bin/bash -c "ping -I uesimtun0 1.1.1.1"
```

<details>
<summary>The output is similar to:</summary>

```console
PING 1.1.1.1 (1.1.1.1) from 1.1.1.1 uesimtun0: 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=64 time=0.044 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=64 time=0.047 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=64 time=0.043 ms
^C
--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3061ms
rtt min/avg/max/mdev = 0.043/0.046/0.050/0.002 ms
```

</details>

Note that our DNN does not actually provide access to the internet, so you won't
be able to reach the other sites.

## Step 9: Change the Capacities of the UPF and SMF NFs

In this step, you will change the capacity requirements for the UPF and SMF, and
see how the operator reconfigures the Kubernetes resources used by the network
functions.

## Footnotes
[^capikubeconfig]: The install process sets up a shortcut for this. You can get
    retrieve a cluster's kubeconfig with `get_capi_kubeconfig <clustername>`.
    For example, `get_capi_kubeconfig regional` will create a file
    `regional-kubeconfig` in your home directory.
