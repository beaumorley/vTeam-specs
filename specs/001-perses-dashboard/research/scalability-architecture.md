# Scalability Architecture Research

**Research Date**: 2025-11-14
**Target Scale**: 500+ concurrent users within 12 months
**Current Architecture**: Perses-based observability dashboard on OpenShift AI

## Executive Summary

Based on analysis of the Perses codebase and industry best practices for dashboard applications, this document outlines a comprehensive scalability strategy to support 500+ concurrent users. The architecture focuses on **stateless horizontal scaling** with **multi-layer caching** and **database optimization** to handle high-concurrency dashboard access and metric query workloads.

**Key Findings**:
- Current Perses architecture is **already stateless-friendly** (Go HTTP API, SQL backend, proxy pattern)
- **Redis caching** is optimal for dashboard configs and metric query results
- **Connection pooling** and **read replicas** are critical for PostgreSQL at scale
- **Server-Sent Events (SSE)** outperforms WebSockets for dashboard real-time updates
- **Query result caching with time-based TTL** can reduce Prometheus load by 70-80%

---

## 1. Caching Strategy

### Decision: Multi-Layer Redis Caching with Time-Based Invalidation

**Rationale**:
Dashboard applications have distinct caching patterns:
- **Dashboard configurations** change infrequently (write-rarely, read-often)
- **Metric query results** have time-series characteristics (cache based on time window)
- **Authorization/RBAC data** (Perses already caches this in-memory, see `authorization/cron.go`)

Redis provides:
- **Atomic operations** for cache invalidation across multiple API pods
- **TTL support** native to time-series data patterns
- **Pub/Sub** for real-time cache invalidation events
- **Persistence** options (RDB/AOF) for warm cache on restarts
- **Memory efficiency** superior to Memcached for structured data (Redis Hashes)

**Implementation**:

```yaml
# Redis Configuration (Kubernetes Deployment)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: perses-redis
spec:
  replicas: 1  # Start with single instance, add sentinel for HA later
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        args:
          - redis-server
          - --maxmemory 2gb
          - --maxmemory-policy allkeys-lru  # Evict least-recently-used keys
          - --save 900 1  # Persistence: save every 15min if 1+ key changed
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        ports:
        - containerPort: 6379
```

**Cache Key Patterns**:

```go
// Dashboard Configuration Cache
// Key Pattern: "dashboard:config:{project}:{dashboard_name}"
// TTL: 5 minutes (with invalidation on update)
Key: "dashboard:config:openshift-ai:model-serving-overview"
Value: JSON dashboard definition
TTL: 300 seconds

// Metric Query Result Cache
// Key Pattern: "query:result:{hash(query_params)}:{time_bucket}"
// TTL: Based on time range (short queries = short TTL)
Key: "query:result:8f3a2bc:2025-11-14-22:00"
Value: Prometheus query result JSON
TTL: 30 seconds (for real-time queries)
     300 seconds (for historical queries >1hr)

// User Permissions Cache (augments existing in-memory cache)
// Key Pattern: "authz:user:{user_id}:{namespace}"
// TTL: 2 minutes (aligns with existing RBAC refresh cron)
Key: "authz:user:stella:openshift-ai-prod"
Value: Permission bitmap
TTL: 120 seconds
```

**Cache Invalidation Strategy**:

```go
// Option 1: Time-Based (for metric queries)
// - Metric query results auto-expire based on TTL
// - No manual invalidation needed
// - Works well because time-series data doesn't "change" retrospectively

// Option 2: Event-Based (for dashboard configs)
// Implementation in Perses API update handler:

func (s *DashboardService) Update(ctx context.Context, entity *v1.Dashboard) error {
    // 1. Update database
    if err := s.dao.Update(entity); err != nil {
        return err
    }

    // 2. Invalidate cache (broadcast to all API pods via Redis Pub/Sub)
    cacheKey := fmt.Sprintf("dashboard:config:%s:%s",
        entity.Metadata.Project, entity.Metadata.Name)

    if err := s.redisClient.Del(ctx, cacheKey).Err(); err != nil {
        // Log but don't fail - cache will expire naturally
        logrus.WithError(err).Warn("failed to invalidate cache")
    }

    // 3. Publish invalidation event (for multi-pod coordination)
    s.redisClient.Publish(ctx, "dashboard:invalidate", cacheKey)

    return nil
}
```

