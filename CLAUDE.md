# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Test Commands

```bash
make test          # Run full test suite (lint + tests + examples verification)
make lint          # Run golangci-lint
make lint-fix      # Auto-fix linting issues
make modules       # Tidy all go.mod files
```

Run a single test:
```bash
go test -v -race ./pkg/client/... -run TestClientGet
```

Run tests for a specific package:
```bash
go test -v -race ./pkg/controller/...
```

## Required Import Aliases

The linter enforces these import aliases (see `.golangci.yml`):

```go
import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    kerrors "k8s.io/apimachinery/pkg/util/errors"
    ctrl "sigs.k8s.io/controller-runtime"
)
```

## Architecture Overview

Controller-runtime provides libraries for building Kubernetes controllers. The core reconciliation flow:

```
Source (watch events) → EventHandler (enqueue requests) → Controller (worker queue) → Reconciler (business logic)
```

### Key Packages

- **pkg/manager**: Creates and coordinates controllers, caches, and clients. All components share a Manager.
- **pkg/controller**: Worker queue implementation that processes reconcile.Requests.
- **pkg/reconcile**: Defines the `Reconciler` interface that implements business logic.
- **pkg/builder**: High-level API for constructing controllers and webhooks.
- **pkg/client**: Split client - reads from cache, writes to API server.
- **pkg/cache**: Local informer-based cache with field indexing support.
- **pkg/source**: Event sources (watches) that feed the controller.
- **pkg/handler**: Event handlers that create reconcile.Requests. Key implementations:
  - `EnqueueRequestForObject` - reconcile the object itself
  - `EnqueueRequestForOwner` - reconcile the owning object
  - `EnqueueRequestsFromMapFunc` - custom mapping logic
- **pkg/predicate**: Event filters to reduce unnecessary reconciliations.
- **pkg/webhook**: Admission webhook server for mutating/validating requests.
- **pkg/envtest**: Integration testing with real etcd and kube-apiserver.

### Controller Design Principles

1. **Reconcilers should be idempotent** - read full state and reconcile to desired state each time
2. **One object type per controller** - use owner references and event handlers to map related objects
3. **Cache reads, API writes** - the client reads from cache (may be stale) and writes to API server
4. **Event batching** - multiple events for same object are deduplicated in the work queue

## Testing Approach

Use `envtest.Environment` for integration tests rather than mocking the API server. Tests should verify system state, not specific API call sequences.

## PR Labels

All PRs require one of: 🐛 (patch fix), ✨ (feature), ⚠️ (breaking change)
