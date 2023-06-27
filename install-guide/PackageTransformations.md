Package transformations work in Nephio Sandbox Installation
===========================================================

# Table of Contents
- [Vanilla kpt for the management cluster](#vanilla-kpt-for-the-management-cluster)
  - [kpt pkg get](#kpt-pkg-get)
  - [kpt fn render](#kpt-fn-render)
  - [kpt live init](#kpt-live-init)
- [kpt alpha rpkg for workload clusters](#kpt-alpha-rpkg-for-workload-clusters)
  - [Create workload cluster package](#create-workload-cluster-package)
  - [Configure the package](#configure-the-package)
  - [Propose the package](#propose-the-package)
  - [Approve the package and trigger configsync](#approve-the-package-and-trigger-configsync)
- [Transformations in workload cluster creation](#transformations-in-workload-cluster-creation)

# Vanilla kpt for the management cluster

Before reading this, please read [the kpt book](https://kpt.dev/book/).

## kpt pkg get

The `kpt pkg get --for-deployment https://<repo-path>/<repo-pkg-name>@<repo-pkg-name>/<pkg-version> <local-pkg-name>` command downloads a kpt package from a repository.

The fields in the command above are as follows:

| Field            | Description                                                                |
| ---------------- | -------------------------------------------------------------------------- |
| `repo-path`      | The path in the repo to the kpt package                                    |
| `repo-pkg-name`  | The name of the kpt package in the repo                                    |
| `pkg-version`    | The version of the kpt package                                             |
| `local-pkg-name` | The local name of the kpt package in the repo, defaults to `repo-pkg-name` |

`kpt pkg get` make the following transformations:

1. The `metadata.name` field in the root `Kptfile` in the package is changed from whatever value it has to `local-pkg-name`
2. The `metadata.namespace` field in the root `Kptfile` in the package is removed
3. `upstream` and `upstreamlock` root fields are added to the root `Kptfile` as follows:

```
upstream:
  type: git
  git:
    repo: https://<repo-path>/<repo-pkg-name>
    directory: /<local-pkg-name>
    ref: <pkg-version>
  updateStrategy: resource-merge
upstreamLock:
  type: git
  git:
    repo: https://<repo-path>/<repo-pkg-name>
    directory: /<local-pkg-name>
    ref: <pkg-version>
    commit: 0123456789abcdef0123456789abcdef01234567
```
4. The `data.name` field in the root  `package-context.yaml` files is changed to be `local-pkg-name`
5. The `package-context.yaml` file is added if it does not exist with the following content:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kptfile.kpt.dev
  annotations:
    config.kubernetes.io/local-config: "true"
data:
  name: <local-pkg-name|sub-pkg-name>
```

6. The `data.name` field in `package-context.yaml` files in sub kpt packages is changed to be the name of the sub package
7. All other sub-fields under the `data:` field are deleted
8. The comment `metadata: # kpt-merge: <namespace>/<name>` is added to root `metadata` fields on all yaml documents in the kpt package and enclosed sub-packages that have a root `apiVersion` and `kind` field if such a comment does not already exist. The `namespace` and `name` values used are the values of those fields in the `metadata` field. Note that a yaml file can contain multiple yaml documents and each root `metadata` field is commented. For example:

```
metadata: # kpt-merge: cert-manager/cert-manager-cainjector
  name: cert-manager-cainjector
  namespace: cert-manager
```

9. The annotation `internal.kpt.dev/upstream-identifier: '<apiVersion>|<kind>|<namespace>|<name>'` is added to root `metadata.annotations:` fields on all yaml documents in the kpt package and enclosed sub-packages that have a root `apiVersion:` and `kind:` field if such an annotation does not already exist. The `namespace` and `name` values used are the values of those fields in the `metadata` field. Note that a yaml file can contain multiple yaml documents and each root `metadata` field is commented. For example:

```
metadata: # kpt-merge: cert-manager/cert-manager-cainjector
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: # kpt-merge: capi-kubeadm/leader-election-rolebinding
  name: leader-election-rolebinding
  namespace: capi-kubeadm
  annotations:
    internal.kpt.dev/upstream-identifier: 'rbac.authorization.k8s.io|RoleBinding|capi-kubeadm|leader-election-rolebinding'
```

## kpt fn render

The `kpt fn render <local-pkg-name>` runs kpt functions on a local package, thus applying local changes to the package.

In the Nephio sandbox install, kpt fn render only acts on the `repository` and `rootsync` kpt packages from [nephio-example-packages](https://github.com/nephio-project/nephio-example-packages).

### repository package

The `repository` package has a kpt function written in [starlark](https://github.com/bazelbuild/starlark), which is invoked by a pipeline specified in the `kptfile`.

```
pipeline:
  mutators:
  - image: gcr.io/kpt-fn/starlark:v0.4.3
    configPath: set-values.yaml
```

The starlark function is specified in the `set-values.yaml` file. It makes the following transformations on repositories:

1. in the file `repo-gitea.yaml`
  - the `metadata.name` field gets the value of `<local-pkg-name>`
  - the `spec.description` field gets the value of `<local-pkg-name> repository`
2. in the file `repo-porch.yaml`
  - the `metadata.name` field gets the value of `<local-pkg-name>`
  - the `spec.git.repo` field gets the value of `"http://172.18.0.200:3000/nephio/<local-pkg-name>.git`
  - the `spec.git.secretRef.name` field gets the value of `<local-pkg-name>-access-token-porch`
  - if the `<local-pkg-name>` is called `mgmt-staging`, then the following extra changes are made:
    - the `spec.deployment` field is set to `false` (it defaults to `true`)
    - the annotation `metadata.annotations.nephio.org/staging: "true"` is added

3. in the file `token-configsync.yaml`
  - the `metadata.name` field gets the value of `<local-pkg-name>-access-token-configsync`
  - the `metadata.namespace` field gets the value of `config-management-system`
4. in the file `token-porch.yaml`
  - the `metadata.name` field gets the value of `<local-pkg-name>-access-token-porch`

### rootsync package

The `rootsync` package also has a kpt function written in [starlark](https://github.com/bazelbuild/starlark) specified in the `set-values.yaml` file. It makes the following transformations on repositories:

1. in the file `rootsync.yaml`
  - the `metadata.name` field gets the value of `<local-pkg-name>`
  - the `spec.git.repo` field gets the value of `"http://172.18.0.200:3000/nephio/<local-pkg-name>.git`
  - the `spec.git.secretRef.name` field gets the value of `<local-pkg-name>-access-token-configsync`
  
## kpt live init

The `kpt live init <local-pkg-name>` initializes a local package, making it ready for application to a cluster. This command creates a `resourcegroup.yaml` in the kpt package with content similar to:

```
apiVersion: kpt.dev/v1alpha1
kind: ResourceGroup
metadata:
  name: inventory-12345678
  namespace: default
  labels:
    cli-utils.sigs.k8s.io/inventory-id: 0123456789abcdef0123456789abcdef01234567-0123456789abcdef012
```
# kpt alpha rpkg for workload clusters

The `kpt alpha rpkg` suite of commands that act on `Repository` resources on the kubernetes cluster in scope. The packages in the `Repository` resources are *remote packages (rpkg)*.

To see what repositories are in scope:

```
NAME                      TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
free5gc-packages          git    Package   false        True    https://github.com/nephio-project/free5gc-packages.git
mgmt                      git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging              git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
nephio-example-packages   git    Package   false        True    https://github.com/nephio-project/nephio-example-packages.git
```

To see all the remote packages that are available:

<details>
<summary>$ kpt alpha rpkg get</summary>

```
NAME                                                               PACKAGE                              WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
free5gc-packages-daba5a241a23af3ce777b256c39dcedc3c5e3917          free5gc-cp                           v1              v1         true     Published   free5gc-packages
free5gc-packages-ab9c21f547bdf11832d683a01dde851305ab151e          free5gc-operator                     v3              v3         true     Published   free5gc-packages
free5gc-packages-bc41f4d0a9f984b7a7aa8ef6623a615e2aa75993          free5gc-upf                          v1              v1         true     Published   free5gc-packages
free5gc-packages-4c66951196b8727569070db9f0d19f83bdb21adf          pkg-example-amf-bp                   v3              v3         true     Published   free5gc-packages
free5gc-packages-0292da186776c49eb0f72489a33d67cb6f0f1da2          pkg-example-smf-bp                   v3              v3         true     Published   free5gc-packages
free5gc-packages-a845d4297634409d813c882a307f033999b05d03          pkg-example-upf-bp                   v3              v3         true     Published   free5gc-packages
nephio-example-packages-ae679b5ac4fbb7719e7302fe6726e26005b004e0   5g-core-topology                     v2              v2         true     Published   nephio-example-packages
nephio-example-packages-5d6969cb98e4d6494ff6eadd6c52e94368f8261c   cert-manager                         v2              v2         true     Published   nephio-example-packages
nephio-example-packages-2144369250410c527ac4716b50fce8c88656f231   cluster-capi                         v4              v4         true     Published   nephio-example-packages
nephio-example-packages-6be85e6942a3f9a2e2e4d6e6b9a4c95172142c8e   cluster-capi-infrastructure-docker   v1              v1         true     Published   nephio-example-packages
nephio-example-packages-0bb045b3458818144b6596c1239e8cf307ed10d0   cluster-capi-kind                    v5              v5         true     Published   nephio-example-packages
nephio-example-packages-4204d1ed31d71ac2147a4bd7dad614bf3ab0dd50   cluster-capi-kind-docker-templates   v1              v1         true     Published   nephio-example-packages
nephio-example-packages-73a2b11f81e580686fc01773809b4050958177e3   configsync                           v1              v1         true     Published   nephio-example-packages
nephio-example-packages-12b4164a8aed5ffc6e197b8af97edb34b31a11bf   free5gc-smf-org-example              v2              v2         true     Published   nephio-example-packages
nephio-example-packages-e3f5dce5fc98f9c06c32fa0342f9b0ae78eca562   gitea                                v3              v3         true     Published   nephio-example-packages
nephio-example-packages-073f3256d9ba4c32a3cf491c1cb9ee683b14b5c0   kindnet                              v1              v1         true     Published   nephio-example-packages
nephio-example-packages-7f4c57efd2a8707ef17f3ff1565da41b39ef6412   local-path-provisioner               v1              v1         true     Published   nephio-example-packages
nephio-example-packages-14dcd072fa92334197a1f23af02d9a4b55e8bae5   metallb                              v1              v1         true     Published   nephio-example-packages
nephio-example-packages-a48da09e5a4d99fae4ad5f54261e53ef1ee98b25   metallb-sandbox-config               v1              v1         true     Published   nephio-example-packages
nephio-example-packages-a226e2e41725db980f780812d9ab42f653a1a049   multus                               v1              v1         true     Published   nephio-example-packages
nephio-example-packages-943d66ad8063c09731550ab456bd93e099a55a8d   nephio-controllers                   v5              v5         true     Published   nephio-example-packages
nephio-example-packages-aa903df4b3652fdb5fdec2fd599b5b6275397b28   nephio-stock-repos                   v1              v1         true     Published   nephio-example-packages
nephio-example-packages-7895e28d847c0296a204007ed577cd2a4222d1ea   nephio-workload-cluster              v8              v8         true     Published   nephio-example-packages
nephio-example-packages-0e363ac47ac4612c2d41d37e976a783abe849067   network                              v1              v1         true     Published   nephio-example-packages
nephio-example-packages-7264e782c747fe720fb41b2b55a3234ffcd053f0   porch-dev                            v2              v2         true     Published   nephio-example-packages
nephio-example-packages-bb0001a2735f809281be792895a6a410abbe60fd   repository                           v3              v3         true     Published   nephio-example-packages
nephio-example-packages-6bf353f2bfe4b09bfd69d62ca169f30a35404d33   resource-backend                     v2              v2         true     Published   nephio-example-packages
nephio-example-packages-b0e4d5eb9e14d3a49d1e3a386b58e4eb9450243c   rootsync                             v3              v3         true     Published   nephio-example-packages
nephio-example-packages-fb6e4adecc13c50da838953ece623cf04de21884   ueransim                             v1              v1         true     Published   nephio-example-packages
nephio-example-packages-dc0b55fb7a17d107e834417a2c9d8fb37f36d7cb   vlanindex                            v1              v1         true     Published   nephio-example-packages
```

</details>

<details>
<summary>To see the versions of a particular package:</summary>

```
$ kpt alpha rpkg get --name nephio-workload-cluster
NAME                                                               PACKAGE                   WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
nephio-example-packages-05707c7acfb59988daaefd85e3f5c299504c2da1   nephio-workload-cluster   main            main       false    Published   nephio-example-packages
nephio-example-packages-781e1c17d63eed5634db7b93307e1dad75a92bce   nephio-workload-cluster   v1              v1         false    Published   nephio-example-packages
nephio-example-packages-5929727104f2c62a2cb7ad805dabd95d92bf727e   nephio-workload-cluster   v2              v2         false    Published   nephio-example-packages
nephio-example-packages-cdc6d453ae3e1bd0b64234d51d575e4a30980a77   nephio-workload-cluster   v3              v3         false    Published   nephio-example-packages
nephio-example-packages-c78ecc6bedc8bf68185f28a998718eed8432dc3b   nephio-workload-cluster   v4              v4         false    Published   nephio-example-packages
nephio-example-packages-46b923a6bbd09c2ab7aa86c9853a96cbd38d1ed7   nephio-workload-cluster   v5              v5         false    Published   nephio-example-packages
nephio-example-packages-17bffe318ac068f5f9ef22d44f08053e948a3683   nephio-workload-cluster   v6              v6         false    Published   nephio-example-packages
nephio-example-packages-0fbaccf6c5e75a3eff7976a523bb4f42bb0118ce   nephio-workload-cluster   v7              v7         false    Published   nephio-example-packages
nephio-example-packages-7895e28d847c0296a204007ed577cd2a4222d1ea   nephio-workload-cluster   v8              v8         true     Published   nephio-example-packages
```

</details>

## Create workload cluster package 

The workload cluster package contains `PackageVariant` files for configuring the new cluster. 

Clone the `nephio-workload-cluster` package into the `mgmt` repo. This creates the blueprint package for the workload cluster in the management repo.

```
kpt alpha rpkg clone -n default nephio-example-packages-7895e28d847c0296a204007ed577cd2a4222d1ea --repository mgmt regional
```

During the clone operation, the command above performs the following operations:
1. It creates a `drafts/regional/v1` branch on the `mgmt` repo
2. It does the equivalent of a [kpt pkg get](#kpt-pkg-get) on the `nephio-workload-cluster` package into a directory called `regional` on that branch, with the same transformations on package files carried out as the [kpt pkg get](#kpt-pkg-get) command above, this content is checked into the new branch in the initial commit
3. The pipeline specified in the `Kptfile`of the `nephio-workload-cluster` package specifies an `apply-replacements` specified in the `apply-replacements.yaml` file in the package and uses the value of the `package-context.yaml:data.name` field set in 2. above (which is the workload cluster name) as follows:
  <br><br>
  a. In all `PackageVariant` files, the `metadata.name` and `spec.downstream.package` field before the '-' is replaced with that field value. In this way, the downstream package names for all the packages to be pulled from the `mgmt-staging` repo for the workload cluster are specified.
  <br>b. In all `PackageVariant` files, the `spec.injectors.WorkloadCluster.name` field is replaced with the workload cluster name. This gives us the handle for `packageVariant` injection for the workload cluster in question.
  <br>c. In all `PackageVariant` files, the `spec.pipeline.mutators.[image=gcr.io/kpt-fn/set-annotations:v0.1.4].configMap.[nephio.org/cluster-name]` field is repalced with the workload cluster name.
  <br>d. In all `WorkloadCluster` files, the `metadata.name` and `spec.clusterName` fields are replaced with the workload cluster name.

We now have a draft blueprint package for our workload cluster ready for further configuration.

## Configure the package

We follow the instructions in the [installation README file](https://github.com/nephio-project/test-infra/tree/main/e2e/provision).

1. Get the name of the package:

```
kpt alpha rpkg get | egrep '(NAME|regional)'
NAME                                           PACKAGE  WORKSPACENAME REVISION LATEST LIFECYCLE REPOSITORY
mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868  regional v1                     false  Draft     mgmt
```

2. Pull the package to get a local copy of it

```
kpt alpha rpkg pull -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
```
3. Set the nephio labels on the package

```
kpt fn eval --image gcr.io/kpt-fn/set-labels:v0.2.0 regional -- "nephio.org/site-type=regional" "nephio.org/region=us-west1"
```

4. Check that the labels have been set. In all `PackageVariant` and `WorkloadCluster` files, the following `metadata.labels` fields  have been added:

```
labels:
  nephio.org/region: us-west1
  nephio.org/site-type: regional
```

5. Push the updated package back to the draft branch on the repo:

```
kpt alpha rpkg push -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
[RUNNING] "gcr.io/kpt-fn/apply-replacements:v0.1.1" 
[PASS] "gcr.io/kpt-fn/apply-replacements:v0.1.1"
```

## Propose the package

Propose the package:

```
kpt alpha rpkg propose -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 proposed
```

Proposing the package changes the name of the `drafts/regional/v1` to `proposed/regional/v1`. There are no changes to the content of the branch.

## Approve the package and trigger configsync

Approving the package triggers `configsync`, which triggers creation of the new workload cluster using all the `PackageVariant` components specified in the `nephio-workload-cluster` kpt package.

```
kpt alpha rpkg approve -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 approved
```
The new cluster comes up after a number of minutes.

# Transformations in workload cluster creation

Approving the `regional` workload cluster package in the `mgmt` repo triggered configsync to apply the `PackageVariant` configurations in the `mgmt/regional` package. Let's examine those `PackageVariant` configurations one by one

## pv-cluster.yaml: create the workload cluster

## pv-rootsync.yaml:

## pv-repo.yaml: create the workload cluster repo


## pv-configsync.yaml:


## pv-kindnet.yaml:

## pv-local-path-provisioner.yaml:

## pv-multus.yaml:


## pv-vlanindex.yaml:

