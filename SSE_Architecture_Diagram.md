# SSE Architecture Diagram

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OpenShift AI Dashboard                            │
│                          Frontend (Browser)                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ EventSource API
                                    │ GET /api/v1/metrics/stream?query=...&interval=15
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           API Gateway / Router                           │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │                    SSE Middleware (T050)                       │      │
│  │  - Set SSE headers (text/event-stream, no-cache)             │      │
│  │  - Verify http.Flusher support                                │      │
│  │  - Track request ID                                           │      │
│  │  - Log connection lifecycle                                   │      │
│  └───────────────────────────────────────────────────────────────┘      │
│                                    │                                     │
│                                    ▼                                     │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │                     Auth Middleware                            │      │
│  │  - Validate OAuth token                                        │      │
│  │  - Extract user context                                        │      │
│  │  - Load RBAC permissions                                       │      │
│  └───────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        SSE Handler (T048)                                │
│  pkg/api/handlers/sse_handlers.go                                       │
│                                                                          │
│  StreamMetrics(c echo.Context) error                                    │
│  ┌────────────────────────────────────────────────────────────┐         │
│  │ 1. Parse query parameters (query, interval)                │         │
│  │ 2. Validate interval (5-300s)                              │         │
│  │ 3. Inject namespace filters (RBAC)                         │         │
│  │ 4. Create SSE connection via ConnectionManager             │         │
│  │ 5. Send initial "connected" event                          │         │
│  │ 6. Start query worker (background goroutine)               │         │
│  │ 7. Stream events to client                                 │         │
│  └────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
                    │                                   │
                    │                                   │
        ┌───────────▼────────────┐         ┌───────────▼──────────────┐
        │   Query Worker         │         │   Event Streamer         │
        │   (goroutine)          │         │   (main thread)          │
        │                        │         │                          │
        │  ┌──────────────────┐  │         │  ┌────────────────────┐ │
        │  │ Every N seconds  │  │         │  │ Read from          │ │
        │  │ 1. Execute query │  │         │  │ MessageChan        │ │
        │  │ 2. Format result │  │         │  │                    │ │
        │  │ 3. Send event    │  │         │  │ Write SSE event    │ │
        │  └──────────────────┘  │         │  │ Flush response     │ │
        └────────────────────────┘         │  └────────────────────┘ │
                    │                      └──────────────────────────┘
                    │                                   │
                    └───────────────┬───────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    SSE Connection Manager (T049)                         │
│  pkg/sse/connection_manager.go                                          │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │                   Connection Registry                         │       │
│  │   map[connectionID]*Connection (thread-safe with RWMutex)    │       │
│  │                                                               │       │
│  │   Connection {                                                │       │
│  │     ID              string                                    │       │
│  │     UserID          string                                    │       │
│  │     MessageChan     chan *Event                               │       │
│  │     CloseChan       chan struct{}                             │       │
│  │     MetricsQuery    string                                    │       │
│  │     QueryInterval   time.Duration                             │       │
│  │   }                                                            │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                                                          │
│  Background Workers:                                                     │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐        │
│  │   Heartbeat Worker      │    │   Inactivity Checker         │        │
│  │   (every 30s)           │    │   (every 1 min)              │        │
│  │                         │    │                              │        │
│  │   - Send heartbeat to   │    │   - Find stale connections   │        │
│  │     all connections     │    │   - Close inactive (>5min)   │        │
│  │   - Detect timeouts     │    │   - Cleanup resources        │        │
│  └─────────────────────────┘    └──────────────────────────────┘        │
│                                                                          │
│  Prometheus Metrics:                                                     │
│  - sse_active_connections                                                │
│  - sse_messages_sent_total                                               │
│  - sse_connection_duration_seconds                                       │
│  - sse_connection_errors_total                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Query Execution Layer                            │
│                                                                          │
│  ┌──────────────────────┐      ┌─────────────────────────────┐          │
│  │   MetricsHandler     │      │      QueryRouter            │          │
│  │                      │      │                             │          │
│  │ - injectNamespace    │◄─────┤ - RouteQuery()              │          │
│  │   Filters (RBAC)     │      │ - ExecuteInstantQuery()     │          │
│  │ - convertPrometheus  │      │                             │          │
│  │   Result()           │      │                             │          │
│  └──────────────────────┘      └─────────────────────────────┘          │
│                                            │                             │
│                                            ▼                             │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │              Prometheus Client                               │        │
│  │  - Query()           (instant queries)                       │        │
│  │  - QueryRange()      (range queries)                         │        │
│  │  - Authentication    (service account token)                 │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Prometheus / Thanos                                │
│  - Metric storage and query engine                                      │
│  - Time series database                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Event Flow Diagram

