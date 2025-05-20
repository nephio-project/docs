---
title: O-RAN O-Cloud K8s Cluster deployment
description: >
  A step by step guide to deploying a workload cluster using the o2ims interface.
weight: 2
---

## Prerequisites

- A Nephio Management cluster: 
  - See the [installation guides](/content/en/docs/guides/install-guides/_index.md#introduction) for detailed demo environment options.
- With the following *optional* operator pkgs deployed:
  - [o2ims operator](/content/en/docs/guides/install-guides/optional-components.md#o2ims-operator)


This exercise will take us from a system with only the Nephio Management cluster setup, to a deployment with:

- A [workload cluster](https://github.com/nephio-project/catalog/tree/main/infra/capi/nephio-workload-cluster) specified 
 as a template when instantiating an instance of the [ProvisioningRequest CRD](https://github.com/nephio-project/api/blob/main/config/crd/bases/o2ims.provisioning.oran.org_provisioningrequests.yaml).
- A repository for said cluster, registered with Nephio Porch.

To perform these exercises, we will need:

- Access to the installed demo VM environment as the ubuntu user.

The exercise will attempt to simulate the [O-RAN O-Cloud](/content/en/docs/network-architecture/o-ran-integration.md#overview) 
architecture by deploying the [focom operator](/content/en/docs/guides/install-guides/optional-components.md#focom-operator)
to a separate KinD cluster, which will represent the SMO Nephio Management Cluster.

### Step 1: Verify the O2IMS operator deployment

Check the operator pod status
```bash
kubectl get po -n o2ims
```
Example output
```
NAME                              READY   STATUS    RESTARTS   AGE
o2ims-operator-5595cd78b7-mwdxz   1/1     Running   0          5m
```
Verify that the o2ims CRD is installed
```bash
kubectl get crd | grep provisioningrequests
```
Example output
```
NAME                                                         CREATED AT
provisioningrequests.o2ims.provisioning.oran.org             2025-03-19T00:41:57Z
```

## Step 2: Deploy the Focom Operator to separate cluster

Create a default kind cluster to simulate the *SMO cluster* env
```bash
kind create cluster -n focom-cluster --kubeconfig /tmp/focom-kubeconfig
```

Verify the cluster is healthy
```bash
kubectl get po -A --kubeconfig /tmp/focom-kubeconfig
```

Deploy the [focom operator](/content/en/docs/guides/install-guides/optional-components.md#focom-operator) to the new cluster
```bash
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/optional/focom-operator@origin/main /tmp/focom
kpt fn render /tmp/focom
kpt live init /tmp/focom --kubeconfig /tmp/focom-kubeconfig
kpt live apply /tmp/focom --reconcile-timeout=15m --output=table --kubeconfig /tmp/focom-kubeconfig
```

Verify the pod is healthy
```bash
kubectl get po -n focom-operator-system --kubeconfig /tmp/focom-kubeconfig
```
Example output
```
NAME                                                READY   STATUS    RESTARTS   AGE
focom-operator-controller-manager-d8f4d5cb6-dbxjh   1/1     Running   0          99s
```

Verify that the relevant CRDs are available
```bash
kubectl get crd --kubeconfig /tmp/focom-kubeconfig
```
Example output
```bash
NAME                                         CREATED AT
focomprovisioningrequests.focom.nephio.org   2025-03-19T08:30:07Z
oclouds.focom.nephio.org                     2025-03-19T08:30:07Z
resourcegroups.kpt.dev                       2025-03-19T08:29:57Z
templateinfoes.provisioning.oran.org         2025-03-19T08:30:07Z
```

## Step 3: Create  kubeconfig Secret

This step may vary depending on the networking of the demo environment.

Get the kube-api IP Address of the target O-Cloud cluster where the o2ims operator is deployed
```bash
IP=$(docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kind-control-plane)
```

Create a temporary kubeconfig, substituting the *IP* Address
```bash
sed "s|https://127.0.0.1:[^ ]*|https://$IP:6443|" ~/.kube/config > /tmp/kubeconfig-bak
```

Verify the edit
```bash
cat /tmp/kubeconfig-bak | grep server
```
Example output
```
server: https://172.18.0.2:6443
```

Create the secret on the *SMO cluster* using the demo kubeconfig
```bash
kubectl create secret generic ocloud-kubeconfig --from-file=kubeconfig=/tmp/kubeconfig-bak --kubeconfig /tmp/focom-kubeconfig
```
Example output
```bash
secret/ocloud-kubeconfig created
```

## Step 4: Create the demo CRs to trigger the Focom Provisioning Request

Create the *OCloud* CR, referencing the K8s Secret created previously
```bash
cat << EOF | kubectl apply --kubeconfig /tmp/focom-kubeconfig -f - 
apiVersion: focom.nephio.org/v1alpha1
kind: OCloud
metadata:
  name: ocloud-1
  namespace: focom-operator-system
spec:
  o2imsSecret:
    secretRef:
      name: ocloud-kubeconfig
      namespace: default

EOF
```
Example output
```bash
ocloud.focom.nephio.org/ocloud-1 created
```

Create the required *TemplateInfo* CR
```bash
cat << EOF | kubectl apply --kubeconfig /tmp/focom-kubeconfig -f - 
apiVersion: provisioning.oran.org/v1alpha1
kind: TemplateInfo
metadata:
  name: nephio-workload-cluster
  namespace: focom-operator-system
spec:
  templateName: nephio-workload-cluster
  templateVersion: main
  templateParameterSchema: |
    {
      "type": "object",
      "infra": {
      "param1": {
          "type": "string"
        },
        "params": {
          "type": "integer"
        }
      },
      "required": ["param1"]
    }

EOF
```
Example output
```bash
templateinfo.provisioning.oran.org/nephio-workload-cluster created
```

Create the *FocomProvisioningRequest*
```bash
cat << EOF | kubectl apply --kubeconfig /tmp/focom-kubeconfig -f - 
apiVersion: focom.nephio.org/v1alpha1
kind: FocomProvisioningRequest
metadata:
  name: focom-cluster-prov-req-nephio
  namespace: focom-operator-system
spec:
  name: sample-edge
  description: "Provisioning request for setting up a sample edge kind cluster"
  oCloudId: ocloud-1
  oCloudNamespace: focom-operator-system
  templateName: nephio-workload-cluster
  templateVersion: main
  templateParameters:
    clusterName: edge
    labels:
      nephio.org/site-type: edge
      nephio.org/region: europe-paris-west
      nephio.org/owner: nephio-o2ims

EOF
```
Example output
```bash
focomprovisioningrequest.focom.nephio.org/focom-cluster-prov-req-nephio created
```

## Step 5: Monitor the progress of the *ProvisioningRequest* 

Verify that the *provisioningrequest* has been created on the target O-Cloud cluster
```bash
kubectl get provisioningrequests
```
Example output
```
NAME                            AGE
focom-cluster-prov-req-nephio   15s
```

Verify that the *packagevariants* are created for the edge-cluster. This may take some time.
```bash
kubectl get packagevariants
```
Example output
```bash
NAME                            AGE
edge-cluster                    1m
edge-configsync                 1m
edge-crds                       1m
edge-kindnet                    1m
edge-local-path-provisioner     1m
edge-metallb                    1m
edge-multus                     1m
edge-repo                       1m
edge-rootsync                   1m
edge-vlanindex                  1m
focom-cluster-prov-req-nephio   2m18s
```

Examine the details of the *provisioningrequest* CR created
```bash
kubectl get provisioningrequests focom-cluster-prov-req-nephio -o yaml
```
Example output
```yaml
apiVersion: o2ims.provisioning.oran.org/v1alpha1
kind: ProvisioningRequest
metadata:
  annotations:
    provisioningrequests.o2ims.provisioning.oran.org/kopf-managed: "yes"
    provisioningrequests.o2ims.provisioning.oran.org/last-ha-a.A3qw: |
      {"spec":{"description":"Provisioning request for setting up a sample edge kind cluster","name":"sample-edge","templateName":"nephio-workload-cluster","templateParameters":{"clusterName":"edge","labels":{"nephio.org/owner":"nephio-o2ims","nephio.org/region":"europe-paris-west","nephio.org/site-type":"edge"}},"templateVersion":"main"}}
    provisioningrequests.o2ims.provisioning.oran.org/last-handled-configuration: |
      {"spec":{"description":"Provisioning request for setting up a sample edge kind cluster","name":"sample-edge","templateName":"nephio-workload-cluster","templateParameters":{"clusterName":"edge","labels":{"nephio.org/owner":"nephio-o2ims","nephio.org/region":"europe-paris-west","nephio.org/site-type":"edge"}},"templateVersion":"main"}}
  creationTimestamp: "2025-03-19T09:10:23Z"
  generation: 1
  name: focom-cluster-prov-req-nephio
  resourceVersion: "186743"
  uid: 95ba053e-7020-483b-bfc4-41f3cddb39d2
spec:
  description: Provisioning request for setting up a sample edge kind cluster
  name: sample-edge
  templateName: nephio-workload-cluster
  templateParameters:
    clusterName: edge
    labels:
      nephio.org/owner: nephio-o2ims
      nephio.org/region: europe-paris-west
      nephio.org/site-type: edge
  templateVersion: main
status:
  provisionedResourceSet:
    oCloudInfrastructureResourceIds:
    - dd62bdc9-ecea-4b83-9484-a34c9cddbf4a
    oCloudNodeClusterId: d10b046d-38bd-4efe-bd08-bb4c4a380371
  provisioningStatus:
    provisioningMessage: Cluster resource created
    provisioningState: fulfilled
    provisioningUpdateTime: "2025-03-19T09:10:31Z"

```

Check the workload cluster deployment status. This may take some time.

```bash
kubectl get cl
```
Example output:
```bash
NAME       CLUSTERCLASS   PHASE         AGE    VERSION
edge       docker         Provisioned   1m     v1.31.0
```

You should also check that the Workload KinD cluster has come up fully by checking the `machinesets`
You should see READY and AVAILABLE replicas. This may take some time.

```bash
kubectl get machinesets
```
Example output:
```
NAME                        CLUSTER    REPLICAS   READY   AVAILABLE   AGE    VERSION
edge-md-0-7gp6p-h8gmg       edge       1          1       1           3h1m   v1.31.0
```

Check that the associated gitea Porch *repository* is ready for the edge cluster
```bash
kubectl get repository | grep edge
```
Example output:
```
edge       git    Package   true         True    http://172.18.0.200:3000/nephio/edge.git
```