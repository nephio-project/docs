---
title: Helm to Operator Codegen Sdk Developer Guide
weight: 1
---

The [Helm to Operator Codegen SDK](https://github.com/nephio-project/nephio-sdk/tree/main/helm-to-operator-codegen-sdk) offers a streamlined solution for translating existing Helm charts into Kubernetes operators with minimal effort and cost.

## The Flow Diagram
In a nutshell, Firstly, the Helm-Charts are converted to Yamls using the values provided in "values.yaml". Then, each Kubernetes Resource Manifest (KRM) in the YAML is translated into Go code, employing one of two methods.
1) If the resource is Runtime-Supported, it undergoes a conversion process where the KRM resource is first transformed into a Runtime Object, then into JSON, and finally into Go code.
2) Otherwise, if the resource is not Runtime-Supported, it is converted into an Unstructured Object and then into Go code.

After the conversion process, all the generated Go code is gathered and compiled into a single Go file. This resulting file contains functions that can be readily utilized by Kubernetes operators.

![alt Flow Diagram](/static/images/user-guides/helm-to-operator-codegen-sdk-flow-diagram.jpg)

-----
### Flow-1: Helm to Yaml
Helm to Yaml conversion is achieved by running the command
`helm template <chart>  --namespace <namespace>  --output-dir “temp/templated/”` Internally. As of now, It retrieves the values from default "values.yaml"

### Flow-2: Yaml Split
The SDK iterates over each YAML file in the "converted-yamls" directory. If a YAML file contains multiple Kubernetes Resource Manifests (KRM), separated by "---", the SDK splits the YAML file accordingly to isolate each individual KRM resource. This ensures that each KRM resource is processed independently.

### Runtime-Object and Unstruct-Object
The SDK currently employs the "runtime-object method" to handle Kubernetes resources whose structure is recognized by Kubernetes by default. Examples of such resources include Deployment, Service, and ConfigMap. Conversely, resources that are not inherently known to Kubernetes and require explicit installation or definition, such as Third-Party Custom Resource Definitions (CRDs) like NetworkAttachmentDefinition or PrometheusRule, are processed using the "unstructured-object" method. Such examples are given below:

<details>
<summary>Example</summary>

```
// Runtime-Object Example
service1 := &corev1.Service{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "v1",
			Kind:       "Service",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name: "my-service",
		},
		Spec: corev1.ServiceSpec{
			Selector: map[string]string{
				"app.kubernetes.io/name": "MyApp",
			},
			Ports: []corev1.ServicePort{
				corev1.ServicePort{
					Port:     80,
					Protocol: corev1.Protocol("TCP"),
					TargetPort: intstr.IntOrString{
						IntVal: 9376,
					},
				},
			},
		},
	}

// Unstruct-Object Example
networkAttachmentDefinition1 := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "k8s.cni.cncf.io/v1",
			"kind":       "NetworkAttachmentDefinition",
			"metadata": map[string]interface{}{
				"name": "macvlan-conf",
			},
			"spec": map[string]interface{}{
				"config": "some-config",
			},
		},
	}
```
</details>

### Flow-3.1: KRM to Runtime-Object
The conversion process relies on the "k8s.io/apimachinery/pkg/runtime" package. Currently, only the API version "v1" is supported. The supported kinds for the Runtime Object method include:
`Deployment, Service, Secret, Role, RoleBinding, ClusterRoleBinding, PersistentVolumeClaim, StatefulSet, ServiceAccount, ClusterRole, PriorityClass, ConfigMap`

### Flow-3.2: Runtime-Object to JSON
Firstly, the SDK performs a typecast of the runtime object to its actual data type. For instance, if the Kubernetes Kind is "Service," the SDK typecasts the runtime object to the specific data type corev1.Service. Then, it conducts a Depth-First Search (DFS) traversal over the corev1.Service object using reflection. During this traversal, the SDK generates a JSON structure that encapsulates information about the struct hierarchy, including corresponding data types and values. This transformation results in a JSON representation of the corev1.Service object's structure and content.
<details>
<summary>DFS Algorithm Cases</summary>

The DFS function iterates over the runtime object, traversing its structure in a Depth-First Search manner. During this traversal, it constructs the JSON structure while inspecting each attribute for its data type and value. Attributes that have default values in the runtime object but are not explicitly set in the YAML file are omitted from the conversion process. This ensures that only explicitly defined attributes with their corresponding values are included in the resulting JSON structure. The function follows this flow to accurately capture the structure, data types, and values of the Kubernetes resource while excluding default attributes that are not explicitly configured in the YAML file.

