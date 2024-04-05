---
title: "Nephio Architecture"
type: docs
weight: 5
description: Reference for the Nephio Architecture
---

Some experiments on working with [C4 model](https://c4model.com/) to document Nephio.

## Prerequisites
To work with these PlantUML files, you will need [Graphviz](https://graphviz.org/download/) installed.

## System Context View


![System Context](/images/architecture/level1-nephio-system.png)

The system context view gives a high level perspective of the Nephio software system and the external entities that it interacts with. There are no deployment considerations in this view - the main purpose of the picture is to depict what is the responsibility and scope of Nephio, and the key interfaces and capabilities it exposes to deliver on that responsibility.

## System Landscape View

![System Landscape](diagrams/gen/level2-nephio-container.png)

Nephio is an amalgamation of software systems, so a system landscape provides a high-level view of how those software systems interoperate.

## Component Views

### Nephio Core

![Nephio Core Component View](diagrams/gen/level3-nephio-core-component.png)

Nephio core is a collection of operators and functions that perform the fundamental aspects of Nephio use cases, independent of the specifics of vendor implementations. 

The controllers for OAI and Free5GC are represented here as while they are vendor extensions to Nephio they are for now part of the Nephio system.


### Porch

![Nephio Porch Component View](diagrams/gen/nephio-porch-component-view.png)

### ConfigSync

TBD - is this a component of Nephio or a dependency?

## Deployment View

TBD

## Representative Use Cases

TBD - use cases between the major components
