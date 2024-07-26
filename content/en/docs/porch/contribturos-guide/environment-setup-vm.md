---
title: "Setting up a VM environment"
type: docs
weight: 2
description:
---

This tutorial gives short instructions on how to set up a development environment for Porch on a Nephio VM. It outlines the steps to
get a [kind](https://kind.sigs.k8s.io/) cluster up and running to which a Porch instance running in Visual Studio Code
can connect to and interact with. If you are not familiar with how porch works, it is highly recommended that you go
through the [Starting with Porch tutorial](../using-porch/install-and-using-porch.md) before going through this one.

## Setting up the environment

1. The first step is to install the Nephio sandbox environment on your VM using the procedure described in
[Installation on a single VM](../../guides/install-guides/install-on-single-vm.md). In short, log onto your VM and give the command
below:

```
wget -O - https://raw.githubusercontent.com/nephio-project/test-infra/main/e2e/provision/init.sh |  \
sudo NEPHIO_DEBUG=false   \
     NEPHIO_BRANCH=main \
     NEPHIO_USER=ubuntu   \
     bash
```

2. Set up your VM for development (optional but recommended step).

```
echo ''                                         >> ~/.bashrc
echo 'source <(kubectl completion bash)'        >> ~/.bashrc
echo 'source <(kpt completion bash)'            >> ~/.bashrc
echo 'source <(porchctl completion bash)'       >> ~/.bashrc
echo ''                                         >> ~/.bashrc
echo 'alias h=history'                          >> ~/.bashrc
echo 'alias k=kubectl'                          >> ~/.bashrc
echo ''                                         >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

sudo usermod -a -G syslog ubuntu
sudo usermod -a -G docker ubuntu
```

3. Log out of your VM and log in again so that the group changes on the `ubuntu` user are picked up.

```
> exit

> ssh ubuntu@thevmhostname
> groups
ubuntu adm dialout cdrom floppy sudo audio dip video plugdev syslog netdev lxd docker
```

4. Install `go` so that you can build Porch on the VM:

```
wget -O - https://go.dev/dl/go1.22.5.linux-amd64.tar.gz | sudo tar -C /usr/local -zxvf -

echo ''                                   >> ~/.profile
echo '# set PATH for go'                  >> ~/.profile
echo 'if [ -d "/usr/local/go" ]'          >> ~/.profile
echo 'then'                               >> ~/.profile
echo '    PATH="/usr/local/go/bin:$PATH"' >> ~/.profile
echo 'fi'                                 >> ~/.profile 
```

5. Log out of your VM and log in again so that the `go` is added to your path. Verify that `go` is in the path:

```
> exit

> ssh ubuntu@thevmhostname

> go version
go version go1.22.5 linux/amd64
```

6. Install `go delve` for debugging on the VM:

```
go install -v github.com/go-delve/delve/cmd/dlv@latest
```

7. Clone Porch onto the VM

```
mkdir -p git/github/nephio-project
cd ~/git/github/nephio-project

# Clone porch
git clone https://github.com/nephio-project/porch.git
cd porch
```

8. Change the Kind cluster name in the Porch Makefile to match the Kind cluster name on the VM:

```
sed -i "s/^KIND_CONTEXT_NAME ?= porch-test$/KIND_CONTEXT_NAME ?= "$(kind get clusters)"/" Makefile
```

You have now set up the VM so that it can be used for remove debugging of Porch.

## Getting started with actual development

You can find a detailed description of the actual development process [here](dev-process.md).

