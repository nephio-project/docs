---
title: "Installing Porch"
type: docs
weight: 1
description: Installing prerequisites, the porchctl CLI and an instance of Porch on a K8s cluster.
---

## Prerequisites

{{% alert color="primary" %}}
Note that Porch and this guide assumes a K8s cluster is set up and a Unix Operating system is used as the basis for operation. These include Linux OS's such as Ubuntu 24.04 LTS or MacOS.

If you are using a Windows OS please install and follow this guide using the [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) layer and ensure the prerequisites are installed on that subsystem.
{{% /alert %}}

Before porch and its CLI can be installed a few prerequisites are required to be present on the system.
These are as follows:

1. [git](https://git-scm.com/) ({{< version_git >}})
2. [Docker](https://www.docker.com/get-started/) - either Docker Desktop or Docker Engine ({{< version_docker >}})
3. [kubectl](https://kubernetes.io/docs/reference/kubectl/) - make sure that [kubectl context](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) configured with your cluster ({{< version_kube >}})
4. [kpt](https://kpt.dev/installation/kpt-cli/) ({{< version_kpt >}})
5. [The go programming language](https://go.dev/) ({{< version_go >}})

## Installing the porchctl CLI

The porchctl CLI can be obtained through the following means:

### Download the latest porchctl binary

<!-- linkchecker-disable -->
{{< tabpane lang="bash" >}}
{{< tab header="Linux AMD64" >}}
curl -LO "https://github.com/nephio-project/porch/releases/download/v{{% latestTag %}}/porchctl_{{% latestTag %}}_linux_amd64.tar.gz"
{{< /tab >}}
{{< tab header="Linux ARM64" >}}
curl -LO "https://github.com/nephio-project/porch/releases/download/v{{% latestTag %}}/porchctl_{{% latestTag %}}_linux_arm64.tar.gz"
{{< /tab >}}
{{< tab header="macOS AMD64" >}}
curl -LO "https://github.com/nephio-project/porch/releases/download/v{{% latestTag %}}/porchctl_{{% latestTag %}}_darwin_amd64.tar.gz"
{{< /tab >}}
{{< tab header="macOS ARM64" >}}
curl -LO "https://github.com/nephio-project/porch/releases/download/v{{% latestTag %}}/porchctl_{{% latestTag %}}_darwin_arm64.tar.gz"
{{< /tab >}}
{{< /tabpane >}}
<!-- linkchecker-enable -->

{{% alert color="primary" title="Note:" %}}
To download a specific version of porch and its porchctl binary you can do so by replacing the version number and machine type its for in the curl link above.

For example, to download the **[1.5.0](https://github.com/nephio-project/porch/releases/tag/v1.5.0)** release version of porch on **macOS AMD64** the URL would be:

```bash
curl -LO "https://github.com/nephio-project/porch/releases/download/v1.5.0/porchctl_1.5.0_darwin_amd64.tar.gz"
```

{{% /alert %}}

### Install the porchctl binary

This extracts the tar file containting the binary executable and installs it into the root binary directory of the machine.

{{% alert color="primary" title="Note:" %}}
That this requires **root** permissions on the host machine.
{{% /alert %}}

```bash
tar -xzf porchctl_{{% latestTag %}}_linux_amd64.tar.gz | sudo install -o root -g root -m 0755 porchctl /usr/local/bin/
```

{{% alert color="primary" title="Note:" %}}
If you do not have root access on the target system, you can still install porchctl to the `~/.local/bin` directory:
{{% /alert %}}

```bash
tar -xzf porchctl_{{% latestTag %}}_linux_amd64.tar.gz
chmod +x ./porchctl
mkdir -p ~/.local/bin
mv ./porchctl ~/.local/bin/porchctl
# and then append (or prepend) ~/.local/bin to $PATH
```

You can test that the CLI has been installed correctly by doing `porchctl version` in your terminal and you should be prompted with a printout that looks similar to this.

```bash
Version: {{% latestTag %}}
Git commit: cddc13bdcd569141142e2b632f09eb7a3e4988c9 (dirty)
```

### Enable porchctl autocompletion (optional)

Create the completions directory (if it doesn’t already exist):

```bash
mkdir -p ~/.local/share/bash-completion/completions
```

{{% alert color="primary" title="Note:" %}}
This is the auto-completion directory for Ubuntu 24.04 LTS and a few other distributions.
Please do your due diligence and use/create the directory for your appropriate OS/distribution.
{{% /alert %}}

Generate and install the completion script:

```bash
porchctl completion bash > ~/.local/share/bash-completion/completions/porchctl
```

Reload your shell:

```bash
exec bash
```

{{% alert color="primary" title="Note:" %}}
You can just reload/refresh your terminal manually without the command by just closing the terminal and starting a new one. Either works as intended.
{{% /alert %}}

Test that the auto-completion works with the following command and pressing the auto-complete key usually `<TAB>` twice.

```bash
porchctl
```

If auto-completion is working as correctly this should return a similar output to the one below

```bash
completion  (Generate the autocompletion script for the specified shell)
help        (Help about any command)
repo        (Manage package repositories.)
rpkg        (Manage packages.)
version     (Print the version number of porchctl)
```

## Deploying Porch on a cluster

Create a new directory for the kpt package and path inside of it

```bash
mkdir porch-{{% latestTag %}} && cd porch-{{% latestTag %}}
```

Download the latest Porch kpt package blueprint

```bash
curl -LO "https://github.com/nephio-project/porch/releases/download/v{{% latestTag %}}/porch_blueprint.tar.gz"
```

Extract the Porch kpt package contents

```bash
tar -xzf porch_blueprint.tar.gz
```

Initialize and apply the Porch kpt package

```bash
kpt live init && kpt live apply
```

You can check that porch is up and running by doing

```bash
kubectl get all -n porch-system
```

A healthy porch install should look as such

```bash
NAME                                   READY   STATUS    RESTARTS   AGE
pod/function-runner-567ddc76d-7k8sj    1/1     Running   0          4m3s
pod/function-runner-567ddc76d-x75lv    1/1     Running   0          4m3s
pod/porch-controllers-d8dfccb4-8lc6j   1/1     Running   0          4m3s
pod/porch-server-7dc5d7cd4f-smhf5      1/1     Running   0          4m3s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)            AGE
service/api               ClusterIP   10.96.108.221   <none>        443/TCP,8443/TCP   4m3s
service/function-runner   ClusterIP   10.96.237.108   <none>        9445/TCP           4m3s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/function-runner     2/2     2            2           4m3s
deployment.apps/porch-controllers   1/1     1            1           4m3s
deployment.apps/porch-server        1/1     1            1           4m3s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/function-runner-567ddc76d    2         2         2       4m3s
replicaset.apps/porch-controllers-d8dfccb4   1         1         1       4m3s
replicaset.apps/porch-server-7dc5d7cd4f      1         1         1       4m3s
```