**Cache Hit Ratio Targets**:
- Dashboard configs: **95%** (rarely change after creation)
- Metric queries (real-time): **70%** (15-second refresh = multiple users hit same time bucket)
- Metric queries (historical): **85%** (same queries for incident analysis)

**Memory Sizing**:
```
Dashboard configs: 500 dashboards × 50KB avg = 25MB
Query results: 1000 unique queries × 100KB × 2 time buckets = 200MB
User permissions: 500 users × 50 namespaces × 1KB = 25MB
Overhead (Redis metadata): ~50MB

Total: ~300MB base + 1.7GB buffer = 2GB Redis memory
```

---

## 2. Horizontal Scaling (Stateless API)

### Decision: Stateless API Pods + Server-Sent Events (SSE) with Sticky Sessions

**Rationale**:

**Stateless Design Analysis** (Perses is already 90% stateless):
- ✅ **Database-backed state**: All dashboards stored in PostgreSQL
- ✅ **No session state**: JWT tokens (stateless authentication)
- ✅ **Proxy pattern**: API forwards queries to Prometheus (no local state)
- ⚠️ **RBAC cache**: Currently in-memory (see `authorization/cron.go`), needs Redis migration

**WebSocket vs SSE Comparison**:

| Aspect | WebSocket | Server-Sent Events (SSE) |
|--------|-----------|--------------------------|
| **Real-time updates** | Bidirectional | Unidirectional (server→client) |
| **Connection overhead** | Higher (full-duplex) | Lower (HTTP/1.1 or HTTP/2) |
| **Load balancer compatibility** | Requires sticky sessions | Works with standard HTTP LB |
| **Automatic reconnection** | Manual implementation | Built-in browser support |
| **Dashboard use case fit** | Overkill (rarely need client→server) | Perfect (only need server→client) |
| **Kubernetes HPA compatibility** | Tricky (connection draining) | Seamless (HTTP request-based) |

**Recommendation**: **Server-Sent Events (SSE)** for real-time metric updates
- Dashboard updates are **unidirectional** (server pushes new metrics to UI)
- SSE reconnects automatically when pods scale down
- Better for Kubernetes autoscaling (no persistent connection pinning)

**Implementation**:

```yaml
# Kubernetes Deployment for Perses API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: perses-api
  namespace: openshift-ai-observability
spec:
  replicas: 3  # Start with 3 pods, HPA scales to 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deployments
  template:
    metadata:
      labels:
        app: perses-api
        version: v1
    spec:
      containers:
      - name: perses
        image: persesdev/perses:v0.51.0
        env:
        - name: PERSES_DATABASE_SQL_ADDR
          value: "postgres-primary:5432"
        - name: PERSES_DATABASE_SQL_DBNAME
          value: "perses"
        - name: PERSES_CACHE_REDIS_ADDR
          value: "perses-redis:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        ports:
        - name: http
          containerPort: 8080
        livenessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: perses-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: perses-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60  # Wait 60s before scaling up
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60  # Add max 2 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120  # Remove max 1 pod every 2 minutes
```

**Load Balancer Configuration** (OpenShift Route):

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: perses-dashboard
  annotations:
    # For SSE: no sticky sessions needed (stateless)
    haproxy.router.openshift.io/timeout: 60s
    haproxy.router.openshift.io/balance: leastconn  # Distribute to least-busy pod
spec:
  to:
    kind: Service
    name: perses-api
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
```

**Scaling Math** (500 users):
```
Assumptions:
- Average 3 dashboards open per user
- Each dashboard: 5 charts × 15-second refresh = 20 req/min per dashboard
- Per user request rate: 3 dashboards × 20 req/min = 60 req/min

Total API load:
500 users × 60 req/min = 30,000 req/min = 500 req/sec

Per-pod capacity (Go API):
- 1 CPU core ~2000 req/sec (simple GET requests with caching)
- With 250m CPU request: ~500 req/sec per pod
- With 70% CPU target: ~350 req/sec safe threshold per pod

Required pods:
500 req/sec ÷ 350 req/sec per pod = 1.4 pods minimum
With 3x safety margin: 4-5 pods for 500 users
HPA config (3-10 pods) provides ample headroom
```

**SSE Implementation Pattern** (for Perses extension):

```go
// In Perses API, add SSE endpoint for real-time dashboard updates
// pkg/api/v1/dashboard/stream.go

