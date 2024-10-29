---
title: "Using authenticated private registries"
type: docs
weight: 4
description: ""
---

To enable the Porch function runner to pull kpt function images from authenticated private registries, the system requires:

1. Creating a kubernetes secret using a JSON file according to the Docker config schema, containing valid credentials for each authenticated registry.
2. Mounting this new secret as a volume on the function runner.
3. Providing the path of the mounted secret to the function runner using the argument `--registry-auth-secret-path`

An example template of what a docker *config.json* file looks like is as follows below. The base64 encoded value *bXlfdXNlcm5hbWU6bXlfcGFzc3dvcmQ=* of the *auth* key decodes to *my_username:my_password*, which is the format used by the config when authenticating.

```json
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "bXlfdXNlcm5hbWU6bXlfcGFzc3dvcmQ="
        },
        "ghcr.io": {
            "auth": "bXlfdXNlcm5hbWU6bXlfcGFzc3dvcmQ="
        }
    }
}
```

A quick way to generate this secret for your use using your docker *config.json* would be to run the following command:

```bash
kubectl create secret generic <SECRET_NAME> --from-file=.dockerconfigjson=/path/to/your/config.json --type=kubernetes.io/dockerconfigjson --dry-run=client -o yaml -n porch-system
```

{{% alert title="Note" color="primary" %}}
The secret must be in the same namespace as the function runner deployment. By default, this is the *porch-system* namespace.
{{% /alert %}}

This should generate a secret template, similar to the one below, which you can add to the *2-function-runner.yaml* file in the Porch catalog package found [here](https://github.com/nephio-project/catalog/tree/main/nephio/core/porch)

```yaml
apiVersion: v1
data:
  .dockerconfigjson: <base64-encoded-data>
kind: Secret
metadata:
  creationTimestamp: null
  name: <SECRET_NAME>
  namespace: porch-system
type: kubernetes.io/dockerconfigjson
```

Next you must mount the secret as a volume on the function runner deployment. Add the following snippet to the Deployment object in the *2-function-runner.yaml* file:

```yaml
    volumeMounts:
      - mountPath: /pod-cache-config
        name: pod-cache-config-volume
      - mountPath: /var/tmp/auth-secret
        name: docker-config
        readOnly: true
volumes:
  - name: pod-cache-config-volume
    configMap:
      name: pod-cache-config
  - name: docker-config
    secret:
      secretName: <SECRET_NAME>
```

You may specify your desired `mountPath:` so long as the function runner can access it.

{{% alert title="Note" color="primary" %}}
The chosen `mountPath:` should use its own, dedicated sub-directory, so that it does not overwrite access permissions of the existing directory. For example, if you wish to mount on `/var/tmp` you should use `mountPath: /var/tmp/<SUB_DIRECTORY>` etc.
{{% /alert %}}

Lastly you must add the `--registry-auth-secret-path` to the arguments of the function-runner Deployment object in the *2-function-runner.yaml* file, giving the path of the secret file mount:

```yaml
command:
  - /server
  - --config=/config.yaml
  - --registry-auth-secret-path=/var/tmp/auth-secret/.dockerconfigjson
  - --functions=/functions
  - --pod-namespace=porch-fn-system
```

With this last step, if your Porch package uses a custom kpt function image stored in an authenticated private registry (for example `- image: ghcr.io/private-registry/set-namespace:customv2`), the function runner will now use the secret info as an `imagePullSecret` for the function pods as documented [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).
