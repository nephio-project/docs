Exploring the Nephio Sandbox
============================

You've installed the Nephio sandbox on your VM
[using the installation instructions](https://github.com/nephio-project/test-infra/blob/main/e2e/provision/README.md). The installation has done a really good job of installing a pretty complex software stack without any fuss. Let's take a look around.

![Software installed by the Nephio Sandbox installation](ExploreSandbox-diagrams/ManagementCLuster.png)

# Components installed on the VM itself

The following components are installed on the VM itself. These components are installed directly on the VM by the ansible install scripts.

| Component | Purpose                                                                                  |
| --------- | ---------------------------------------------------------------------------------------- |
| docker    | Docker is used to host kubernetes clusters created by kind                               |
| kind      | Used to create clusters in docker                                                        |
| kubectl   | Used to control clusters created by kind                                                 |
| kpt       | Used to install packages (software and metadata) on k8s clusters                         |
| cni       | Used to implement the k8s network model for the kind clusters                            |
| gtp5g     | A linux module that supports the 3GPP GPRS tunnelling protocol (required by free5gc NFs) |

The ansible install scripts use kind to create the management cluster. Once the kind cluster is created, the install uses kpt packages to install the remainder of the software.

# Components installed on the kind management cluster

Everything is installed on the management kind cluster by ansible scripts using kpt packages.

The install unpacks each kpt package in the */tmp* directory. It then applies kpt functions to the packages, and applies the packages to the kind management cluster. This is important because we can check the status of the kpt packages in the cluster using the *kpt live status* command on the unpacked packages in the */tmp* directory.

The rendered kpt packages containing components are unpacked in the */tmp/kpt-pkg* directory. The rendered kpt packages that create the *mgmt* and *mgmt-staging* repos are unpacked in the */tmp/repository* directory and the rendered kpt package containing the rootsync configuration for the *mgmt* repository is unpacked in the */tmp/rootsync* directory. You can examine the contents of any rendered kpt packager by examining the contents of these directories.

```
/tmp/kpt-pkg/                           /tmp/repository     /tmp/rootsync/
├── cert-manager                        ├── mgmt            └── mgmt
├── cluster-capi                        └── mgmt-staging
├── cluster-capi-infrastructure-docker
├── cluster-capi-kind-docker-templates
├── configsync
├── gitea
├── metallb
├── metallb-sandbox-config
├── nephio-controllers
├── nephio-stock-repos
├── nephio-webui
├── porch-dev
└── resource-backend

```
<details>
 <summary>You can check the status of an applied kpt package using a *kpt live status \<kpt-package-dir\>* command.</summary>
```
sudo kpt live status /tmp/kpt-pkg/nephio-controllers/
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-approval-role is Current: Resource is current
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-bootstrap-role is Current: Resource is current
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-controller-role is Current: Resource is current
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-network-role is Current: Resource is current
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-porch-role is Current: Resource is current
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-repository-role is Current: Resource is current
inventory-38069595/clusterrole.rbac.authorization.k8s.io//nephio-controller-token-role is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-approval-role-binding is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-bootstrap-role-binding is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-controller-role-binding is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-network-role-binding is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-porch-role-binding is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-repository-role-binding is Current: Resource is current
inventory-38069595/clusterrolebinding.rbac.authorization.k8s.io//nephio-controller-token-role-binding is Current: Resource is current
inventory-38069595/deployment.apps/nephio-system/nephio-controller is Current: Deployment is available. Replicas: 1
inventory-38069595/deployment.apps/nephio-system/token-controller is Current: Deployment is available. Replicas: 1
inventory-38069595/role.rbac.authorization.k8s.io/nephio-system/nephio-controller-leader-election-role is Current: Resource is current
inventory-38069595/rolebinding.rbac.authorization.k8s.io/nephio-system/nephio-controller-leader-election-role-binding is Current: Resource is current
inventory-38069595/serviceaccount/nephio-system/nephio-controller is Current: Resource is current
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//networks.config.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//clustercontexts.infra.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//networkconfigs.infra.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//networks.infra.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//repositories.infra.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//tokens.infra.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//workloadclusters.infra.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//capacities.req.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//datanetworknames.req.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//datanetworks.req.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//interfaces.req.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//amfdeployments.workload.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//smfdeployments.workload.nephio.org is Current: CRD is established
inventory-38069595/customresourcedefinition.apiextensions.k8s.io//upfdeployments.workload.nephio.org is Current: CRD is established
inventory-38069595/namespace//nephio-system is Current: Resource is current
```
</details>

## Base Components

The following base components are installed on the kind management cluster. Base components are standard components in the k8s ecosystem, which Nephio uses out of the box.

| Component    | Purpose                                                            |
| ------------ | -------------------------------------------------------------------|
| Metal LB     | Load balances requests to the cluster                              |
| Cert Manager | Used for certificate management                                    |
| Gitea        | Used to allow creation and management of local git repos by Nephio |
| Postgres     | Used by Gitea to store repositories                                |
| Cluster CAPI | Used deploy kind workload clusters                                 |
| IPAM         | A system used to allocate and manage IP addresses                  |
| VLAM         | A system used to allocate and manage VLANs                         |

## Specific Components

The following specific components are installed on the kind management cluster. Specific components are Nephio components and components from
[Google Container Tools](https://github.com/GoogleContainerTools) that Nephio uses heavily and interacts closely with.

| Component          | Purpose                                                                                                                                                           |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Porch              | Google Container Tools Package Orchestration Server, provides an API that is used by Nephio to work with packages in git repos                                    |
| Configsync         | Google Container Tools configuration synchronization, used by Nephio to deploy configurations from repos on the management cluster out to workload clusters       |
| Nephio Controllers | The Nephio controllers, which implement the Nephio functionality to fetch, manipulate, and deploy NFs                                                             |
| Nephio WebUI       | The Nephio web client                                                                                                                                             |

# Some useful commands

<details>
<summary>
You can query docker to see the docker images running kind clusters:
</summary>

```console
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED      STATUS      PORTS                       NAMES
350b4a7e29f8   kindest/node:v1.27.1   "/usr/local/bin/entr…"   4 days ago   Up 4 days   127.0.0.1:44695->6443/tcp   kind-control-plane
```
</details>

<details>
<summary>Querying kind clusters running after the install produces output similar to:</summary>

```
$ kind get clusters
kind
```
</details>

<details>
<summary>Querying the k8s pods running after the install produces output similar to:</summary>

```
$ kubectl get pods -A
NAMESPACE                           NAME                                                            READY   STATUS    RESTARTS        AGE
backend-system                      resource-backend-controller-6c7cc59945-5kxjq                    2/2     Running   4 (24h ago)     4d21h
capd-system                         capd-controller-manager-c479754b7-kv4c2                         1/1     Running   2 (34h ago)     4d21h
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-bcdfbf4c5-6jgg7       1/1     Running   3 (24h ago)     4d21h
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-b9485b857-cszkj   1/1     Running   1 (4d21h ago)   4d21h
capi-system                         capi-controller-manager-9d9548dc8-59n5t                         1/1     Running   3 (24h ago)     4d21h
cert-manager                        cert-manager-7476c8fcf4-r2f4w                                   1/1     Running   1 (4d21h ago)   4d21h
cert-manager                        cert-manager-cainjector-bdd866bd4-bf95d                         1/1     Running   1 (4d21h ago)   4d21h
cert-manager                        cert-manager-webhook-5655dcfb4b-2ztqj                           1/1     Running   1 (4d21h ago)   4d21h
config-management-monitoring        otel-collector-798c8784bd-t4m4k                                 1/1     Running   1 (4d21h ago)   4d21h
config-management-system            config-management-operator-6946b77565-9ssmd                     1/1     Running   1 (4d21h ago)   4d21h
config-management-system            reconciler-manager-5b5d8557-kffnv                               2/2     Running   2 (4d21h ago)   4d21h
config-management-system            root-reconciler-mgmt-6fdf94dfd4-f5rhs                           4/4     Running   0               4d21h
gitea                               gitea-0                                                         1/1     Running   1 (4d21h ago)   4d21h
gitea                               gitea-memcached-6777864fbd-mhvz5                                1/1     Running   1 (4d21h ago)   4d21h
gitea                               gitea-postgresql-0                                              1/1     Running   1 (4d21h ago)   4d21h
kube-system                         coredns-5d78c9869d-8lxtd                                        1/1     Running   1 (4d21h ago)   4d21h
kube-system                         coredns-5d78c9869d-vlxhr                                        1/1     Running   1 (4d21h ago)   4d21h
kube-system                         etcd-kind-control-plane                                         1/1     Running   1 (4d21h ago)   4d21h
kube-system                         kindnet-tgfwd                                                   1/1     Running   1 (4d21h ago)   4d21h
kube-system                         kube-apiserver-kind-control-plane                               1/1     Running   1 (4d21h ago)   4d21h
kube-system                         kube-controller-manager-kind-control-plane                      1/1     Running   3 (24h ago)     4d21h
kube-system                         kube-proxy-q9q7q                                                1/1     Running   1 (4d21h ago)   4d21h
kube-system                         kube-scheduler-kind-control-plane                               1/1     Running   3 (24h ago)     4d21h
local-path-storage                  local-path-provisioner-6bc4bddd6b-79mmd                         1/1     Running   1 (4d21h ago)   4d21h
metallb-system                      controller-7948676b95-fg7bp                                     1/1     Running   1 (4d21h ago)   4d21h
metallb-system                      speaker-8gpcr                                                   1/1     Running   2 (4d21h ago)   4d21h
nephio-system                       nephio-controller-76db4b45b7-hr4rh                              2/2     Running   4 (4d21h ago)   4d21h
nephio-system                       token-controller-75c98bd77-h657j                                2/2     Running   2 (4d21h ago)   4d21h
nephio-webui                        nephio-webui-7df7bb7c45-zmn9t                                   1/1     Running   1 (4d21h ago)   4d21h
porch-system                        function-runner-5d4f65476d-25l9h                                1/1     Running   1 (4d21h ago)   4d21h
porch-system                        function-runner-5d4f65476d-rkw4p                                1/1     Running   1 (4d21h ago)   4d21h
porch-system                        porch-controllers-646dfb5f6-n8jtx                               1/1     Running   4 (4d21h ago)   4d21h
porch-system                        porch-server-69445b4d58-rkkrn                                   1/1     Running   62 (77m ago)    4d21h
resource-group-system               resource-group-controller-manager-6c9d56d88-g2h7g               3/3     Running   5 (24h ago)     4d21h
</details>
```
</details>

<details>
<summary>Querying the repositories that exist after the install produces output similar to:</summary>

```
$ kubectl get repositories
NAME                      TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
free5gc-packages          git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
mgmt                      git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging              git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
nephio-example-packages   git    Package   false        True    https://github.com/nephio-project/nephio-example-packages.git
```
</details>