---
title: Installing optional Baremetal components
description: >
  Step by step guide to configure and install components supporting Baremetal cluster installation. 

weight: 2
---

{{% alert title="Note" color="primary" %}}

Note that the instructions are for installing on Nephio management non-KIND cluster (has been verified on kubeadm cluster). The ironic pods use host networking that KIND
clusters do not support.

{{% /alert %}}

## Pre-requisites

- Access to non-kind cluster such as kubeadm cluster.
  - Refer [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) for installing kubeadm.
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
Refer to the following links for further explanation of the input fields in the configmaps.
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
The purpose of this step is to update the data.CACHEURL field with a placeholder value that can be customized in a later step.
Refer the function at https://catalog.kpt.dev/search-replace/v0.2/ to understand how this will be used by gcr.io/kpt-fn/search-replace:v0.2.0 function.
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
This will help later when applying our customized values for those fields.
Refer the function at https://catalog.kpt.dev/create-setters/v0.1/ to understand how this will be used by gcr.io/kpt-fn/create-setters:v0.1.0 function.
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
```

Sample output of above command below
```
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
```

Sample output
```
baremetal-operator-system           baremetal-operator-controller-manager-5c55b458cf-bcbkd           1/1     Running   0          28d
baremetal-operator-system           baremetal-operator-ironic-6888bb9cd5-782px                       4/4     Running   0          28d
capm3-system                        capm3-controller-manager-65c5895cc4-kznsm                        1/1     Running   0          28d
capm3-system                        ipam-controller-manager-69f4dd4cdf-zgl55                         1/1     Running   0          28d
```

## Baremetal cluster creation .

{{% alert title="Note" color="primary" %}}

Note that only a single node cluster with IPv4 static IP addressing is supported at this time.

{{% /alert %}}

### Pre-requisites

1. Refer the link at https://book.metal3.io/bmo/supported_hardware#supported-hardware for the Baremetal pre-requisites
2. Access to bare metal servers with BMCs connected to a network and accessible to the Nephio management cluster. 
3. Download the kubeadm node image found at https://artifactory.nordix.org/ui/native/metal3/images/ and copy the image to the ironic pod.
   Note that the image has to be copied all over again if the pod restarts.
   `kubectl cp <IMAGE_NAME>.qcow2 baremetal-operator-system/<ironic-pod-name>:/shared/html/images/<IMG_NAME>.img -c ironic-httpd -n baremetal-operator-system`

### Repository creation to access the packages in the baremetal folder
```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  labels:
    kpt.dev/repository-access: read-only
    kpt.dev/repository-content: external-blueprints
  name: baremetal-packages
  namespace: default
spec:
  content: Package
  deployment: false
  git:
    branch: main
    directory: /infra/baremetal
    repo: https://github.com/nephio-project/catalog.git
  type: git
EOF
```

Confirm that repository is created using the below command
```bash
kubectl get repositories
```

Sample output
```
NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
baremetal-packages          git    Package   false        True    https://github.com/nephio-project/catalog.git
```

Confirm that packagerevisions are available for the baremetal creation
```bash
kubectl get packagerevisions
```

Sample output
```
NAME                                                           PACKAGE                              WORKSPACENAME   REVISION   LATEST   LIFECYCLE   REPOSITORY
baremetal-packages.bmh-template.main                           bmh-template                         main            -1         false    Published   baremetal-packages
baremetal-packages.kubeadm-cluster-template-staticip.main      kubeadm-cluster-template-staticip    main            -1         false    Published   baremetal-packages
```

### BareMetalHost CR creation
Create a new PackageVariant CR for creating a BareMetalHost CR that is needed to be created before creating a cluster.

Have the below information ready for the baremetal server to be used in packagevariant CR creation.
- BMC username that is base64 encoded
- BMC password that is base64 encoded
- Boot MAC address (can be obtained from Redfish APIs or Server vendor BMC CLI/UI )
- Serial number of the disk drive where OS will be installed (can be obtained from Redfish APIs or Server vendor BMC CLI/UI )

Note: Replace the values under `spec.pipeline.mutators.configMap` with values specific to your server.
```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: bmh-sno1
spec:
  upstream:
    repo: baremetal-packages
    package: bmh-template
    revision: -1
    workspaceName: main
  downstream:
    repo: mgmt
    package: bmh-sno1
  annotations:
    approval.nephio.org/policy: initial
  pipeline:
    mutators:
      - image: "gcr.io/kpt-fn/apply-setters:v0.2.0"
        configMap:
          bmc-username: ZXhwZXJpbWVudFVzZXJuYW1l
          bmc-password: ZXhwZXJpbWVudFBhc3N3b3Jk
          bmh-name: bmh1
          boot-mac-address: 00:11:22:33:44:55
          root-hints-serial-number: PQZ123
          bmc-address: idrac-redfish://10.0.0.1/redfish/v1/Systems/1
          bmc-creds-name: bmc-secret1
