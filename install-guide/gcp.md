# GCP Nephio Installation

We can manage GCP infrastructure, including GKE clusters and many other GCP
resources using Kubernetes Config Connector, an open source project from Google.
The easiest way to run it, though, is by using the hosted version running in
[Anthos Config
Controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview),
so that is what we will do.

You can follow the instructions to [create a Config Controller
instance](https://cloud.google.com/anthos-config-management/docs/how-to/config-controller-setup)
in your project.

Note that the project that holds Config Controller (CC) does not need to be the
same project that contains your Nephio management cluster, nor do the workload
clusters need to be in the same project. If using different projects, be sure to
grant the appropriate privileges to Config Controller in the relevant projects
for your workloads.

## Connecting Config Controller To Nephio

In R1, the `mgmt` repository is used to manage workloads in the Nephio
management cluster. Since Cluster API runs in that same cluster in our sandbox
setup, this is repository is sufficient to create clusters. When using Config
Controller, we are using a *separate* cluster from our Nephio management cluster
for managing GCP infrastructure. Thus, we need a repository just for our Config
Controller.

If you are using Gitea and the repository provisioning controller, you can
create a `gcp-infra` repository. You will need to configure access from your
Config Controller (which runs as a private GKE cluster attached to your VPC) to
the Gitea instance running in the Nephio management cluster. The details of this
will vary based upon your project and VPC structure.

For purposes of this setup guide, we will make the following opinionated
decisions:
- All resources (Nephio management cluster, Config Controller, and workload
  clusters) will be in the same GCP project.
- The Nephio and CC clusters are both attached to the default VPC.
- Gitea running in the Nephio management cluster will be the git provider. Gitea
  will be accessible to CC via an internal load balancer IP address (note that
  CC itself by default does not have outbound Internet access).
