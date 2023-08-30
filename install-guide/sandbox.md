# Sandbox Nephio Installation

*Work-in-Progress*

In this guide, you will set up Nephio running in a single VM with:
- **Management Cluster**: [kind](https://kind.sigs.k8s.io/)
- **Cluster Provisioner**: [Cluster API](https://cluster-api.sigs.k8s.io/)
- **Workload Clusters**: kind
- **Gitops Tool**: ConfigSync
- **Git Provider**: Gitea running in the Nephio management cluster will be the
  git provider for cluster deployment repositories. Some external repositories
  will be on GitHub.
- **Web UI Auth**: None
- **Ingress/Load Balancer**: [MetalLB](https://metallb.universe.tf/), but only internally to the VM.

## Provisioning Your Sandbox VM

## Provisioning Your Management Cluster

