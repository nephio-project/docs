---
title: Nephio controller user's guide
description: >

weight: 2
---

## Reconcilers

[Reconcilers](https://kubebyexample.com/learning-paths/operator-framework/operator-sdk-go/controller-reconcile-function)
are used by Kubernetes to enforce the desired state of a Custom Resource (CR).

## The Nephio Reconciler Interface

The Nephio Controller deploys multiple reconcilers and also supports pluggable reconcilers. To plug into the Nephio
controller, a reconciler must implement the
[Nephio reconciler interface](https://github.com/nephio-project/nephio/tree/main/controllers/pkg/reconcilers/reconciler-interface).
Reconcilers register with the Nephio reconciler interface when they start up.

The reconcilers below are currently deployed by default in the Nephio controller:

```bash
./controllers/pkg/reconcilers/token/reconciler.go
./controllers/pkg/reconcilers/repository/reconciler.go
./controllers/pkg/reconcilers/bootstrap-packages/reconciler.go
./controllers/pkg/reconcilers/vlan-specializer/reconciler.go
./controllers/pkg/reconcilers/generic-specializer/reconciler.go
./controllers/pkg/reconcilers/network/reconciler.go
./controllers/pkg/reconcilers/ipam-specializer/reconciler.go
./controllers/pkg/reconcilers/bootstrap-secret/reconciler.go
./controllers/pkg/reconcilers/approval/reconciler.go
```

## Enabling Reconcilers

To enable a particular reconciler, you pass an environment variable to the
Nephio Controller at startup. The environment variable is of the form
*ENABLE_\<RECONCILER\>* where *\<RECONCILER\>* is the name of the reconciler to
be enabled in upper case. Therefore, to enable the bootstrap-packages reconciler,
pass the ENABLE_BOOTSTRAPPACKAGES to the Nephio controller. Reconcilers are
disabled by default.


You can see what reconcilers are enabled on the Nephio Controller using
`kubectl`.

```bash
$ kubectl describe pod -n nephio-system nephio-controller-6565fd695d-44kld

*** Truncated output ***

Name:             nephio-controller-6565fd695d-44kld
Containers:
  controller:
    Container ID:  containerd://37d3eff53c1944a659e5a7ab913173db42f34b44347072e6c9b51e5671f35ea2
    Environment:
      POD_NAMESPACE:              nephio-system (v1:metadata.namespace)
      POD_IP:                      (v1:status.podIP)
      POD_NAME:                   nephio-controller-6565fd695d-44kld (v1:metadata.name)
      NODE_NAME:                   (v1:spec.nodeName)
      NODE_IP:                     (v1:status.hostIP)
      GIT_URL:                    http://172.18.0.200:3000
      GIT_NAMESPACE:              gitea
      ENABLE_APPROVAL:            true
      ENABLE_REPOSITORIES:        true
      ENABLE_BOOTSTRAPSECRETS:    true
      ENABLE_BOOTSTRAPPACKAGES:   true
      ENABLE_GENERICSPECIALIZER:  true
      ENABLE_NETWORKS:            true
      CLIENT_PROXY_ADDRESS:       resource-backend-controller-grpc-svc.backend-system.svc.cluster.local:9999
```

To check that the reconcilers are actually deployed, you can examine the logs
from the Nephio Controller. The log rolls over so you may need to redeploy the
Nephio Controller to see what reconcilers are being deployed.

```bash
$ kubectl rollout restart deployment nephio-controller -n nephio-system

$ kubectl logs -n nephio-system nephio-controller-59487989bf-md845 --all-containers | \
grep enable
2023-06-27T11:37:58.646Z	INFO	setup	enabled reconcilers	{"reconcilers": "repositories,approval,bootstrappackages,bootstrapsecrets,genericspecializer,networks"}
```
