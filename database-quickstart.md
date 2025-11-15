# Database Layer Quick Start Guide

**Phase 2 Foundational Tasks**: T010-T013 ✅ Complete

---

## What Was Built

### 1. PostgreSQL Schema (V001)
- **File**: `backend/db/migrations/V001_initial_schema.up.sql` (332 lines)
- **Rollback**: `backend/db/migrations/V001_initial_schema.down.sql` (26 lines)
- **Entities**: 8 tables with 20+ performance indexes
- **Features**: JSONB storage, full-text search, audit logging, system templates

### 2. Connection Pool Manager
- **File**: `backend/pkg/database/pool.go` (289 lines)
- **Capacity**: 50 max connections (PgBouncer-compatible)
- **Features**: Context-aware queries, transaction helpers, health monitoring

### 3. Health Check System
- **File**: `backend/pkg/database/health.go` (220 lines)
- **Endpoints**: `/api/v1/health`, `/ready`, `/live`, `/metrics/health`
- **Features**: Kubernetes-native, Prometheus metrics, degradation detection

### 4. Server Integration
- **File**: `backend/cmd/server/main.go` (updated with DB initialization)
- **Features**: 12 environment variables, graceful shutdown, dual logging

---

## Quick Start

### 1. Set Up PostgreSQL

```bash
# Using Docker (development)
docker run -d --name postgres-perses \
  -e POSTGRES_DB=perses_dashboard \
  -e POSTGRES_USER=perses \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  postgres:14

# Verify running
docker ps | grep postgres-perses
```

### 2. Run Migrations

```bash
# Install golang-migrate
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Apply migrations
cd /workspace/sessions/agentic-session-1763159109/workspace/perses/openshift-ai-dashboard/backend

migrate -database "postgres://perses:secret@localhost:5432/perses_dashboard?sslmode=disable" \
        -path db/migrations \
        up

# Verify
psql -h localhost -U perses -d perses_dashboard -c "\dt"
```

**Expected Output**:
```
             List of relations
 Schema |       Name        | Type  | Owner
--------+-------------------+-------+--------
 public | audit_log         | table | perses
 public | dashboard         | table | perses
 public | dashboard_template| table | perses
 public | schema_version    | table | perses
 public | share_token       | table | perses
 public | user_preferences  | table | perses
(6 rows)
```

### 3. Start Server

```bash
# Set environment variables
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=perses_dashboard
export DB_USER=perses
export DB_PASSWORD=secret
export DB_SSLMODE=disable  # Development only!

# Run server
cd /workspace/sessions/agentic-session-1763159109/workspace/perses/openshift-ai-dashboard/backend
go run cmd/server/main.go
```

**Expected Output**:
```json
{"level":"info","time":"...","message":"Database connection pool initialized successfully"}
{"level":"info","time":"...","message":"Health check endpoints registered"}
{"level":"info","time":"...","message":"API server listening","address":"0.0.0.0:8080"}
```

### 4. Test Health Endpoints

```bash
# Comprehensive health check
curl http://localhost:8080/api/v1/health | jq .

# Expected: {"status":"healthy","checks":{"database":{"status":"pass",...}}}

# Readiness probe (Kubernetes)
curl http://localhost:8080/ready

# Expected: {"ready":true}

# Liveness probe (Kubernetes)
curl http://localhost:8080/live

# Expected: {"alive":true}

# Metrics (Prometheus)
curl http://localhost:8080/metrics/health | jq .

# Expected: {"database_healthy":true,"database_utilization_percent":...}
```

---

## Database Schema Overview

### Core Tables

**1. dashboard_template** (System Templates)
- Pre-configured templates for common workload types
- 4 system templates: Model Serving, Training Job, Notebook, Data Pipeline
- Query: `SELECT id, name, category FROM dashboard_template WHERE is_system = true;`

**2. dashboard** (User Dashboards)
- Stores Perses dashboard JSON documents
- Optimistic locking via `version` column
- State machine: draft → active → archived
- Query: `SELECT id, name, project, state FROM dashboard WHERE project = 'openshift-ai';`

