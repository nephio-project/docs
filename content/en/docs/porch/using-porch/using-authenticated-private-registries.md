---
title: "Using authenticated private registries"
type: docs
weight: 4
description: ""
---

To enable the Porch function runner to pull kpt function images from authenticated private registries, the system requires:

{{% alert title="Note" color="primary" %}}
Please note items 4, 5 and 6 are only required if your private registries are set up to use TLS with custom/local certificate authorities (CAs) or self-signed certificates (Not recommended for security).
{{% /alert %}}

1. Creating a Kubernetes secret using a JSON file according to the Docker config schema, containing valid credentials for each authenticated registry.
2. Mounting this new secret as a volume on the function runner.
3. Configuring private registry functionality in the function runner's arguments:
   1. Enabling the functionality using the argument *--enable-private-registry*.
   2. Providing the path and name of the mounted secret using the arguments *--registry-auth-secret-path* and *--registry-auth-secret-name* respectively.
4. Creating a Kubernetes secret using TLS information valid for all registries you wish to use.
5. Mounting the secret containing the registries' TLS information to the function runner similarly to step 2.
6. Enabling TLS functionality and providing the path of the mounted secret to the function runner using the arguments *--enable-tls-registry* and *--tls-secret-path* respectively.

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
  <CA_FILE_NAME>: <PEM_CERT_DATA>
type: kubernetes.io/tls
```

{{% alert title="Note" color="primary" %}}
The content in *<PEM_CERT_DATA>* must be in PEM (Privacy Enhanced Mail) format, and the *<CA_FILE_NAME>* must be *ca.crt* or *ca.pem*. No other values are accepted.
{{% /alert %}}

Next you must mount the secrets as volumes on the function runner deployment. Add the following snippet to the Deployment object in the *2-function-runner.yaml* file (Note that the `tls-registry-config` entries in the `volumes` and `volumeMounts` fields are only required for TLS-secured private registries):

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

You may specify your desired paths for each `mountPath:` so long as the function runner can access them.

{{% alert title="Note" color="primary" %}}
The chosen `mountPath:` should use its own, dedicated sub-directory, so that it does not overwrite access permissions of the existing directory. For example, if you wish to mount on `/var/tmp` you should use `mountPath: /var/tmp/<SUB_DIRECTORY>` etc.
{{% /alert %}}

he *--enable-tls-registry* and *--tls-secret-path* variables are only required if a private registry has TLS enabled. They indicate to the function runner that it should attempt authentication to the registry using TLS, and should use the TLS certificate information found on the path provided in *--tls-secret-path*.

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

The *--enable-tls-registry* and *--tls-secret-path* arguments have default values of *false* and */var/tmp/tls-secret/* respectively; however, these should be configured by the user and are only necessary when using a private registry secured with TLS.

It is important to note that enabling TLS registry functionality makes the function runner attempt connection to the registry provided in the porch file using the mounted tls certificate. if this certificate is invalid for the provided registry it will attempt again but with the Intermediate Certificates stored on the machine for use in TLS with "well-known websites" e.g. Github. If this also fails then it will attempt without TLS as a last resort which if also failed will error out letting the user know the error.

{{% alert title="Note" color="primary" %}}
It is vital that the user has pre-configured the Kubernetes node which the function runner is operating on with the same TLS certificate information as is used in the *<TLS_SECRET_NAME>* secret. If this is not configured correctly, then even if the certificate is correctly configured in the function runner, the kpt function will not run - the function runner will be able to pull the image, but the KRM function pod created to run the function will fail with the error *x509 certificate signed by unknown authority*.
This pre-configuration setup is heavily cluster/implementation-dependent - consult your cluster's specific documentation about adding self-signed certificates or private/internal CA certs to your cluster.
{{% /alert %}}

With this last step, if your Porch package uses a custom kpt function image stored in an authenticated private registry (for example `- image: ghcr.io/private-registry/set-namespace:customv2`), the function runner will now use the secret info to replicate your secret on the `porch-fn-system` namespace and specify it as an `imagePullSecret` for the function pods, as documented [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).
