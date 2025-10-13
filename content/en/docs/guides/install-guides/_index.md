---
title: Nephio installation guides
description: >
  The following sections take you through the installation of Nephio in different environments.
weight: 1
---

{{% pageinfo %}}
This page is draft and the separation of the content to different categories is not done. 
{{% /pageinfo %}}

## Introduction

This installation guide will help you to set up and run a Nephio demonstration environment. This
environment is a single virtual machine (VM) that will be used in the exercises to simulate a
topology with a Nephio management cluster and three workload clusters.

## Installing on the Google Compute Engine

### GCE prerequisites

To install Nephio on the Google Compute Engine (GCE), you will need an account in the Google Cloud
Platform (GCP). You will also need to have gcloud installed on your local environment.

### Create a virtual machine on the GCE

```bash
gcloud compute instances create --machine-type e2-standard-16 \
                                    --boot-disk-size 200GB \
                                    --image-family=ubuntu-2204-lts \
                                    --image-project=ubuntu-os-cloud \
                                    --metadata=startup-script-url=https://raw.githubusercontent.com/nephio-project/test-infra/main/e2e/provision/init.sh,nephio-test-infra-branch=main \
                                    nephio-main-e2e

```

{{% alert title="Note" color="primary" %}}

We recommend that you use e2-standard-16. The minimum requirement is e2-standard-8. 

{{% /alert %}}

### Following the progress of the installation on the GCE

To watch the progress of the installation, you need to allow approximately 30 seconds for Nephio to
reach a network-accessible state. Then log in with SSH and investigate the script execution using
tail:

```bash
gcloud compute ssh ubuntu@nephio-main-e2e -- \
                sudo journalctl -u google-startup-scripts.service --follow
```

## Installing on a preprovisioned virtual machine

This installation has been verified on VMs running on vSphere, OpenStack, AWS, and Azure.

### VM prerequisites

Order or create a VM with the following specifications:

- Linux Flavour: Ubuntu-22.04-focal
- a minimum of eight cores and recommended 16 Cores
- 32 GB memory
- a disk size of 200 GB
- a default user with sudo passwordless permissions

**Configuring a route for Kubernetes**

In some installations, the IP range used by Kubernetes in the sandbox can clash with the IP address
that your VPN is using. In such cases, the VM will become unreachable during the sandbox
installation. If this situation arises, add the route detailed below on your VM.

Log on to your VM and run the following commands, replacing *\<interface-name\>* and
*\<interface-gateway-ip\>* with your VM's values:

```bash
sudo bash -c 'cat << EOF > /etc/netplan/99-cloud-init-network.yaml
network:
  ethernets:
    <interface-name>:
      routes:
        - to: 172.18.2.6/32
          via: <interface-gateway-ip>
          metric: 100
  version: 2
EOF'

sudo netplan apply
```

### Kicking off an installation on a virtual machine
The commands set out below use default values for the GitHub path, GitHub branch/tag, username, K8s
context, and so on. See the table of variables below for information on how to set custom
installation parameters and make changes to commands as required.

**Kind cluster**

Log on to your VM and run the following command :

```bash
wget -O - https://raw.githubusercontent.com/nephio-project/test-infra/main/e2e/provision/init.sh |  \
sudo NEPHIO_DEBUG=false   \
     NEPHIO_BRANCH=main \
     NEPHIO_USER=ubuntu   \
     bash
```

**Preinstalled K8s cluster**

Log on to your VM/system and run the following command:

{{% alert title="Note" color="primary" %}}
The VM or system should be able to access the K8S API server via the kubeconfig file and have
Docker installed. You need Docker to run the KRM container functions specified in the *rootsync*
and *repository* packages.
{{% /alert %}}


```bash
wget -O - https://raw.githubusercontent.com/nephio-project/test-infra/main/e2e/provision/init.sh |  \
sudo NEPHIO_DEBUG=false   \
     NEPHIO_BRANCH=main \
     NEPHIO_USER=ubuntu   \
     DOCKERHUB_USERNAME=username \
     DOCKERHUB_TOKEN=password \
     K8S_CONTEXT=kubernetes-admin@kubernetes \
     bash
```

