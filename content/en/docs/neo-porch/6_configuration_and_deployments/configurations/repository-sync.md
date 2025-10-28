---
title: "Repository Sync Configuration"
type: docs
weight: 1
description: Repository Sync Configuration
---

# Repository Sync Configuration

This document describes how to configure repository synchronization in Porch using the Repository Custom Resource (CR).

## Sync Configuration Fields

The `spec.sync` field in a Repository CR controls how Porch synchronizes with the external repository:

```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: example-repo
  namespace: default
spec:
  sync:
    schedule: "*/10 * * * *"           # Periodic sync using cron expression
    runOnceAt: "2024-01-15T10:30:00Z"  # One-time sync at specific time
```

### Schedule Field

The `schedule` field accepts standard cron expressions for periodic synchronization:

- **Format**: Standard 5-field cron expression (`minute hour day month weekday`)
- **Examples**:
  - `"*/10 * * * *"` - Every 10 minutes
  - `"0 */2 * * *"` - Every 2 hours
  - `"0 9 * * 1-5"` - 9 AM on weekdays
  - `"0 0 * * 0"` - Weekly on Sunday at midnight

### RunOnceAt Field

The `runOnceAt` field schedules a one-time sync at a specific timestamp:

- **Format**: RFC3339 timestamp (`metav1.Time`)
- **Behavior**: 
  - Executes once at the specified time
  - Ignored if timestamp is in the past
  - Independent of periodic schedule
  - Can be updated to reschedule

## Complete Examples

### Git Repository with Periodic Sync

```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: blueprints
  namespace: default
spec:
  description: Blueprints with hourly sync
  type: git
  sync:
    schedule: "0 * * * *"  # Every hour
  git:
    repo: https://github.com/example/blueprints.git
    branch: main
    directory: packages
```

### Combined Periodic and One-time Sync

```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: combined-sync
  namespace: default
spec:
  type: git
  sync:
    schedule: "0 */6 * * *"            # Every 6 hours
    runOnceAt: "2024-01-15T09:00:00Z"  # Sync once
  git:
    repo: https://github.com/example/repo.git
    branch: main
```

## Sync Behavior

### Default Behavior
- Without `spec.sync`: Uses system default sync frequency()
- Empty `schedule`: Falls back to default frequency
- Invalid cron expression: Falls back to default frequency

### Sync Manager Operation
Each repository runs two independent goroutines:

1. **Periodic Sync** (`syncForever`):
   - Syncs once at startup
   - Follows cron schedule or default frequency
   - Updates repository conditions after each sync

2. **One-time Sync** (`handleRunOnceAt`):
   - Monitors `runOnceAt` field changes
   - Creates/cancels timers as needed
   - Executes independently of periodic sync

### Status Updates
Repository sync status is reflected in the Repository CR conditions:

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    reason: Ready
    message: "Repository Ready"
    lastTransitionTime: "2024-01-15T10:30:00Z"
```

## Troubleshooting

### Common Issues

1. **Invalid Cron Expression**:
   - Check logs for parsing errors
   - Verify 5-field format
   - Repository falls back to default frequency

2. **Past RunOnceAt Time**:
   - One-time sync is skipped
   - Update to future timestamp
   - Check repository logs

3. **Sync Failures**:
   - Check repository conditions
   - Verify authentication credentials
   - Review repository accessibility

### Monitoring
- Repository conditions show sync status
- Logs contain detailed sync information
- Next sync time is logged for periodic syncs


## CLI Commands

### Repository Registration with Sync

Use `porchctl repo reg` to register repositories with sync configuration:

```bash
# Register Git repository with periodic sync
porchctl repo reg https://github.com/example/repo.git \
  --name my-repo \
  --namespace default \
  --sync-schedule "*/10 * * * *"

# Register OCI repository with sync
porchctl repo reg oci://gcr.io/example/packages \
  --name oci-repo \
  --sync-schedule "0 */2 * * *"

# Register with authentication
porchctl repo reg https://github.com/private/repo.git \
  --name private-repo \
  --repo-basic-username myuser \
  --repo-basic-password mytoken \
  --sync-schedule "0 9 * * 1-5"
```

#### Registration Flags
- `--sync-schedule`: Cron expression for periodic sync
- `--name`: Repository name (defaults to last URL segment)
- `--description`: Repository description
- `--branch`: Git branch (default: main)
- `--directory`: Package directory within repo
- `--deployment`: Mark as deployment repository
- `--repo-basic-username/--repo-basic-password`: Basic auth
- `--repo-workload-identity`: Use workload identity

### Repository Sync Commands

Use `porchctl repo sync` to trigger immediate syncs:

```bash
# Sync specific repository (schedules 1-minute delayed sync)
porchctl repo sync my-repo --namespace default

# Sync multiple repositories
porchctl repo sync repo1 repo2 repo3 --namespace default

# Sync all repositories in namespace
porchctl repo sync --all --namespace default

# Sync all repositories across all namespaces
porchctl repo sync --all --all-namespaces

# Schedule sync with custom delay
porchctl repo sync my-repo --run-once 5m
porchctl repo sync my-repo --run-once 2h30m

# Schedule sync at specific time
porchctl repo sync my-repo --run-once "2024-01-15T14:30:00Z"
```

#### Sync Command Flags
- `--all`: Sync all repositories in namespace
- `--all-namespaces`: Include all namespaces
- `--run-once`: Schedule one-time sync (duration or RFC3339 timestamp)
- `--namespace`: Target namespace

#### Sync Behavior
- Minimum delay: 1 minute from command execution
- Updates `spec.sync.runOnceAt` field in Repository CR
- Independent of existing periodic sync schedule
- Past timestamps automatically adjusted to minimum delay