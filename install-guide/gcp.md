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
</details>

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
Operation "operations/acat.p2-NNNNNNNNNNNNN-c1aeadbe-3593-48a4-b4a9-e765e18a3009" finished successfully.
```
</details>

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
Default change: VPC-native is the default mode during cluster creation for versions greater than 1.21.0-gke.1500. To create advanced routes based clusters, please pass the `--no-enable-ip-alias` flag
Default change: During creation of nodepools or autoscaling configuration changes for cluster versions greater than 1.24.1-gke.800 a default location policy is applied. For Spot and PVM it defaults to ANY, and for all other VM kinds a BALANCED policy is used. To change the default values use the `--location-policy` flag.
Note: Your Pod address range (`--cluster-ipv4-cidr`) can accommodate at most 1008 node(s).
Creating cluster nephio in us-central1-c... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1/projects/your-nephio-project-id/zones/us-central1-c/clusters/nephio].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-central1-c/nephio?project=your-nephio-project-id
kubeconfig entry generated for nephio.
NAME    LOCATION       MASTER_VERSION  MASTER_IP      MACHINE_TYPE  NODE_VERSION    NUM_NODES  STATUS
nephio  us-central1-c  1.27.3-gke.100  34.xx.xxx.xxx  e2-medium     1.27.3-gke.100  3          RUNNING
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
kpt pkg get --for-deployment https://github.com/johnbelamaric/nephio-gcp-packages.git/gitea@v1.0.1
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

## Common Dependencies

There are a few dependencies that are common across most installations, and do
not require any installation-specific setup. You should install these next, as
described in the [common dependencies documentation](common-dependencies.md).

## Common Components

With the necessary dependencies now installed, you can now install the essential
Nephio components. This is documented in the [common components
documentation](common-components.md).

## GCP Package Repositories

A repository of GCP-installation specific packages must be registered with
Nephio. This repository contains packages derived from the standard Nephio
packages, but with GCP-specific modifications, as well as packages that are used
to integrate with specific GCP functionality.

You can register this package as a read-only external repository by applying the
`gcp-repository` package:

```bash
kpt pkg get --for-deployment https://github.com/johnbelamaric/nephio-gcp-packages.git/gcp-repository@v1.0.1
kpt fn render gcp-repository
kpt live init gcp-repository
kpt live apply gcp-repository --reconcile-timeout=15m --output=table
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

Note that Config Controller clusters are always regional and are not available
in all regions. See the link above for a list of available regions. The Config
Controller creation may take up to fifteen minutes.

<details>
<summary>The output is similar to:</summary>

```console
Create request issued for: [nephio-cc]
Waiting for operation [projects/your-nephio-project-id/locations/us-central1/operations/operation-1693351134043-6041808d31cac-44c9513a-128be132] to complete...done.
Created instance [nephio-cc].
Fetching cluster endpoint and auth data.
kubeconfig entry generated for krmapihost-nephio-cc.
```
</details>

After completing, your `kubectl` context will be pointing to the Config
Controller cluster:

```bash
kubectl config get-contexts
```

<details>
<summary>The output is similar to:</summary>

```console
CURRENT   NAME                                                          CLUSTER                                                  AUTHINFO                                                 NAMESPACE
          gke_your-nephio-project-id_us-central1-c_nephio               gke_your-nephio-project-id_us-central1-c_nephio               gke_your-nephio-project-id_us-central1-c_nephio
*         gke_your-nephio-project-id_us-central1_krmapihost-nephio-cc   gke_your-nephio-project-id_us-central1_krmapihost-nephio-cc   gke_your-nephio-project-id_us-central1_krmapihost-nephio-cc
```

</details>

If not, you should retrieve the credentials with:

```bash
gcloud anthos config controller get-credentials nephio-cc --location us-central1
```

There is one more step - granting privileges to the CC cluster to manage GCP
resources in this project. With `kubectl` pointing at the CC cluster, retrieve
the service account email address used by CC:

```bash
export SA_EMAIL="$(kubectl get ConfigConnectorContext -n config-control \
    -o jsonpath='{.items[0].spec.googleServiceAccount}' 2> /dev/null)"
echo $SA_EMAIL
```

<details>
<summary>The output is similar to:</summary>
```console
service-NNNNNNNNNNNN@gcp-sa-yakima.iam.gserviceaccount.com
```
</details>

And then grant that service account `roles/editor`, which allows full management
access to the project, except for IAM:

```bash
gcloud projects add-iam-policy-binding $PROJECT \
    --member "serviceAccount:${SA_EMAIL}" \
    --role roles/editor \
    --project $PROJECT
```

<details>
<summary>The output is similar to:</summary>
```console
Updated IAM policy for project [your-nephio-project-id].
bindings:
- members:
  - serviceAccount:NNNNNNNNNNNNN@cloudbuild.gserviceaccount.com
  role: roles/cloudbuild.builds.builder
- members:
  - serviceAccount:service-NNNNNNNNNNNNN@gcp-sa-cloudbuild.iam.gserviceaccount.com
  role: roles/cloudbuild.serviceAgent
- members:
  - serviceAccount:service-NNNNNNNNNNNNN@compute-system.iam.gserviceaccount.com
  role: roles/compute.serviceAgent
- members:
  - group:admins@example.com
  role: roles/compute.storageAdmin
- members:
  - serviceAccount:service-NNNNNNNNNNNNN@container-engine-robot.iam.gserviceaccount.com
  role: roles/container.serviceAgent
- members:
  - serviceAccount:service-NNNNNNNNNNNNN@containerregistry.iam.gserviceaccount.com
  role: roles/containerregistry.ServiceAgent
- members:
  - serviceAccount:NNNNNNNNNNNNN-compute@developer.gserviceaccount.com
  - serviceAccount:NNNNNNNNNNNNN@cloudservices.gserviceaccount.com
  - serviceAccount:service-NNNNNNNNNNNNN@gcp-sa-yakima.iam.gserviceaccount.com
  role: roles/editor
- members:
  - serviceAccount:service-NNNNNNNNNNNNN@gcp-sa-krmapihosting.iam.gserviceaccount.com
  role: roles/krmapihosting.serviceAgent
- members:
  - serviceAccount:service-NNNNNNNNNNNNN@gcp-sa-yakima.iam.gserviceaccount.com
  - user:your-gcp-account@example.com
  role: roles/owner
- members:
  - serviceAccount:config-sync-sa@your-nephio-project-id.iam.gserviceaccount.com
  role: roles/source.reader
etag: BwYEGPcbq9U=
version: 1
```
</details>

You should now switch your `kubectl` context back to the Nephio management
cluster:

```bash
kubectl config use-context "gke_${PROJECT}_us-central1-c_nephio"
```

## Finishing the GCP Installation

Since you now have the core Nephio pieces up and running, you can use Nephio
itself to complete the installation. The remaining pieces needed are all in a
single package,
[gcp-components](http://github.com/johnbelamaric/nephio-gcp-packages/tree/v1.0.1/gcp-components),
which you can apply to the management repository using a PackageVariant
resource (be sure your `kubectl` context points to the Nephio management
cluster):

```bash
kubectl apply -f - <<EOF
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: gcp-components
spec:
  upstream:
    repo: nephio-gcp-packages
    package: gcp-components
    revision: v1.0.1
  downstream:
    repo: mgmt
    package: gcp-components
  annotations:
    approval.nephio.org/policy: initial
EOF
```

This package contains additional `PackageVariant` resources that deploy all the
remaining GCP Nephio packages including:
- Creation of a `gcp-infra` respository
- The Nephio WebUI, configured to use Google Cloud OAuth 2.0
- A GCP-specific controller for syncing clusters, fleets, and fleet scopes

There is one thing left to do: connect Config Controller to the `gcp-infra`
repository. To do that:

```bash
```
