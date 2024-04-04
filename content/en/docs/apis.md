---
title: "Models and APIs"
type: docs
weight: 5
description: Reference for the Nephio models and APIs
---

# Nephio APIs

## Overview

Nephio APIs primarily consist of a collection of Go API objects, CRDs and other KRM types,
specified and maintained in the [Nephio API repository](https://github.com/nephio-project/api).
this page aims to give a high level overview of the objects that are available and their
relationships, and is based on an
[original document](https://docs.google.com/document/d/1-5nlpY4FbuhWtdKTvIqPOv4bWmA6zx6TdHoEfk9Jc_Q/edit)
developed ny Tal Liron.

The aim is to keep these diagrams as simple as possible for now and only convey the important aspects of the modelled entities. As such they are intended to give a high-level overview of the entities and relationships that can be accessed and modified via the Nephio API, and provide reference to detailed documentation, generated from the code, where available.


## Topology

```mermaid
flowchart TD
    NFTopology-- 1 .. n -->NFInstance
    NFInstance -. refers .-> NFTemplate
    NFTemplate -- 1..n --> NFInterface
    NFInterface -. "refers (by name)" .-> NetworkInstance
    NFTemplate-.refers..->Capacity
    NFTemplate -. "refers (by name)" .-> NFClass
    NFClass -. refers .-> PackageRevisionReference
   click NFTopology "https://doc.crds.dev/github.com/nephio-project/api/topology.nephio.org/NFTopology/v1alpha1@v2.0.0" "NFTopology"
   click NFInstance "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_topology_types.go#L59" "NFInstance"
   click NFTemplate "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_topology_types.go#L45" "NFTemplate"
   click NFInterface "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_topology_types.go#L35" "NFInterface"
   click Capacity "https://doc.crds.dev/github.com/nephio-project/api/req.nephio.org/Capacity/v1alpha1@v2.0.0" "Capacity"
   click NFClass "https://doc.crds.dev/github.com/nephio-project/api/req.nephio.org/NFClass/v1alpha1@v2.0.0" "NFClass"
   click PackageRevisionReference "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_class_types.go#L25" "PackageRevisionReference"
   
```

## Network Requirements

![Network Requirements](diagrams/requirements.svg)

## Capacity Requirements

![Capacity Requirements](diagrams/capacity-requirements.svg)

# Issues
Please raise any issues in the [nephio](https://github.com/nephio-project/nephio) repository
instead of in here, using the prefix "api: " in the issue title.


{{< iframe src="https://doc.crds.dev/github.com/nephio-project/api@v2.0.0" sub="https://doc.crds.dev/github.com/nephio-project/api@v2.0.0">}}