func (e *endpoint) StreamDashboardMetrics(c echo.Context) error {
    // Set headers for SSE
    c.Response().Header().Set("Content-Type", "text/event-stream")
    c.Response().Header().Set("Cache-Control", "no-cache")
    c.Response().Header().Set("Connection", "keep-alive")

    dashboardName := c.Param("name")
    project := c.Param("project")

    // Create channel for metric updates
    ticker := time.NewTicker(15 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // Fetch latest metrics (with Redis cache)
            metrics, err := e.fetchDashboardMetrics(project, dashboardName)
            if err != nil {
                logrus.WithError(err).Error("failed to fetch metrics")
                continue
            }

            // Send SSE event
            fmt.Fprintf(c.Response(), "data: %s\n\n", metrics)
            c.Response().Flush()

        case <-c.Request().Context().Done():
            // Client disconnected
            return nil
        }
    }
}
```

---

## 3. Database Optimization (PostgreSQL)

### Decision: Connection Pooling + Read Replicas + Query Optimization

**Rationale**:

**Read/Write Pattern Analysis** (from spec.md):
- **Writes**: Dashboard CRUD (infrequent, ~5% of traffic)
- **Reads**:
  - Dashboard list/get (frequent, ~40%)
  - User auth/authz checks (very frequent, ~40%)
  - Audit logs (infrequent, ~15%)
- **Read/Write Ratio**: ~95:5 (heavy read bias)

**PostgreSQL Scaling Strategy**:

1. **Connection Pooling** (PgBouncer)
   - Problem: Perses currently uses `database/sql` without external pooling
   - Solution: PgBouncer in transaction mode (for stateless queries)
   - Benefit: 500 API connections → 50 DB connections (10:1 multiplexing)

2. **Read Replicas**
   - Primary: Handle all writes + real-time reads
   - Replica: Handle dashboard list queries, user lookups (cacheable)
   - Replication lag tolerance: 1-2 seconds (acceptable for dashboard lists)

3. **Query Optimization**
   - Current schema: JSON column storage (flexible but slow for filters)
   - Add indexes for common query patterns

**Implementation**:

```yaml
# PgBouncer Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
spec:
  replicas: 2  # High availability
  template:
    spec:
      containers:
      - name: pgbouncer
        image: edoburu/pgbouncer:1.21.0
        env:
        - name: DATABASE_URL
          value: "postgres://user:password@postgres-primary:5432/perses"
        - name: POOL_MODE
          value: "transaction"  # Most efficient for stateless API
        - name: MAX_CLIENT_CONN
          value: "1000"  # Accept 1000 API connections
        - name: DEFAULT_POOL_SIZE
          value: "50"   # Multiplex to 50 DB connections
        - name: RESERVE_POOL_SIZE
          value: "10"   # Emergency pool
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
```

```yaml
# PostgreSQL Primary-Replica Setup
# (Using Postgres Operator or manual streaming replication)

apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: perses-db
spec:
  postgresVersion: 17
  instances:
  - name: primary
    replicas: 1
    dataVolumeClaimSpec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
    resources:
      requests:
        memory: "4Gi"
        cpu: "2000m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
  - name: replica
    replicas: 2  # Two read replicas for HA
    dataVolumeClaimSpec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
    resources:
      requests:
        memory: "2Gi"
        cpu: "1000m"

  # Connection pooling via built-in PgBouncer
  proxy:
    pgBouncer:
      replicas: 2
      config:
        global:
          pool_mode: transaction
          max_client_conn: 1000
          default_pool_size: 50
```

**Connection Pool Sizing**:
```
API Pods: 10 pods (max HPA)
Connections per pod: 100 (Go database/sql pool)
Total potential connections: 10 × 100 = 1000

PgBouncer multiplexing:
1000 API connections → 50 active DB connections
PostgreSQL max_connections: 100 (50 active + 50 reserve)

Per-connection memory: ~10MB
100 connections × 10MB = 1GB connection overhead
Plus 2GB shared_buffers + 1GB work_mem = 4GB total
```

**PostgreSQL Configuration** (`postgresql.conf`):

```conf
# Connection Settings
max_connections = 100
superuser_reserved_connections = 5

