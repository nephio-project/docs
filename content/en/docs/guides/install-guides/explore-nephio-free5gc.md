---
title: Nephio Free5gc Operator
description: >
  Step by step guide to deploy the Free5gc Operator
weight: 6
---

Concepts as [Operator](/content/en/docs/glossary-abbreviations#operator) and
[Controller](/content/en/docs/glossary-abbreviations#controller) and
[CR](/content/en/docs/glossary-abbreviations#custom-resource) have already been explained here.

## Deploy the operator to the cluster

In the [Nephio Free5gc repository](https://github.com/nephio-project/free5gc) the **Makefile** is used to
[deploy the operator to the cluster](https://github.com/nephio-project/free5gc/tree/main#getting-started) automating
tasks such as 

1. Build Targets:
 * build: Builds the operator binary.
 * run: Runs the controller from the host.
 * docker-build: Builds a Docker image with the operator.
 * docker-push: Pushes the Docker image with the operator.

2. Deployment Targets:
 * install: Installs CustomResourceDefinitions (CRDs) into the Kubernetes cluster.
 * uninstall: Uninstalls CRDs from the Kubernetes cluster.
 * deploy: Deploys the controller to the Kubernetes cluster.
 * undeploy: Undeploys the controller from the Kubernetes cluster.

3. Build Dependencies:
 * Targets for installing and managing build dependencies like Kustomize, controller-gen, and envtest.

## The structure of the repository

1. [Operator](https://github.com/nephio-project/free5gc/tree/main/free5gc-operator)
 * Binding metrics and check the health of ports
 * Registering UPF SMF AMF deployments
 * Reconciler and Setup (Creating Controllers)

2. [Controllers](https://github.com/nephio-project/free5gc/tree/main/controllers)
 * **Reconciler**: The XXFDeploymentReconciler struct is responsible for reconciling the state of the XXFDeployment
   resource in the Kubernetes cluster. It implements the Reconcile function, which is called by the Controller Runtime
   framework when changes occur to the XXFDeployment resource. The Reconcile function performs various operations such
   as creating or updating the ConfigMap and Service resources associated with the XXFDeployment.
   Overall, the XXFDeploymentReconciler struct acts as the controller for the XXFDeployment resource, ensuring that the
   cluster state aligns with the desired state specified by the user.
 * **Resources**: functions that provide the necessary logic to create the required Kubernetes resources for an XXF
   deployment, including the deployment, service, and configuration map: 

   * createDeployment: This function creates a Deployment resource for the AMF deployment. It defines the desired
     state of the deployment, including the number of replicas, container image, ports, command, arguments, volume
     mounts, resource requirements, and security context.
   * createService: This function creates a Service resource for the AMF deployment. It defines the desired state of
     the service, including the selector for the associated deployment and the ports it exposes.
   * createConfigMap: This function creates a ConfigMap resource for the AMF deployment. It generates the
     configuration data for the AMF based on the provided template values and renders it into the amfcfg.yaml file.
   * createResourceRequirements: This function calculates the resource requirements (CPU and memory limits and
     requests) for the AMF deployment based on the specified capacity and sets them in a ResourceRequirements object.
   * createNetworkAttachmentDefinitionNetworks: This function creates the network attachment definition networks for
     the AMF deployment. It uses the CreateNetworkAttachmentDefinitionNetworks function from the controllers package to
     generate the network attachment definition YAML based on the provided template name and interface configurations.
 * **Templates**: The configuration template includes various parameters. Example for AMF: version, description,
   ngapIpList, sbi, nrfUri, amfName, serviceNameList, servedGuamiList, supportTaiList, plmnSupportList, supportDnnList,
   security settings, networkName, locality, networkFeatureSupport5GS, timers, and logger configurations.
   The renderConfigurationTemplate function takes a struct (configurationTemplateValues) containing the values for
   placeholders in the template and renders the final configuration as a string. The rendered configuration can then be
   used by the AMF application.
 * **Status**: It holds the logic to get the status of the deployment and displaying it as "Available," "Progressing,"
   and "ReplicaFailure".The function returns the NFDeploymentStatus object and a boolean value indicating whether the
   status has been updated or not.

3. [Config](https://github.com/nephio-project/free5gc/tree/main/config)

   There are [Kustomization](https://github.com/kubernetes-sigs/kustomize) files for a Kubernetes application, specifying various configuration options and resources for the application.

   In the */default* folder there are:

* *Namespace*: Defines the namespace (free5gc) where all resources will be deployed.
* *Name Prefix*: Specifies a prefix (free5gc-operator-) that will be prepended to the names of all resources.
* *Common Labels*: Allows adding labels to all resources and selectors. Currently commented out.
* *Bases*: Specifies the directories (*../crd*, *../rbac*, *../operator*) containing the base resources for the application.
  In the *crd/base* folder there are CRDs for the workload network functions. They define the schema for the
  "XXFDeployment" resource under the "workload.nephio.org" group. Also, there are YAML config files for teaching
  kustomize how to substitute *name* and *namespace* reference in CRD.
* *Patches Strategic Merge*: Specifies the patches that should be applied to the base resources.
  Currently includes a patch file named *manager_auth_proxy_patch.yaml*, which adds authentication protection to the
  */metrics* endpoint.
* **Vars**: Defines variables that can be used for variable substitution.
    
