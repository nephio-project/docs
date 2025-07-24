---
title: "Install and use Porch"
type: docs
weight: 1
description: "A tutorial to install and use Porch"
---

This tutorial is a guide to installing and using Porch. It is based on the
[Porch demo produced by Tal Liron of Google](https://github.com/tliron/klab/tree/main/environments/porch-demo). Users
should be comfortable using *git*, *docker*, and *kubernetes*.

See also [the Nephio Learning Resource](https://github.com/nephio-project/docs/blob/main/learning.md) page for
background help and information.

## Prerequisites

The tutorial can be executed on a Linux VM or directly on a laptop. It has been verified to execute on a MacBook Pro M1
machine and an Ubuntu 20.04 VM.

The following software should be installed prior to running through the tutorial:

1. [git](https://git-scm.com/)
2. [Docker](https://www.docker.com/get-started/)
3. [kubectl](https://kubernetes.io/docs/reference/kubectl/) - make sure that [kubectl context](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) configured with your cluster
4. [kind](https://kind.sigs.k8s.io/)
5. [kpt](https://github.com/kptdev/kpt)
6. [The go programming language](https://go.dev/)
7. [Visual Studio Code](https://code.visualstudio.com/download)
8. [VS Code extensions for go](https://code.visualstudio.com/docs/languages/go)

## Clone the repository and cd into the tutorial

```bash
git clone https://github.com/nephio-project/porch.git

cd porch/examples/tutorials/starting-with-porch/
```

## Create the Kind clusters for management and edge1

Create the clusters:

```bash
kind create cluster --config=kind_management_cluster.yaml
kind create cluster --config=kind_edge1_cluster.yaml
```

Output the *kubectl* configuration for the clusters:

```bash
kind get kubeconfig --name=management > ~/.kube/kind-management-config
kind get kubeconfig --name=edge1 > ~/.kube/kind-edge1-config
```

Toggling *kubectl* between the clusters:

```bash
export KUBECONFIG=~/.kube/kind-management-config

export KUBECONFIG=~/.kube/kind-edge1-config
```

## Install MetalLB on the management cluster

Install the MetalLB load balancer on the management cluster to expose services:

```bash
export KUBECONFIG=~/.kube/kind-management-config
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=component=controller \
                --timeout=90s
```

Check the subnet that is being used by the kind network in docker

```bash
docker network inspect kind | grep Subnet
```

Sample output:

```yaml
"Subnet": "172.18.0.0/16",
"Subnet": "fc00:f853:ccd:e793::/64"
```

Edit the *metallb-conf.yaml* file and ensure the spec.addresses range is in the IPv4 subnet being used by the kind network in docker.

```yaml
...
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
...
```

Apply the MetalLB configuration:

```bash
kubectl apply -f metallb-conf.yaml
```

## Deploy and set up Gitea on the management cluster using kpt

Get the *gitea kpt* package:

```bash
export KUBECONFIG=~/.kube/kind-management-config

cd kpt_packages

kpt pkg get https://github.com/nephio-project/catalog/tree/main/distros/sandbox/gitea
```

Comment out the preconfigured IP address from the *gitea/service-gitea.yaml* file in the *gitea kpt* package:

```bash
11c11
<     metallb.universe.tf/loadBalancerIPs: 172.18.0.200
---
>     #    metallb.universe.tf/loadBalancerIPs: 172.18.0.200
```

Now render, init and apply the *gitea kpt* package:

```bash
kpt fn render gitea
kpt live init gitea # You only need to do this command once
kpt live apply gitea
```

Once the package is applied, all the Gitea pods should come up and you should be able to reach the Gitea UI on the
exposed IP Address/port of the Gitea service.

```bash
kubectl get svc -n gitea gitea

NAME    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                       AGE
gitea   LoadBalancer   10.96.243.120   172.18.255.200   22:31305/TCP,3000:31102/TCP   10m
```

The UI is available at http://172.18.255.200:3000 in the example above.

To login to Gitea, use the credentials nephio:secret.

## Create repositories on Gitea for management and edge1

On the Gitea UI, click the **+** opposite **Repositories** and fill in the form for both the *management* and *edge1*
repositories. Use default values except for the following fields:

- Repository Name: "Management" or "edge1"
- Description: Something appropriate
 
Alternatively, we can create the repositories via curl:

```bash
curl -k -H "content-type: application/json" "http://nephio:secret@172.18.255.200:3000/api/v1/user/repos" --data '{"name":"management"}'

curl -k -H "content-type: application/json" "http://nephio:secret@172.18.255.200:3000/api/v1/user/repos" --data '{"name":"edge1"}'
```

Check the repositories:

```bash
 curl -k -H "content-type: application/json" "http://nephio:secret@172.18.255.200:3000/api/v1/user/repos" | grep -Po '"name": *\K"[^"]*"'
```

Now initialize both repositories with an initial commit.

Initialize the *management* repository:

```bash
cd ../repos
git clone http://172.18.255.200:3000/nephio/management
cd management

touch README.md
git init
git checkout -b main
git config user.name nephio
git add README.md

git commit -m "first commit"
git remote remove origin
git remote add origin http://nephio:secret@172.18.255.200:3000/nephio/management.git
git remote -v
git push -u origin main
cd ..
 ```

Initialize the *edge1* repository:

```bash
git clone http://172.18.255.200:3000/nephio/edge1
cd edge1

touch README.md
git init
git checkout -b main
git config user.name nephio
git add README.md

git commit -m "first commit"
git remote remove origin
git remote add origin http://nephio:secret@172.18.255.200:3000/nephio/edge1.git
git remote -v
git push -u origin main
cd ../../
```

## Install Porch

We will use the *Porch Kpt* package from Nephio catalog repository.

```bash
cd kpt_packages

kpt pkg get https://github.com/nephio-project/catalog/tree/main/nephio/core/porch
```

Now we can install porch. We render the *kpt* package and then init and apply it.

```bash
kpt fn render porch
kpt live init porch # You only need to do this command once
kpt live apply porch
```

Check that the Porch PODs are running on the management cluster:

```bash
kubectl get pod -n porch-system
NAME                                 READY   STATUS    RESTARTS   AGE
function-runner-7994f65554-nrzdh     1/1     Running   0          81s
function-runner-7994f65554-txh9l     1/1     Running   0          81s
porch-controllers-7fb4497b77-2r2r6   1/1     Running   0          81s
porch-server-68bfdddbbf-pfqsm        1/1     Running   0          81s
```

Check that the Porch CRDs and other resources have been created:

```bash
kubectl api-resources | grep porch   
packagerevs                                    config.porch.kpt.dev/v1alpha1          true         PackageRev
packagevariants                                config.porch.kpt.dev/v1alpha1          true         PackageVariant
packagevariantsets                             config.porch.kpt.dev/v1alpha2          true         PackageVariantSet
repositories                                   config.porch.kpt.dev/v1alpha1          true         Repository
packagerevisionresources                       porch.kpt.dev/v1alpha1                 true         PackageRevisionResources
packagerevisions                               porch.kpt.dev/v1alpha1                 true         PackageRevision
packages                                       porch.kpt.dev/v1alpha1                 true         Package
```

## Connect the Gitea repositories to Porch

Create a demo namespace:

```bash
kubectl create namespace porch-demo
```

Create a secret for the Gitea credentials in the demo namespace:

```bash
kubectl create secret generic gitea \
    --namespace=porch-demo \
    --type=kubernetes.io/basic-auth \
    --from-literal=username=nephio \
    --from-literal=password=secret
```

Now, define the Gitea repositories in Porch:

```bash
kubectl apply -f porch-repositories.yaml
```

Check that the repositories have been correctly created:

```bash
kubectl get repositories -n porch-demo
NAME                  TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
edge1                 git    Package   true         True    http://172.18.255.200:3000/nephio/edge1.git
external-blueprints   git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
management            git    Package   false        True    http://172.18.255.200:3000/nephio/management.git
```

## Configure configsync on the workload cluster

configsync is installed on the edge1 cluster so that it syncs the contents of the *edge1* repository onto the edge1
workload cluster. We will use the configsync package from Nephio.

```bash
export KUBECONFIG=~/.kube/kind-edge1-config

cd kpt_packages

kpt pkg get https://github.com/nephio-project/catalog/tree/main/nephio/core/configsync
kpt fn render configsync
kpt live init configsync
kpt live apply configsync
```

Check that the configsync PODs are up and running:

```bash
kubectl get pod -n config-management-system
NAME                                          READY   STATUS    RESTARTS   AGE
config-management-operator-6946b77565-f45pc   1/1     Running   0          118m
reconciler-manager-5b5d8557-gnhb2             2/2     Running   0          118m
```

Now, we need to set up a RootSync CR to synchronize the *edge1* repository:

```bash
kpt pkg get https://github.com/nephio-project/catalog/tree/main/nephio/optional/rootsync
```

Edit the *rootsync/package-context.yaml* file to set the name of the cluster/repo we are syncing from/to:

```bash
9c9
<   name: example-rootsync
---
>   name: edge1
```

Render the package. This configures the *rootsync/rootsync.yaml* file in the Kpt package:

```bash
kpt fn render rootsync
```

Edit the *rootsync/rootsync.yaml* file to set the IP address of Gitea and to turn off authentication for accessing
Gitea:

```bash
11c11
<     repo: http://172.18.0.200:3000/nephio/example-cluster-name.git
---
>     repo: http://172.18.255.200:3000/nephio/edge1.git
13,15c13,16
<     auth: token
<     secretRef:
<       name: example-cluster-name-access-token-configsync
---
>     auth: none
> #    auth: token
> #    secretRef:
> #      name: edge1-access-token-configsync
```

Initialize and apply RootSync:

```bash
export KUBECONFIG=~/.kube/kind-edge1-config

kpt live init rootsync # This command is only needed once
kpt live apply rootsync
```

Check that the RootSync CR is created:

```bash
kubectl get rootsync -n config-management-system
NAME    RENDERINGCOMMIT                            RENDERINGERRORCOUNT   SOURCECOMMIT                               SOURCEERRORCOUNT   SYNCCOMMIT                                 SYNCERRORCOUNT
edge1   613eb1ad5632d95c4336894f8a128cc871fb3266                         613eb1ad5632d95c4336894f8a128cc871fb3266                      613eb1ad5632d95c4336894f8a128cc871fb3266   
```

Check that configsync is synchronized with the repository on the management cluster:

```bash
kubectl get pod -n config-management-system -l app=reconciler
NAME                                     READY   STATUS    RESTARTS   AGE
root-reconciler-edge1-68576f878c-92k54   4/4     Running   0          2d17h

kubectl logs -n config-management-system root-reconciler-edge1-68576f878c-92k54 -c git-sync -f

```

The result should be similar to:

```bash
INFO: detected pid 1, running init handler
I0105 17:50:11.472934      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="" "cmd"="git config --global gc.autoDetach false"
I0105 17:50:11.493046      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="" "cmd"="git config --global gc.pruneExpire now"
I0105 17:50:11.513487      15 main.go:473] "level"=0 "msg"="starting up" "pid"=15 "args"=["/git-sync","--root=/repo/source","--dest=rev","--max-sync-failures=30","--error-file=error.json","--v=5"]
I0105 17:50:11.514044      15 main.go:923] "level"=0 "msg"="cloning repo" "origin"="http://172.18.255.200:3000/nephio/edge1.git" "path"="/repo/source"
I0105 17:50:11.514061      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="" "cmd"="git clone -v --no-checkout -b main --depth 1 http://172.18.255.200:3000/nephio/edge1.git /repo/source"
I0105 17:50:11.706506      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git rev-parse HEAD"
I0105 17:50:11.729292      15 main.go:737] "level"=0 "msg"="syncing git" "rev"="HEAD" "hash"="385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:11.729332      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git fetch -f --tags --depth 1 http://172.18.255.200:3000/nephio/edge1.git main"
I0105 17:50:11.920110      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git cat-file -t 385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:11.945545      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git rev-parse 385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:11.967150      15 main.go:726] "level"=1 "msg"="removing worktree" "path"="/repo/source/385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:11.967359      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git worktree prune"
I0105 17:50:11.987522      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git worktree add --detach /repo/source/385295a2143f10a6cda0cf4609c45d7499185e01 385295a2143f10a6cda0cf4609c45d7499185e01 --no-checkout"
I0105 17:50:12.057698      15 main.go:772] "level"=0 "msg"="adding worktree" "path"="/repo/source/385295a2143f10a6cda0cf4609c45d7499185e01" "branch"="origin/main"
I0105 17:50:12.057988      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/385295a2143f10a6cda0cf4609c45d7499185e01" "cmd"="git reset --hard 385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:12.099783      15 main.go:833] "level"=0 "msg"="reset worktree to hash" "path"="/repo/source/385295a2143f10a6cda0cf4609c45d7499185e01" "hash"="385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:12.099805      15 main.go:838] "level"=0 "msg"="updating submodules"
I0105 17:50:12.099976      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/385295a2143f10a6cda0cf4609c45d7499185e01" "cmd"="git submodule update --init --recursive --depth 1"
I0105 17:50:12.442466      15 main.go:694] "level"=1 "msg"="creating tmp symlink" "root"="/repo/source/" "dst"="385295a2143f10a6cda0cf4609c45d7499185e01" "src"="tmp-link"
I0105 17:50:12.442494      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/" "cmd"="ln -snf 385295a2143f10a6cda0cf4609c45d7499185e01 tmp-link"
I0105 17:50:12.453694      15 main.go:699] "level"=1 "msg"="renaming symlink" "root"="/repo/source/" "old_name"="tmp-link" "new_name"="rev"
I0105 17:50:12.453718      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/" "cmd"="mv -T tmp-link rev"
I0105 17:50:12.467904      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git gc --auto"
I0105 17:50:12.492329      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git cat-file -t HEAD"
I0105 17:50:12.518878      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source" "cmd"="git rev-parse HEAD"
I0105 17:50:12.540979      15 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
I0105 17:50:27.553609      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git rev-parse HEAD"
I0105 17:50:27.600401      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git ls-remote -q http://172.18.255.200:3000/nephio/edge1.git refs/heads/main"
I0105 17:50:27.694035      15 main.go:1065] "level"=1 "msg"="no update required" "rev"="HEAD" "local"="385295a2143f10a6cda0cf4609c45d7499185e01" "remote"="385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:27.694159      15 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
I0105 17:50:42.695482      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git rev-parse HEAD"
I0105 17:50:42.733276      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git ls-remote -q http://172.18.255.200:3000/nephio/edge1.git refs/heads/main"
I0105 17:50:42.826422      15 main.go:1065] "level"=1 "msg"="no update required" "rev"="HEAD" "local"="385295a2143f10a6cda0cf4609c45d7499185e01" "remote"="385295a2143f10a6cda0cf4609c45d7499185e01"
I0105 17:50:42.826611      15 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000

.......

I0108 11:04:05.935586      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git rev-parse HEAD"
I0108 11:04:05.981750      15 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git ls-remote -q http://172.18.255.200:3000/nephio/edge1.git refs/heads/main"
I0108 11:04:06.079536      15 main.go:1065] "level"=1 "msg"="no update required" "rev"="HEAD" "local"="385295a2143f10a6cda0cf4609c45d7499185e01" "remote"="385295a2143f10a6cda0cf4609c45d7499185e01"
I0108 11:04:06.079599      15 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
```

## Exploring the Porch resources

We have configured three repositories in Porch:

```bash
kubectl get repositories -n porch-demo
NAME                  TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
edge1                 git    Package   true         True    http://172.18.255.200:3000/nephio/edge1.git
external-blueprints   git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
management            git    Package   false        True    http://172.18.255.200:3000/nephio/management.git
```

A repository is a CR of the Porch Repository CRD. You can examine the *repositories.config.porch.kpt.dev* CRD with
either of the following commands (both of which are rather verbose):

```bash
kubectl get crd -n porch-system repositories.config.porch.kpt.dev -o yaml
kubectl describe crd -n porch-system repositories.config.porch.kpt.dev 
```

You can examine any other CRD using the commands above and changing the CRD name/namespace.

The full list of Nephio CRDs is as below:

```bash
kubectl api-resources --api-group=porch.kpt.dev         
NAME                       SHORTNAMES   APIVERSION               NAMESPACED   KIND
packagerevisionresources                porch.kpt.dev/v1alpha1   true         PackageRevisionResources
packagerevisions                        porch.kpt.dev/v1alpha1   true         PackageRevision
packages                                porch.kpt.dev/v1alpha1   true         Package
```

The PackageRevision CRD is used to keep track of revision (or version) of each package found in the repositories.

```bash
kubectl get packagerevision -n porch-demo
NAME                                                           PACKAGE              WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
external-blueprints-922121d0bcdd56bfa8cae6c375720e2b5f358ab0   free5gc-cp           main            main       false    Published   external-blueprints
external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9   free5gc-cp           v1              v1         true     Published   external-blueprints
external-blueprints-716aae722092dbbb9470e56079b90ad76ec8f0d5   free5gc-operator     main            main       false    Published   external-blueprints
external-blueprints-d65dc89f7a2472650651e9aea90edfcc81a9afc6   free5gc-operator     v1              v1         false    Published   external-blueprints
external-blueprints-9fee880e8fa52066f052c9cae7aac2e2bc1b5a54   free5gc-operator     v2              v2         false    Published   external-blueprints
external-blueprints-91d60ee31d2d0a1a6d5f1807593d5419434accd3   free5gc-operator     v3              v3         false    Published   external-blueprints
external-blueprints-21f19a0641cf520e7dc6268e64c58c2c30c27036   free5gc-operator     v4              v4         false    Published   external-blueprints
external-blueprints-bf2e7522ee92680bd49571ab309e3f61320cf36d   free5gc-operator     v5              v5         true     Published   external-blueprints
external-blueprints-c1b9ecb73118e001ab1d1213e6a2c94ab67a0939   free5gc-upf          main            main       false    Published   external-blueprints
external-blueprints-5d48b1516e7b1ea15830ffd76b230862119981bd   free5gc-upf          v1              v1         true     Published   external-blueprints
external-blueprints-ed97798b46b36d135cf23d813eccad4857dff90f   pkg-example-amf-bp   main            main       false    Published   external-blueprints
external-blueprints-ed744bfdf4a4d15d4fcf3c46fde27fd6ac32d180   pkg-example-amf-bp   v1              v1         false    Published   external-blueprints
external-blueprints-5489faa80782f91f1a07d04e206935d14c1eb24c   pkg-example-amf-bp   v2              v2         false    Published   external-blueprints
external-blueprints-16e2255bd433ef532684a3c1434ae0bede175107   pkg-example-amf-bp   v3              v3         false    Published   external-blueprints
external-blueprints-7689cc6c953fa83ea61283983ce966dcdffd9bae   pkg-example-amf-bp   v4              v4         false    Published   external-blueprints
external-blueprints-caff9609883eea7b20b73b7425e6694f8eb6adc3   pkg-example-amf-bp   v5              v5         true     Published   external-blueprints
external-blueprints-00b6673c438909975548b2b9f20c2e1663161815   pkg-example-smf-bp   main            main       false    Published   external-blueprints
external-blueprints-4f7dfbede99dc08f2b5144ca550ca218109c52f2   pkg-example-smf-bp   v1              v1         false    Published   external-blueprints
external-blueprints-3d9ab8f61ce1d35e264d5719d4b3c0da1ab02328   pkg-example-smf-bp   v2              v2         false    Published   external-blueprints
external-blueprints-2006501702e105501784c78be9e7d57e426d85e8   pkg-example-smf-bp   v3              v3         false    Published   external-blueprints
external-blueprints-c97ed7c13b3aa47cb257217f144960743aec1253   pkg-example-smf-bp   v4              v4         false    Published   external-blueprints
external-blueprints-3bd78e46b014dac5cc0c58788c1820d043d61569   pkg-example-smf-bp   v5              v5         true     Published   external-blueprints
external-blueprints-c3f660848d9d7a4df5481ec2e06196884778cd84   pkg-example-upf-bp   main            main       false    Published   external-blueprints
external-blueprints-4cb00a17c1ee2585d6c187ba4d0211da960c0940   pkg-example-upf-bp   v1              v1         false    Published   external-blueprints
external-blueprints-5903efe295026124e6fea926df154a72c5bd1ea9   pkg-example-upf-bp   v2              v2         false    Published   external-blueprints
external-blueprints-16142d8d23c1b8e868a9524a1b21634c79b432d5   pkg-example-upf-bp   v3              v3         false    Published   external-blueprints
external-blueprints-60ef45bb8f55b63556e7467f16088325022a7ece   pkg-example-upf-bp   v4              v4         false    Published   external-blueprints
external-blueprints-7757966cc7b965f1b9372370a4b382c8375a2b40   pkg-example-upf-bp   v5              v5         true     Published   external-blueprints
```

The PackageRevisionResources resource is an API Aggregation resource that Porch uses to wrap the GET URL for the package
on its repository.

```bash
kubectl get packagerevisionresources  -n porch-demo
NAME                                                           PACKAGE              WORKSPACENAME   REVISION   REPOSITORY            FILES
external-blueprints-922121d0bcdd56bfa8cae6c375720e2b5f358ab0   free5gc-cp           main            main       external-blueprints   28
external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9   free5gc-cp           v1              v1         external-blueprints   28
external-blueprints-716aae722092dbbb9470e56079b90ad76ec8f0d5   free5gc-operator     main            main       external-blueprints   14
external-blueprints-d65dc89f7a2472650651e9aea90edfcc81a9afc6   free5gc-operator     v1              v1         external-blueprints   11
external-blueprints-9fee880e8fa52066f052c9cae7aac2e2bc1b5a54   free5gc-operator     v2              v2         external-blueprints   11
external-blueprints-91d60ee31d2d0a1a6d5f1807593d5419434accd3   free5gc-operator     v3              v3         external-blueprints   14
external-blueprints-21f19a0641cf520e7dc6268e64c58c2c30c27036   free5gc-operator     v4              v4         external-blueprints   14
external-blueprints-bf2e7522ee92680bd49571ab309e3f61320cf36d   free5gc-operator     v5              v5         external-blueprints   14
external-blueprints-c1b9ecb73118e001ab1d1213e6a2c94ab67a0939   free5gc-upf          main            main       external-blueprints   6
external-blueprints-5d48b1516e7b1ea15830ffd76b230862119981bd   free5gc-upf          v1              v1         external-blueprints   6
external-blueprints-ed97798b46b36d135cf23d813eccad4857dff90f   pkg-example-amf-bp   main            main       external-blueprints   16
external-blueprints-ed744bfdf4a4d15d4fcf3c46fde27fd6ac32d180   pkg-example-amf-bp   v1              v1         external-blueprints   7
external-blueprints-5489faa80782f91f1a07d04e206935d14c1eb24c   pkg-example-amf-bp   v2              v2         external-blueprints   8
external-blueprints-16e2255bd433ef532684a3c1434ae0bede175107   pkg-example-amf-bp   v3              v3         external-blueprints   16
external-blueprints-7689cc6c953fa83ea61283983ce966dcdffd9bae   pkg-example-amf-bp   v4              v4         external-blueprints   16
external-blueprints-caff9609883eea7b20b73b7425e6694f8eb6adc3   pkg-example-amf-bp   v5              v5         external-blueprints   16
external-blueprints-00b6673c438909975548b2b9f20c2e1663161815   pkg-example-smf-bp   main            main       external-blueprints   17
external-blueprints-4f7dfbede99dc08f2b5144ca550ca218109c52f2   pkg-example-smf-bp   v1              v1         external-blueprints   8
external-blueprints-3d9ab8f61ce1d35e264d5719d4b3c0da1ab02328   pkg-example-smf-bp   v2              v2         external-blueprints   9
external-blueprints-2006501702e105501784c78be9e7d57e426d85e8   pkg-example-smf-bp   v3              v3         external-blueprints   17
external-blueprints-c97ed7c13b3aa47cb257217f144960743aec1253   pkg-example-smf-bp   v4              v4         external-blueprints   17
external-blueprints-3bd78e46b014dac5cc0c58788c1820d043d61569   pkg-example-smf-bp   v5              v5         external-blueprints   17
external-blueprints-c3f660848d9d7a4df5481ec2e06196884778cd84   pkg-example-upf-bp   main            main       external-blueprints   17
external-blueprints-4cb00a17c1ee2585d6c187ba4d0211da960c0940   pkg-example-upf-bp   v1              v1         external-blueprints   8
external-blueprints-5903efe295026124e6fea926df154a72c5bd1ea9   pkg-example-upf-bp   v2              v2         external-blueprints   8
external-blueprints-16142d8d23c1b8e868a9524a1b21634c79b432d5   pkg-example-upf-bp   v3              v3         external-blueprints   17
external-blueprints-60ef45bb8f55b63556e7467f16088325022a7ece   pkg-example-upf-bp   v4              v4         external-blueprints   17
external-blueprints-7757966cc7b965f1b9372370a4b382c8375a2b40   pkg-example-upf-bp   v5              v5         external-blueprints   17
```

Let's examine the *free5gc-cp v1* package.

The PackageRevision CR name for *free5gc-cp v1* is external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9.

```bash
kubectl get packagerevision -n porch-demo external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9 -o yaml
```

```yaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevision
metadata:
  creationTimestamp: "2023-06-13T13:35:34Z"
  labels:
    kpt.dev/latest-revision: "true"
  name: external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9
  namespace: porch-demo
  resourceVersion: 5fc9561dcd4b2630704c192e89887490e2ff3c61
  uid: uid:free5gc-cp:v1
spec:
  lifecycle: Published
  packageName: free5gc-cp
  repository: external-blueprints
  revision: v1
  workspaceName: v1
status:
  publishTimestamp: "2023-06-13T13:35:34Z"
  publishedBy: dnaleksandrov@gmail.com
  upstreamLock: {}
```

Getting the *PackageRevisionResources* pulls the package from its repository with each file serialized into a name-value
map of resources in it's spec.

<details>
<summary>Open this to see the command and the result</summary>

```bash
kubectl get packagerevisionresources -n porch-demo external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9 -o yaml
```
```yaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevisionResources
metadata:
  creationTimestamp: "2023-06-13T13:35:34Z"
  name: external-blueprints-dabbc422fdf0b8e5942e767d929b524e25f7eef9
  namespace: porch-demo
  resourceVersion: 5fc9561dcd4b2630704c192e89887490e2ff3c61
  uid: uid:free5gc-cp:v1
spec:
  packageName: free5gc-cp
  repository: external-blueprints
  resources:
    Kptfile: |
      apiVersion: kpt.dev/v1
      kind: Kptfile
      metadata:
        name: free5gc-cp
        annotations:
          config.kubernetes.io/local-config: "true"
      info:
        description: this package represents free5gc NFs, which are required to perform E2E conn testing
      pipeline:
        mutators:
          - image: gcr.io/kpt-fn/set-namespace:v0.4.1
            configPath: package-context.yaml
    README.md: "# free5gc-cp\n\n## Description\nPackage representing free5gc control
      plane NFs.\n\nPackage definition is based on [Towards5gs helm charts](https://github.com/Orange-OpenSource/towards5gs-helm),
      \nand service level configuration is preserved as defined there.\n\n### Network
      Functions (NFs)\n\nfree5gc project implements following NFs:\n\n\n| NF | Description
      | local-config |\n| --- | --- | --- |\n| AMF | Access and Mobility Management
      Function | true |\n| AUSF | Authentication Server Function | false |\n| NRF
      | Network Repository Function | false |\n| NSSF | Network Slice Selection Function
      | false |\n| PCF | Policy Control Function | false |\n| SMF | Session Management
      Function | true |\n| UDM | Unified Data Management | false |\n| UDR | Unified
      Data Repository | false |\n\nalso Database and Web UI is defined:\n\n| Service
      | Description | local-config |\n| --- | --- | --- |\n| mongodb | Database to
      store free5gc data | false |\n| webui | UI used to register UE | false |\n\nNote:
      `local-config: true` indicates that this resources won't be deployed to the
      workload cluster\n\n### Dependencies\n\n- `mongodb` requires `Persistent Volume`.
      We need to assure that dynamic PV provisioning will be available on the cluster\n-
      `NRF` should be running before other NFs will be instantiated\n    - all NFs
      packages contain `wait-nrf` init-container\n- `NRF` and `WEBUI` require DB\n
      \   - packages contain `wait-mongodb` init-container\n- `WEBUI` service is exposed
      as `NodePort` \n    - will be used to register UE on the free5gc side\n- Communication
      via `SBI` between NFs and communication with `mongodb` is defined using K8s
      `ClusterIP` services\n    - it forces you to deploy all NFs on a single cluster
      or consider including `service mesh` in a multi-cluster scenario\n\n## Usage\n\n###
      Fetch the package\n`kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] free5gc-cp`\n\nDetails:
      https://kpt.dev/reference/cli/pkg/get/\n\n### View package content\n`kpt pkg
      tree free5gc-cp`\n\nDetails: https://kpt.dev/reference/cli/pkg/tree/\n\n###
      Apply the package\n```\nkpt live init free5gc-cp\nkpt live apply free5gc-cp
      --reconcile-timeout=2m --output=table\n```\n\nDetails: https://kpt.dev/reference/cli/live/\n\n"
    ausf/ausf-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name:
      ausf-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n    app:
      free5gc\ndata:\n  ausfcfg.yaml: |\n    info:\n      version: 1.0.2\n      description:
      AUSF initial local configuration\n\n    configuration:\n      serviceNameList:\n
      \       - nausf-auth\n      \n      sbi:\n        scheme: http\n        registerIPv4:
      ausf-nausf  # IP used to register to NRF\n        bindingIPv4: 0.0.0.0      #
      IP used to bind the service\n        port: 80\n        tls:\n          key:
      config/TLS/ausf.key\n          pem: config/TLS/ausf.pem\n      \n      nrfUri:
      http://nrf-nnrf:8000\n      plmnSupportList:\n        - mcc: 208\n          mnc:
      93\n        - mcc: 123\n          mnc: 45\n      groupId: ausfGroup001\n      eapAkaSupiImsiPrefix:
      false\n\n    logger:\n      AUSF:\n        ReportCaller: false\n        debugLevel:
      info\n"
    ausf/ausf-deployment.yaml: "---\napiVersion: apps/v1\nkind: Deployment\nmetadata:\n
      \ name: free5gc-ausf\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n
      \   project: free5gc\n    nf: ausf\nspec:\n  replicas: 1\n  selector:\n    matchLabels:\n
      \     project: free5gc\n      nf: ausf\n  template:\n    metadata:\n      labels:\n
      \       project: free5gc\n        nf: ausf\n    spec:\n      initContainers:\n
      \     - name: wait-nrf\n        image: towards5gs/initcurl:1.0.0\n        env:\n
      \       - name: DEPENDENCIES\n          value: http://nrf-nnrf:8000\n        command:
      ['sh', '-c', 'set -x; for dependency in $DEPENDENCIES; do while [ $(curl --insecure
      --connect-timeout 1 -s -o /dev/null -w \"%{http_code}\" $dependency) -ne 200
      ]; do echo waiting for dependencies; sleep 1; done; done;']\n      \n      containers:\n
      \     - name: ausf\n        image: towards5gs/free5gc-ausf:v3.1.1\n        imagePullPolicy:
      IfNotPresent\n        securityContext:\n            {}\n        ports:\n        -
      containerPort: 80\n        command: [\"./ausf\"]\n        args: [\"-c\", \"../config/ausfcfg.yaml\"]\n
      \       env:\n          - name: GIN_MODE\n            value: release\n        volumeMounts:\n
      \       - mountPath: /free5gc/config/\n          name: ausf-volume\n        resources:\n
      \           limits:\n              cpu: 100m\n              memory: 128Mi\n
      \           requests:\n              cpu: 100m\n              memory: 128Mi\n
      \     dnsPolicy: ClusterFirst\n      restartPolicy: Always\n\n      volumes:\n
      \     - name: ausf-volume\n        projected:\n          sources:\n          -
      configMap:\n              name: ausf-configmap\n"
    ausf/ausf-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: ausf-nausf
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: ausf
      spec:
        type: ClusterIP
        ports:
          - port: 80
            targetPort: 80
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: ausf
    mongodb/dep-sts.yaml: "---\napiVersion: apps/v1\nkind: StatefulSet\nmetadata:\n
      \ name: mongodb\n  namespace: default\n  labels:\n    app.kubernetes.io/name:
      mongodb\n    app.kubernetes.io/instance: free5gc\n    app.kubernetes.io/component:
      mongodb\nspec:\n  serviceName: mongodb\n  updateStrategy:\n    type: RollingUpdate\n
      \ selector:\n    matchLabels:\n      app.kubernetes.io/name: mongodb\n      app.kubernetes.io/instance:
      free5gc\n      app.kubernetes.io/component: mongodb\n  template:\n    metadata:\n
      \     labels:\n        app.kubernetes.io/name: mongodb\n        app.kubernetes.io/instance:
      free5gc\n        app.kubernetes.io/component: mongodb\n    spec:\n      \n      serviceAccountName:
      mongodb\n      affinity:\n        podAffinity:\n        podAntiAffinity:\n          preferredDuringSchedulingIgnoredDuringExecution:\n
      \           - podAffinityTerm:\n                labelSelector:\n                  matchLabels:\n
      \                   app.kubernetes.io/name: mongodb\n                    app.kubernetes.io/instance:
      free5gc\n                    app.kubernetes.io/component: mongodb\n                namespaces:\n
      \                 - \"default\"\n                topologyKey: kubernetes.io/hostname\n
      \             weight: 1\n        nodeAffinity:\n          \n      securityContext:\n
      \       fsGroup: 1001\n        sysctls: []\n      containers:\n        - name:
      mongodb\n          image: docker.io/bitnami/mongodb:4.4.4-debian-10-r0\n          imagePullPolicy:
      \"IfNotPresent\"\n          securityContext:\n            runAsNonRoot: true\n
      \           runAsUser: 1001\n          env:\n            - name: BITNAMI_DEBUG\n
      \             value: \"false\"\n            - name: ALLOW_EMPTY_PASSWORD\n              value:
      \"yes\"\n            - name: MONGODB_SYSTEM_LOG_VERBOSITY\n              value:
      \"0\"\n            - name: MONGODB_DISABLE_SYSTEM_LOG\n              value:
      \"no\"\n            - name: MONGODB_ENABLE_IPV6\n              value: \"no\"\n
      \           - name: MONGODB_ENABLE_DIRECTORY_PER_DB\n              value: \"no\"\n
      \         ports:\n            - name: mongodb\n              containerPort:
      27017\n          livenessProbe:\n            exec:\n              command:\n
      \               - mongo\n                - --disableImplicitSessions\n                -
      --eval\n                - \"db.adminCommand('ping')\"\n            initialDelaySeconds:
      30\n            periodSeconds: 10\n            timeoutSeconds: 5\n            successThreshold:
      1\n            failureThreshold: 6\n          readinessProbe:\n            exec:\n
      \             command:\n                - bash\n                - -ec\n                -
      |\n                  mongo --disableImplicitSessions $TLS_OPTIONS --eval 'db.hello().isWritablePrimary
      || db.hello().secondary' | grep -q 'true'\n            initialDelaySeconds:
      5\n            periodSeconds: 10\n            timeoutSeconds: 5\n            successThreshold:
      1\n            failureThreshold: 6\n          resources:\n            limits:
      {}\n            requests: {}\n          volumeMounts:\n            - name: datadir\n
      \             mountPath: /bitnami/mongodb/data/db/\n              subPath: \n
      \     volumes:\n  volumeClaimTemplates:\n    - metadata:\n        name: datadir\n
      \     spec:\n        accessModes:\n          - \"ReadWriteOnce\"\n        resources:\n
      \         requests:\n            storage: \"6Gi\"\n"
    mongodb/serviceaccount.yaml: |
      ---
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: mongodb
        namespace: default
        labels:
          app.kubernetes.io/name: mongodb
          app.kubernetes.io/instance: free5gc
      secrets:
        - name: mongodb
    mongodb/svc.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: mongodb
        namespace: default
        labels:
          app.kubernetes.io/name: mongodb
          app.kubernetes.io/instance: free5gc
          app.kubernetes.io/component: mongodb
      spec:
        type: ClusterIP
        ports:
          - name: mongodb
            port: 27017
            targetPort: mongodb
            nodePort: null
        selector:
          app.kubernetes.io/name: mongodb
          app.kubernetes.io/instance: free5gc
          app.kubernetes.io/component: mongodb
    namespace.yaml: |
      apiVersion: v1
      kind: Namespace
      metadata:
        name: example
        labels:
          pod-security.kubernetes.io/warn: "privileged"
          pod-security.kubernetes.io/audit: "privileged"
          pod-security.kubernetes.io/enforce: "privileged"
    nrf/nrf-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name:
      nrf-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n    app:
      free5gc\ndata:\n  nrfcfg.yaml: |\n    info:\n      version: 1.0.1\n      description:
      NRF initial local configuration\n    \n    configuration:\n      MongoDBName:
      free5gc\n      MongoDBUrl: mongodb://mongodb:27017\n\n      serviceNameList:\n
      \       - nnrf-nfm\n        - nnrf-disc\n\n      sbi:\n        scheme: http\n
      \       registerIPv4: nrf-nnrf  # IP used to serve NFs or register to another
      NRF\n        bindingIPv4: 0.0.0.0    # IP used to bind the service\n        port:
      8000\n        tls:\n          key: config/TLS/nrf.key\n          pem: config/TLS/nrf.pem\n
      \     DefaultPlmnId:\n        mcc: 208\n        mnc: 93\n\n    logger:\n      NRF:\n
      \       ReportCaller: false\n        debugLevel: info\n"
    nrf/nrf-deployment.yaml: "---\napiVersion: apps/v1\nkind: Deployment\nmetadata:\n
      \ name: free5gc-nrf\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n
      \   project: free5gc\n    nf: nrf\nspec:\n  replicas: 1\n  selector:\n    matchLabels:\n
      \     project: free5gc\n      nf: nrf\n  template:\n    metadata:\n      labels:\n
      \       project: free5gc\n        nf: nrf\n    spec:\n      initContainers:\n
      \     - name: wait-mongo\n        image: busybox:1.32.0\n        env:\n        -
      name: DEPENDENCIES\n          value: mongodb:27017\n        command: [\"sh\",
      \"-c\", \"until nc -z $DEPENDENCIES; do echo waiting for the MongoDB; sleep
      2; done;\"]\n      containers:\n      - name: nrf\n        image: towards5gs/free5gc-nrf:v3.1.1\n
      \       imagePullPolicy: IfNotPresent\n        securityContext:\n            {}\n
      \       ports:\n        - containerPort: 8000\n        command: [\"./nrf\"]\n
      \       args: [\"-c\", \"../config/nrfcfg.yaml\"]\n        env: \n          -
      name: DB_URI\n            value: mongodb://mongodb/free5gc\n          - name:
      GIN_MODE\n            value: release\n        volumeMounts:\n        - mountPath:
      /free5gc/config/\n          name: nrf-volume\n        resources:\n            limits:\n
      \             cpu: 100m\n              memory: 128Mi\n            requests:\n
      \             cpu: 100m\n              memory: 128Mi\n        readinessProbe:\n
      \         initialDelaySeconds: 0\n          periodSeconds: 1\n          timeoutSeconds:
      1\n          failureThreshold:  40\n          successThreshold: 1\n          httpGet:\n
      \           scheme: \"HTTP\"\n            port: 8000\n        livenessProbe:\n
      \         initialDelaySeconds: 120\n          periodSeconds: 10\n          timeoutSeconds:
      10\n          failureThreshold: 3\n          successThreshold: 1\n          httpGet:\n
      \           scheme: \"HTTP\"\n            port: 8000\n      dnsPolicy: ClusterFirst\n
      \     restartPolicy: Always\n\n      volumes:\n      - name: nrf-volume\n        projected:\n
      \         sources:\n          - configMap:\n              name: nrf-configmap\n"
    nrf/nrf-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: nrf-nnrf
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: nrf
      spec:
        type: ClusterIP
        ports:
          - port: 8000
            targetPort: 8000
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: nrf
    nssf/nssf-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name:
      nssf-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n    app:
      free5gc\ndata:\n  nssfcfg.yaml: |\n    info:\n      version: 1.0.1\n      description:
      NSSF initial local configuration\n\n    configuration:\n      serviceNameList:\n
      \       - nnssf-nsselection\n        - nnssf-nssaiavailability\n\n      sbi:\n
      \       scheme: http\n        registerIPv4: nssf-nnssf  # IP used to register
      to NRF\n        bindingIPv4: 0.0.0.0      # IP used to bind the service\n        port:
      80\n        tls:\n          key: config/TLS/nssf.key\n          pem: config/TLS/nssf.pem\n
      \     \n      nrfUri: http://nrf-nnrf:8000\n      \n      nsiList:\n        -
      snssai:\n            sst: 1\n          nsiInformationList:\n            - nrfId:
      http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId: 10\n        -
      snssai:\n            sst: 1\n            sd: 1\n          nsiInformationList:\n
      \           - nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId:
      11\n        - snssai:\n            sst: 1\n            sd: 2\n          nsiInformationList:\n
      \           - nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId:
      12\n            - nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId:
      12\n        - snssai:\n            sst: 1\n            sd: 3\n          nsiInformationList:\n
      \           - nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId:
      13\n        - snssai:\n            sst: 2\n          nsiInformationList:\n            -
      nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId: 20\n
      \       - snssai:\n            sst: 2\n            sd: 1\n          nsiInformationList:\n
      \           - nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId:
      21\n        - snssai:\n            sst: 1\n            sd: 010203\n          nsiInformationList:\n
      \           - nrfId: http://nrf-nnrf:8000/nnrf-nfm/v1/nf-instances\n              nsiId:
      22\n      amfSetList:\n        - amfSetId: 1\n          amfList:\n            -
      ffa2e8d7-3275-49c7-8631-6af1df1d9d26\n            - 0e8831c3-6286-4689-ab27-1e2161e15cb1\n
      \           - a1fba9ba-2e39-4e22-9c74-f749da571d0d\n          nrfAmfSet: http://nrf-nnrf:8081/nnrf-nfm/v1/nf-instances\n
      \         supportedNssaiAvailabilityData:\n            - tai:\n                plmnId:\n
      \                 mcc: 466\n                  mnc: 92\n                tac:
      33456\n              supportedSnssaiList:\n                - sst: 1\n                  sd:
      1\n                - sst: 1\n                  sd: 2\n                - sst:
      2\n                  sd: 1\n            - tai:\n                plmnId:\n                  mcc:
      466\n                  mnc: 92\n                tac: 33457\n              supportedSnssaiList:\n
      \               - sst: 1\n                - sst: 1\n                  sd: 1\n
      \               - sst: 1\n                  sd: 2\n        - amfSetId: 2\n          nrfAmfSet:
      http://nrf-nnrf:8084/nnrf-nfm/v1/nf-instances\n          supportedNssaiAvailabilityData:\n
      \           - tai:\n                plmnId:\n                  mcc: 466\n                  mnc:
      92\n                tac: 33456\n              supportedSnssaiList:\n                -
      sst: 1\n                - sst: 1\n                  sd: 1\n                -
      sst: 1\n                  sd: 3\n                - sst: 2\n                  sd:
      1\n            - tai:\n                plmnId:\n                  mcc: 466\n
      \                 mnc: 92\n                tac: 33458\n              supportedSnssaiList:\n
      \               - sst: 1\n                - sst: 1\n                  sd: 1\n
      \               - sst: 2\n      nssfName: NSSF\n      supportedPlmnList:\n        -
      mcc: 208\n          mnc: 93\n      supportedNssaiInPlmnList:\n        - plmnId:\n
      \           mcc: 208\n            mnc: 93\n          supportedSnssaiList:\n
      \           - sst: 1\n              sd: 010203\n            - sst: 1\n              sd:
      112233\n            - sst: 1\n              sd: 3\n            - sst: 2\n              sd:
      1\n            - sst: 2\n              sd: 2\n      amfList:\n        - nfId:
      469de254-2fe5-4ca0-8381-af3f500af77c\n          supportedNssaiAvailabilityData:\n
      \           - tai:\n                plmnId:\n                  mcc: 466\n                  mnc:
      92\n                tac: 33456\n              supportedSnssaiList:\n                -
      sst: 1\n                - sst: 1\n                  sd: 2\n                -
      sst: 2\n            - tai:\n                plmnId:\n                  mcc:
      466\n                  mnc: 92\n                tac: 33457\n              supportedSnssaiList:\n
      \               - sst: 1\n                  sd: 1\n                - sst: 1\n
      \                 sd: 2\n        - nfId: fbe604a8-27b2-417e-bd7c-8a7be2691f8d\n
      \         supportedNssaiAvailabilityData:\n            - tai:\n                plmnId:\n
      \                 mcc: 466\n                  mnc: 92\n                tac:
      33458\n              supportedSnssaiList:\n                - sst: 1\n                -
      sst: 1\n                  sd: 1\n                - sst: 1\n                  sd:
      3\n                - sst: 2\n            - tai:\n                plmnId:\n                  mcc:
      466\n                  mnc: 92\n                tac: 33459\n              supportedSnssaiList:\n
      \               - sst: 1\n                - sst: 1\n                  sd: 1\n
      \               - sst: 2\n                - sst: 2\n                  sd: 1\n
      \       - nfId: b9e6e2cb-5ce8-4cb6-9173-a266dd9a2f0c\n          supportedNssaiAvailabilityData:\n
      \           - tai:\n                plmnId:\n                  mcc: 466\n                  mnc:
      92\n                tac: 33456\n              supportedSnssaiList:\n                -
      sst: 1\n                - sst: 1\n                  sd: 1\n                -
      sst: 1\n                  sd: 2\n                - sst: 2\n            - tai:\n
      \               plmnId:\n                  mcc: 466\n                  mnc:
      92\n                tac: 33458\n              supportedSnssaiList:\n                -
      sst: 1\n                - sst: 1\n                  sd: 1\n                -
      sst: 2\n                - sst: 2\n                  sd: 1\n      taList:\n        -
      tai:\n            plmnId:\n              mcc: 466\n              mnc: 92\n            tac:
      33456\n          accessType: 3GPP_ACCESS\n          supportedSnssaiList:\n            -
      sst: 1\n            - sst: 1\n              sd: 1\n            - sst: 1\n              sd:
      2\n            - sst: 2\n        - tai:\n            plmnId:\n              mcc:
      466\n              mnc: 92\n            tac: 33457\n          accessType: 3GPP_ACCESS\n
      \         supportedSnssaiList:\n            - sst: 1\n            - sst: 1\n
      \             sd: 1\n            - sst: 1\n              sd: 2\n            -
      sst: 2\n        - tai:\n            plmnId:\n              mcc: 466\n              mnc:
      92\n            tac: 33458\n          accessType: 3GPP_ACCESS\n          supportedSnssaiList:\n
      \           - sst: 1\n            - sst: 1\n              sd: 1\n            -
      sst: 1\n              sd: 3\n            - sst: 2\n          restrictedSnssaiList:\n
      \           - homePlmnId:\n                mcc: 310\n                mnc: 560\n
      \             sNssaiList:\n                - sst: 1\n                  sd: 3\n
      \       - tai:\n            plmnId:\n              mcc: 466\n              mnc:
      92\n            tac: 33459\n          accessType: 3GPP_ACCESS\n          supportedSnssaiList:\n
      \           - sst: 1\n            - sst: 1\n              sd: 1\n            -
      sst: 2\n            - sst: 2\n              sd: 1\n          restrictedSnssaiList:\n
      \           - homePlmnId:\n                mcc: 310\n                mnc: 560\n
      \             sNssaiList:\n                - sst: 2\n                  sd: 1\n
      \     mappingListFromPlmn:\n        - operatorName: NTT Docomo\n          homePlmnId:\n
      \           mcc: 440\n            mnc: 10\n          mappingOfSnssai:\n            -
      servingSnssai:\n                sst: 1\n                sd: 1\n              homeSnssai:\n
      \               sst: 1\n                sd: 1\n            - servingSnssai:\n
      \               sst: 1\n                sd: 2\n              homeSnssai:\n                sst:
      1\n                sd: 3\n            - servingSnssai:\n                sst:
      1\n                sd: 3\n              homeSnssai:\n                sst: 1\n
      \               sd: 4\n            - servingSnssai:\n                sst: 2\n
      \               sd: 1\n              homeSnssai:\n                sst: 2\n                sd:
      2\n        - operatorName: AT&T Mobility\n          homePlmnId:\n            mcc:
      310\n            mnc: 560\n          mappingOfSnssai:\n            - servingSnssai:\n
      \               sst: 1\n                sd: 1\n              homeSnssai:\n                sst:
      1\n                sd: 2\n            - servingSnssai:\n                sst:
      1\n                sd: 2\n              homeSnssai:\n                sst: 1\n
      \               sd: 3      \n\n    logger:\n      NSSF:\n        ReportCaller:
      false\n        debugLevel: info\n"
    nssf/nssf-deployment.yaml: "---\napiVersion: apps/v1\nkind: Deployment\nmetadata:\n
      \ name: free5gc-nssf\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n
      \   project: free5gc\n    nf: nssf\nspec:\n  replicas: 1\n  selector:\n    matchLabels:\n
      \     project: free5gc\n      nf: nssf\n  template:\n    metadata:\n      labels:\n
      \       project: free5gc\n        nf: nssf\n    spec:\n      initContainers:\n
      \     - name: wait-nrf\n        image: towards5gs/initcurl:1.0.0\n        env:\n
      \       - name: DEPENDENCIES\n          value: http://nrf-nnrf:8000\n        command:
      ['sh', '-c', 'set -x; for dependency in $DEPENDENCIES; do while [ $(curl --insecure
      --connect-timeout 1 -s -o /dev/null -w \"%{http_code}\" $dependency) -ne 200
      ]; do echo waiting for dependencies; sleep 1; done; done;']\n\n      containers:\n
      \     - name: nssf\n        image: towards5gs/free5gc-nssf:v3.1.1\n        imagePullPolicy:
      IfNotPresent\n        securityContext:\n            {}\n        ports:\n        -
      containerPort: 80\n        command: [\"./nssf\"]\n        args: [\"-c\", \"../config/nssfcfg.yaml\"]\n
      \       env: \n          - name: GIN_MODE\n            value: release\n        volumeMounts:\n
      \       - mountPath: /free5gc/config/\n          name: nssf-volume\n        resources:\n
      \           limits:\n              cpu: 100m\n              memory: 128Mi\n
      \           requests:\n              cpu: 100m\n              memory: 128Mi\n
      \     dnsPolicy: ClusterFirst\n      restartPolicy: Always\n\n      volumes:\n
      \     - name: nssf-volume\n        projected:\n          sources:\n          -
      configMap:\n              name: nssf-configmap\n"
    nssf/nssf-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: nssf-nnssf
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: nssf
      spec:
        type: ClusterIP
        ports:
          - port: 80
            targetPort: 80
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: nssf
    package-context.yaml: |
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: kptfile.kpt.dev
        annotations:
          config.kubernetes.io/local-config: "true"
      data:
        name: free5gc
        namespace: free5gc
    pcf/pcf-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name:
      pcf-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n    app:
      free5gc\ndata:\n  pcfcfg.yaml: |\n    info:\n      version: 1.0.1\n      description:
      PCF initial local configuration\n\n    configuration:\n      serviceList:\n
      \       - serviceName: npcf-am-policy-control\n        - serviceName: npcf-smpolicycontrol\n
      \         suppFeat: 3fff\n        - serviceName: npcf-bdtpolicycontrol\n        -
      serviceName: npcf-policyauthorization\n          suppFeat: 3\n        - serviceName:
      npcf-eventexposure\n        - serviceName: npcf-ue-policy-control\n\n      sbi:\n
      \       scheme: http\n        registerIPv4: pcf-npcf  # IP used to register
      to NRF\n        bindingIPv4: 0.0.0.0    # IP used to bind the service\n        port:
      80\n        tls:\n          key: config/TLS/pcf.key\n          pem: config/TLS/pcf.pem\n
      \     \n      mongodb:       # the mongodb connected by this PCF\n        name:
      free5gc                  # name of the mongodb\n        url: mongodb://mongodb:27017
      # a valid URL of the mongodb\n      \n      nrfUri: http://nrf-nnrf:8000\n      pcfName:
      PCF\n      timeFormat: 2019-01-02 15:04:05\n      defaultBdtRefId: BdtPolicyId-\n
      \     locality: area1\n\n    logger:\n      PCF:\n        ReportCaller: false\n
      \       debugLevel: info\n"
    pcf/pcf-deployment.yaml: |
      ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: free5gc-pcf
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: pcf
      spec:
        replicas: 1
        selector:
          matchLabels:
            project: free5gc
            nf: pcf
        template:
          metadata:
            labels:
              project: free5gc
              nf: pcf
          spec:
            initContainers:
            - name: wait-nrf
              image: towards5gs/initcurl:1.0.0
              env:
              - name: DEPENDENCIES
                value: http://nrf-nnrf:8000
              command: ['sh', '-c', 'set -x; for dependency in $DEPENDENCIES; do while [ $(curl --insecure --connect-timeout 1 -s -o /dev/null -w "%{http_code}" $dependency) -ne 200 ]; do echo waiting for dependencies; sleep 1; done; done;']

            containers:
            - name: pcf
              image: towards5gs/free5gc-pcf:v3.1.1
              imagePullPolicy: IfNotPresent
              ports:
              - containerPort: 80
              command: ["./pcf"]
              args: ["-c", "../config/pcfcfg.yaml"]
              env:
                - name: GIN_MODE
                  value: release
              volumeMounts:
              - mountPath: /free5gc/config/
                name: pcf-volume
              resources:
                  limits:
                    cpu: 100m
                    memory: 128Mi
                  requests:
                    cpu: 100m
                    memory: 128Mi
            dnsPolicy: ClusterFirst
            restartPolicy: Always

            volumes:
            - name: pcf-volume
              projected:
                sources:
                - configMap:
                    name: pcf-configmap
    pcf/pcf-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: pcf-npcf
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: pcf
      spec:
        type: ClusterIP
        ports:
          - port: 80
            targetPort: 80
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: pcf
    udm/udm-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name:
      udm-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n    app:
      free5gc\ndata:\n  udmcfg.yaml: |\n    info:\n      version: 1.0.2\n      description:
      UDM initial local configuration\n\n    configuration:\n      serviceNameList:\n
      \       - nudm-sdm\n        - nudm-uecm\n        - nudm-ueau\n        - nudm-ee\n
      \       - nudm-pp\n      \n      sbi:\n        scheme: http\n        registerIPv4:
      udm-nudm # IP used to register to NRF\n        bindingIPv4: 0.0.0.0  # IP used
      to bind the service\n        port: 80\n        tls:\n          key: config/TLS/udm.key\n
      \         pem: config/TLS/udm.pem\n      \n      nrfUri: http://nrf-nnrf:8000\n
      \     # test data set from TS33501-f60 Annex C.4\n      SuciProfile:\n        -
      ProtectionScheme: 1 # Protect Scheme: Profile A\n          PrivateKey: c53c22208b61860b06c62e5406a7b330c2b577aa5558981510d128247d38bd1d\n
      \         PublicKey: 5a8d38864820197c3394b92613b20b91633cbd897119273bf8e4a6f4eec0a650\n
      \       - ProtectionScheme: 2 # Protect Scheme: Profile B\n          PrivateKey:
      F1AB1074477EBCC7F554EA1C5FC368B1616730155E0041AC447D6301975FECDA\n          PublicKey:
      0472DA71976234CE833A6907425867B82E074D44EF907DFB4B3E21C1C2256EBCD15A7DED52FCBB097A4ED250E036C7B9C8C7004C4EEDC4F068CD7BF8D3F900E3B4\n\n
      \   logger:\n      UDM:\n        ReportCaller: false\n        debugLevel: info\n"
    udm/udm-deployment.yaml: "---\napiVersion: apps/v1\nkind: Deployment\nmetadata:\n
      \ name: free5gc-udm\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n
      \   project: free5gc\n    nf: udm\nspec:\n  replicas: 1\n  selector:\n    matchLabels:\n
      \     project: free5gc\n      nf: udm\n  template:\n    metadata:\n      labels:\n
      \       project: free5gc\n        nf: udm\n    spec:\n      initContainers:\n
      \     - name: wait-nrf\n        image: towards5gs/initcurl:1.0.0\n        env:\n
      \       - name: DEPENDENCIES\n          value: http://nrf-nnrf:8000\n        command:
      ['sh', '-c', 'set -x; for dependency in $DEPENDENCIES; do while [ $(curl --insecure
      --connect-timeout 1 -s -o /dev/null -w \"%{http_code}\" $dependency) -ne 200
      ]; do echo waiting for dependencies; sleep 1; done; done;']\n\n      containers:\n
      \     - name: udm\n        image: towards5gs/free5gc-udm:v3.1.1\n        imagePullPolicy:
      IfNotPresent\n        ports:\n        - containerPort: 80\n        command:
      [\"./udm\"]\n        args: [\"-c\", \"../config/udmcfg.yaml\"]\n        env:
      \n          - name: GIN_MODE\n            value: release\n        volumeMounts:\n
      \       - mountPath: /free5gc/config/\n          name: udm-volume\n        resources:\n
      \           limits:\n              cpu: 100m\n              memory: 128Mi\n
      \           requests:\n              cpu: 100m\n              memory: 128Mi\n
      \     dnsPolicy: ClusterFirst\n      restartPolicy: Always\n\n      volumes:\n
      \     - name: udm-volume\n        projected:\n          sources:\n          -
      configMap:\n              name: udm-configmap\n"
    udm/udm-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: udm-nudm
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: udm
      spec:
        type: ClusterIP
        ports:
          - port: 80
            targetPort: 80
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: udm
    udr/udr-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name:
      udr-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n    app:
      free5gc\ndata:\n  udrcfg.yaml: |\n    info:\n      version: 1.0.1\n      description:
      UDR initial local configuration\n\n    configuration:\n      sbi:\n        scheme:
      http\n        registerIPv4: udr-nudr # IP used to register to NRF\n        bindingIPv4:
      0.0.0.0  # IP used to bind the service\n        port: 80\n        tls:\n          key:
      config/TLS/udr.key\n          pem: config/TLS/udr.pem\n\n      mongodb:\n        name:
      free5gc\n        url: mongodb://mongodb:27017       \n      \n      nrfUri:
      http://nrf-nnrf:8000\n\n    logger:\n      MongoDBLibrary:\n        ReportCaller:
      false\n        debugLevel: info\n      OpenApi:\n        ReportCaller: false\n
      \       debugLevel: info\n      PathUtil:\n        ReportCaller: false\n        debugLevel:
      info\n      UDR:\n        ReportCaller: false\n        debugLevel: info\n"
    udr/udr-deployment.yaml: "---\napiVersion: apps/v1\nkind: Deployment\nmetadata:\n
      \ name: free5gc-udr\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n
      \   project: free5gc\n    nf: udr\nspec:\n  replicas: 1\n  selector:\n    matchLabels:\n
      \     project: free5gc\n      nf: udr\n  template:\n    metadata:\n      labels:\n
      \       project: free5gc\n        nf: udr\n    spec:\n      initContainers:\n
      \     - name: wait-nrf\n        image: towards5gs/initcurl:1.0.0\n        env:\n
      \       - name: DEPENDENCIES\n          value: http://nrf-nnrf:8000\n        command:
      ['sh', '-c', 'set -x; for dependency in $DEPENDENCIES; do while [ $(curl --insecure
      --connect-timeout 1 -s -o /dev/null -w \"%{http_code}\" $dependency) -ne 200
      ]; do echo waiting for dependencies; sleep 1; done; done;']\n\n      containers:\n
      \     - name: udr\n        image: towards5gs/free5gc-udr:v3.1.1\n        imagePullPolicy:
      IfNotPresent\n        ports:\n        - containerPort: 80\n        command:
      [\"./udr\"]\n        args: [\"-c\", \"../config/udrcfg.yaml\"]\n        env:
      \n          - name: DB_URI\n            value: mongodb://mongodb/free5gc\n          -
      name: GIN_MODE\n            value: release\n        volumeMounts:\n        -
      mountPath: /free5gc/config/\n          name: udr-volume\n        resources:\n
      \           limits:\n              cpu: 100m\n              memory: 128Mi\n
      \           requests:\n              cpu: 100m\n              memory: 128Mi\n
      \     dnsPolicy: ClusterFirst\n      restartPolicy: Always\n\n      volumes:\n
      \     - name: udr-volume\n        projected:\n          sources:\n          -
      configMap:\n              name: udr-configmap\n"
    udr/udr-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: udr-nudr
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: udr
      spec:
        type: ClusterIP
        ports:
          - port: 80
            targetPort: 80
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: udr
    webui/webui-configmap.yaml: "---\napiVersion: v1\nkind: ConfigMap\nmetadata:\n
      \ name: webui-configmap\n  labels:\n    app.kubernetes.io/version: \"v3.1.1\"\n
      \   app: free5gc\ndata:\n  webuicfg.yaml: |\n    info:\n      version: 1.0.0\n
      \     description: WEBUI initial local configuration\n\n    configuration:\n
      \     mongodb:\n        name: free5gc\n        url: mongodb://mongodb:27017\n
      \       \n    logger:\n      WEBUI:\n        ReportCaller: false\n        debugLevel:
      info\n"
    webui/webui-deployment.yaml: |
      ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: free5gc-webui
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: webui
      spec:
        replicas: 1
        selector:
          matchLabels:
            project: free5gc
            nf: webui
        template:
          metadata:
            labels:
              project: free5gc
              nf: webui
          spec:
            initContainers:
            - name: wait-mongo
              image: busybox:1.32.0
              env:
              - name: DEPENDENCIES
                value: mongodb:27017
              command: ["sh", "-c", "until nc -z $DEPENDENCIES; do echo waiting for the MongoDB; sleep 2; done;"]
            containers:
            - name: webui
              image: towards5gs/free5gc-webui:v3.1.1
              imagePullPolicy: IfNotPresent
              ports:
              - containerPort: 5000
              command: ["./webconsole"]
              args: ["-c", "../config/webuicfg.yaml"]
              env:
                - name: GIN_MODE
                  value: release
              volumeMounts:
              - mountPath: /free5gc/config/
                name: webui-volume
              resources:
                  limits:
                    cpu: 100m
                    memory: 128Mi
                  requests:
                    cpu: 100m
                    memory: 128Mi
              readinessProbe:
                initialDelaySeconds: 0
                periodSeconds: 1
                timeoutSeconds: 1
                failureThreshold:  40
                successThreshold: 1
                httpGet:
                  scheme: HTTP
                  port: 5000
              livenessProbe:
                initialDelaySeconds: 120
                periodSeconds: 10
                timeoutSeconds: 10
                failureThreshold: 3
                successThreshold: 1
                httpGet:
                  scheme: HTTP
                  port: 5000
            dnsPolicy: ClusterFirst
            restartPolicy: Always

            volumes:
            - name: webui-volume
              projected:
                sources:
                - configMap:
                    name: webui-configmap
    webui/webui-service.yaml: |
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: webui-service
        labels:
          app.kubernetes.io/version: "v3.1.1"
          project: free5gc
          nf: webui
      spec:
        type: NodePort
        ports:
          - port: 5000
            targetPort: 5000
            nodePort: 30500
            protocol: TCP
            name: http
        selector:
          project: free5gc
          nf: webui
  revision: v1
  workspaceName: v1
status:
  renderStatus:
    error: ""
    result:
      exitCode: 0
      metadata:
        creationTimestamp: null
```
</details>