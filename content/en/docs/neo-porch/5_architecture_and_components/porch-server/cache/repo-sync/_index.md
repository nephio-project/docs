---
title: "Repository Sync"
type: docs
weight: 4
---

# Porch Repository Sync Architecture

## Overview

The Porch sync system manages the synchronization of package repositories between external sources (Git/OCI) and the internal cache. It consists of two main cache implementations that both utilize a common sync manager to handle periodic and one-time synchronization operations. The architecture consists of two main flows: **SyncManager-driven synchronization** for package content and **Background process** for Repository CR lifecycle management.

### High-Level Architecture

![Repository Sync Architecture](/static/images/porch/repository-sync.svg)

## Core Components

### 1. SyncManager (`pkg/cache/sync/sync.go`)

**Purpose**: Central orchestrator for repository synchronization operations.

**Components**:
- `handler` (SyncHandler interface)
- `coreClient` (Kubernetes client)
- `nextSyncTime` (scheduling)
- `lastSyncError` (error tracking)

**Goroutines**:

1. **syncForever()** - Periodic sync with cron scheduling
   - Syncs once at startup, then uses ticker to check countdown
   - Supports both cron expressions (spec.sync.schedule) and default frequency fallback
   - Recalculates next sync time when cron expression changes
   - Updates repository conditions after each sync

2. **handleRunOnceAt()** - One-time sync with timer-based execution
   - Monitors spec.sync.runOnceAt field for scheduled one-time syncs
   - Creates/cancels timers when the runOnceAt time changes
   - Skips past timestamps and handles timer cleanup
   - Independent of periodic sync schedule


### 2. Cache Handlers (Implements SyncHandler)

Both cache implementations follow the same interface pattern:

#### repositorySync (DB Cache)
- Database-backed repository cache
- External repository synchronization
- Mutex-based thread safety
- Sync statistics tracking

#### cachedRepository (CR Cache)  
- In-memory repository cache
- External repository synchronization
- Mutex-based thread safety
- Metadata store integration

### 3. Background Process (`pkg/registry/porch/background.go`)

**Purpose**: Manages Repository CR lifecycle and cache updates.

**Components**:
- **K8S API** - Source of Repository CRs
- **Repository CRs** - Custom resources defining repositories
- **Watch Events** - Real-time CR change notifications
- **Periodic Ticker** - RepoCrSyncFrequency-based updates

## Architecture Flows

### Package Content Synchronization

<pre>
SyncManager → Goroutines   →   Cache Handlers   → Condition Management
     ↓              ↓              ↓                  ↓
  Start()     syncForever()     SyncOnce()      Set/Build/Apply
             handleRunOnceAt()                  RepositoryCondition
</pre>

**Process**:
1. SyncManager starts two goroutines
2. Goroutines call handler.SyncOnce() on cache implementations
3. Cache handlers perform sync operations
4. All components update repository conditions

### Repository Lifecycle Management

<pre>
K8S API   →  Repository CRs   →  Watch Events   →  Background.go    →  Cache Spec Update
    ↓             ↓                 ↓                    ↓                  ↓
Kubernetes    CR Changes        Added/Modified/      Event Handler      OpenRepository/
 Cluster                           Deleted          cacheRepository     CloseRepository
</pre>

**Process**:
1. Repository CRs created/modified/deleted in Kubernetes
2. Watch events generated for CR changes
3. Background.go receives and processes events
4. Cache updated via OpenRepository/CloseRepository calls
5. Periodic ticker ensures consistency

### Event-Driven Status Updates

<pre>
Background.go     ←-→   Watch Events   ←-→  Repository CRs
     ↓                                            ↑
Cache Repository Update                    Status Updates
     ↓                                            ↑
Repository SyncManagers     --→        Condition Management
</pre>

## Sync Process Details

### Common Sync Process (Both Caches)
1. Acquire mutex lock (if applicable)
2. Set condition to "sync-in-progress"
3. Fetch cached package revisions
4. Fetch external package revisions
5. Compare and identify differences
6. Update cache (add/remove packages)
7. Release mutex and update final condition

### Background Event Handling
1. **Added/Modified Events**: Call `cache.OpenRepository()`
2. **Deleted Events**: Call `cache.CloseRepository()`
3. **Bookmark Events**: Update resource version for watch continuity
4. **Status Updates**: Update Repository CR conditions

## Condition Management

### Condition States
- **sync-in-progress**: Repository synchronization actively running
- **ready**: Repository synchronized and ready for use
- **error**: Synchronization failed with error details

### Condition Functions
- `SetRepositoryCondition()`: Updates repository status
- `BuildRepositoryCondition()`: Creates condition objects
- `ApplyRepositoryCondition()`: Applies conditions to CRs

## Interface Contracts

### SyncHandler Interface
```go
type SyncHandler interface {
    SyncOnce(ctx context.Context) error
    Key() repository.RepositoryKey
    GetSpec() *configapi.Repository
}
```

The SyncHandler interface is implemented by two cache types:

- **Database Cache**: `repositorySync` struct in `pkg/cache/dbcache/dbreposync.go`
- **Custom Resource Cache**: `cachedRepository` struct in `pkg/cache/crcache/repository.go`

## Configuration

### Repository Sync Specification
```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
spec:
  sync:
    schedule: "0 */30 * * *"  # Cron expression
    runOnceAt: "2024-01-15T10:00:00Z"  # One-time sync
```

### Background Process Configuration
- **RepoCrSyncFrequency**: Periodic sync interval
- **Watch Reconnection**: Exponential backoff (1s - 30s)

## Error Handling & Resilience

### SyncManager Errors
- Captured in `lastSyncError`
- Reflected in repository conditions
- Retried on next sync cycle

### Background Process Errors
- Watch connection failures → Exponential backoff reconnection
- Repository validation errors → Status condition with error message
- API conflicts on status updates → Retry with backoff

### Condition Update Errors
- Logged as warnings
- Don't block sync operations
- Include retry logic with conflict resolution

## Concurrency & Safety

### Thread Safety
- **DB Cache**: `sync.Mutex` for sync operations
- **CR Cache**: `sync.Mutex` for cache access
- **Background**: Watch event serialization

### Context Management
- Cancellable contexts for graceful shutdown
- Separate contexts for sync operations
- Timeout handling for long-running operations

## Monitoring & Observability

### Logging
- Sync start/completion times with duration
- Package revision statistics (cached/external/both)
- Error conditions and warnings
- Schedule changes and next sync times
- Background event processing
- Watch connection status

### Key Metrics (via logging)
- Sync duration and frequency
- Package counts and changes
- Success/failure rates
- Condition transition events
- Background event processing rates