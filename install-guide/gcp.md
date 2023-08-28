# GCP Nephio Installation

*Work-in-Progress*

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

## Prerequisites

In addition to the general prerequisites, you will need:
- A GCP account. This account should have enough privileges to create projects,
  enable APIs in those projects, and create the necessary resources.
- [Google Cloud CLI](https://cloud.google.com/sdk/docs) (`gcloud`) installed and
  set up on your workstation.

## Setup Your Environment

To make the instructions (and possibly your life) simpler, you can create a
`gcloud` configuration and a project for Nephio.

In the commands below, two environment variables are used. You can set them to
appropriate values for you.

```bash
PROJECT=your-nephio-project-id
ACCOUNT=your-gcp-account@example.com
```

First, create the configuration. You can view and switch between gcloud
configurations with `gcloud config configurations list` and `gcloud config
configurations activate`.

```bash
gcloud config configurations create nephio
```

<details>
<summary>The output is similar to:</summary>

```console
Created [nephio].
Activated [nephio].
```
</details>

Next, set the configuration to use your account.

```bash
gcloud config set account $ACCOUNT
```

<details>
<summary>The output is similar to:</summary>

```console
Updated property [core/account].
```

Now, create a project for your Nephio resources. The instructions here work in
the simplest environments. However, your organization may have specific
processes and method for creating projects see the GCP [project creation
documentation](https://cloud.google.com/resource-manager/docs/creating-managing-projects#creating_a_project)
or consult with the GCP administrators in your organization.

```bash
gcloud projects create $PROJECT
```

<details>
<summary>The output is similar to:</summary>

```console
Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/your-nephio-project-id].
Waiting for [operations/cp.6666041359205885403] to finish...done.
Enabling service [cloudapis.googleapis.com] on project [your-nephio-project-id]...
Operation "operations/acat.p2-971752707070-f5dd29ea-a6c1-424d-ad15-5d563f7c68d1" finished successfully.
```
</details>

Projects must be associated with a billing account, which may be done in the
[console](https://console.cloud.google.com/billing/projects). Again, your
organization may have specific processes and method for selecting and assigning
billing accounts.  See the [project billing account
documentation](https://cloud.google.com/billing/docs/how-to/modify-project#how-to-change-ba),
or consult with the GCP administrators in your organization.

Next, set the new project as the default in your `gcloud` configuration:

```bash
gcloud config set project $PROJECT
```

<details>
<summary>The output is similar to:</summary>

```console
Updated property [core/project].
```
</details>

Next, enable the GCP services you will need:

```bash
gcloud services enable krmapihosting.googleapis.com \
    container.googleapis.com  \
    cloudresourcemanager.googleapis.com \
    serviceusage.googleapis.com
```

<details>
<summary>The output is similar to:</summary>

```console
Operation "operations/acat.p2-1067498212994-c1aeadbe-3593-48a4-b4a9-e765e18a3009" finished successfully.
```

Your project should now be ready to proceed with the installation.

## Provisioning Your Management Cluster

You can now create your management cluster. The command below sets up a GKE
cluster with a single auto-scaling node pool. This will be used for the Nephio
management workloads, and so there are no specific network function related node
configurations needed. The cluster is a zonal cluster in the `us-central1-c`
zone. Regional clusters are recommended for high availability, so you may wish
to consider using a regional cluster instead. You may also use a different zone,
if you wish.

```
gcloud container clusters create nephio --zone us-central1-c \
  --enable-autoscaling --min-nodes 3 --max-nodes 10 \
  --workload-pool ${PROJECT}.svc.id.goog \
  --enable-managed-prometheus \
  --gateway-api=standard \
  --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver
```

<details>
<summary>The output is similar to:</summary>

```console
```
</details>

Once the management cluster is up and running, `gcloud` will create an
associated `kubectl` context and make it the current context. To double-check
this:

```bash
kubectl config get-contexts
```

<details>
<summary>The output is similar to:</summary>

```console
CURRENT   NAME                                              CLUSTER                                           AUTHINFO                                          NAMESPACE
          gke_some-project_us-central1-c_cluster-1          gke_some-project_us-central1-c_cluster-1          gke_some-project_us-central1-c_cluster-1
*         gke_your-nephio-project-id_us-central1-c_nephio   gke_your-nephio-project-id_us-central1-c_nephio   gke_your-nephio-project-id_us-central1-c_nephio
```
</details>

If the context is present but not current, use:

```bash
kubectl config use-context "gke_${PROJECT}_us-central1-c_nephio"
```

If the context is not present, use:

```bash
gcloud container clusters get-credentials --zone us-central1-c nephio
```

## Gitea Installation

While you may use other Git providers as well, Gitea is required in the R1
setup. To install Gitea, use `kpt`. From your `nephio-install` directory, run:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/gitea@v1.0.1
```

We need to make a few changes. The R1 Gitea package is designed for the sandbox
environment with Metal LB. Let's change that to:
- Use Secrets Manager to manage the Gitea secrets
- Use an internal load balancer for the Gitea git Service so that it is
  accessible in our VPC
- Use Gateway API to expose the Gitea Web UI to the Internet for
  consumption by our workstation


```bash
kpt fn render gitea/
kpt live init gitea/
kpt live apply gitea/ --reconcile-timeout 15m --output=table
```

## Provisioning Config Controller

You can manage GCP infrastructure, including GKE clusters and many other GCP
resources using Kubernetes Config Connector, an open source project from Google.
The easiest way to run it, though, is by using the hosted version running in
[Anthos Config
Controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview).

You can use the commands below, or for additional details, see the instructions
to [create a Config Controller
instance](https://cloud.google.com/anthos-config-management/docs/how-to/config-controller-setup)
in your project. If you follow that guide, do not configure ConfigSync yet; you
will do that later in these instructions, once we have the Gitea repo created.

```bash
gcloud anthos config controller create nephio-cc \
    --location=us-central1 \
    --full-management
```

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
