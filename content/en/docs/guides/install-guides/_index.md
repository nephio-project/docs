---
title: Nephio install guides
description: >
  Demonstration Environment Installation
weight: 1
---

## Introduction

This Installation Guide will set up and run a Nephio demonstration environment. This environment is a single VM that
will be used in the exercises to simulate a topology with a Nephio Management cluster, a Regional Workload
cluster, and two Edge Workload clusters.

## Installing on GCE

### GCE Prerequisites

You will need a account in GCP and `gcloud` installed on your local environment.

### Create a Virtual Machine on GCE

```bash
gcloud compute instances create --machine-type e2-standard-16 \
                                    --boot-disk-size 200GB \
                                    --image-family=ubuntu-2004-lts \
                                    --image-project=ubuntu-os-cloud \
                                    --metadata=startup-script-url=https://raw.githubusercontent.com/nephio-project/test-infra/v2.0.0/e2e/provision/init.sh,nephio-test-infra-branch=v2.0.0 \
                                    nephio-r2-e2e
```

**NOTE**: e2-standard-16 is recommended and e2-standard-8 is minimum. 

### Follow the Installation on GCE

If you want to watch the progress of the installation, give it about 30 seconds to reach a network accessible state, and
then ssh in and tail the startup script execution:

```bash
gcloud compute ssh ubuntu@nephio-r2-e2e -- \
                sudo journalctl -u google-startup-scripts.service --follow
```

## Installing on a Pre-Provisioned VM

This install has been verified on VMs running on vSphere, OpenStack, AWS, and Azure.

### VM Prerequisites

Order or create a VM with the following specification:

- Linux Flavour: Ubuntu-20.04-focal
- Minimum 8 cores and recommended 16 Cores
- 32 GB memory
- 200 GB disk size
- Default user with sudo passwordless permissions

**Configure a Route for Kubernetes**

In some installations, the IP range used by Kubernetes in the sandbox can clash with the IP address used by your VPN. In
such cases, the VM will become unreachable during the sandbox installation. If you have this situation, add the route
below on your VM.

Log onto your VM and run the following commands,
replacing **\<interface-name\>** and **\<interface-gateway-ip\>** with your VMs values:

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

### Kick Off an Installation on VM

Log onto your VM and run the following command:

```bash
wget -O - https://raw.githubusercontent.com/nephio-project/test-infra/v2.0.0/e2e/provision/init.sh |  \
sudo NEPHIO_DEBUG=false   \
     NEPHIO_BRANCH=v2.0.0 \
     NEPHIO_USER=ubuntu   \
     bash
```

The following environment variables can be used to configure the installation:

| Variable               | Values           | Default Value      | Description                                                                  |
|------------------------|------------------| -------------------|------------------------------------------------------------------------------|
| NEPHIO_USER            | userid           | ubuntu             | The user to install the sandbox on (must have sudo passwordless permissions) |
| NEPHIO_DEBUG           | false or true    | false              | Controls debug output from the install                                       |
| NEPHIO_HOME            | path             | /home/$NEPHIO_USER | The directory to check out the install scripts into                          |
| RUN_E2E                | false or true    | false              | Specifies whether end-to-end tests should be executed or not                 |
| DOCKERHUB_USERNAME     | alpha-num string |                    | Specifies the dockerhub username                                             |
| DOCKERHUB_TOKEN        | alpha-num string |                    | Specifies the password or token                                              |
| NEPHIO_REPO            | URL              | https://github.com/nephio-project/test-infra.git | URL of the repository to be used for installation |
| NEPHIO_BRANCH          | branch or tag    | main/v2.0.0               | Tag or branch name to use in NEPHIO_REPO                                     |
| DOCKER_REGISTRY_MIRRORS | list of URLs in JSON format |        | List of docker registry mirrors in JSON format, or empty for no mirrors to be set. Example value: ``["https://docker-registry-remote.mycompany.com", "https://docker-registry-remote2.mycompany.com"]`` |

### Follow the Installation on VM

Monitor the installation on your terminal.

Log onto your VM using ssh on another terminal and use commands *docker* and *kubectl* to monitor the installation.

## Access to the User Interfaces

Once it is completed, ssh in and port forward the port to the UI (7007) and to Gitea's HTTP interface, if desired
(3000):

Using GCE:

```bash
gcloud compute ssh ubuntu@nephio-r2-e2e -- \
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

You can now navigate to:
- [http://localhost:7007/config-as-data](http://localhost:7007/config-as-data) to
browse the Nephio Web UI
- [http://localhost:3000/nephio](http://localhost:3000/nephio) to browse the Gitea UI

## Open Terminal

You will probably want a second ssh window open to run `kubectl` commands, etc., without the port forwarding (which
would fail if you try to open a second ssh connection with that setting).

Using GCE:

```bash
gcloud compute ssh ubuntu@nephio-r2-e2e
```

Using a VM:

```bash
ssh <user>@<vm-address>
```

## Next Steps

* Step through the exercises
  * [Free5GC Testbed Deployment and E2E testing with UERANSIM](/content/en/docs/guides/user-guides/exercise-1-free5gc.md)
  * [OAI Core and RAN Testbed Deployment and E2E testing](/content/en/docs/guides/user-guides/exercise-2-oai.md)
* Learn more about the [Nephio demo sandbox](/content/en/docs/guides/install-guides/explore-sandbox.md)
* Dig into the [user guide](/content/en/docs/guides/user-guides/_index.md)
* Create a [Bring-Your-Own-Cluster](/content/en/docs/guides/install-guides/install-on-byoc.md) Nephio Installation