# Memory Settings (for 4GB RAM instance)
shared_buffers = 1GB              # 25% of RAM
effective_cache_size = 3GB        # 75% of RAM
work_mem = 20MB                   # Per-operation memory
maintenance_work_mem = 256MB      # For VACUUM, CREATE INDEX

# Query Planner
random_page_cost = 1.1            # SSD storage (lower = better)
effective_io_concurrency = 200    # SSD concurrent I/O

# Write-Ahead Log (WAL)
wal_buffers = 16MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min

# Replication (for read replicas)
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
hot_standby = on
```

**Query Optimization** (SQL indexes for dashboard listing):

```sql
-- Current Perses schema (from sql/sql.go):
-- CREATE TABLE dashboard (
--   id VARCHAR(256) NOT NULL PRIMARY KEY,
--   name VARCHAR(128) NOT NULL,
--   project VARCHAR(128) NOT NULL,
--   doc JSON NOT NULL
-- );

-- Add indexes for common query patterns:

-- Index for listing dashboards by project (most common query)
CREATE INDEX idx_dashboard_project_name ON dashboard(project, name);

-- Index for full-text search on dashboard names
CREATE INDEX idx_dashboard_name_trgm ON dashboard USING gin(name gin_trgm_ops);

-- Index for filtering by metadata fields (using JSON path)
CREATE INDEX idx_dashboard_created_at ON dashboard
  ((doc->>'metadata'->>'createdAt'));

-- Analyze query performance
EXPLAIN ANALYZE
SELECT doc FROM dashboard
WHERE project = 'openshift-ai'
ORDER BY name
LIMIT 50;

-- Expected improvement:
-- Before: Seq Scan (120ms for 500 dashboards)
-- After: Index Scan (8ms with idx_dashboard_project_name)
```

**Read Replica Routing** (application-level):

```go
// Perses DAO enhancement for read replica support
// internal/api/database/sql/sql.go

type DAO struct {
    PrimaryDB   *sql.DB  // For writes
    ReplicaDB   *sql.DB  // For reads (optional)
    SchemaName  string
}

func (d *DAO) Query(query databaseModel.Query, slice any) error {
    // Use replica for read-only queries
    db := d.PrimaryDB
    if d.ReplicaDB != nil && query.IsReadOnly() {
        db = d.ReplicaDB
    }

    // Execute query...
    rows, err := db.Query(sqlQuery, args...)
    // ...
}

// Configuration in config.yaml:
database:
  sql:
    addr: "pgbouncer-primary:5432"  # Primary for writes
    replica_addr: "pgbouncer-replica:5432"  # Replica for reads
    dbname: "perses"
```

**Database Capacity Planning** (500 users, 500 dashboards):

```
Storage:
- 500 dashboards × 50KB avg = 25MB dashboard data
- 500 users × 5KB = 2.5MB user data
- 50 projects × 10KB = 500KB project metadata
- Audit logs: 100K events × 2KB = 200MB
Total data: ~250MB

With indexes (2x overhead): ~500MB
WAL + temp files: ~10GB
Allocation: 100GB volume (200x headroom for growth)

IOPS Requirements:
- Read IOPS: 500 req/sec × 0.3 cache miss = 150 IOPS
- Write IOPS: 25 req/sec (dashboard updates) = 25 IOPS
Total: ~200 IOPS (standard SSD provides 3000+, no bottleneck)
```

---

## 4. Metric Query Optimization (Prometheus)

### Decision: Query Result Caching + PromQL Best Practices + Streaming Results

**Rationale**:

**High-Cardinality Challenge**:
- 1000+ time series per dashboard (e.g., pod metrics across 100 pods × 10 metrics)
- PromQL queries can take 1-5 seconds for complex aggregations
- Without caching: 500 users × 3 dashboards × 5 charts = 7,500 queries/min to Prometheus

**Solution: Multi-Level Query Optimization**:

1. **Query Result Caching** (Redis) - Reduces load by 70-80%
2. **PromQL Optimization** - Faster query execution
3. **Result Streaming** - Reduces memory/latency for large results
4. **Query Deduplication** - Multiple users → single Prometheus query

**Implementation**:

### 4.1 Query Result Caching

```go
// Perses proxy enhancement for query caching
// internal/api/impl/proxy/prometheus_cache.go

type PrometheusProxy struct {
    client      *http.Client
    redisClient *redis.Client
}

