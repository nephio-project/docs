---
title: Baremetal cluster install
description: >
  Step by step guide to configure and install components supporting Baremetal cluster installation. 

weight: 3
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

## Pre-requisites

- Access to non-kind cluster such as kubeadm cluster.
- Nephio management components and porch installed.
  - Refer the "Preinstalled K8s cluster" section at [Kicking off an installation on a virtual machine](/docs/guides/install-guides/#kicking-off-an-installation-on-a-virtual-machine)
  - Refer installing base Nephio components at [Common Components](/docs/guides/install-guides/common-components/)

## Metal3, BMO and Ironic packages install

Create a directory and pull the packages into that directory. Note that the instructions are for installing on 
Nephio management non-KIND cluster (has been verified on kubeadm cluster). The ironic pods use host networking that KIND
clusters do not support.

```bash
#create a directory
mkdir -p /tmp/baremetal_kpt
cd /tmp/baremetal_kpt

# The below three commands will create three directories named metal3, ironic and bmo
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-metal3@main metal3
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-ironic@main ironic
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-bmo@main bmo

# The below command creates a Kptfile, README.md and package-context.yaml file
kpt pkg init

# create search-replace.yaml file using the below command.
# Refer the function at https://catalog.kpt.dev/search-replace/v0.2/ to understand how this will be used by gcr.io/kpt-fn/search-replace:v0.2.0 function.
cat <<EOF > search-replace.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: search-replace-fn-config
  annotations:
    config.kubernetes.io/local-config: "true"
data:
  by-path: data.CACHEURL
  by-value: 'http://172.22.0.1/images'
  put-value: 'http://CACHEURL_IP_PLACEHOLDER/images'
EOF

# create create-setters.yaml file using the below command.
# Refer the function at https://catalog.kpt.dev/create-setters/v0.1/ to understand how this will be used by gcr.io/kpt-fn/create-setters:v0.1.0 function.
cat <<EOF > create-setters.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: create-setters-fn-config
  annotations:
    config.kubernetes.io/local-config: "true"
data:
  CACHEURL_IP: 'CACHEURL_IP_PLACEHOLDER' 
  CTRL_PLANE_IP: '172.22.0.2'
  PROVISIONING_INTERFACE: 'eth2' 
  DHCP_RANGE: '172.22.0.10,172.22.0.100' 
  DHCP_HOSTS: 'b4:96:91:c0:31:64,id:*;b4:96:91:c4:e3:f4,id:*' 
  DNS_IP: '172.22.0.3'
  GATEWAY_IP: '172.22.0.4' 
  IRONIC_SSH_KEY: 'PLACEHOLDER_SSH_KEY'
EOF

# create apply-setters.yaml file using the below command.
# Refer the function at https://catalog.kpt.dev/apply-setters/v0.2/ to undestand how this will be used by gcr.io/kpt-fn/apply-setters:v0.2.0 function.
cat <<EOF > apply-setters.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apply-setters-fn-config
  annotations:
    config.kubernetes.io/local-config: "true"
data:
  CACHEURL_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  CTRL_PLANE_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT'
  PROVISIONING_INTERFACE: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  DHCP_RANGE: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  DHCP_HOSTS: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  DNS_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT'
  GATEWAY_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  IRONIC_SSH_KEY: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT'
EOF

# Edit the Kptfile to add content using the below command.
cat <<EOF >> Kptfile
pipeline:
  mutators:
    - image: gcr.io/kpt-fn/search-replace:v0.2.0
      configPath: search-replace.yaml
    - image: gcr.io/kpt-fn/create-setters:v0.1.0
      configPath: create-setters.yaml
    - image: gcr.io/kpt-fn/apply-setters:v0.2.0
      configPath: apply-setters.yaml
EOF

```

The next step before deploying the packages is to update the fields of the ConfigMaps that are specific to the
deployment environment.

Below are the ConfigMap fields and values in the bmo/cluster-capi-infrastructure-bmo.yaml. 
Refer the README at https://github.com/metal3-io/ironic-image for explanation of the fields in the ConfigMaps.
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

Below are the ConfigMap fields and values in the ironic/cluster-capi-infrastructure-ironic.yaml.
Refer the README at https://github.com/metal3-io/ironic-image for explanation of the fields in the ConfigMaps.
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

![Network Diagram](/static/images/install-guides/CapiMetal3.png)

Next, make sure to update the contents of the `apply-setters.yaml` file with environment specific values.
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: apply-setters-fn-config
  annotations:
    config.kubernetes.io/local-config: "true"
data:
  CACHEURL_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  CTRL_PLANE_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT'
  PROVISIONING_INTERFACE: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  DHCP_RANGE: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  DHCP_HOSTS: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  DNS_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT'
  GATEWAY_IP: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT' 
  IRONIC_SSH_KEY: 'INSERT_VALUE_FOR_YOUR_ENVIRONMENT'
```

NOTE: 
- Even if using static ip addressing for the nodes, DHCP is still needed during the PXE boot phase to get an initial IP and boot file information.
  The DHCP_HOSTS should contain the mac address info of the baremetal provisioning interface(s) that would be part of cluster, 
  DHCP_RANGE needs to be set to the IP addresses that does not conflict with static IP address range and will be used during the PXE boot phase of the ironic python agent.
- The PROVISIONING_INTERFACE is the interface where the Nephio Management cluster control plane IP can be reached.
- The values for DNS_IP and GATEWAY_IP can be gathered from the output of `resolvectl` and `ip route` commands on Nephio management cluster host.
- The value for CACHEURL_IP and IRONIC_SSH_KEY can be ignored if you do not have it setup or don't want to provide them.

Once, the `apply-setters.yaml` file is updated, issue the below command that updates the configMap fields in the bmo and ironic directories.
```bash
kpt fn render --truncate-output=false

# Sample output of above command below
Package "kpt_test/bmo": 
Package "kpt_test/ironic": 
Package "kpt_test/metal3": 
Package "kpt_test": 
[RUNNING] "gcr.io/kpt-fn/search-replace:v0.2.0"
[PASS] "gcr.io/kpt-fn/search-replace:v0.2.0" in 900ms
  Results:
    [info] data.CACHEURL: Mutated field value to "http://CACHEURL_IP_PLACEHOLDER/images"
    [info] data.CACHEURL: Mutated field value to "http://CACHEURL_IP_PLACEHOLDER/images"
[RUNNING] "gcr.io/kpt-fn/create-setters:v0.1.0"
[PASS] "gcr.io/kpt-fn/create-setters:v0.1.0" in 900ms
  Results:
    [info] data.CACHEURL: Added line comment "kpt-set: http://${CACHEURL_IP}/images" for field with value "http://CACHEURL_IP_PLACEHOLDER/images"
    [info] data.DEPLOY_KERNEL_URL: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6180/images/ironic-python-agent.kernel" for field with value "http://172.22.0.2:6180/images/ironic-python-agent.kernel"
    [info] data.DEPLOY_RAMDISK_URL: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6180/images/ironic-python-agent.initramfs" for field with value "http://172.22.0.2:6180/images/ironic-python-agent.initramfs"
    [info] data.DHCP_RANGE: Added line comment "kpt-set: ${DHCP_RANGE}" for field with value "172.22.0.10,172.22.0.100"
    [info] data.IRONIC_ENDPOINT: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6385/v1/" for field with value "http://172.22.0.2:6385/v1/"
    [info] data.PROVISIONING_INTERFACE: Added line comment "kpt-set: ${PROVISIONING_INTERFACE}" for field with value "eth2"
    [info] data.CACHEURL: Added line comment "kpt-set: http://${CACHEURL_IP}/images" for field with value "http://CACHEURL_IP_PLACEHOLDER/images"
    [info] data.DEPLOY_KERNEL_URL: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6180/images/ironic-python-agent.kernel" for field with value "http://172.22.0.2:6180/images/ironic-python-agent.kernel"
    [info] data.DEPLOY_RAMDISK_URL: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6180/images/ironic-python-agent.initramfs" for field with value "http://172.22.0.2:6180/images/ironic-python-agent.initramfs"
    [info] data.DHCP_HOSTS: Added line comment "kpt-set: ${DHCP_HOSTS}" for field with value "b4:96:91:c0:31:64,id:*;b4:96:91:c4:e3:f4,id:*"
    [info] data.DHCP_RANGE: Added line comment "kpt-set: ${DHCP_RANGE}" for field with value "172.22.0.10,172.22.0.100"
    [info] data.DNS_IP: Added line comment "kpt-set: ${DNS_IP}" for field with value "172.22.0.3"
    [info] data.GATEWAY_IP: Added line comment "kpt-set: ${GATEWAY_IP}" for field with value "172.22.0.4"
    [info] data.IRONIC_BASE_URL: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6385" for field with value "http://172.22.0.2:6385"
    [info] data.IRONIC_ENDPOINT: Added line comment "kpt-set: http://${CTRL_PLANE_IP}:6385/v1/" for field with value "http://172.22.0.2:6385/v1/"
    [info] data.IRONIC_RAMDISK_SSH_KEY: Added line comment "kpt-set: ${IRONIC_SSH_KEY}" for field with value "PLACEHOLDER_SSH_KEY"
    [info] data.PROVISIONING_INTERFACE: Added line comment "kpt-set: ${PROVISIONING_INTERFACE}" for field with value "eth2"
    [info] data.CACHEURL_IP: Added line comment "kpt-set: ${CACHEURL_IP}" for field with value "CACHEURL_IP_PLACEHOLDER"
    [info] data.CTRL_PLANE_IP: Added line comment "kpt-set: ${CTRL_PLANE_IP}" for field with value "172.22.0.2"
    [info] data.PROVISIONING_INTERFACE: Added line comment "kpt-set: ${PROVISIONING_INTERFACE}" for field with value "eth2"
    [info] data.DHCP_RANGE: Added line comment "kpt-set: ${DHCP_RANGE}" for field with value "172.22.0.10,172.22.0.100"
    [info] data.DHCP_HOSTS: Added line comment "kpt-set: ${DHCP_HOSTS}" for field with value "b4:96:91:c0:31:64,id:*;b4:96:91:c4:e3:f4,id:*"
    [info] data.DNS_IP: Added line comment "kpt-set: ${DNS_IP}" for field with value "172.22.0.3"
    [info] data.GATEWAY_IP: Added line comment "kpt-set: ${GATEWAY_IP}" for field with value "172.22.0.4"
    [info] data.IRONIC_SSH_KEY: Added line comment "kpt-set: ${IRONIC_SSH_KEY}" for field with value "PLACEHOLDER_SSH_KEY"
    [info] data.put-value: Added line comment "kpt-set: http://${CACHEURL_IP}/images" for field with value "http://CACHEURL_IP_PLACEHOLDER/images"
[RUNNING] "gcr.io/kpt-fn/apply-setters:v0.2.0"
[PASS] "gcr.io/kpt-fn/apply-setters:v0.2.0" in 900ms
  Results:
    [info] data.CACHEURL: set field value to "http://192.22.0.22/images"
    [info] data.DEPLOY_KERNEL_URL: set field value to "http://192.22.0.2:6180/images/ironic-python-agent.kernel"
    [info] data.DEPLOY_RAMDISK_URL: set field value to "http://192.22.0.2:6180/images/ironic-python-agent.initramfs"
    [info] data.DHCP_RANGE: set field value to "192.22.0.10,192.22.0.100"
    [info] data.IRONIC_ENDPOINT: set field value to "http://192.22.0.2:6385/v1/"
    [info] data.PROVISIONING_INTERFACE: set field value to "ens4"
    [info] data.CACHEURL: set field value to "http://192.22.0.22/images"
    [info] data.DEPLOY_KERNEL_URL: set field value to "http://192.22.0.2:6180/images/ironic-python-agent.kernel"
    [info] data.DEPLOY_RAMDISK_URL: set field value to "http://192.22.0.2:6180/images/ironic-python-agent.initramfs"
    [info] data.DHCP_HOSTS: set field value to "a4:96:91:c0:31:64,id:*;a4:96:91:c4:e3:f4,id:*"
    [info] data.DHCP_RANGE: set field value to "192.22.0.10,192.22.0.100"
    [info] data.DNS_IP: set field value to "192.22.0.3"
    [info] data.GATEWAY_IP: set field value to "192.22.0.4"
    [info] data.IRONIC_BASE_URL: set field value to "http://192.22.0.2:6385"
    [info] data.IRONIC_ENDPOINT: set field value to "http://192.22.0.2:6385/v1/"
    [info] data.IRONIC_RAMDISK_SSH_KEY: set field value to "P_SSH_KEY"
    [info] data.PROVISIONING_INTERFACE: set field value to "ens4"
    [info] data.CACHEURL_IP: set field value to "192.22.0.22"
    [info] data.CTRL_PLANE_IP: set field value to "192.22.0.2"
    [info] data.PROVISIONING_INTERFACE: set field value to "ens4"
    [info] data.DHCP_RANGE: set field value to "192.22.0.10,192.22.0.100"
    [info] data.DHCP_HOSTS: set field value to "a4:96:91:c0:31:64,id:*;a4:96:91:c4:e3:f4,id:*"
    [info] data.DNS_IP: set field value to "192.22.0.3"
    [info] data.GATEWAY_IP: set field value to "192.22.0.4"
    [info] data.IRONIC_SSH_KEY: set field value to "P_SSH_KEY"
    [info] data.put-value: set field value to "http://192.22.0.22/images"

Successfully executed 3 function(s) in 4 package(s).
```

Once, the environment specific fields in the ConfigMap for the packages in bmo and ironic directory have been updated,
issue the below commands that deploys the packages in metal3, bmo and ironic directories
```bash
kpt live init
kpt live apply --reconcile-timeout=15m --output=table
```