```	
A) Base-Cases:
1. Float32, Float64, Int8, Int16, Int32, Int64
 	Typecast the Float, Int value to String and returns. (0 is considered as default value)
2. Bool
	Returns the bool value as it is. 
3. String
	Replaces (“ with \”) and (\ with \\) and returns. (“” is considered as default value) 

B) Composite-Cases:
1. Slice/ Array:
	Iterates over each element of slice and calls the DFS fxn again with the element.
	Returns the list of all backtrack-values. ([] is considered as default value)
2. Map
	Iterates over each key-value pairs, calls the DFS(value).
	Returns the map containing key-backtrack_values. (Empty Map is considered as default value).
3. Struct
	Iterates over each attribute-value and calls the DFS(value).
	Returns map[Attribute-Name] = {“type” : “Data-type of Attribute”, “val”: “Backtracked-Value of 	Attribute”}.

C) Special-Cases:
We have assumed in the DFS function, that every path (structure) will end at the basic-data-types (string, int, bool etc), But there lies some cases when we can’t traverse further because the attributes of struct are private. Such cases are handled specially. (Converted to String and then return appropriately)
1. V1.Time and resource.Quantity
2. []byte/[]uint8: []byte is generally used in kind: Secret. It is seen that we provide 64base encoded secret-value in yaml, but on converting the yaml to runtime-obj, the secret-val automatically get decoded to actual value, Since, It is not good to show decoded/actual secret value in the code, therefore, we decode it again and store this decoded-value as secret-value in json.



		
```

</details>

<details>
<summary>JSON Conversion Example</summary>

```
// For A KRM Resource
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80

// The Converted JSON Representation looks-like the following: 
{
    "ObjectMeta": {
        "type": "v1.ObjectMeta",
        "val": {
            "Name": {
                "type": "string",
                "val": "my-service"
            }
        }
    },
    "Spec": {
        "type": "v1.ServiceSpec",
        "val": {
            "Ports": {
                "type": "[]v1.ServicePort",
                "val": [
                    {
                        "Port": {
                            "type": "int32",
                            "val": "80"
                        },
                        "Protocol": {
                            "type": "v1.Protocol",
                            "val": "TCP"
                        }
                    }
                ]
            }
        }
    }
}
// It shows the hierarchical structure along with the specific data types and corresponding values for each attribute
```
</details>

### Flow-3.3: JSON to String (Go-Code)
The SDK reads the JSON file containing the information about the Kubernetes resource and then translates this information into a string of Go code. This process involves parsing the JSON structure and generating corresponding Go code strings based on the structure, data types, and values extracted from the JSON representation. Ultimately, this results in a string that represents the Kubernetes resource in a format compatible with Go code.

<details>
<summary>TraverseJSON Cases (Json-to-String)</summary>


```
The traverse JSON function is responsible for converting JSON data into Go code. Here's how it handles base cases:
The JSON structure contains type as well as value information. Based on the type the following case are formulated.
A) Base Cases:
1. Bool: Returns the boolean value as a string.
2. String: if SingleLine, then return the string with enclosed quotes i.e. \”mystring\”,
	   If MultiLine, then it is handled using Concatenated Line strings.
	Line-1       ---> "Line-1\n" + 
	Line-2        	  "Line-2\n"	


B) Composite Cases:
1. Slice/Array: Iterate over each element, run the TraverseJson(element), capture the backtrackVal & format it accordingly using FormatTypeVal(backtrack-Val):
		“Formatted-backtrackVal1”,
		“Formatted-backtrackVal2”,

2. Map: Iterate over each key-value pair, run the TraverseJson(value), capture the backtrackVal & format it accordingly using FormatTypeVal(backtrack-Val):
	map[string]string{}{
		“key1”: “Formatted-backtrackVal1”,
		“key2”: “Formatted-backtrackVal2”,	
	}

3. : Any Data-type Other Than Map (Signifies it is a Struct with attributes)
	Iterate over each attribute value, run the TraverseJson(attribute-value), capture the backtrackVal & format it accordingly using FormatTypeVal(backtrack-Val)
	Attribute-Name1: “Formatted-backtrackVal1”,
	Attribute-Name2: “Formatted-backtrackVal2”,

```

</details>



<details>
<summary>GoCode Conversion Example</summary>

```
// For a JSON structure Like the following: 
{
    "ObjectMeta": {
        "type": "v1.ObjectMeta",
        "val": {
            "Name": {
                "type": "string",
                "val": "my-service"
            }
        }
    },
    "Spec": {
        "type": "v1.ServiceSpec",
        "val": {
            "Ports": {
                "type": "[]v1.ServicePort",
                "val": [
                    {
                        "Port": {
                            "type": "int32",
                            "val": "80"
                        },
                        "Protocol": {
                            "type": "v1.Protocol",
                            "val": "TCP"
                        }
                    }
                ]
            }
        }
    }
}

// The Go-code will look like as
 &corev1.Service{
	ObjectMeta: metav1.ObjectMeta{
		Name: "my-service",
	},
	
	Spec: corev1.ServiceSpec{
		Ports: []corev1.ServicePort{
	
			corev1.ServicePort{
				Port:     80,
				Protocol: corev1.Protocol("TCP"),
				TargetPort: intstr.IntOrString{
					IntVal: 9376,
				},
			},
		},
	},
}
```
</details>

### Significance of Config-Jsons: (Struct_Module_mapping.json & Enum_module_mapping.json)
Based on the data type, Values are formatted accordingly,
| Data-Type | Value    | Formatted-Value    |
| :---:   | :---: | :---: |
| int32 | 5   | 5   |
| string | 5   | \"5\" |
| *int32 | 5   | int32Ptr(5) |

The Config-Jsons are required for more package-specific-types (such as : v1.Service, v1.Deployment)

#### i) Struct_Module_mapping.json
Mostly, It is seen that inspecting the type of struct(using reflect) would tell us that the struct belong to package “v1”, but there are multiple v1 packages (appsv1, metav1, rbacv1, etc), So, the actual package remains unknown. 

Solution: To solve the above problems, we build a “structModuleMapping” which is a map that takes “struct-name” as the key and gives “package/module name” as a value.
```
v1.Deployment -->  appsv1.Deployment
v1.Service --> corev1.Service
```

#### ii) Enum_Module_mapping.json
Structs need to be initialized using curly brackets {}, whereas enums need Parenthesis (), Since, reflect doesn’t tell us which data-type is struct or enum, We:

Solution: We solve the above problems by building an “enumModuleMapping” which is a set that stores all data types that are enums. i.e. If a data type belongs to the set, then It is an Enum.

There is an automation-script that takes the types.go files of packages and build the config-json. For details, Please refer [here](https://github.com/nephio-project/nephio-sdk/tree/main/helm-to-operator-codegen-sdk/config)


### Flow-4: KRM to Unstruct-Obj to String(Go-code)
All Kubernetes resource kinds that are not supported by the runtime-object method are handled using the unstructured method. In this approach, the Kubernetes Resource Manifest (KRM) is converted to an unstructured object using the package "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured". 
Then, We traverse the unstructured-Obj in a DFS fashion and build the gocode-string.
<details>
<summary>DFS Algorithm Cases (Unstruct-Version)</summary>


```	
A) Base Cases:
1. Bool: Convert the Bool value to string and return.
2. Int & Float: Convert the value to string and return.
3. String: if SingleLine, then return the string with enclosed quotes i.e. \”mystring\”,
	If MultiLine, then it handled using Concatenated Line strings, (as done previously in Flow 3.3).

B) Composite Cases:
1. Slice/Array: Iterate over each element, runs the DFS(element), captures the backtrackVal & return as:
	[]interface{}{
		“backtrackVal1”,
		“backtrackVal2”,
	}
2. Map: Iterate over each key-value pair, runs the DFS(value), capture the backtrackVal & returns as:
	map[string]interface{}{
		“key1”: “backtrackVal1”,
		“key2”: “backtrackVal2”,	
	}

```

</details>

### Flow-5: Go-Codes to Gofile
The process of generating the final Go file consists of the following steps:

1. Collecting Go Code: Go code for each Kubernetes Resource Manifest (KRM) is collected and stored in a map where the key represents the kind of resource (e.g., "Service", "Deployment"), and the value is a slice containing the corresponding Go code strings.

2. Aggregation and Writing to Runnable Go Function: Iterate over the map and for each kind of resource, assign the collected Go code to a variable and write it into its corresponding runnable Go function. All services are written in one function, and the same applies to other kinds of resources.

3. Adding Helper Functions and Import Statements: Include import statements at the beginning of the file. Additionally, add helper functions such as "deleteMeAfterDeletingUnusedImportedModules" to handle unused import errors, as well as functions like "int32Ptr", "int64Ptr", etc., which return pointers to values of specific types. Also, include functions like "getDataForSecret()" for decoding base64-encoded bytes and master functions like "CreateAll()" and "DeleteAll()" for creating and deleting all KRM resources, respectively.

4. Pluggable Functions: Generate pluggable functions such as "Getxxx()", where "xxx" represents the specific kind of resource. These functions are designed to create KRM resources belonging to a particular kind and are intended to be called by a Reconciler. Examples include "GetDeployment()", "GetService()", etc.

By following these steps, the final Go file is created, containing all necessary import statements, helper functions, runnable functions for each kind of resource, and pluggable functions for creating specific types of resources. This comprehensive file is ready for use by a Reconciler in managing Kubernetes resources within an operator.