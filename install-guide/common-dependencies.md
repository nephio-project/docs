# Common Dependencies

This guide describes how to install some required dependencies that are the
same across all environments.

Some of these, like the resource-backend, will move out of the "required"
category in later releases.  Even if you do not use these directly in your
installation, the CRDs that come along with them are necessary.

### Network Config Operator

This component is a controller for applying configuration to routers and
switches. 

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/network-config@v1.0.1
kpt fn render network-config
kpt live init network-config
kpt live apply network-config --reconcile-timeout=15m --output=table
```

### Resource Backend

The resource backend provides IP and VLAN allocation.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/resource-backend@v1.0.1
kpt fn render resource-backend
kpt live init resource-backend
kpt live apply resource-backend --reconcile-timeout=15m --output=table
```
