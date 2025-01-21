---
title: Minimal Environment install for development
description: >
  How to set up a minimal environment for development in the Nephio project.
weight: 2
---

The following environment install works on a MacBook Pro M1 or via SSH to Ubuntu 22.04 to get the nephio-operators
running in VS Code talking to a local kind cluster running in Docker. Note that depending on what part of Nephio you are
working on, you may wish to install less or more of the components below. It should work on other environments with
appropriate tweaking.

## Install Docker, kind, and kpt packages

This script automates steps 3 to 9 below.

```bash
#!/usr/bin/env bash

cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.29.0
    extraPortMappings:
      - containerPort: 3000
        hostPort: 3000
EOF
pushd "$(mktemp -d -t "kpt-pkg-XXX")" >/dev/null || exit
for pkg in "distros/sandbox/gitea" "/nephio/core/porch" "nephio/core/configsync" "nephio/optional/resource-backend"; do
    kpt pkg get "https://github.com/nephio-project/catalog.git/${pkg}@main" "${pkg##*/}"
    kpt live init "${pkg##*/}"
    kpt live apply "${pkg##*/}"
done

kpt pkg get https://github.com/nephio-project/catalog/tree/main/nephio/core/nephio-operator nephio-operator
find nephio-operator/crd/bases/*.yaml -exec kubectl apply -f {} \;

popd >/dev/null || exit
```

### Installation steps:

1. [Install Docker](https://docs.docker.com/engine/install/) using the appropriate method for your system.
2. [Install kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) using the appropriate method for your
   system (homebrew on mac)
3. [Install kpt CLI](https://kpt.dev/installation/kpt-cli) using the appropriate method for your system (e.g. homebrew
   on mac)
4. Create the management cluster

```bash
kind create cluster -n mgmt
```

5. Install Gitea

```bash
kpt pkg get https://github.com/nephio-project/catalog.git/distros/sandbox/gitea@main gitea
kpt live init gitea
kpt live apply gitea
```

6. Make Gitea visible on host machine

```bash
kubectl port-forward -n gitea svc/gitea 3000:3000
```

7. Install Porch

```bash
kpt pkg get https://github.com/nephio-project/catalog.git/nephio/core/porch@main porch
kpt live init porch
kpt live apply porch
```

8. Install configsync

```bash
kpt pkg get https://github.com/nephio-project/catalog.git/nephio/core/configsync@main configsync
kpt live init configsync
kpt live apply configsync
```

9. Install the resource backend

```bash
kpt pkg get https://github.com/nephio-project/catalog.git/nephio/optional/resource-backend@main resource-backend
kpt live init resource-backend
kpt live apply resource-backend
```

10. Load the Nephio CRDs

```bash
kpt pkg get https://github.com/nephio-project/catalog/tree/main/nephio/core/nephio-operator nephio-operator
ls nephio-operator/crd/bases/*.yaml | xargs -n1 kubectl apply -f
```

## Connect to Gitea on your browser

Connecting to Gitea allows you to see the actions that Nephio takes on Gitea.

1. Port forward the Gitea port to your localhost

```bash
kubectl port-forward -n gitea svc/gitea 3000:3000
```

2. Browse to the Gitea web client at http://localhost:3000 and log on.

## VS Code Configuration

Set up a launch configuration in VS Code *launch.json* similar to the configuration below:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch Nephio Controller Manager",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}/operators/nephio-controller-manager",
            "args": [
                "--reconcilers",
                "*"
            ],
            "env": {
				"GIT_URL": "http://localhost:3000",
                "GIT_NAMESPACE": "gitea"
			},
        }
    ]
}
```

You can now launch the Nephio operators in VS Code using the launch configuration above. You can specify a list of the reconcilers you wish to run in the `--reconcilers` argument value or `*` to run all the reconcilers.

