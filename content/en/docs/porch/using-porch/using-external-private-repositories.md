---
title: "Using external private registries"
type: docs
weight: 4
description: ""
---

To enable the Porch function runner to communicate with authenticated private registries, we must:

1. Create a kubernetes secret using the a docker `config.json` file.
2. Mount this new secret as a volume on the function runner.
3. Provide this secret mount path to the function runner using the argument `--registry-auth-secret-path`

 A quick way to generate this secret for your use using your docker `config.json` would be to run the following.

```bash
kubectl create secret generic <SECRET_NAME> --from-file=.dockerconfigjson=/path/to/your/config.json --type=kubernetes.io/dockerconfigjson --dry-run=client -o yaml
```

This should generate a secret template similar to the one below which you can add to the `2-function-runner.yaml` file present on the Porch deployment found [here](https://github.com/nephio-project/catalog/tree/main/nephio/core/porch)

```yaml
apiVersion: v1
data:
  .dockerconfigjson: <base64-encoded-data>
kind: Secret
metadata:
  creationTimestamp: null
  name: <SECRET_NAME>
type: kubernetes.io/dockerconfigjson
```

Next you must mount the secret as a volume on the function runner in the `2-function-runner.yaml` as follows

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
      secretName: <secret-name>
```

You may specify your desired `mountPath:` so long as the function runner can access it.

Lastly you must add the `--registry-auth-secret-path` to the function runner arguments, giving the path of the secret file mount.

```yaml
command:
  - /server
  - --config=/config.yaml
  - --registry-auth-secret-path=/var/tmp/auth-secret/.dockerconfigjson
  - --functions=/functions
  - --pod-namespace=porch-fn-system
```

With this last step, if your Porch package uses a custom kpt function image stored in a private registry (e.g. `- image: ghcr.io/private-registry/set-namespace:customv2`), the function runner will now use the secret info as an `imagePullSecret` for the function pods as documented [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).
