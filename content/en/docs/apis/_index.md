---
title: "Models and APIs"
type: docs
weight: 5
description: Reference for the Nephio models and APIs
---

## Overview

Nephio APIs consist primarily of a collection of Go API objects, CRDs, and other KRM types,
specified and maintained in the [Nephio API repository](https://github.com/nephio-project/api).
This section aims to give a high-level overview of the objects that are available and their
relationships, and is based on an
[original document](https://docs.google.com/document/d/1-5nlpY4FbuhWtdKTvIqPOv4bWmA6zx6TdHoEfk9Jc_Q/edit)
developed by Tal Liron.

The aim is to keep the diagrams as simple as possible for now and only convey the important aspects
of the modeled entities.
As such, they are intended to give a high-level overview of the entities and relationships that can
be accessed and modified via the Nephio API, and provide reference to detailed documentation,
generated from the code, where available.

## Topology and network APIs

This is a high-level overview of the Nephio models and their relationships, with links to the
relevant API documentation where available, and to the source code where not.

{{< mermaid >}}
flowchart TD
    %% Topology constructs
    NFTopology-- 1 .. n -->NFInstance
    NFInstance -. refers .-> NFTemplate
    NFTemplate -- 1..n --> NFInterface
    NFInterface -. "refers (by name)" .-> NetworkInstance
    NFTemplate-.refers..->Capacity
    NFTemplate -. "refers (by name)" .-> NFClass
    NFClass -. refers .-> PackageRevisionReference
    style NFTopology fill:#00CC00
    click NFTopology "https://doc.crds.dev/github.com/nephio-project/api/topology.nephio.org/NFTopology/v1alpha1@v2.0.0" "NFTopology"
    style NFInstance fill:#00CC00
    click NFInstance "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_topology_types.go#L59" "NFInstance"
    style NFTemplate fill:#00CC00
    click NFTemplate "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_topology_types.go#L45" "NFTemplate"
    style NFInterface fill:#00CC00
    click NFInterface "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_topology_types.go#L35" "NFInterface"
    style Capacity fill:#00CC00
    click Capacity "https://doc.crds.dev/github.com/nephio-project/api/req.nephio.org/Capacity/v1alpha1@v2.0.0" "Capacity"
    style NFClass fill:#00CC00
    click NFClass "https://doc.crds.dev/github.com/nephio-project/api/req.nephio.org/NFClass/v1alpha1@v2.0.0" "NFClass"
    style PackageRevisionReference fill:#00CC00
    click PackageRevisionReference "https://github.com/nephio-project/api/blob/main/nf_topology/v1alpha1/nf_class_types.go#L25" "PackageRevisionReference"

    %% Workload constructs

    DataNetwork -- 1..n --> Pool
    DataNetwork -. refers .-> NetworkInstance
    NFDeployment -. refers .-> Provider
    NFDeployment -. refers .-> Capacity
    NFDeployment -- 1..n --> InterfaceConfig
    NFDeployment -- 1..n --> NetworkInstance
    NFDeployment -. refers .-> ParametersRefs
    style DataNetwork fill:#CCCCFF
    click DataNetwork "https://doc.crds.dev/github.com/nephio-project/api/req.nephio.org/DataNetwork/v1alpha1@v2.0.0" "DataNetwork"
    style NetworkInstance fill:#CCCCFF
    click NetworkInstance "https://doc.crds.dev/github.com/nephio-project/api/req.nephio.org/NetworkInstance/v1alpha1@v2.0.0" "NetworkInstance"
    style Pool fill:#CCCCFF
    click Pool "https://github.com/nephio-project/api/blob/main/workload/v1alpha1/nf_deployment_types.go#L165" "Pool"
{{< /mermaid >}}

A detailed API description can be found [here](topology-and-networking/).

## Porch

A detailed API description of Porch can be found [here](porch/).
