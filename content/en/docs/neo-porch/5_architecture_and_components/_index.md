---
title: "Architecture & Components"
type: docs
weight: 5
description: Porch Architecture and its underlying components
---

## Overview

This section provides developer-level documentation of Porch's internal architecture. It explains how the main components work together to provide package orchestration capabilities, and details the implementation of each component and its subcomponents.

Porch is built as a **Kubernetes extension** consisting of multiple deployable components that work together to manage the lifecycle of KRM configuration packages stored in Git or OCI repositories.

## High-Level Architecture

Porch's architecture follows a modular design with clear separation of concerns. The diagram below shows the main components and their relationships:

![Porch Architecture](/static/images/porch/Porch-Architecture.drawio.svg)

The architecture includes:

- **Porch Server**: Kubernetes Aggregated API Server containing the Engine, Cache, and Repository Adapters
- **Function Runner**: Separate gRPC service for executing KRM functions
- **Controllers**: Kubernetes controllers for automation (PackageVariant, PackageVariantSet)
- **External Repositories**: Git or OCI repositories storing the actual packages

## Deployable Components

Porch consists of the following independently deployable components:

### 1. Porch Server

The main API server that extends Kubernetes with Porch-specific resources. It handles all package operations, coordinates with other components, and manages package lifecycle.

**Deployment**: Single pod (can scale horizontally)  
**Port**: 4443 (API), 8443 (webhooks)

**Internal Components**:

- **API Server**: Kubernetes Aggregated API Server implementation
- **Engine (CaD Engine)**: Core orchestration logic for package lifecycle operations
- **Task Handler**: Executes package operations (init, clone, edit, update, render)
- **Repository Adapters**: Abstraction layer for Git/OCI repository interactions
- **Cache Client**: Interface to the caching layer
- **Watcher Manager**: Manages watch streams for real-time package revision updates
- **Webhooks**: Validation and mutation webhooks for API resources

### 2. Function Runner

A gRPC service that executes KRM functions to transform and validate package contents.

**Deployment**: 2+ replicas for availability  
**Port**: 9445 (gRPC)

**Internal Components**:

- **gRPC Server**: Exposes function evaluation endpoint
- **Executable Evaluator**: Executes built-in functions directly in the runner pod
- **Pod Evaluator**: Spawns function pods for external functions
- **Pod Cache Manager**: Manages lifecycle of cached function pods
- **Multi-Evaluator**: Coordinates between different evaluation strategies

### 3. Controllers

Kubernetes controllers that automate package operations by watching Porch resources and reconciling desired state.

**Deployment**: Single pod with leader election  

**Internal Components**:

- **PackageVariant Controller**: Automates package cloning and updates
- **PackageVariantSet Controller**: Manages sets of package variants
- **Reconciliation Logic**: Watches resources and maintains desired state

### 4. Cache

Storage backend for repository content caching. Can be either CR-based (using Kubernetes resources) or PostgreSQL-based.

**Deployment**:

- **CR Cache**: No separate deployment (uses K8s API)
- **DB Cache**: PostgreSQL StatefulSet

**Internal Components**:

- **Repository Sync Manager**: Periodic synchronization with external repositories
- **Cache Storage**: Stores package contents and metadata
- **Cache Invalidation**: Handles cache updates and cleanup

## Component Interaction Flow

Here's how components interact during a typical package operation (e.g., creating and rendering a package):

1. **User Request**: Client sends API request to Porch Server
2. **API Server**: Validates request, authenticates user
3. **Engine**: Orchestrates the operation
4. **Cache**: Opens repository, checks current state
5. **Repository Adapter**: Interacts with Git/OCI if needed
6. **Task Handler**: Applies tasks (init, edit, etc.)
7. **Function Runner**: Executes KRM functions (if rendering)
8. **Repository Adapter**: Commits changes to Git/OCI
9. **Cache**: Updates cached state
10. **Watcher Manager**: Notifies watching clients
11. **Response**: Returns updated package revision to client

## Design Principles

Porch's architecture follows these key principles:

1. **Configuration as Data**: Packages are treated as versioned data, not imperative scripts
2. **Git-Native**: Leverages Git's versioning and collaboration features
3. **Kubernetes-Native**: Extends Kubernetes API, uses standard patterns
4. **Separation of Concerns**: Clear boundaries between components
5. **Extensibility**: Function-based transformation pipeline
6. **Performance**: Caching layer for scalability
7. **Automation**: Controllers enable GitOps workflows
