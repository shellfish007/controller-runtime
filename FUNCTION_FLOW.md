# Controller-Runtime Function Call Flow

This document traces how functions call each other from startup through reconciliation.

## Table of Contents

1. [Visual Overview](#visual-overview)
2. [Startup Flow](#startup-flow)
   - Manager Startup
   - Runnable Groups
   - Controller Registration
   - Watch Registration
3. [Cache & Sync](#cache--sync)
   - Source Sync at Controller Startup
   - Kind Source Startup
   - Two-Level Sync Explained
4. [Event Flow](#event-flow)
   - Event Handler Implementation
   - Informer to Queue
5. [Reconciliation](#reconciliation)
   - Worker Loop
   - Reconciliation Handler
   - How Your Reconcile Gets Called
6. [Shutdown Flow](#shutdown-flow)
7. [Key Files Reference](#key-files-reference)

---

## Visual Overview

```
+--------------------------------------------------------------+
|                      STARTUP                                  |
|  Manager.Start() -> Cache.Start() -> Controller.Start()      |
|                         |                                     |
|              +----------+----------+                          |
|              v                     v                          |
|        Informers sync         Workers spawn                   |
|              |                     |                          |
+--------------+---------------------+--------------------------+
               |                     |
+--------------+---------------------+--------------------------+
|              |     RUNTIME LOOP    |                          |
|              v                     |                          |
|   +---------------------+          |                          |
|   |  API Server Event   |          |                          |
|   +----------+----------+          |                          |
|              v                     |                          |
|   +---------------------+          |                          |
|   |  Informer callback  |          |                          |
|   +----------+----------+          |                          |
|              v                     |                          |
|   +---------------------+          |                          |
|   |  Predicates filter  |          |                          |
|   +----------+----------+          |                          |
|              v                     |                          |
|   +---------------------+          |                          |
|   |  Handler enqueues   |----------+--> Work Queue            |
|   +---------------------+          |         |                |
|                                    |         |                |
|                                    v         v                |
|                              +-------------------+            |
|                              |  Worker dequeues  |            |
|                              +---------+---------+            |
|                                        v                      |
|                              +-------------------+            |
|                              | Reconciler.       |            |
|                              |   Reconcile()     |            |
|                              |   (YOUR CODE)     |            |
|                              +---------+---------+            |
|                                        v                      |
|                              +-------------------+            |
|                              | Result handling   |            |
|                              | (requeue/forget)  |            |
|                              +-------------------+            |
+---------------------------------------------------------------+
```

---

## Startup Flow

### Manager Startup (Summary)

```
Manager.Start(ctx)
+-- cm.runnables.HTTPServers.Start()    // Health, metrics, pprof
+-- cm.runnables.Webhooks.Start()       // Admission webhooks
+-- cm.runnables.Caches.Start()         // Informer cache sync
+-- cm.runnables.Others.Start()         // Non-leader-election runnables
+-- cm.runnables.Warmup.Start()         // Optional warmup
+-- leaderElector.Run()                 // If leader election enabled
    +-- OnStartedLeading callback
        +-- cm.runnables.LeaderElection.Start()  // Controllers start here
```

### Manager.Start() Full Trace

**File:** `pkg/manager/internal.go:347-495`

```
Manager.Start(ctx context.Context) error
+-- controllerManager.Start(ctx context.Context) error
    |
    +-- Initialize internal context
    |   +-- cm.internalCtx, cm.internalCancel = context.WithCancel(ctx)
    |
    +-- Initialize leader election (if configured)
    |   +-- cm.initLeaderElector()
    |
    +-- Add cluster runnable
    |   +-- cm.add(cm.cluster)
    |
    +-- Add metrics server (if configured)
    |   +-- cm.runnables.HTTPServers.Add(cm.metricsServer, nil)
    |
    +-- Add health probe server (if configured)
    |   +-- cm.addHealthProbeServer()
    |
    +-- Add pprof server (if configured)
    |   +-- cm.addPprofServer()
    |
    +-- Start runnables in order:
        1. HTTP Servers (health probes, metrics, pprof)
           +-- cm.runnables.HTTPServers.Start(logCtx)

        2. Webhooks (conversion, validation, defaulting)
           +-- cm.runnables.Webhooks.Start(cm.internalCtx)

        3. Caches
           +-- cm.runnables.Caches.Start(cm.internalCtx)

        4. Non-leader election runnables
           +-- cm.runnables.Others.Start(cm.internalCtx)

        5. Warmup runnables (optional, for controller cache pre-warming)
           +-- cm.runnables.Warmup.Start(cm.internalCtx)

        6. Leader election + controllers
           +-- if leaderElector != nil:
              +-- leaderElector.Run(leaderCtx)
              |   +-- OnStartedLeading callback:
              |       +-- cm.startLeaderElectionRunnables()
              |           +-- cm.runnables.LeaderElection.Start(cm.internalCtx)
              |
              +-- else: (no leader election)
                  +-- cm.startLeaderElectionRunnables()
```

### Runnable Groups

The Manager organizes runnables into 6 groups that start in order:

| Order | Group | Description |
|-------|-------|-------------|
| 1 | **HTTPServers** | Health probes, metrics, pprof (always running) |
| 2 | **Webhooks** | Conversion/validation webhooks (before cache) |
| 3 | **Caches** | Cluster cache (synced before workers start) |
| 4 | **Warmup** | Optional controller warmup (before leader election) |
| 5 | **Others** | Controllers not needing leader election |
| 6 | **LeaderElection** | Controllers requiring leader election |

### Controller Registration

**File:** `pkg/controller/controller.go:184-199`

```
New(name string, mgr manager.Manager, options Options) (Controller, error)
+-- NewTyped(name, mgr, options)
    +-- options.DefaultFromConfig(mgr.GetControllerOptions())
    +-- NewTypedUnmanaged(name, options)
    |   +-- controller.New[request](controller.Options)
    |       +-- Creates *Controller[request] with:
    |           +-- work queue (priority queue by default)
    |           +-- rate limiter (exponential backoff)
    |           +-- reconciler (c.Do)
    |
    +-- mgr.Add(c)  [Register controller as Runnable]
        +-- Calls: cm.add(controller)
            +-- Determines controller type via interface matching:
                +-- warmupRunnable? -> Add to Warmup group
                +-- LeaderElectionRunnable?
                |   +-- NeedLeaderElection() == true?
                |       +-- Add to LeaderElection group
                |       +-- else: Add to Others group
                +-- else: Add to LeaderElection group (default)
```

### Watch Registration (Builder Pattern)

**File:** `pkg/builder/controller.go:265-327`

When you call `ctrl.NewControllerManagedBy(mgr).For(...).Owns(...).Complete(reconciler)`:

```
Builder.Build(reconciler)                       // :266
+-- blder.doController(r)                       // :278 - creates controller with reconciler
|
+-- blder.doWatch()                             // :283 - registers all watches
    |
    +-- For each For():                         // :309-327
    |   +-- src := source.TypedKind(cache, obj, EnqueueRequestForObject, predicates)
    |   +-- ctrl.Watch(src)                     // adds to c.startWatches
    |
    +-- For each Owns():                        // :333-356
    |   +-- src := source.TypedKind(cache, obj, EnqueueRequestForOwner, predicates)
    |   +-- ctrl.Watch(src)
    |
    +-- For each Watches():                     // :358-370
        +-- src := source.TypedKind(cache, obj, handler, predicates)
        +-- ctrl.Watch(src)
```

**Note:** `ctrl.Watch(src)` just appends to `c.startWatches` slice - sources are not started yet.

---

## Cache & Sync

### Controller Startup (Summary)

```
Controller.Start(ctx)
+-- startEventSourcesAndQueueLocked()
|   +-- c.NewQueue()                    // Create work queue
|   +-- for each source:
|       +-- source.Start(ctx, queue)    // Start watches
|           +-- Kind.Start()
|               +-- cache.GetInformer() // Get/create informer
|               +-- informer.AddEventHandler(EventHandler)
|               +-- cache.WaitForCacheSync()
|
+-- for i := 0; i < MaxConcurrentReconciles; i++:
    +-- go worker()
        +-- for { c.processNextWorkItem(ctx) }
```

### Source Sync at Controller Startup

**File:** `pkg/internal/controller/controller.go:332-416`

When controller starts, it syncs all registered sources:

```
Controller.Start(ctx)
+-- startEventSourcesAndQueueLocked()           // :299
    |
    +-- Create work queue                       // :338-343
    |
    +-- For each source in c.startWatches (in parallel):
        |
        +-- watch.Start(ctx, queue)             // :372 - start the source
        |   |
        |   +-- Kind.Start()                    // pkg/internal/source/kind.go:45
        |       +-- cache.GetInformer(type)     // get/create informer
        |       +-- informer.AddEventHandler()  // register event handler
        |
        +-- If source implements SyncingSource: // :376-385
            +-- syncingSource.WaitForSync()     // :381
                |
                +-- Kind.WaitForSync()          // pkg/internal/source/kind.go:127
                    |
                    +-- Two-level sync:
                        1. cache.WaitForCacheSync()           // :103 - objects loaded
                        2. toolscache.WaitForCacheSync(       // :109 - handler synced
                             handlerRegistration.HasSynced)
```

### Kind Source Startup

**File:** `pkg/internal/source/kind.go:45-116`

```
Kind[object, request].Start(ctx, queue) error
+-- Validate Type, Cache, Handler are non-nil
+-- Create cancelable context: ctx, ks.startCancel = context.WithCancel(ctx)
+-- ks.startedErr = make(chan error, 1)
|
+-- Go routine:  [Async source startup]
    +-- Retry until success or context done:
    |   +-- ks.Cache.GetInformer(ctx, ks.Type)
    |       [Gets or creates informer for the resource type]
    |
    +-- handlerRegistration := i.AddEventHandlerWithOptions(
    |       NewEventHandler(ctx, queue, ks.Handler, ks.Predicates),
    |       HandlerOptions{Logger: &logKind}
    |   )
    |   [Registers handler to receive events from informer]
    |
    +-- ks.Cache.WaitForCacheSync(ctx)
    |   [Wait for initial list to be fetched]
    |
    +-- toolscache.WaitForCacheSync(ctx.Done(), handlerRegistration.HasSynced)
    |   [Wait for this specific handler to receive all initial events]
    |
    +-- close(ks.startedErr)
```

### Two-Level Sync Explained

| Level | What It Waits For | Why It Matters |
|-------|-------------------|----------------|
| **Cache sync** | Informer has fetched initial list from API server | Objects exist in local store |
| **Handler sync** | Event handler received `OnAdd()` for all existing objects | Work queue has initial reconcile requests |

Without handler sync, the controller could start processing before all existing objects are enqueued, causing missed reconciliations on startup.

---

## Event Flow

### Event Flow (Summary)

```
[API Server change detected]
    |
    v
Informer.OnAdd/OnUpdate/OnDelete(obj)
    |
    v
EventHandler.OnAdd/OnUpdate/OnDelete()           // pkg/internal/source/event_handler.go
+-- Create TypedEvent{Object: obj}
+-- for each predicate:
|   +-- predicate.Create/Update/Delete(event)    // Filter events
|       +-- return false -> event dropped
|
+-- handler.Create/Update/Delete(ctx, event, queue)
    +-- EnqueueRequestForObject                  // pkg/handler/enqueue.go
        +-- queue.Add(reconcile.Request{Name, Namespace})
```

### Event Handler Implementation

**File:** `pkg/internal/source/event_handler.go:37-169`

```
NewEventHandler[object, request](ctx, queue, handler, predicates)
+-- &EventHandler[object, request]{
        ctx: ctx,
        handler: handler,
        queue: queue,
        predicates: predicates,
    }
```

When informer detects object changes:

**OnAdd (resource created):**
```
EventHandler.OnAdd(obj, isInInitialList)
+-- Create TypedCreateEvent[object]
+-- Check all predicates:
|   +-- for p in predicates: p.Create(event)
|
+-- If all predicates pass:
    +-- e.handler.Create(ctx, event, e.queue)
```

**OnUpdate (resource modified):**
```
EventHandler.OnUpdate(oldObj, newObj)
+-- Create TypedUpdateEvent[object]
+-- Check all predicates:
|   +-- for p in predicates: p.Update(event)
|
+-- If all predicates pass:
    +-- e.handler.Update(ctx, event, e.queue)
```

**OnDelete (resource deleted):**
```
EventHandler.OnDelete(obj)
+-- Create TypedDeleteEvent[object]
+-- Handle tombstone events (DeleteFinalStateUnknown)
+-- Check all predicates:
|   +-- for p in predicates: p.Delete(event)
|
+-- If all predicates pass:
    +-- e.handler.Delete(ctx, event, e.queue)
```

---

## Reconciliation

**File:** `pkg/internal/controller/controller.go`

```
Controller.Start()                                      // :276
    |
    +-- startEventSourcesAndQueueLocked()               // :299 - start watches, sync cache
    |
    +-- Launch worker goroutines                        // :308-316
            |
            +-- for c.processNextWorkItem(ctx) {}       // :313
                        |
                        v
                    processNextWorkItem()               // :420
                        |
                        +-- c.Queue.GetWithPriority()   // :421 - blocks until item
                        |
                        +-- defer c.Queue.Done(obj)
                        |
                        +-- reconcileHandler()          // :462
                                |
                                +-- log := c.LogConstructor(&req)
                                +-- reconcileID := uuid.NewUUID()
                                +-- ctx = logf.IntoContext(ctx, log)
                                |
                                +-- c.Reconcile()       // :195 - wrapper
                                |       |
                                |       +-- defer recover()           // panic recovery
                                |       +-- ctx = WithTimeout(...)    // timeout
                                |       +-- c.Do.Reconcile(ctx, req)  // YOUR CODE
                                |
                                +-- Handle result:
                                    +-- err != nil:
                                    |   +-- TerminalError -> don't requeue
                                    |   +-- else -> queue.Add(RateLimited)
                                    +-- RequeueAfter > 0 -> queue.Add(After: duration)
                                    +-- Requeue -> queue.Add(RateLimited)
                                    +-- success -> queue.Forget(req)
```

---

## Shutdown Flow

```
engageStopProcedure()
+-- Stop warmup runnables
+-- Stop non-leader runnables
+-- Stop leader election runnables (controllers)
|   +-- All controllers receive ctx.Done()
|   +-- Workers finish current items
|   +-- Work queue shuts down
|   +-- processNextWorkItem returns false
+-- Stop caches
+-- Stop webhooks
+-- Stop HTTP servers
```

---

## Key Files Reference

| Component | File | Key Functions |
|-----------|------|---------------|
| Manager startup | `pkg/manager/internal.go` | `Start()` line 347 |
| Controller startup | `pkg/internal/controller/controller.go` | `Start()` line 276 |
| Worker loop | `pkg/internal/controller/controller.go` | `processNextWorkItem()` line 420 |
| Reconcile wrapper | `pkg/internal/controller/controller.go` | `Reconcile()` line 195 |
| Event bridge | `pkg/internal/source/event_handler.go` | `OnAdd/Update/Delete()` |
| Kind source | `pkg/internal/source/kind.go` | `Start()` line 45 |
| Enqueue handler | `pkg/handler/enqueue.go` | `Create/Update/Delete()` |
| Builder | `pkg/builder/controller.go` | `Build()`, `doWatch()` |
