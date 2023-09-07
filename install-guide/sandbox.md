# Sandbox Nephio Installation

*Work-in-Progress*

In this guide, you will set up Nephio running in a single VM with:
- **Management Cluster**: [kind](https://kind.sigs.k8s.io/)
- **Cluster Provisioner**: [Cluster API](https://cluster-api.sigs.k8s.io/)
- **Workload Clusters**: kind
- **Gitops Tool**: ConfigSync
- **Git Provider**: Gitea running in the Nephio management cluster will be the
  git provider for cluster deployment repositories. Some external repositories
  will be on GitHub.
- **Web UI Auth**: None
- **Ingress/Load Balancer**: [MetalLB](https://metallb.universe.tf/), but only internally to the VM.

## Provisioning Your Sandbox VM

In addition to the general prerequisites, you will need:

* Access to a Virtual Machine provided by an hypervisor ([VirtualBox](https://www.virtualbox.org/), [Libvirt](https://libvirt.org/)) and running an OS supported by Nephio (Ubuntu 20.04/22.04, Fedora 34) with a minimum of 8vCPUs and 8 GB in RAM.
* [Kubernetes IN Docker](https://kind.sigs.k8s.io/) (`kind`) installed and set up your workstation.

## Provisioning Your Management Cluster

The Cluster API services require communication with the Docker socket for creation of workload clusters. The command below creates an All-in-One Nephio management cluster through the KinD tool, mapping the `/var/run/docker.sock` socket file for Cluster API communication.

```bash
cat << EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.27.1
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
EOF
```

## Gitea Installation

While you may use other Git providers as well, Gitea is required in the R1
setup. To install Gitea, use `kpt`. From your `nephio-install` directory, run:

Gitea package has services that use front-end and database services, these services require authentication information.
This information needs to be created as secret resources that belong to the `gitea` namespace.

```bash
kubectl create namespace gitea
kubectl create secret generic gitea-postgresql -n gitea \
    --from-literal=postgres-password=secret \
    --from-literal=password=secret
kubectl label secret -n gitea gitea-postgresql app.kubernetes.io/name=postgresql
kubectl label secret -n gitea gitea-postgresql app.kubernetes.io/instance=gitea
kubectl create secret generic git-user-secret -n gitea \
    --type='kubernetes.io/basic-auth' \
    --from-literal=username=nephio \
    --from-literal=password=secret
```

Once those resources are created, you can proceed to install the gitea package.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages/gitea@v1.0.1
kpt fn render gitea
kpt live init gitea
kpt live apply gitea --reconcile-timeout 15m --output=table
```

## Common Dependencies

There are a few dependencies that are common across most installations, and do
not require any installation-specific setup. You should install these next, as
described in the [common dependencies documentation](common-dependencies.md).

## Common Components

With the necessary dependencies now installed, you can now install the essential
Nephio components. This is documented in the [common components
documentation](common-components.md).

## Provisioning Cluster API

For managing the Kubernetes cluster infrastructure, it is necessary to install [Cluster API project](https://cluster-api.sigs.k8s.io/). This package depends on [cert-manager project](https://cert-manager.io/) to generate certificates.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages/cert-manager@v1.0.1
kpt fn render cert-manager
kpt live init cert-manager
kpt live apply cert-manager --reconcile-timeout 15m --output=table
```

Once `cert-manager` is installed, you can proceed with the installation of Cluster API components

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages/cluster-capi@v1.0.1
kpt fn render cluster-capi
kpt live init cluster-capi
kpt live apply cluster-capi --reconcile-timeout 15m --output=table
```

Cluster API uses infrastructure providers to provision cloud resources required by the clusters. You can manage local
resources with the Docker provider, it can be installed with the followed package.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages/cluster-capi-infrastructure-docker@v1.0.1
kpt fn render cluster-capi-infrastructure-docker
kpt live init cluster-capi-infrastructure-docker
kpt live apply cluster-capi-infrastructure-docker --reconcile-timeout 15m --output=table
```

The last step is required for defining cluster, machine and kubeadmin templates for controller and worker docker machines.
These templates define the kubelet args, etcd and coreDNS configuration and image repository as other things.


```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages/cluster-capi-kind-docker-templates@v1.0.1
kpt fn render cluster-capi-kind-docker-templates
kpt live init cluster-capi-kind-docker-templates
kpt live apply cluster-capi-kind-docker-templates --reconcile-timeout 15m --output=table
```