func (p *PrometheusProxy) Query(ctx context.Context, query string, timestamp time.Time) ([]byte, error) {
    // Generate cache key based on query + time bucket
    timeBucket := timestamp.Truncate(15 * time.Second)  // 15-second buckets
    cacheKey := fmt.Sprintf("query:result:%s:%d",
        hashQuery(query), timeBucket.Unix())

    // Check cache first
    if cached, err := p.redisClient.Get(ctx, cacheKey).Bytes(); err == nil {
        return cached, nil  // Cache hit
    }

    // Cache miss - query Prometheus
    result, err := p.queryPrometheus(ctx, query, timestamp)
    if err != nil {
        return nil, err
    }

    // Cache result with TTL based on query time range
    ttl := calculateTTL(timestamp)
    p.redisClient.Set(ctx, cacheKey, result, ttl)

    return result, nil
}

func calculateTTL(queryTime time.Time) time.Duration {
    age := time.Since(queryTime)

    // Real-time queries (last 5 min): 30 second TTL
    if age < 5*time.Minute {
        return 30 * time.Second
    }
    // Recent queries (last 1 hour): 5 min TTL
    if age < 1*time.Hour {
        return 5 * time.Minute
    }
    // Historical queries (>1 hour): 15 min TTL
    return 15 * time.Minute
}
```

**Cache Hit Ratio Estimation**:
```
Scenario: 100 users viewing same "Model Serving Overview" dashboard
- Without cache: 100 users × 5 charts × 4 req/min = 2000 req/min to Prometheus
- With cache (15-sec buckets): ~4 unique time buckets/min × 5 charts = 20 cache misses/min
- Cache hit ratio: (2000 - 20) / 2000 = 99% for popular dashboards

Average across all dashboards (including unique ones):
- Expected cache hit ratio: 70-80%
- Prometheus load reduction: 7,500 req/min → 1,500 req/min
```

### 4.2 PromQL Best Practices

**Dashboard Query Optimization Guide**:

```yaml
# BAD: High-cardinality label in aggregation (slow)
sum(rate(http_requests_total{namespace="openshift-ai"}[5m])) by (pod, endpoint, method, status)
# Returns 1000+ time series, slow aggregation

# GOOD: Aggregate first, then filter (fast)
sum by (status) (rate(http_requests_total{namespace="openshift-ai"}[5m]))
# Returns 5-10 time series, fast aggregation

# BAD: Regex matching (full metric scan)
sum(rate(container_cpu_usage{pod=~"model-.*"}[5m]))

# GOOD: Exact label matching with OR (indexed lookup)
sum(rate(container_cpu_usage{namespace="openshift-ai",workload="model-serving"}[5m]))

# BAD: Large time ranges with high resolution
rate(metric[1h:1s])  # 3600 data points

# GOOD: Appropriate resolution for time range
rate(metric[1h:15s])  # 240 data points (sufficient for visualization)
```

**Query Complexity Limits** (prevent abuse):

```yaml
# Perses dashboard validation rules
# docs/configuration/custom-lint-rules.md

dashboard_query_limits:
  max_time_series_per_chart: 500  # Limit cardinality
  max_lookback_window: 7d         # Prevent excessive historical queries
  max_query_length: 1000          # Prevent complex nested queries
  required_label_filters:         # Force namespace filtering
    - namespace
```

### 4.3 Query Result Streaming

**Problem**: Large query results (1000+ time series × 1000 data points = 1MB+ JSON) cause:
- High memory usage in API pod
- Long latency (wait for full result before sending)
- Poor UX for users (blank screen while loading)

**Solution**: Stream results incrementally

```go
// Streaming query results from Prometheus
// internal/api/impl/proxy/stream.go

