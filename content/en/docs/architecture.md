---
title: "Nephio Architecture"
type: docs
weight: 5
description: Reference for the Nephio Architecture
---

Some experiments on working with [C4 model](https://c4model.com/) to document Nephio.

## Prerequisites
1. [Graphviz](https://graphviz.org/download/) is required to render some of the diagrams in this document.

## System Context View


![System Context](/images/architecture/level1-nephio-system.png)

The system context view gives a high level perspective of the Nephio software system and the external entities that it interacts with. There are no deployment considerations in this view - the main purpose of the picture is to depict what is the responsibility and scope of Nephio, and the key interfaces and capabilities it exposes to deliver on that responsibility. 

- Comments, 1) it will be good to clarify the meaning of the color coding of the functional components used in the Architecture; 2) "Supported Network Functions" is tagged as "external system",  would it be better to tag it as "external vendor network functions", since these are more deployment artifacts than systems ?

## System Landscape View

![System Landscape](/images/architecture/level2-nephio-container.png)

Nephio is an amalgamation of software systems, so a system landscape provides a high-level view of how those software systems interoperate.

- Comments, 1) Assume from this figure on, its R2 view, so change the title to "R2 system Landscape View"; 2) Significance of the color codes could be clarified too.

## Component Views

### Nephio Core

![Nephio Core Component View](/images/architecture/level3-nephio-core-component.png)

Nephio core is a collection of operators and functions that perform the fundamental aspects of Nephio use cases, independent of the specifics of vendor implementations. 

The controllers for OAI and Free5GC are represented here. Although they are vendor extensions to Nephio, they are for now part of the Nephio system.

- Comments, 1) Change the title to "R2 Nephio Core Component View" ; 2) Is there a "Inventory" implementation in R2 ?

### Porch

![Nephio Porch Component View](/images/architecture/nephio-porch-component-view.png)

### ConfigSync

TBD - is this a component of Nephio or a dependency?

## Deployment View

TBD

## Representative Use Cases

TBD - use cases between the major components