```
Client                    SSE Handler              ConnectionManager           QueryWorker
  │                            │                           │                        │
  │ GET /metrics/stream        │                           │                        │
  ├───────────────────────────>│                           │                        │
  │                            │                           │                        │
  │                            │ CreateConnection()        │                        │
  │                            ├──────────────────────────>│                        │
  │                            │                           │                        │
  │                            │ Connection created        │                        │
  │                            │<──────────────────────────┤                        │
  │                            │                           │                        │
  │ event: connected           │                           │                        │
  │<───────────────────────────┤                           │                        │
  │ data: {...}                │                           │                        │
  │                            │                           │                        │
  │                            │ Start worker ─────────────┼───────────────────────>│
  │                            │                           │                        │
  │                            │                           │                Execute │
  │                            │                           │                query   │
  │                            │                           │<───────────────────────┤
  │                            │                           │ SendMessage(event)     │
  │                            │                           │                        │
  │ event: metric-update       │                           │                        │
  │<───────────────────────────┼───────────────────────────┤                        │
  │ data: {...}                │                           │                        │
  │                            │                           │                        │
  │                            │                           │ Heartbeat (30s)        │
  │ event: heartbeat           │                           │                        │
  │<───────────────────────────┼───────────────────────────┤                        │
  │                            │                           │                        │
  │                            │                           │                Execute │
  │                            │                           │                query   │
  │                            │                           │<───────────────────────┤
  │                            │                           │ SendMessage(event)     │
  │ event: metric-update       │                           │                        │
  │<───────────────────────────┼───────────────────────────┤                        │
  │                            │                           │                        │
  │ (client disconnect)        │                           │                        │
  ├────────────────────────────X                           │                        │
  │                            │                           │                        │
  │                            │ CloseConnection()         │                        │
  │                            ├──────────────────────────>│                        │
  │                            │                           │ Stop worker ───────────X
  │                            │                           │ Cleanup                │
  │                            │                           │                        │
```

## SSE Message Format

```
event: metric-update
data: {"timestamp":1234567890,"query_time_ms":45,"status":"success","data":{"resultType":"vector","result":[...]}}

event: heartbeat
data: {"timestamp":1234567920}

```

## Connection Lifecycle States

```
     ┌──────────┐
     │  IDLE    │
     └────┬─────┘
          │
          │ CreateConnection()
          ▼
     ┌──────────┐
     │  ACTIVE  │◄──────────┐
     └────┬─────┘           │
          │                 │
          │ SendMessage()   │ Activity
          ├─────────────────┘
          │
          │ Timeout (5min) / Client disconnect / Explicit close
          ▼
     ┌──────────┐
     │  CLOSING │
     └────┬─────┘
          │
          │ Cleanup resources
          ▼
     ┌──────────┐
     │  CLOSED  │
     └──────────┘
```

## Resource Limits

```
┌─────────────────────────────────────────────────────────────┐
│                    Resource Limits                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Per Instance:                                               │
│    Max Connections: 100                                      │
│    Memory per conn: ~10KB                                    │
│    Total memory:    ~1MB (100 connections)                   │
│                                                              │
│  Per Connection:                                             │
│    Buffer size:     10 messages                              │
│    Goroutines:      1 (query worker)                         │
│    Min interval:    5 seconds                                │
│    Max interval:    300 seconds (5 minutes)                  │
│    Inactivity:      5 minutes → auto-close                   │
│                                                              │
│  Query Limits:                                               │
│    Query timeout:   10 seconds                               │
│    Write timeout:   10 seconds                               │
│    Max query len:   1000 characters                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## RBAC Flow

```
User Request
    │
    ▼
┌──────────────────────┐
│  Extract UserContext │
│  - Username          │
│  - Namespaces        │
│  - Permissions       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐      ┌─────────────────────────┐
│  Original Query      │      │  Cluster Admin?         │
│  "DCGM_FI_DEV_GPU_   │      │  YES: No filter         │
│   UTIL"              │      │  NO:  Add namespace     │
└──────────┬───────────┘      └─────────┬───────────────┘
           │                            │
           │                            ▼
           │                  ┌─────────────────────────┐
           │                  │  Inject namespace filter│
           │                  │  namespace=~"ns1|ns2|.."│
           │                  └─────────┬───────────────┘
           │                            │
           └────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────────┐
                    │  Filtered Query      │
                    │  "DCGM_FI_DEV_GPU_   │
                    │   UTIL{namespace=~   │
                    │   'ml-platform'}"    │
                    └──────────┬───────────┘
                               │
                               ▼
                      Execute on Prometheus
```

## Monitoring Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│                    SSE Monitoring Metrics                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Active Connections                                              │
│  sse_active_connections                                          │
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░  65 / 100                               │
│                                                                  │
│  Messages Sent (rate)                                            │
│  rate(sse_messages_sent_total[5m])                               │
│  ▓▓▓▓▓▓▓▓░░░░░░░░░░░░░  450 msg/min                             │
│                                                                  │
│  Connection Duration (p95)                                       │
│  histogram_quantile(0.95, sse_connection_duration_seconds)       │
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░  245 seconds                             │
│                                                                  │
│  Error Rate                                                      │
│  rate(sse_connection_errors_total[5m])                           │
│  ▓░░░░░░░░░░░░░░░░░░░░  2 errors/min                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Implementation Highlights

1. **W3C Compliant**: Follows Server-Sent Events specification exactly
2. **Thread-Safe**: All shared state protected with sync.RWMutex
3. **Graceful Shutdown**: Notifies clients before closing connections
4. **Production-Ready**: Comprehensive error handling, timeouts, limits
5. **Observable**: Prometheus metrics and structured logging
6. **Scalable**: Stateless design, horizontal scaling support
7. **Secure**: RBAC enforcement, token validation, namespace isolation
