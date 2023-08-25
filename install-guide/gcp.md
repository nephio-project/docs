# GCP Nephio Installation

In this guide, you will set up Nephio with:
- **Management Cluster**: GKE Standard with auto scaling enabled
- **Cluster Provisioner**: Kubernetes Config Connector (KCC), hosted as a
  managed service via Anthos Config Controller (CC).
- **Workload Clusters**: GKE, optionally with the Network Function Optimization
  feature enabled
- **Gitops Tool**: ConfigSync
- **Git Provider**: Gitea running in the Nephio management cluster will be the
  git provider for cluster deployment repositories. Some external repositories
  will be on GitHub.
- **Web UI Auth**: Google OAuth 2.0
- **Ingress/Load Balancer**: Gateway API will be used to provide access to the
  Nephio and Gitea Web UIs from you workstation.

Additionally, this guide makes the following simplifying choices:
- All resources (Nephio management cluster, Config Controller, and workload
  clusters) will be in the same GCP project. 
- All clusters attached to the default VPC as their primary VPC.

## Provisioning Your Management Cluster

## Provisioning Config Controller

You can manage GCP infrastructure, including GKE clusters and many other GCP
resources using Kubernetes Config Connector, an open source project from Google.
The easiest way to run it, though, is by using the hosted version running in
[Anthos Config
Controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview).

You can follow the instructions to [create a Config Controller
instance](https://cloud.google.com/anthos-config-management/docs/how-to/config-controller-setup)
in your project.

### Connecting Config Controller To Nephio

In R1, the `mgmt` repository is used to manage workloads in the Nephio
management cluster. Since Cluster API runs in that same cluster in our sandbox
setup, this is repository is sufficient to create clusters. When using Config
Controller, we are using a *separate* cluster from our Nephio management cluster
for managing GCP infrastructure. Thus, we need a repository just for our Config
Controller.

```
 kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/gitea@v1.0.1
 kpt fn render gitea/
 kpt live init gitea/
 kpt live apply gitea/ --reconcile-timeout 15m --output=table
```

If you are using Gitea and the repository provisioning controller, you can
create a `gcp-infra` repository. You will need to configure access from your
Config Controller (which runs as a private GKE cluster attached to your VPC) to
the Gitea instance running in the Nephio management cluster. The details of this
will vary based upon your project and VPC structure.
