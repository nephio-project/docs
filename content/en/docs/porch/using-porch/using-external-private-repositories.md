---
title: "Using an external private repository"
type: docs
weight: 4
description: ""
---

To enable the porch function runner to communicate with authenticated private repositories, we must:

1. Create a kubernetes secret using the docker `config.json` file.
2. Mount this new secret as a volume on the fn runner.
3. Provide this secret mount path to the fn runner using the argument `--custom-repo-secret-path`

 A quick way to generate this secret for your use using your docker `config.json` would be to run the following.

```bash
kubectl create secret generic secret-name --from-file=.dockerconfigjson=/path/to/your/config.json --type=kubernetes.io/dockerconfigjson --dry-run=client -o yaml
```

This should generate a secret template similar to the one below which you can insert in your function runner deployment file `2-function-runner.yaml`.

```yaml
apiVersion: v1
data:
  .dockerconfigjson: <base64-encoded-data>
kind: Secret
metadata:
  creationTimestamp: null
  name: secret-name
type: kubernetes.io/dockerconfigjson
```

Next we must mount the secret as a volume on the `2-function-runner.yaml` deployment as follows.

```yaml
    volumeMounts:
      - mountPath: /pod-cache-config
        name: pod-cache-config-volume
      - mountPath: /var/tmp
        name: docker-config
        readOnly: true
volumes:
  - name: pod-cache-config-volume
    configMap:
      name: pod-cache-config
  - name: docker-config
    secret:
      secretName: secret-name
```

You may mount this on whatever path you wish so long as its usable for the fn-runner and you specify it correctly in the environment variable.

Lastly you must add the `--custom-repo-secret-path` to the fn runner agruments giving the path of the secret file mount.

```yaml
command:
  - /server
  - --config=/config.yaml
  - --custom-repo-secret-path=/var/tmp/.dockerconfigjson
  - --functions=/functions
  - --pod-namespace=porch-fn-system
```

With this last step the function runner should be set up such that if you use a custom kpt fn image in your porch packages stored on a private repo e.g. `- image: ghcr.io/private-repo/set-namespace:customv2`. The function runner will now use the secret info as an `imagePullSecret` for the fn pods to allow them to pull from these registries.