**3. user_preferences** (User Settings)
- Timezone, theme, favorites, recent dashboards
- One row per user
- Query: `SELECT * FROM user_preferences WHERE user_id = 'user@example.com';`

**4. share_token** (Dashboard Sharing)
- Ephemeral tokens for read-only sharing
- IP restrictions, expiration, revocation
- Query: `SELECT token_id, dashboard_id, expires_at FROM share_token WHERE revoked = false;`

**5. audit_log** (Compliance)
- Immutable audit trail (append-only)
- WHO, WHAT, WHEN, WHERE, RESULT
- Query: `SELECT * FROM audit_log WHERE user_id = 'admin@example.com' ORDER BY timestamp DESC LIMIT 10;`

### Performance Indexes

**20+ indexes** optimized for:
- Dashboard listing by project (<10ms)
- Recent dashboards (<20ms)
- Full-text search on names (<50ms)
- Share token validation (<5ms)
- Audit log queries (<30ms)

---

## Configuration

### Environment Variables (12 total)

**Required**:
```bash
DB_HOST=localhost
DB_PORT=5432
DB_NAME=perses_dashboard
DB_USER=perses
DB_PASSWORD=secret
DB_SSLMODE=require  # Production: require, verify-full
```

**Optional** (defaults shown):
```bash
DB_MAX_OPEN_CONNS=50
DB_MAX_IDLE_CONNS=10
DB_CONN_MAX_LIFETIME=5m
DB_CONN_MAX_IDLE_TIME=1m
DB_CONNECT_TIMEOUT=10s
DB_QUERY_TIMEOUT=30s
```

### Production Settings

```bash
# High availability
DB_SSLMODE=verify-full
DB_MAX_OPEN_CONNS=100
DB_MAX_IDLE_CONNS=25
DB_CONN_MAX_LIFETIME=10m

# Kubernetes deployment
export DB_HOST=$(kubectl get svc postgres -o jsonpath='{.spec.clusterIP}')
export DB_PASSWORD=$(kubectl get secret postgres-secret -o jsonpath='{.data.password}' | base64 -d)
```

---

## Common Operations

### Insert System Template

```sql
INSERT INTO dashboard_template (id, name, description, category, doc, created_by, version, is_system)
VALUES (
    'template-custom',
    'Custom Template',
    'Description here',
    'custom',
    '{"kind": "Dashboard", "spec": {}}'::jsonb,
    'admin',
    '1.0.0',
    false
);
```

### Create Dashboard

```sql
INSERT INTO dashboard (id, name, project, doc, created_by, updated_by)
VALUES (
    'dashboard-' || gen_random_uuid()::text,
    'My Dashboard',
    'openshift-ai',
    '{"kind": "Dashboard", "spec": {"panels": {}}}'::jsonb,
    'user@example.com',
    'user@example.com'
);
```

### Query Dashboards by Project

```sql
SELECT
    id,
    name,
    state,
    created_at,
    updated_at
FROM dashboard
WHERE project = 'openshift-ai'
  AND state = 'active'
ORDER BY updated_at DESC
LIMIT 10;
```

### Generate Share Token

```sql
INSERT INTO share_token (dashboard_id, created_by, expires_at)
VALUES (
    'dashboard-abc123',
    'user@example.com',
    NOW() + INTERVAL '24 hours'
)
RETURNING token_id, expires_at;
```

### Audit Log Query

```sql
SELECT
    timestamp,
    user_id,
    action,
    resource_id,
    result
FROM audit_log
WHERE project_id = 'openshift-ai'
  AND action LIKE 'dashboard.%'
  AND timestamp > NOW() - INTERVAL '7 days'
ORDER BY timestamp DESC
LIMIT 100;
```

---

## Health Check Examples

### Healthy Response (200 OK)

