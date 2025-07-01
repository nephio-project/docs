---
title: Installing optional Baremetal components
description: >
  Step by step guide to configure and install components supporting Baremetal cluster installation. 

weight: 3
---

Note that the instructions are for installing on Nephio management non-KIND cluster (has been verified on kubeadm cluster). The ironic pods use host networking that KIND
clusters do not support.


## Pre-requisites

- Access to non-kind cluster such as kubeadm cluster.
  - Refer https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ for installing kubeadm.
- Nephio management components and porch installed.
  - Refer to the "Preinstalled K8s cluster" section in [Kicking off an installation on a virtual machine](./_index.md/#kicking-off-an-installation-on-a-virtual-machine)

## Metal3, BMO and Ironic packages install on Nephio management cluster (non-kind cluster)

### Create a directory to pull the packages and initialize them

```bash
mkdir -p /tmp/baremetal_kpt
cd /tmp/baremetal_kpt

# The below three commands will create three directories named metal3, ironic and bmo
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-metal3@main metal3
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-ironic@main ironic
kpt pkg get https://github.com/nephio-project/catalog/infra/capi/cluster-capi-infrastructure-bmo@main bmo

# The below command creates a Kptfile, README.md and package-context.yaml file
kpt pkg init
```

### Prepare for customizing the packages for your environment by executing the below steps.

Note that the below referenced configmaps `bmo-configmap.yaml` and `ironic-configmap.yaml` in the `ironic` and `bmo` packages are to be customized for the deployment environment.
Refer the links for explanation of some of the fields in the configmap at
- https://github.com/metal3-io/baremetal-operator/blob/main/docs/configuration.md
- https://book.metal3.io/quick-start
- https://book.metal3.io/ironic/ironic_installation#environmental-variables
- https://github.com/metal3-io/ironic-image

#### bmo-configmap
```bash
---
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
metadata:
  name: ironic
  namespace: baremetal-operator-system
```

#### ironic-configmap
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
metadata: 
  name: baremetal-operator-ironic-bmo-configmap-6cf9t7484b
  namespace: baremetal-operator-system
```

#### 1. Create search-replace.yaml file using the below command.
The purpose of this step to update the value for the data.CACHEURL field with a placeholder value that can be customized in a later step.
Note: Refer the function at https://catalog.kpt.dev/search-replace/v0.2/ to understand how this will be used by gcr.io/kpt-fn/search-replace:v0.2.0 function.
```bash
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
```

#### 2. Create create-setters.yaml file using the below command.
The purpose of this step is to add comments to the fields matching the setter values using setter names as parameters.
This will help later with apply customized values for those fields.
Note: Refer the function at https://catalog.kpt.dev/create-setters/v0.1/ to understand how this will be used by gcr.io/kpt-fn/create-setters:v0.1.0 function.
```bash
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
```

#### 3. Create `apply-setters.yaml` file using the below command.
The purpose of this step is to update the field values parameterized by setters in the above step.
Note: Refer the function at https://catalog.kpt.dev/apply-setters/v0.2/ to understand how this will be used by gcr.io/kpt-fn/apply-setters:v0.2.0 function.

```bash
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
```

{{% alert title="Note" color="primary" %}}

- Even if static ip addressing is being used for the nodes, DHCP is still needed during the PXE boot phase to get an initial IP and boot file information.
  The DHCP_HOSTS should contain the mac address info of the baremetal provisioning interface(s) that would be part of cluster,
  DHCP_RANGE needs to be set to an IP addresses that does not conflict with static IP address range, and will be used during the PXE boot phase of the ironic python agent.
- The PROVISIONING_INTERFACE is the interface where the Nephio Management cluster control plane IP can be reached.
- The values for DNS_IP and GATEWAY_IP can be gathered from the output of `resolvectl` and `ip route` commands on Nephio management cluster host.
- The value for CACHEURL_IP and IRONIC_SSH_KEY can be ignored if unused.

{{% /alert %}}


#### 4. Edit the Kptfile to add content using the below command.
The purpose of this step is to add containerized functions to be invoked by the pipeline when rendering the packages.
```bash
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

### Render and deploy the package with values customized for your environment
The next step before deploying the packages is to review that the values in the `apply-setters.yaml` file are specific to the
deployment environment.
Once the `apply-setters.yaml` file is reviewed and updated, issue the below command to update the configMap fields in the bmo and ironic directories.
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

Once the ConfigMap values for the packages have been updated,
issue the below commands to deploy the packages to the cluster.
```bash
kpt live init
kpt live apply --reconcile-timeout=15m --output=table
```

Verify that the pods are up and running using the below command
```bash
kubectl get pods --all-namespaces | grep -E '^(capm3-system|baremetal-operator-system)'

baremetal-operator-system           baremetal-operator-controller-manager-5c55b458cf-bcbkd           1/1     Running   0          28d
baremetal-operator-system           baremetal-operator-ironic-6888bb9cd5-782px                       4/4     Running   0          28d
capm3-system                        capm3-controller-manager-65c5895cc4-kznsm                        1/1     Running   0          28d
capm3-system                        ipam-controller-manager-69f4dd4cdf-zgl55                         1/1     Running   0          28d
```

The network diagram highlights simple connection between Nephio management cluster (provisioning cluster) and workload cluster (target). The diagram does not consider switches or routers. If you are using switches then make sure the connectivity is properly configured.

![Network Diagram](/static/images/install-guides/CapiMetal3.png)
