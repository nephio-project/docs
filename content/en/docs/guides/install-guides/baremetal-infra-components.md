---
title: Baremetal cluster install
description: >
  Step by step guide to configure and install components supporting Baremetal cluster installation. 

weight: 2
---

{{% alert title="Note" color="primary" %}}

If you want to use a version other than that of v3.0.0 of Nephio *catalog* repo, then replace the *@origin/v3.0.0*
suffix on the package URLs on the `kpt pkg get` commands below with the tag/branch of the version you wish to use.

While using KPT you can [either pull a branch or a tag](https://kpt.dev/book/03-packages/01-getting-a-package) from a
git repository. By default, it pulls the tag. In case, you have branch with the same name as a tag then to:

```bash
#pull a branch 
kpt pkg get --for-deployment <git-repository>@origin/v3.0.0
#pull a tag
kpt pkg get --for-deployment <git-repository>@v3.0.0
```

{{% /alert %}}

## Metal3, BMO and Ironic packages install

Create a directory and pull the packages into that directory

```bash
#create a directory
mkdir -p /tmp/baremetal_kpt
cd /tmp/baremetal_kpt

kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-metal3@main metal3
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-ironic@main ironic
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-bmo@main bmo

```
The above commands will create three directories metal3, ironic and bmo that contains the package files.

The next step before deploying the packages is to update the fields of the ConfigMaps that are specific to the
deployment environment.

Below are the ConfigMap fields and values in the bmo/cluster-capi-infrastructure-bmo.yaml
```bash
apiVersion: v1
data:
  CACHEURL: http://172.22.0.1/images
  DEPLOY_KERNEL_URL: http://172.22.0.2:6180/images/ironic-python-agent.kernel
  DEPLOY_RAMDISK_URL: http://172.22.0.2:6180/images/ironic-python-agent.initramfs
  DHCP_RANGE: 172.22.0.10,172.22.0.100
  HTTP_PORT: "6180"
  IRONIC_ENDPOINT: http://172.22.0.2:6385/v1/
  PROVISIONING_INTERFACE: eth2
kind: ConfigMap
metadata: # kpt-merge: baremetal-operator-system/ironic
  name: ironic
  namespace: baremetal-operator-system
  annotations:
    internal.kpt.dev/upstream-identifier: '|ConfigMap|baremetal-operator-system|ironic'
```

Below are the ConfigMap fields and values in the ironic/cluster-capi-infrastructure-ironic.yaml
```bash
apiVersion: v1
data:
  CACHEURL: http://172.22.0.1/images
  DEPLOY_KERNEL_URL: http://172.22.0.2:6180/images/ironic-python-agent.kernel
  DEPLOY_RAMDISK_URL: http://172.22.0.2:6180/images/ironic-python-agent.initramfs
  DHCP_HOSTS: b4:96:91:c0:31:64,id:*;b4:96:91:c4:e3:f4,id:*
  DHCP_IGNORE: tag:!known
  DHCP_RANGE: 172.22.0.10,172.22.0.100
  DNS_IP: 172.22.0.3
  GATEWAY_IP: 172.22.0.4
  HTTP_PORT: "6180"
  IRONIC_BASE_URL: http://172.22.0.2:6385
  IRONIC_ENDPOINT: http://172.22.0.2:6385/v1/
  IRONIC_INSPECTOR_VLAN_INTERFACES: all
  IRONIC_KERNEL_PARAMS: console=ttyS0
  IRONIC_RAMDISK_SSH_KEY: PLACEHOLDER_SSH_KEY
  PROVISIONING_INTERFACE: eth2
  USE_IRONIC_INSPECTOR: "false"
kind: ConfigMap
metadata: # kpt-merge: baremetal-operator-system/baremetal-operator-ironic-bmo-configmap-6cf9t7484b
  name: baremetal-operator-ironic-bmo-configmap-6cf9t7484b
  namespace: baremetal-operator-system
  annotations:
    internal.kpt.dev/upstream-identifier: '|ConfigMap|baremetal-operator-system|baremetal-operator-ironic-bmo-configmap-6cf9t7484b'
```

The IP address 172.22.0.2 that is present in the fields DEPLOY_KERNEL_URL, DEPLOY_RAMDISK_URL, IRONIC_BASE_URL, 
IRONIC_ENDPOINT across files in bmo and ironic directories can be modified as per the deployment environment using
```bash
kpt fn eval --match-namespace=baremetal-operator-system --match-kind=ConfigMap --image gcr.io/kpt-fn/search-replace:v0.2.0 -- by-value-regex='(http://)172.22.0.2(.*)' put-value='${1}10.128.0.13${2}'
```

The value for the PROVISIONING_INTERFACE can be modified as below
```bash
 kpt fn eval --match-namespace=baremetal-operator-system --match-kind=ConfigMap --image gcr.io/kpt-fn/search-replace:v0.2.0 -- by-path="data.PROVISIONING_INTERFACE" put-value=ens4
```

The other fields that need modification depending on the deployment environment can use the command similar
to what was used for modifying the PROVISIONING_INTERFACE field above. Refer the README at https://github.com/metal3-io/ironic-image
for explanation of the fields in the ConfigMaps.

Once, the environment specific fields in the ConfigMap for the packages in bmo and ironic directory have been updated,
issue the below commands that deploys the packages in metal3, bmo and ironic directories
```bash
kpt live init
kpt live apply --reconcile-timeout=15m --output=table
```

