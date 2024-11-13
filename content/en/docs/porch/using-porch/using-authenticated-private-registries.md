---
title: "Using authenticated private registries"
type: docs
weight: 4
description: ""
---

To enable the Porch function runner to pull kpt function images from authenticated private registries, the system requires:

{{% alert title="Note" color="primary" %}}
Please note sections 4, 5 and 6 are only required if your private registries are set up to use TLS with either self signed or custom/local CA's
{{% /alert %}}

1. Creating a kubernetes secret using a JSON file according to the Docker config schema, containing valid credentials for each authenticated registry.
2. Mounting this new secret as a volume on the function runner.
3. Enabling the external registries feature, providing the path and name of the mounted secret to the function runner using the arguments `--enable-private-registry`, `--registry-auth-secret-path` and `--registry-auth-secret-name` respectively.
4. Creating a kubernetes secret using the TLS information for the registry you wish to use.
5. Mounting the secret containing the registries TLS information to the function runner similarly to step 2.
6. Enabling the TLS feature and providing the path of the mounted secret to the funtion runner using the arguments `--enable-private-registry` and `--tls-secret-path` respectively.

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

A typical secret containing TLS information will take on the a similar format to the following:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <TLS_SECRET_NAME>
  namespace: porch-system
data:
  ca.crt: base64_encoded_ca.crt_data
  tls.crt: base64_encoded_tls.crt_data
  tls.key: base64_encoded_tls.key_data
type: kubernetes.io/tls
```

{{% alert title="Note" color="primary" %}}
The function runner will look specifically for a *ca.crt* or *ca.pem* file for TLS transport and so the ca TLS data must use that key/file naming.
{{% /alert %}}

Next you must mount the secrets as a volume on the function runner deployment. Add the following snippet to the Deployment object in the *2-function-runner.yaml* file noting that the last entries in the `volumes` and `volumeMounts` fields are only required for TLS private registries:

```yaml
    volumeMounts:
      - mountPath: /pod-cache-config
        name: pod-cache-config-volume
      - mountPath: /var/tmp/auth-secret
        name: docker-config
        readOnly: true
      - mountPath: /var/tmp/tls-secret/
        name: tls-registry-config
        readOnly: true
volumes:
  - name: pod-cache-config-volume
    configMap:
      name: pod-cache-config
  - name: docker-config
    secret:
      secretName: <SECRET_NAME>
  - name: tls-registry-config
    secret:
      secretName: <TLS_SECRET_NAME>
```

You may specify your desired `mountPath:` so long as the function runner can access it.

{{% alert title="Note" color="primary" %}}
The chosen `mountPath:` should use its own, dedicated sub-directory, so that it does not overwrite access permissions of the existing directory. For example, if you wish to mount on `/var/tmp` you should use `mountPath: /var/tmp/<SUB_DIRECTORY>` etc.
{{% /alert %}}

The `--enable-tls-registry` and `--tls-secret-path` variables are only required if the user has a private TLS registry. they are used to indicate to the function runner that it should try and autenticate to the registry with TLS transport and should use the TLS certificate information found on the path provided in `--tls-secret-path`.

Lastly you must enable private registry functionality along with providing the path and name of the secret. Add the `--enable-private-registry`, `--registry-auth-secret-path` and `--registry-auth-secret-name` arguments to the function-runner Deployment object in the *2-function-runner.yaml* file:

```yaml
command:
  - /server
  - --config=/config.yaml
  - --enable-private-registry=true
  - --registry-auth-secret-path=/var/tmp/auth-secret/.dockerconfigjson
  - --registry-auth-secret-name=<SECRET_NAME>
  - --enable-tls-registry=true
  - --tls-secret-path=/var/tmp/tls-secret/
  - --functions=/functions
  - --pod-namespace=porch-fn-system
```

The `--enable-private-registry`, `--registry-auth-secret-path` and `--registry-auth-secret-name` arguments have default values of *false*, */var/tmp/auth-secret/.dockerconfigjson* and *auth-secret* respectively; however, these should be overridden to enable the functionality and match user specifications.

The `--enable-tls-registry` and `--tls-secret-path` arguments have default values of *false* and */var/tmp/tls-secret/* respectively; however these should be configured by the user and are only necessary when the user has a private TLS registry.

{{% alert title="Note" color="primary" %}}
It is vital that the user has configured the node which the function runner is operating on with the certificate information which is used in the `<TLS_SECRET_NAME>`. If this is not configured correctly even if the certificate is correct the function runner will be able to pull the image but the krm function pod spun up to run the function will error out with a *x509 certificate signed by unknown authority*.
{{% /alert %}}

With this last step, if your Porch package uses a custom kpt function image stored in an authenticated private registry (for example `- image: ghcr.io/private-registry/set-namespace:customv2`), the function runner will now use the secret info to replicate your secret on the `porch-fn-system` namespace and specify it as an `imagePullSecret` for the function pods, as documented [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).
