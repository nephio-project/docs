---
title: Installation on GCE
description: >
  Step by step guide to install Nephio on GCE
weight: 3
---

## Introduction

This Installation Guide will set up and run a Nephio demonstration
environment. This environment is a single VM that will be used in the exercises
to simulate a topology with a Nephio management cluster and three workload clusters.

## Installing on GCE

### GCE Prerequisites

You will need an account in GCP and `gcloud` installed on your local environment.

### Create a Virtual Machine on GCE

```bash
gcloud compute instances create --machine-type e2-standard-16 \
                                    --boot-disk-size 200GB \
                                    --image-family=ubuntu-2004-lts \
                                    --image-project=ubuntu-os-cloud \
                                    --metadata=startup-script-url=https://raw.githubusercontent.com/nephio-project/test-infra/v3.0.0/e2e/provision/init.sh,nephio-test-infra-branch=v3.0.0 \
                                    nephio-r3-e2e
```

{{% alert title="Note" color="primary" %}}

e2-standard-16 is recommended and e2-standard-8 is minimum.

{{% /alert %}}

### Follow the Installation on GCE

If you want to watch the progress of the installation, give it about 30
seconds to reach a network accessible state, and then ssh in and tail the
startup script execution:

```bash
gcloud compute ssh ubuntu@nephio-r3-e2e -- \
                sudo journalctl -u google-startup-scripts.service --follow
```

## Access to the User Interfaces

Once it is completed, ssh in and port forward the port to the UI (7007) and to
Gitea's HTTP interface, if desired (3000):

```bash
gcloud compute ssh ubuntu@nephio-r3-e2e -- \
                -L 7007:localhost:7007 \
                -L 3000:172.18.0.200:3000 \
                kubectl port-forward --namespace=nephio-webui svc/nephio-webui 7007
```

You can now navigate to:
- [http://localhost:7007/config-as-data](http://localhost:7007/config-as-data) to
browse the Nephio Web UI
- [http://localhost:3000/nephio](http://localhost:3000/nephio) to browse the Gitea UI

## Open Terminal

You will probably want a second ssh window open to run `kubectl` commands, etc.,
without the port forwarding (which would fail if you try to open a second ssh
connection with that setting).

```bash
gcloud compute ssh ubuntu@nephio-r3-e2e
```

## Next Steps

* Step through the exercises
  * [Free5GC Testbed Deployment and E2E testing with UERANSIM](/content/en/docs/guides/user-guides/exercise-1-free5gc.md)
  * [OAI Core and RAN Testbed Deployment and E2E testing](/content/en/docs/guides/user-guides/exercise-2-oai.md)
* Dig into the [user guide](/content/en/docs/guides/user-guides/_index.md)
* In case you want to install Nephio on pre-provisioned VMs:
  * [Single VM](/content/en/docs/guides/install-guides/install-on-single-vm.md)
  * [Multiple VM](/content/en/docs/guides/install-guides/install-on-multiple-vm.md) 
  