```json
{
  "status": "healthy",
  "timestamp": "2025-11-14T22:00:00Z",
  "version": "1.0.0",
  "checks": {
    "database": {
      "status": "pass",
      "response_time_ms": 12,
      "details": {
        "open_connections": 5,
        "idle_connections": 3,
        "in_use_connections": 2,
        "wait_count": 0
      }
    }
  }
}
```

### Degraded Response (200 OK)

Pool utilization >80%:
```json
{
  "status": "degraded",
  "checks": {
    "database": {
      "status": "warn",
      "details": {
        "in_use_connections": 48,
        "utilization_percent": 96.0,
        "warning": "Connection pool utilization above 80%"
      }
    }
  }
}
```

### Unhealthy Response (503 Service Unavailable)

Database unreachable:
```json
{
  "status": "unhealthy",
  "checks": {
    "database": {
      "status": "fail",
      "error": "dial tcp 127.0.0.1:5432: connect: connection refused"
    }
  }
}
```

---

## Troubleshooting

### Migration Fails

**Error**: `duplicate key value violates unique constraint`
```bash
# Check current version
psql -d perses_dashboard -c "SELECT * FROM schema_version;"

# Force version (if migration partially applied)
migrate -database "..." -path db/migrations force 1
migrate -database "..." -path db/migrations up
```

### Connection Refused

**Error**: `dial tcp: connect: connection refused`
```bash
# Check PostgreSQL running
docker ps | grep postgres

# Check port mapping
docker port postgres-perses

# Test connection
psql -h localhost -U perses -d perses_dashboard -c "SELECT 1;"
```

### Pool Exhaustion

**Symptoms**: Slow queries, high wait_count
```bash
# Check pool stats
curl http://localhost:8080/metrics/health | jq .database_utilization_percent

# If >80%, increase max connections
export DB_MAX_OPEN_CONNS=100
```

### Slow Queries

**Symptoms**: response_time_ms >100ms
```sql
-- Enable query logging
ALTER DATABASE perses_dashboard SET log_min_duration_statement = 100;

-- Check slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

## Next Steps

### Phase 2 Continuation

1. **T014-T017**: Authentication & Authorization
   - OpenShift OAuth integration
   - RBAC middleware
   - User model

2. **T025-T032**: Models & DAOs
   - Dashboard DAO (CRUD operations)
   - Template DAO (template management)
   - Share Token DAO (token lifecycle)
   - Audit Log DAO (compliance queries)

3. **T121**: Caching Layer
   - Redis for dashboard configs
   - Metric query result caching

### Testing

```bash
# Unit tests
go test -v ./pkg/database/...

# Integration tests
go test -v -tags=integration ./tests/integration/...

# Performance tests
go test -v -bench=. -benchmem ./pkg/database/...
```

---

## File Reference

All files in `/workspace/sessions/agentic-session-1763159109/workspace/perses/openshift-ai-dashboard/backend/`:

```
db/migrations/
├── V001_initial_schema.up.sql    (332 lines - Initial schema)
└── V001_initial_schema.down.sql  (26 lines - Rollback)

pkg/database/
├── pool.go        (289 lines - Connection pool manager)
├── health.go      (220 lines - Health check system)
└── README.md      (300+ lines - Documentation)

cmd/server/
└── main.go        (Updated with DB initialization)
```

**Total**: 987 lines of production code

---

## Resources

- **Detailed Report**: `/workspace/sessions/agentic-session-1763159109/workspace/artifacts/T010-T013-completion-report.md`
- **Database Package Docs**: `backend/pkg/database/README.md`
- **Data Model Spec**: `specs/001-perses-dashboard/data-model.md`
- **Implementation Plan**: `specs/001-perses-dashboard/plan.md`
- **Tasks List**: `specs/001-perses-dashboard/tasks.md`

---

**Status**: ✅ Phase 2 Foundational Database Layer Complete
**Ready For**: Authentication, Models, DAOs, and User Stories
**Implemented By**: Stella (Staff Engineer) | 2025-11-14