EOF
```

Verify that the packagevariant and bmh CRs are created using the below commands
```bash
kubectl get packagevariants
kubectl get bmh
```

### Single Node Cluster creation using IPv4 Static IP addressing
Create a new PackageVariant CR for creating a single node cluster.

Have the below information ready from the server (either via Redfish APIs or the server vendors BMC UI/CLI) and/or from your lab admin.
- Static address IP pool information (IPv4) that includes start IP address, end IP address and network prefix
- Network address, DNS IP(s), Gateway IP
- `net-link-eth-id` can be obtained from looking up the `status.hardware.nics` field of the `bmh` CR created in above step

Note: Replace the values under `spec.pipeline.mutators.configMap` with values specific to your environment.
```bash
cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: cluster-staticip
spec:
  upstream:
    repo: baremetal-packages
    package: kubeadm-cluster-template-staticip
    revision: -1
    workspaceName: main
  downstream:
    repo: mgmt
    package: cluster-staticip
  annotations:
    approval.nephio.org/policy: initial
  pipeline:
    mutators:
      - image: "gcr.io/kpt-fn/apply-setters:v0.2.0"
        configMap:
          cluster-name: staticip-cluster
          namespace-value: default
          ctrl-plane-ip: 172.168.14.37
          ctrl-plane-port: "6443" 
          k8s-version: v1.29.0
          img-checksum: 5f1aea8dba3d7c5e0c4db8b2a83747f296adab6463e57a45610a5e544058aaca
          img-checksum-type: sha256
          img-format: qcow2
          img-url: http://172.168.14.61:6180/images/ubuntu-22.04-server-cloudimg-amd64-updated.img
          ippool-start-ip: 172.168.14.37
          ippool-end-ip: 172.168.14.37
          ippool-prefix: "27"
          net-link-eth-id: "ens1f0"
          net-link-eth-mac-addr: "00:11:22:33:44:55"
          net-svc-dns: |
            - 172.168.14.35
          net-ipv4-gw: 172.168.14.33
          net-ipv4-nw-addr: 0.0.0.0
          ctrl-node-user: ubuntu
          ctrl-node-hashed-passwd: $6$n7odPOnQCIM0c4qB$px8Vm7z/xj4TRAIvd3WTfIv4ZNbvelpXAZT1Sv3XppYquLbJ3abUUnXTR0vvOr/eeQJnmBxTSELoTPZnQ3ni50
          ctrl-node-ssh-key: 'ssh-rsa key nobody@nobody'
EOF
```

Verify that the packagevariant and cluster CRs are created using the below commands
```bash
kubectl get packagevariants
kubectl get clusters
```

{{% alert title="Note" color="primary" %}}

NOTE: The network diagram highlights simple connection between Nephio management cluster (provisioning cluster) and workload cluster (target). The diagram does not consider switches or routers. If you are using switches then make sure the connectivity is properly configured.

{{% /alert %}}

![Network Diagram](/static/images/install-guides/CapiMetal3.png)
