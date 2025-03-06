---
title: "Git Authentication Configuration"
type: docs
weight: 4
description: ""
---

## 1. Porch Server to Git Interaction

The porch server handles interaction with associated git repositories through the use of porch repository CR (Custom Resource) which act as a link between the porch server and the git repositories the server is meant to interact with and store packages on.

More information on porch repositories can be found [here](../package-orchestration.md#repositories).

### 1.1 Porch Repository Basic Authentication Configuration

A porch repository object can be created through the use of the `porchctl repo reg porch-test-repository -n porch-test http://example-ip:example-port/repo.git --repo-basic-password=password --repo-basic-username=username` command which creates a secret and repository object.

The basic authentication secret must meet the following criteria:

- Exist in the same namespace as the Repository CR (Custom Resource) that requires it.
- Have a Data keys named *username* and *password* containing the relevant information.
- Be of type *basic-auth*.

Which would be the equivalent of doing a `kubectl apply -f` on a yaml file with the following content (assuming the porch-test namespace exists on the cluster):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-auth-secret
  namespace: porch-test
data:
  password: base-64-encoded-password
  username: base-64-encoded-username
type: kubernetes.io/basic-auth

---
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository

metadata:
  name: porch-test-repository
  namespace: porch-test

spec:
  description: porch test repository
  content: Package
  deployment: false
  type: git
  git:
    repo: http://example-ip:example-port/repo.git
    directory: /
    branch: main
    secretRef:
      name: git-auth-secret
```

### 1.2 Porch Repository Token Authentication Configuration

The authentication to the git repository can be configured to be in token format by altering the secret used in the porch repository object.

The token authentication secret must meet the following criteria:

- Exist in the same namespace as the Repository CR (Custom Resource) that requires it
- Have a Data key named *token* containing the relevant git token information.
- Be of type *Opaque*.

For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-auth-secret
  namespace: porch-test
data:
  token: base-64-encoded-token
type: Opaque
```

{{% alert title="Note" color="primary" %}}
The Porch server caches the authentication credentials from the secret, so if the secret's contents are updated they may in fact not be the credentials which are used in the authentication until the old secret credentials are no longer valid which triggers the porch server to query the secret again use the new credentials which if valid become the new cached authentication values.
{{% /alert %}}

### 1.3 Porch Repository HTTPS Configuration

To enable the porch server to communicate with a custom git deployment over HTTPS, we must:

1. Provide an additional arguments flag `use-git-cabundle=true` to the porch-server deployment.
2. Provide an additional Kubernetes secret containing the relevant certificate chain in the form of a cabundle.

The secret itself must meet the following criteria:

- Exist in the same namespace as the Repository CR that requires it.
- Be named specifically \<namespace\>-ca-bundle.
- Have a Data key named *ca.crt* containing the relevant ca certificate (chain).

For example, a Git Repository is hosted over HTTPS at the *https://my-gitlab.com/joe.bloggs/blueprints.git* URL:

`https://my-gitlab.com/joe.bloggs/blueprints.git`

Before creating the new Repository in the **GitLab** namespace, we must create a secret that fulfils the criteria above.

`kubectl create secret generic gitlab-ca-bundle --namespace=gitlab --from-file=ca.crt`

Which would produce the following:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-ca-bundle
  namespace: gitlab
type: Opaque
data:
  ca.crt: FAKE1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUNuakNDQWdHZ0F3SUJBZ0lRTEdmUytUK3YyRDZDczh1MVBlUUlKREFLQmdncWhrak9QUVFEQkRBZE1Sc3cKR1FZRFZRUURFeEpqWlhKMExXMWhibUZuWlhJdWJHOWpZV3d3SGhjTk1qUXdOVE14TVRFeU5qTXlXaGNOTWpRdwpPREk1TVRFeU5qTXlXakFWTVJNd0VRWURWUVFGRXdveE1qTTBOVFkzT0Rrd01JSUJJakFOQmdrcWhraUc5dzBCCkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXhCUUtWMEVzQ1JOOGxuV3lQR1ZWNXJwam5QZkI2emszK0N4cEp2NVMKUWhpMG1KbDI0elV1WWZjRzNxdFUva1NuREdjK3NQRUY0RmlOcUlsSTByWHBQSXBPazhKbjEvZU1VT3RkZUUyNgpSWEZBWktjeDVvdUJyZVNja3hsN2RPVkJnOE1EM1h5RU1PQU5nM0hJZ1J4ZWx2U2p1dy8vMURhSlRnK0lBS0dUCkgrOVlRVFcrZDIwSk5wQlR3NkdnQlRsYmdqL2FMRWEwOXVYSVBjK0JUSkpXRThIeDhkVjFNbEtHRFlDU29qZFgKbG9TN1FIa0dsSVk3M0NPZVVGWEVnTlFVVmZaZHdreXNsT3F4WmdXUTNZTFZHcEFyRitjOVdyUGpQQU5NQWtORQpPdHRvaG8zTlRxQ3FST3JEa0RMYWdsU1BKSUd1K25TcU5veVVxSUlWWkV5R1dRSURBUUFCbzJBd1hqQU9CZ05WCkhROEJBZjhFQkFNQ0JhQXdEQVlEVlIwVEFRSC9CQUl3QURBZkJnTlZIU01FR0RBV2dCUitFZTVDTnVJSkcwZjkKV3J3VzdqYUZFeVdzb1RBZEJnTlZIUkVFRmpBVWdoSm5hWFJzWVdJdVpYaGhiWEJzWlM1amIyMHdDZ1lJS29aSQp6ajBFQXdRRGdZb0FNSUdHQWtGLzRyNUM4bnkwdGVIMVJlRzdDdXJHYk02SzMzdTFDZ29GTkthajIva2ovYzlhCnZwODY0eFJKM2ZVSXZGMEtzL1dNUHNad2w2bjMxUWtXT2VpM01aYWtBUUpCREw0Kyt4UUxkMS9uVWdqOW1zN2MKUUx3NXVEMGxqU0xrUS9mOTJGYy91WHc4QWVDck5XcVRqcDEycDJ6MkUzOXRyWWc1a2UvY2VTaWFPUm16eUJuTwpTUTg9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0=
```
