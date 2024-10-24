---
title: Helm to Operator Codegen SDK
description: >
  Deploying helm charts in Nephio using Helm To Operator Codegen SDK
weight: 1
---

The [Helm to Operator Codegen SDK](https://github.com/nephio-project/nephio-sdk/tree/main/helm-to-operator-codegen-sdk)
offers a streamlined solution for translating existing Helm charts into Kubernetes operators with minimal effort and
cost.

By utilizing the Helm to Operator Codegen SDK, users can efficiently convert existing Helm charts into Kubernetes
operators. This transition enables users to leverage the advanced capabilities provided by Kubernetes operators, such as
enhanced automation, lifecycle management, and custom logic handling. Overall, the SDK facilitates a smooth migration
process, empowering users to embrace the operator model for managing their Kubernetes resources effectively.

## Exercise: Deploying Free5gc using operator

In the following exercise, the objective is to convert the Free5gc Helm chart to Go code suitable for a Kubernetes
operator using the SDK. Once the conversion is complete, the generated Go code will be used to deploy all the resources
defined in the Free5gc Helm chart using a Kubernetes operator.

### Step 0: Prerequisite

1. GoLang Version: 1.21
2. Helm : v3.9.3
3. Kubebuilder
4. A Kubernetes Cluster with Calico CNI and Multus CNI plugin (Can Refer
   [here](https://medium.com/rahasak/deploying-5g-core-network-with-free5gc-kubernets-and-helm-charts-29741cea3922),
   Before "Deploy Helm-Chart Part" )
5. Go Packages:

    ```bash
    # Clone the Repo
    git clone https://github.com/nephio-project/nephio-sdk.git
    cd nephio-sdk/helm-to-operator-codegen-sdk/
    go mod tidy
    ```

### Step 1: Convert the helm-chart to Go-Code using Helm-to-operator-codegen-sdk

Currently, only Local-Helm charts are supported by SDK, Therefore, the first step would be to download the
free5gc-Helm-Chart. (Refer [here](https://github.com/Orange-OpenSource/towards5gs-helm/tree/main))

To initiate the conversion process using the SDK, you can use the following command:

```bash
go run main.go <path_to_local_helm_chart> <namespace> <logging-level>
where:
    <path_to_local_helm_chart>: Path to your local chart, the folder must contain a chart.yaml file.
    <namespace>: The namespace you want to deploy the resources
    <logging-level>: debug, info (default), error, warn

```


#### Example Run

```
go run main.go /home/ubuntu/free5gccharts/towards5gs-helm/charts/free5gc/ free5gcns info
```
<details>
<summary>The output is similar to:</summary>

```bash
INFO[0000] free5gcns ../../testing_helpers/free5gc_helm_chart/towards5gs-helm/charts/free5gc/
INFO[0000]  ----------------- Converting Helm to Yaml --------------------------
WARN[0000] Duplication Detected in Struct Mapping | For Preconditions
WARN[0000] Duplication Detected in Struct Mapping | For ConditionStatus
WARN[0000] Duplication Detected in Enum Mapping | For ConditionStatus
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-amf/templates/amf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-amf/templates/amf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-amf/templates/amf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-amf/templates/amf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-amf/templates/amf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-amf/templates/amf-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-amf/templates/amf-n2-nad.yaml
INFO[0000] Kind | NetworkAttachmentDefinition Would Be Treated as Third Party Kind
INFO[0000]       Converting Unstructured to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-amf/templates/amf-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-amf/templates/amf-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-ausf/templates/ausf-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-dbpython/templates/dbpython-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-dbpython/templates/dbpython-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-nrf/templates/nrf-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-nssf/templates/nssf-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-pcf/templates/pcf-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-smf/templates/smf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-smf/templates/smf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-smf/templates/smf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-smf/templates/smf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-smf/templates/smf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-smf/templates/smf-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-smf/templates/smf-n4-nad.yaml
INFO[0000] Kind | NetworkAttachmentDefinition Would Be Treated as Third Party Kind
INFO[0000]       Converting Unstructured to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-smf/templates/smf-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-smf/templates/smf-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udm/templates/udm-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-udm/templates/udm-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udm/templates/udm-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-udm/templates/udm-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udm/templates/udm-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udm/templates/udm-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udm/templates/udm-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-udm/templates/udm-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udr/templates/udr-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-udr/templates/udr-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udr/templates/udr-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-udr/templates/udr-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udr/templates/udr-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udr/templates/udr-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-udr/templates/udr-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-udr/templates/udr-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf/upf-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-upf/templates/upf/upf-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf/upf-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-upf/templates/upf/upf-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf/upf-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf-n3-nad.yaml
INFO[0000] Kind | NetworkAttachmentDefinition Would Be Treated as Third Party Kind
INFO[0000]       Converting Unstructured to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf-n4-nad.yaml
INFO[0000] Kind | NetworkAttachmentDefinition Would Be Treated as Third Party Kind
INFO[0000]       Converting Unstructured to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf-n6-nad.yaml
INFO[0000] Kind | NetworkAttachmentDefinition Would Be Treated as Third Party Kind
INFO[0000]       Converting Unstructured to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf-n9-nad.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf1/upf1-configmap.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf1/upf1-deployment.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf1/upf1-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf2/upf2-configmap.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf2/upf2-deployment.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upf2/upf2-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upfb/upfb-configmap.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upfb/upfb-deployment.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-upf/templates/upfb/upfb-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-webui/templates/webui-configmap.yaml
INFO[0000]  Current KRM Resource| Kind : ConfigMap| YamlFilePath : temp/templated/free5gc/charts/free5gc-webui/templates/webui-configmap.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-webui/templates/webui-deployment.yaml
INFO[0000]  Current KRM Resource| Kind : Deployment| YamlFilePath : temp/templated/free5gc/charts/free5gc-webui/templates/webui-deployment.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-webui/templates/webui-hpa.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-webui/templates/webui-ingress.yaml
ERRO[0000] Unable to convert yaml to unstructured |Object 'Kind' is missing in 'null'
INFO[0000] CurFile --> | temp/templated/free5gc/charts/free5gc-webui/templates/webui-service.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/free5gc-webui/templates/webui-service.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/mongodb/templates/serviceaccount.yaml
INFO[0000]  Current KRM Resource| Kind : ServiceAccount| YamlFilePath : temp/templated/free5gc/charts/mongodb/templates/serviceaccount.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/mongodb/templates/standalone/dep-sts.yaml
INFO[0000]  Current KRM Resource| Kind : StatefulSet| YamlFilePath : temp/templated/free5gc/charts/mongodb/templates/standalone/dep-sts.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] CurFile --> | temp/templated/free5gc/charts/mongodb/templates/standalone/svc.yaml
INFO[0000]  Current KRM Resource| Kind : Service| YamlFilePath : temp/templated/free5gc/charts/mongodb/templates/standalone/svc.yaml
INFO[0000]       Converting Runtime to Json Completed
INFO[0000]       Converting Json to String Completed
INFO[0000] ----------------- Writing GO Code ---------------------------------
INFO[0000] ----------------- Program Run Successful| Summary ---------------------------------
INFO[0000] Deployment            |11
INFO[0000] NetworkAttachmentDefinition           |5
INFO[0000] Service               |10
INFO[0000] ServiceAccount                |1
INFO[0000] StatefulSet           |1
INFO[0000] ConfigMap             |10

```
</details>

The generated Go-Code would be written to the "outputs/generated_code.go" file

The Generated Go-Code shall contain the following functions:

#### A) Pluggable functions

1. CreateAll():  When called, it will create all the k8s resources(services, deployment) on the Kubernetes cluster.
    {{% alert title="Note" color="primary" %}}
      For your Reconciler to call the function, Replace *YourKindReconciler* with the type of your Reconciler
    {{% /alert %}}

2. DeleteAll(): When called, it will delete all the k8s resources(services, deployment) on the Kubernetes cluster.
    {{% alert title="Note" color="primary" %}}
      For your Reconciler to call the function, Replace *YourKindReconciler* with the type of your Reconciler
    {{% /alert %}}
3. Getxxx(): Shall return the list of a particular resource.
    1. GetService(): Shall return the list of all services.
    2. GetDeployment(): Shall return the list of all deployments. & so on

#### B) Helper Functions: (For internal use only)

1. *deleteMeAfterDeletingUnusedImportedModules*: This function is included in the generated Go code to handle the
   scenario where a module is imported but not used. Once the user has removed the non-required modules from the import
   statements, they can safely delete this function as well.
2. Pointer Functions: *intPtr()*, *int16Ptr()*, *int32Ptr()*, *int64Ptr()*, *boolPtr()*, *stringPtr()*: Takes the value
   and returns the pointer to that value.
3. *getDataForSecret*: This function takes the *encodedVal* of Secret, decodes it, and returns.

### Step 2: Using Kubebuilder to develop the operator

Please refer [here](https://book.kubebuilder.io/quick-start) to develop & deploy the operator.

After the basic structure of the operator is created, users can proceed to add their business logic. The *CreateAll()*
and *DeleteAll()* functions generated by the SDK can be leveraged for Day-0 resource deployments, allowing users to
manage the creation and deletion of resources defined in the Helm chart. By integrating their business logic with
these functions, users can ensure that their operator effectively handles resource lifecycle management and
orchestration within a Kubernetes environment.

In the end, all the resources created could be checked by:
`kubectl get pods -n free5gcns`

The output Should be:

```bash
mongodb-0                                                1/1     Running             0          87m
release-name-free5gc-amf-amf-8649b95f6-hkrqr             1/1     Running             0          87m
release-name-free5gc-ausf-ausf-78864bd7c4-cgc8v          1/1     Running             0          87m
release-name-free5gc-dbpython-dbpython-6f7fbbbd4-qmg4b   1/1     Running             0          87m
release-name-free5gc-nrf-nrf-7f99d8bbcc-kkzzv            1/1     Running             0          87m
release-name-free5gc-nssf-nssf-bdbf85c77-2dkd4           1/1     Running             0          87m
release-name-free5gc-pcf-pcf-c447dcfb6-ftnnv             1/1     Running             0          87m
release-name-free5gc-smf-smf-797db658cf-ndnss            1/1     Running             0          87m
release-name-free5gc-udm-udm-64d6c55855-6sl9n            1/1     Running             0          87m
release-name-free5gc-udr-udr-7566b84c8b-5n7dv            1/1     Running             0          87m
release-name-free5gc-upf-upf-76bd87cc76-bfqsw            1/1     Running             0          87m
release-name-free5gc-webui-webui-7dccf6c877-lj5p5        1/1     Running             0          87m

```

----
For advanced requirements, Please refer to the developer guide [here](/content/en/docs/guides/contributor-guides/helm-to-operator-codegen-sdk-developer-guide.md)