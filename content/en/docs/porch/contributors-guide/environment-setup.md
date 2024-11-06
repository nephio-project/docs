---
title: "Setting up a local environment"
type: docs
weight: 2
description:
---

This tutorial gives short instructions on how to set up a development environment for Porch on your local machine. It outlines the steps to
get a [kind](https://kind.sigs.k8s.io/) cluster up and running to which a Porch instance running in Visual Studio Code
can connect to and interact with. If you are not familiar with how porch works, it is highly recommended that you go
through the [Starting with Porch tutorial](../using-porch/install-and-using-porch.md) before going through this one.

{{% alert title="Note" color="primary" %}}

As your Dev environment, you can run the code on a remote VM and use the
[VSCode Remote SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
plugin to connect to it.

{{% /alert %}}

## Extra steps for MacOS users

The script the `make deployment-config` target to generate the deployment files for porch. The scripts called by this
make target use recent `bash` additions. MacOS comes with `bash` 3.x.x

1. Install `bash` 4.x.x or better of `bash` using homebrew, see
   [this this post for details](https://apple.stackexchange.com/questions/193411/update-bash-to-version-4-0-on-osx)
2. Ensure that `/opt/homebrew/bin` is earlier in your path than `/bin` and `/usr/bin`

{{% alert title="Note" color="primary" %}}

The changes above **permanently** change the `bash` version for **all** applications and may cause side
effects. 

{{% /alert %}}


## Setup the environment automatically

The [`./scripts/setup-dev-env.sh`](https://github.com/nephio-project/porch/blob/main/scripts/setup-dev-env.sh) setup
script automatically builds a porch development environment.

{{% alert title="Note" color="primary" %}}

This is only one of many possible ways of building a working porch development environment so feel free
to customize it to suit your needs.

{{% /alert %}}

The setup script will perform the following steps:

1. Install a kind cluster. The name of the cluster is read from the PORCH_TEST_CLUSTER environment variable, otherwise
   it defaults to `porch-test`. The configuration of the cluster is taken from
   [here](https://github.com/nephio-project/porch/blob/main/deployments/local/kind_porch_test_cluster.yaml).
1. Install the MetalLB load balancer into the cluster, in order to allow `LoadBalancer` typed Services to work properly.
1. Install the Gitea git server into the cluster. This can be used to test porch during development, but it is not used
   in automated end-to-end tests. Gitea is exposed to the host via port 3000. The GUI is accessible via
   <http://localhost:3000/nephio>, or <http://172.18.255.200:3000/nephio> (username: nephio, password: secret).
   {{% alert title="Note" color="primary" %}}
   
   If you are using WSL2 (Windows Subsystem for Linux), then Gitea is also accessible from the Windows host via the
   <http://localhost:3000/nephio> URL.
   
   {{% /alert %}}
1. Generate the PKI resources (key pairs and certificates) required for end-to-end tests.
1. Build the porch CLI binary. The result will be generated as `.build/porchctl`.

That's it! If you want to run the steps manually, please use the code of the script as a detailed description.

The setup script is idempotent in the sense that you can rerun it without cleaning up first. This also means that if the
script is interrupted for any reason, and you run it again it should effectively continue the process where it left off.

## Extra manual steps

Copy the `.build/porchctl` binary (that was built by the setup script) to somewhere in your $PATH, or add the `.build`
directory to your PATH.

## Build and deploy porch

You can build all of porch, and also deploy it into your newly created kind cluster with this command.

```bash
make run-in-kind
```

See more advanced variants of this command in the [detailed description of the development process](dev-process.md).

## Check that everything works as expected

At this point you are basically ready to start developing porch, but before you start it is worth checking that
everything works as expected.

### Check that the apiservice is ready

```bash
kubectl get apiservice v1alpha1.porch.kpt.dev
```

Sample output:

```bash
NAME                     SERVICE            AVAILABLE   AGE
v1alpha1.porch.kpt.dev   porch-system/api   True        18m
```

### Check the porch api-resources

```bash
kubectl api-resources | grep porch
```

Sample output:

```bash
packagerevs                                      config.porch.kpt.dev/v1alpha1     true         PackageRev
packagevariants                                  config.porch.kpt.dev/v1alpha1     true         PackageVariant
packagevariantsets                               config.porch.kpt.dev/v1alpha2     true         PackageVariantSet
repositories                                     config.porch.kpt.dev/v1alpha1     true         Repository
functions                                        porch.kpt.dev/v1alpha1            true         Function
packagerevisionresources                         porch.kpt.dev/v1alpha1            true         PackageRevisionResources
packagerevisions                                 porch.kpt.dev/v1alpha1            true         PackageRevision
packages                                         porch.kpt.dev/v1alpha1            true         PorchPackage
```

## Create Repositories using your local Porch server

To connect Porch to Gitea, follow [step 7 in the Starting with Porch](../using-porch/install-and-using-porch.md)
tutorial to create the repositories in Porch.

You will notice logging messages in VSCode when you run the `kubectl apply -f porch-repositories.yaml` command.

You can check that your locally running Porch server has created the repositories by running the `porchctl` command:

```bash
porchctl repo get -A
```

Sample output:

```bash
NAME                  TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
external-blueprints   git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
management            git    Package   false        True    http://172.18.255.200:3000/nephio/management.git
```

You can also check the repositories using kubectl.

```bash
kubectl get  repositories -n porch-demo
```

Sample output:

```bash
NAME                  TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
external-blueprints   git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
management            git    Package   false        True    http://172.18.255.200:3000/nephio/management.git
```

You now have a locally running Porch (api)server. Happy developing!

## Restart from scratch

Sometimes the development cluster gets cluttered and you may experience weird behavior from porch.
In this case you might want to restart with a clean slate, by deleting the development cluster with the following
command:

```bash
kind delete cluster --name porch-test
```

and running the [setup script](https://github.com/nephio-project/porch/blob/main/scripts/setup-dev-env.sh) again:

```bash
./scripts/setup-dev-env.sh
```

## Getting started with actual development

You can find a detailed description of the actual development process [here](dev-process.md).

## Enabling Open Telemetry/Jaeger tracing

### Enabling tracing on a Porch deployment

Follow the steps below to enable Open Telemetry/Jaeger tracing on your Porch deployment.

1. Apply the Porch *deployment.yaml* manifest for Jaeger.

```bash
kubectl apply -f https://raw.githubusercontent.com/nephio-project/porch/refs/heads/main/deployments/tracing/deployment.yaml
```

2. Add the environment variable *OTEL* to the porch-server manifest:

```
kubectl edit deployment -n porch-system porch-server
```

```
env:
- name: OTEL
  value: otel://jaeger-oltp:4317
```

3. Set up port forwarding of the Jaeger http port to your local machine:

```
kubectl port-forward -n porch-system service/jaeger-http 16686
```

4. Open the Jaeger UI in your browser at http://localhost:16686

### Enable tracing on a local Porch server

Follow the steps below to enable Open Telemetry/Jaeger tracing on a porch server running locally on your machine, such as in VSCode.

1. Download the Jaeger binary tarball for your local machine architecture from [the Jaeger download page](https://www.jaegertracing.io/download/#binaries) and untar the tarball in some suitable directory.

2. Run Jaeger:

```
cd jaeger
./jaeger-all-in-one
```

3. Configure the Porch server to output Open Telemetry traces:

Set the `OTEL` environment variable to point at the Jaeger server

In `.vscode/launch.json`:

```
"env": {
   ...
   ...
"OTEL": "otel://localhost:4317",
   ...
   ...
}
```

In a shell:

```
export OTEL="otel://localhost:4317"
```

4. Open the Jaeger UI in your browser at http://localhost:16686

5. Run the Porch Server.