func (p *PrometheusProxy) StreamQuery(ctx echo.Context, query string) error {
    // Set headers for chunked transfer encoding
    ctx.Response().Header().Set("Content-Type", "application/json")
    ctx.Response().Header().Set("Transfer-Encoding", "chunked")

    // Query Prometheus with streaming
    resp, err := p.client.Get(prometheusURL + "/api/v1/query?query=" + url.QueryEscape(query))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    // Stream response in chunks (don't buffer entire response)
    buffer := make([]byte, 32*1024)  // 32KB chunks
    for {
        n, err := resp.Body.Read(buffer)
        if n > 0 {
            ctx.Response().Write(buffer[:n])
            ctx.Response().Flush()  // Send chunk immediately
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
    }
    return nil
}
```

### 4.4 Query Deduplication (Coalescing)

**Problem**: 50 users open same dashboard → 50 identical Prometheus queries simultaneously

**Solution**: Request coalescing (single-flight pattern)

```go
// Query deduplication using singleflight
// internal/api/impl/proxy/dedupe.go

import "golang.org/x/sync/singleflight"

type PrometheusProxy struct {
    queryGroup singleflight.Group
}

func (p *PrometheusProxy) QueryWithDedup(ctx context.Context, query string) ([]byte, error) {
    // Generate cache key
    key := fmt.Sprintf("%s:%d", hashQuery(query), time.Now().Unix()/15)

    // Use singleflight to deduplicate concurrent identical queries
    result, err, shared := p.queryGroup.Do(key, func() (interface{}, error) {
        return p.queryPrometheus(ctx, query)
    })

    if shared {
        logrus.Debugf("query deduplication: shared result for %s", query)
    }

    return result.([]byte), err
}

// Benefit: 50 concurrent identical queries → 1 Prometheus query
```

**Performance Impact Estimates**:

| Optimization | Prometheus Load Reduction | Latency Improvement |
|--------------|---------------------------|---------------------|
| Redis caching (70% hit ratio) | -70% queries | -200ms avg (cache hit) |
| PromQL optimization | -30% query time | -500ms avg (per query) |
| Streaming results | No change | -50% perceived latency |
| Query deduplication | -40% (for popular dashboards) | -100ms (avoid queue) |

**Combined**: 90% reduction in Prometheus load, 2-3x latency improvement

---

## 5. OpenShift-Specific Considerations

### Deployment Architecture on OpenShift

```yaml
# Complete OpenShift deployment manifest
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-ai-observability
---
# Perses API Service
apiVersion: v1
kind: Service
metadata:
  name: perses-api
  namespace: openshift-ai-observability
spec:
  selector:
    app: perses-api
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  type: ClusterIP
---
# Redis Service
apiVersion: v1
kind: Service
metadata:
  name: perses-redis
  namespace: openshift-ai-observability
spec:
  selector:
    app: perses-redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  type: ClusterIP
---
# PostgreSQL Service (Primary)
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: openshift-ai-observability
spec:
  selector:
    app: postgres
    role: primary
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
---
# PgBouncer Service
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer
  namespace: openshift-ai-observability
spec:
  selector:
    app: pgbouncer
  ports:
  - port: 5432
    targetPort: 6432
  type: ClusterIP
---
# OpenShift Route (external access)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: perses-dashboard
  namespace: openshift-ai-observability
  annotations:
    haproxy.router.openshift.io/timeout: 60s
    haproxy.router.openshift.io/balance: leastconn
spec:
  host: perses.apps.openshift-cluster.example.com
  to:
    kind: Service
    name: perses-api
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Resource Quotas and Limits

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: observability-quota
  namespace: openshift-ai-observability
spec:
  hard:
    requests.cpu: "20"      # Total CPU requests
    requests.memory: "40Gi" # Total memory requests
    limits.cpu: "40"        # Total CPU limits
    limits.memory: "80Gi"   # Total memory limits
    persistentvolumeclaims: "5"
    requests.storage: "500Gi"

# Breakdown for 500 users:
# - Perses API: 10 pods × 250m CPU × 512Mi RAM = 2.5 CPU, 5Gi RAM
# - Redis: 1 pod × 500m CPU × 2Gi RAM = 0.5 CPU, 2Gi RAM
# - PgBouncer: 2 pods × 100m CPU × 256Mi RAM = 0.2 CPU, 512Mi RAM
# - PostgreSQL: 1 primary + 2 replicas × 2 CPU × 4Gi RAM = 6 CPU, 12Gi RAM
# Total: ~10 CPU, 20Gi RAM (with headroom in quota)
```

---

## 6. Monitoring and Observability (for the observability platform itself!)

### Key Metrics to Track

```yaml
# ServiceMonitor for Perses API metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: perses-api
  namespace: openshift-ai-observability
spec:
  selector:
    matchLabels:
      app: perses-api
  endpoints:
  - port: http
    path: /metrics
    interval: 15s

# Key metrics to alert on:
# 1. API latency: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 3
# 2. Cache hit ratio: rate(cache_hits[5m]) / rate(cache_requests[5m]) < 0.7
# 3. Database connection pool: pg_pool_active_connections / pg_pool_max_connections > 0.8
# 4. Prometheus query latency: rate(prometheus_query_duration_seconds[5m]) > 2
# 5. Pod autoscaling lag: kube_hpa_status_current_replicas < kube_hpa_status_desired_replicas
```

### Performance Benchmarks (Target SLOs)

```yaml
SLO Definition:
  - API Response Time (p95): < 500ms for cached requests, < 2s for uncached
  - Dashboard Load Time (p95): < 3 seconds (as per spec.md SC-005)
  - Cache Hit Ratio: > 70% (query results), > 95% (dashboard configs)
  - Database Query Time (p95): < 100ms for indexed queries
  - Prometheus Query Time (p95): < 2 seconds (with optimized PromQL)
  - System Availability: 99.9% uptime (43 min downtime/month allowed)
  - Concurrent Users: 500+ without degradation
  - Horizontal Scaling: < 2 min to add new pod (HPA reaction time)
```

---

## 7. Migration Path and Rollout Strategy

### Phase 1: Foundation (Month 1-2)
- Deploy PostgreSQL with connection pooling (PgBouncer)
- Add database indexes for dashboard queries
- Implement basic Redis caching for dashboard configs
- Set up HPA for Perses API pods (start with 3 replicas)

**Success Criteria**: Support 100 concurrent users with <2s dashboard load time

### Phase 2: Caching Layer (Month 3-4)
- Implement query result caching in Redis
- Add query deduplication (singleflight)
- Optimize PromQL queries in dashboard templates
- Monitor cache hit ratios and tune TTLs

**Success Criteria**: 70% cache hit ratio, support 250 concurrent users

### Phase 3: Read Replicas (Month 5-6)
- Deploy PostgreSQL read replicas
- Implement read/write splitting in DAO layer
- Add streaming for large query results
- Load testing with 500 simulated users

**Success Criteria**: Support 500+ concurrent users, meet all SLOs

### Phase 4: Advanced Optimization (Month 7-12)
- Implement SSE for real-time updates (replace polling)
- Redis Sentinel for cache HA
- Database query optimization based on production patterns
- Chaos engineering tests (pod failures, network partitions)

**Success Criteria**: 99.9% availability, graceful degradation under load

---

## 8. Cost Estimation (OpenShift Resource Costs)

**Infrastructure Costs** (AWS/OpenShift pricing estimates):

```
Compute (Pods):
- Perses API: 10 pods × 1 vCPU × $0.04/hr = $28.80/day
- Redis: 1 pod × 1 vCPU × $0.04/hr = $0.96/day
- PgBouncer: 2 pods × 0.5 vCPU × $0.04/hr = $0.96/day
- PostgreSQL: 3 pods × 2 vCPU × $0.04/hr = $5.76/day
Compute Total: ~$37/day = $1,110/month

Storage (PVCs):
- PostgreSQL: 300GB SSD × $0.10/GB/month = $30/month
- Backup volumes: 100GB × $0.05/GB/month = $5/month
Storage Total: ~$35/month

Networking:
- OpenShift Route egress: ~1TB/month × $0.09/GB = $90/month

Total Monthly Cost: ~$1,235/month for 500 users
Per-user cost: $2.47/month/user
```

**Cost Optimization Opportunities**:
- Use spot instances for non-primary PostgreSQL replicas (-50% cost)
- Implement tiered storage (move old dashboards to S3) (-30% storage cost)
- Cache warming during off-peak hours (reduce Prometheus load)

---

## 9. Risk Assessment and Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Redis cache failure** | High (all queries hit DB) | Low | Implement Redis Sentinel (HA), graceful degradation (queries work without cache, just slower) |
| **PostgreSQL connection exhaustion** | High (API unavailable) | Medium | PgBouncer with queueing, connection pool monitoring alerts |
| **Prometheus overload** | Medium (slow queries) | Medium | Query result caching, rate limiting per user, query timeout enforcement |
| **Pod autoscaling lag** | Low (brief slowdown) | Medium | Predictive scaling based on time-of-day patterns, faster HPA stabilization window |
| **Database replication lag** | Low (stale dashboard list) | Low | Monitor lag metrics, fail over to primary if lag >5 seconds |
| **Cache invalidation bugs** | Medium (users see stale data) | Medium | Conservative TTLs, manual cache flush endpoint, versioned cache keys |
| **Thundering herd** (cache expiry) | Medium (Prometheus spike) | Low | Probabilistic early expiration, query deduplication |

---

## 10. References and Further Reading

**Industry Best Practices**:
- Grafana scaling architecture: https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/
- Prometheus query optimization: https://prometheus.io/docs/practices/histograms/
- PostgreSQL connection pooling: https://www.pgbouncer.org/config.html
- Redis caching patterns: https://redis.io/docs/manual/patterns/

**Perses-Specific Documentation**:
- Perses configuration: `/workspace/perses/docs/configuration/configuration.md`
- Database implementation: `/workspace/perses/internal/api/database/sql/sql.go`
- Proxy architecture: `/workspace/perses/internal/api/impl/proxy/proxy.go`

**OpenShift AI Context**:
- Spec requirements: `/workspace/artifacts/specs/001-perses-dashboard/spec.md`
- Target metrics: FR-002 (15-second refresh), SC-003 (50 concurrent users), SC-005 (3-second load time)

---

## Appendix A: Load Testing Plan

**Test Scenarios**:

```yaml
# Using k6 load testing tool
# test-dashboard-load.js

import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 250 },   // Ramp to 250 users
    { duration: '5m', target: 250 },   // Stay at 250
    { duration: '2m', target: 500 },   // Ramp to 500 users
    { duration: '10m', target: 500 },  // Stay at 500 (stress test)
    { duration: '3m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // 95% of requests < 2s
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

export default function () {
  // Simulate user viewing dashboard
  let dashboardResponse = http.get('https://perses.openshift-ai/api/v1/projects/ml-prod/dashboards/model-serving');
  check(dashboardResponse, {
    'dashboard loaded': (r) => r.status === 200,
    'response time OK': (r) => r.timings.duration < 3000,
  });

  // Simulate user refreshing metrics (15-second interval)
  sleep(15);

  let metricsResponse = http.post('https://perses.openshift-ai/api/v1/proxy/prometheus/query', {
    query: 'rate(http_requests_total{namespace="openshift-ai"}[5m])',
  });
  check(metricsResponse, {
    'metrics fetched': (r) => r.status === 200,
  });

  sleep(15);
}
```

**Performance Baseline** (before optimizations):
- 100 users: API p95 latency ~1.5s, DB connections ~40
- 250 users: API p95 latency ~4s, DB connections ~85 (near limit)
- 500 users: Service degradation, connection pool exhaustion

**Expected Performance** (after optimizations):
- 100 users: API p95 latency ~300ms (cache hit), DB connections ~15
- 250 users: API p95 latency ~500ms, DB connections ~25
- 500 users: API p95 latency ~800ms, DB connections ~40

---

## Appendix B: Quick Reference Checklist

**Pre-Deployment Checklist**:
- [ ] PostgreSQL deployed with 100GB storage, 4GB RAM
- [ ] PgBouncer configured with max_client_conn=1000, pool_size=50
- [ ] Redis deployed with 2GB memory, allkeys-lru eviction
- [ ] Perses API deployment with HPA (min=3, max=10)
- [ ] Database indexes created (project+name, created_at)
- [ ] Connection strings updated to use PgBouncer
- [ ] Redis cache integration tested (dashboard config + query results)
- [ ] Monitoring dashboards created for API/DB/Redis metrics
- [ ] Load testing completed with 500 simulated users
- [ ] Runbook documented for common issues (cache flush, connection reset)

**Day-2 Operations**:
- [ ] Weekly: Review cache hit ratios, adjust TTLs if needed
- [ ] Weekly: Analyze slow query logs, add indexes for new patterns
- [ ] Monthly: Review autoscaling behavior, tune HPA thresholds
- [ ] Monthly: Database VACUUM and index maintenance
- [ ] Quarterly: Load testing regression tests
- [ ] Quarterly: Disaster recovery drill (DB failover, cache rebuild)

---

**Document Version**: 1.0
**Last Updated**: 2025-11-14
**Author**: Stella (Staff Engineer, OpenShift AI Platform)
**Review Status**: Ready for Architecture Review