The following environment variables can be used to configure the installation:

| Variable                  | Values           | Default value      | Description                                                                  |
|---------------------------|------------------| -------------------|------------------------------------------------------------------------------|
| *NEPHIO_USER*             | userid           | ubuntu             | This is the user on which the sandbox needs to be installed (the user must have sudo passwordless permissions). |
| *NEPHIO_DEBUG*            | true or false    | false              | This variable controls the debug output from the install.                    |
| *NEPHIO_HOME*             | path             | /home/$NEPHIO_USER | This is the directory into which the install scripts should be checked out.  |
| *RUN_E2E*                 | true or false    | false              | This variable specifies whether or not end-to-end tests should be run.       |
| *DOCKERHUB_USERNAME*      | alpha-num string |                    | This variable specifies the Docker Hub username.                             |
| *DOCKERHUB_TOKEN*         | alpha-num string |                    | This variable specifies the password or token.                               |
| *NEPHIO_REPO*             | URL              | https://github.com/nephio-project/test-infra.git | This variable specifies the URL of the repository to be used for installation. |
| *NEPHIO_BRANCH*           | branch           | main/v4.0.0        | This variable specifies the tag or branch name to use in NEPHIO_REPO         |
| *DOCKER_REGISTRY_MIRRORS* | List of URLs in JSON format |         | This variable specifies the list of Docker registry mirrors in JSON format. If there are no mirrors to be set, then the variable remains empty. Here are two example values: ``["https://docker-registry-remote.mycompany.com", "https://docker-registry-remote2.mycompany.com"]``|
| *K8S_CONTEXT*             | K8s context      | kind-kind          | This variable defines the Kubernetes context for the existing non-kind cluster (gathered from `kubectl config get-contexts`, for example, *kubernetes-admin@kubernetes*). |

### Following the progress of the installation on a virtual machine

Monitor the installation on your terminal.

Log on to your VM using SSH on another terminal. Use the `docker` and `kubectl` commands to monitor
the installation.

## Access to the user interfaces

Once the installation is complete, log in with SSH and forward the port to the user interface (UI)
(7007) and to Gitea’s HTTP interface (3000), if desired:

Using the GCE:

```bash
gcloud compute ssh ubuntu@nephio-main-e2e -- \
                -L 7007:localhost:7007 \
                -L 3000:172.18.0.200:3000 \
                kubectl port-forward --namespace=nephio-webui svc/nephio-webui 7007
```

Using a VM:

```bash
ssh <user>@<vm-address> \
                -L 7007:localhost:7007 \
                -L 3000:172.18.0.200:3000 \
                kubectl port-forward --namespace=nephio-webui svc/nephio-webui 7007
```

You can now navigate to the following URLs:
- [http://localhost:7007/config-as-data](http://localhost:7007/config-as-data), to browse the Nephio
  web UI.
- [http://localhost:3000/nephio](http://localhost:3000/nephio), to browse the Gitea UI.

## Open terminal

You may want a second SSH window open to run `kubectl` commands, and so on, without port forwarding
(which would fail if you tried to open a second SSH connection with that setting).

Using the GCE:

```bash
gcloud compute ssh ubuntu@nephio-main-e2e
```

Using a VM:

```bash
ssh <user>@<vm-address>
```

## Next steps

* Go through the following exercises:
  * [Free5GC Testbed Deployment and E2E testing with UERANSIM](/docs/guides/user-guides/usecase-user-guides/exercise-1-free5gc.md)
  * [OAI Core and RAN Testbed Deployment and E2E testing](/docs/guides/user-guides/usecase-user-guides/exercise-2-oai.md)
* Dig in to the [user guide](/docs/guides/user-guides/_index.md).
* Nephio sandbox environment:
  * Install on preprovisioned single VM.
  * Install on a GCE.
  * [Explore sandbox environment](/docs/guides/install-guides/explore-sandbox.md)
* [Bring-Your-Own-Cluster](/docs/guides/install-guides/install-on-byoc.md) 
* [Install using vagrant on Windows (for development)](/docs/guides/install-guides/demo-vagrant-windows.md) 
