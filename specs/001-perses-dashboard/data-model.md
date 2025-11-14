# Data Model: Perses Observability Dashboard

**Feature**: Perses Observability Dashboard for OpenShift AI
**Phase**: Phase 1 - Data Model Design
**Date**: 2025-11-14
**Author**: Stella (Staff Engineer)
**Status**: Ready for Review

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Entity Definitions](#entity-definitions)
3. [Database Schema](#database-schema)
4. [Entity Relationships](#entity-relationships)
5. [Validation Rules](#validation-rules)
6. [Indexes and Query Optimization](#indexes-and-query-optimization)
7. [Migration Strategy](#migration-strategy)
8. [Data Retention and Archival](#data-retention-and-archival)

---

## Executive Summary

This data model supports the Perses Observability Dashboard feature for OpenShift AI, designed to handle **500+ concurrent users** and **500+ custom dashboards** while maintaining **<3s p95 load times** and **99.9% persistence reliability** (SC-007).

**Key Design Principles**:
- **Perses Compatibility**: Extends existing Perses schema with OpenShift AI-specific entities
- **Performance-First**: Optimized indexes for dashboard listing, metric queries, and RBAC checks
- **Security-Embedded**: RBAC, audit logging, and share token management at schema level
- **Scalability-Ready**: Read replica support, caching layer integration, partitioning for audit logs

**Database Technology**: PostgreSQL 17+ (required for Perses, supports JSON columns, excellent indexing)

**Schema Versioning**: Follows Perses migration pattern with version tracking table

---

## Entity Definitions

### 1. Dashboard

**Description**: Core entity representing a collection of visualization panels organized on a canvas. Extends Perses' native Dashboard model with OpenShift AI-specific metadata.

**Ownership**: Belongs to a project/namespace with RBAC-based access control

**State Lifecycle**: Draft → Active → Archived

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(256) | PK, NOT NULL | UUID v4 format (e.g., `dashboard-abc123`) |
| `name` | VARCHAR(128) | NOT NULL, Unique per project | Human-readable name (e.g., "Model Serving Overview") |
| `project` | VARCHAR(128) | NOT NULL, FK to Project | Namespace/project identifier |
| `doc` | JSONB | NOT NULL | Full Perses dashboard definition (panels, layout, datasources) |
| `created_at` | TIMESTAMP | NOT NULL, Default NOW() | Creation timestamp (UTC) |
| `created_by` | VARCHAR(256) | NOT NULL | User identity (OpenShift username/email) |
| `updated_at` | TIMESTAMP | NOT NULL, Default NOW() | Last modification timestamp (UTC) |
| `updated_by` | VARCHAR(256) | NOT NULL | User who last modified dashboard |
| `version` | INTEGER | NOT NULL, Default 1 | Version number for optimistic locking |
| `state` | VARCHAR(32) | NOT NULL, Default 'active' | Lifecycle state: 'draft', 'active', 'archived' |
| `template_id` | VARCHAR(256) | FK to DashboardTemplate, Nullable | If created from template, reference to template |
| `refresh_interval` | INTEGER | Default 15 | Metric refresh interval in seconds (5-300) |
| `time_range_default` | VARCHAR(64) | Default 'last_1h' | Default time range: 'last_15m', 'last_1h', 'last_24h', 'last_7d' |

**Validation Rules** (see [Validation Rules](#validation-rules) section):
- FR-005: User must have `Dashboard:create` permission in project
- FR-006: Dashboard persisted across sessions (99.9% reliability SLA)
- FR-015: Dashboard layout must be responsive (validated at application layer)

**JSON Document Structure** (`doc` field):
```json
{
  "kind": "Dashboard",
  "metadata": {
    "name": "model-serving-overview",
    "project": "openshift-ai",
    "createdAt": "2025-11-14T22:00:00Z",
    "version": 1
  },
  "spec": {
    "display": {
      "name": "Model Serving Overview",
      "description": "Real-time metrics for deployed models"
    },
    "datasources": {
      "prometheus": {
        "kind": "PrometheusDatasource",
        "spec": {
          "directUrl": "https://thanos-querier.openshift-monitoring.svc:9091"
        }
      }
    },
    "variables": [],
    "panels": {
      "panel-1": {
        "kind": "Panel",
        "spec": {
          "display": {
            "name": "Inference Latency (p95)"
          },
          "plugin": {
            "kind": "TimeSeriesChart",
            "spec": {
              "queries": [
                {
                  "kind": "PrometheusTimeSeriesQuery",
                  "spec": {
                    "query": "histogram_quantile(0.95, rate(model_inference_duration_seconds_bucket[5m]))"
                  }
                }
              ]
            }
          }
        }
      }
    },
    "layouts": [
      {
        "kind": "Grid",
        "spec": {
          "items": [
            {
              "x": 0, "y": 0, "width": 12, "height": 6,
              "content": {"$ref": "#/spec/panels/panel-1"}
            }
          ]
        }
      }
    ]
  }
}
```

**Relationships**:
- One-to-many with `ChartPanel` (embedded in `doc.spec.panels`)
- Many-to-one with `DashboardTemplate` (optional)
- One-to-many with `ShareToken`
- One-to-many with `AuditLogEntry`

---

### 2. Chart Panel

**Description**: Individual visualization component within a dashboard. Stored as part of Dashboard's JSON document (Perses native pattern), but extracted here for clarity.

**Storage**: Embedded in `Dashboard.doc.spec.panels` (not a separate table)

**Fields** (within JSON):

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | String | Unique within dashboard | Panel identifier (e.g., "panel-1") |
| `kind` | String | Enum: Panel | Perses resource type |
| `spec.display.name` | String | Required | User-visible panel title |
| `spec.plugin.kind` | String | Required | Chart type: TimeSeriesChart, GaugeChart, StatPanel, Table, BarChart |
| `spec.plugin.spec.queries` | Array | Required | Array of metric query definitions |
| `spec.plugin.spec.visual` | Object | Optional | Visualization properties (colors, thresholds, axes) |

**Validation Rules**:
- FR-004: Chart type must be one of: time series, area, bar, stat, gauge, table
- FR-018: Visualization properties (colors, axes, thresholds) validated per chart type

**Supported Chart Types** (FR-004):
```yaml
TimeSeriesChart:     # Line/area graphs for time-series data
  - line styles: solid, dashed, dotted
  - axes: linear, logarithmic
  - thresholds: warning/critical overlays

GaugeChart:          # Single-value gauge with min/max/thresholds
  - min/max: configurable
  - thresholds: color zones (green/yellow/red)

StatPanel:           # Single numeric value with trend
  - value: current metric value
  - sparkline: mini time-series trend

TableView:           # Tabular metric display
  - columns: metric labels + values
  - sorting: ascending/descending

BarChart:            # Bar chart for comparative metrics
  - orientation: horizontal/vertical
  - stacking: none/normal/percent
```

---

### 3. Metric Query

**Description**: Definition of time-series data to retrieve from Prometheus/Thanos. Embedded in Chart Panel specification.

**Storage**: Embedded in `Dashboard.doc.spec.panels[*].spec.plugin.spec.queries` (not a separate table)

**Fields** (within JSON):

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `kind` | String | Required | Query type: "PrometheusTimeSeriesQuery" |
| `spec.datasource` | Object | Required | Reference to datasource (default: Prometheus) |
| `spec.query` | String | Required, Max 1000 chars | PromQL query string |
| `spec.minStep` | String | Optional | Minimum query resolution (e.g., "15s") |
| `spec.legend` | String | Optional | Legend template for series labels |

**Validation Rules**:
- FR-007: Query must include namespace filter (enforced at API layer for security)
- FR-011: Query supports aggregations: sum, avg, min, max, percentiles
- Scalability: Query complexity limits (max 500 time series per chart)

**Example Queries**:
```promql
# Real-time inference latency (p95)
histogram_quantile(0.95, rate(model_inference_duration_seconds_bucket{namespace="openshift-ai"}[5m]))

# CPU utilization aggregated by workload
sum by (workload) (rate(container_cpu_usage_seconds_total{namespace="openshift-ai"}[5m]))

# Request rate with labels
sum(rate(http_requests_total{namespace="openshift-ai",workload="model-serving"}[5m])) by (status)
```

---

### 4. Dashboard Template

**Description**: Pre-configured dashboard definition for common OpenShift AI workload types. Users instantiate templates to create new dashboards with baseline configurations.

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(256) | PK, NOT NULL | Template identifier (e.g., "template-model-serving") |
| `name` | VARCHAR(128) | NOT NULL, Unique | Template name (e.g., "Model Serving Dashboard") |
| `description` | TEXT | NOT NULL | Template description and use case |
| `category` | VARCHAR(64) | NOT NULL | Category: "model_serving", "training_job", "notebook", "data_pipeline" |
| `doc` | JSONB | NOT NULL | Template dashboard definition (same structure as Dashboard.doc) |
| `variables` | JSONB | Optional | Template variables for instantiation (e.g., namespace, workload_name) |
| `created_at` | TIMESTAMP | NOT NULL, Default NOW() | Template creation timestamp |
| `created_by` | VARCHAR(256) | NOT NULL | Template author (system or user) |
| `version` | VARCHAR(32) | NOT NULL | Template version (semver: "1.0.0") |
| `is_system` | BOOLEAN | NOT NULL, Default false | System-provided template (cannot be deleted by users) |
| `usage_count` | INTEGER | NOT NULL, Default 0 | Number of dashboards created from this template |

**Validation Rules**:
- FR-003: System templates for model serving, training jobs, notebook servers, data pipelines
- SC-009: 90% of users successfully locate and use templates on first attempt

**System Templates** (FR-003):

1. **Model Serving Dashboard**
   - Metrics: Inference latency (p50/p95/p99), request rate, error rate, CPU/memory utilization
   - Panels: 6 charts (time series + gauges)
   - Variables: `namespace`, `model_name`

2. **Training Job Dashboard**
   - Metrics: Training loss/accuracy, GPU utilization, data throughput, epoch duration
   - Panels: 8 charts (time series + stat panels)
   - Variables: `namespace`, `job_name`

3. **Notebook Server Dashboard**
   - Metrics: CPU/memory utilization, kernel status, I/O operations
   - Panels: 5 charts (gauges + time series)
   - Variables: `namespace`, `notebook_name`

4. **Data Pipeline Dashboard**
   - Metrics: Pipeline throughput, stage latency, error rate, data volume processed
   - Panels: 7 charts (time series + tables)
   - Variables: `namespace`, `pipeline_name`

**Template Instantiation Flow**:
```
1. User selects template from gallery
2. System presents variable input form (namespace, workload name)
3. User fills in variables + provides dashboard name
4. System substitutes variables into template.doc
5. New Dashboard created with template_id reference
6. usage_count incremented on template
```

**Relationships**:
- One-to-many with `Dashboard` (dashboards created from template)

---

### 5. Time Range

**Description**: Temporal window for metric data visualization. Stored as part of dashboard configuration or user preferences.

**Storage**: Embedded in `Dashboard.doc` or `UserPreferences` (not a separate table)

**Types**:

**Relative Time Ranges** (most common, FR-008):
```yaml
last_5m:     "Last 5 minutes"
last_15m:    "Last 15 minutes"
last_1h:     "Last 1 hour"
last_6h:     "Last 6 hours"
last_24h:    "Last 24 hours"
last_7d:     "Last 7 days"
last_30d:    "Last 30 days"
```

**Absolute Time Ranges** (FR-008):
```json
{
  "type": "absolute",
  "start": "2025-11-14T00:00:00Z",
  "end": "2025-11-14T23:59:59Z"
}
```

**Validation Rules**:
- FR-008: Time range selector supports relative (last 15m to last 7d) and absolute ranges
- Maximum lookback: 30 days (configurable per deployment)
- Time range affects query caching TTL (see scalability research)

---

### 6. User Preferences

**Description**: User-specific settings for dashboard interaction and personalization.

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `user_id` | VARCHAR(256) | PK, NOT NULL | OpenShift user identity (username or email) |
| `default_time_range` | VARCHAR(64) | Default 'last_1h' | Preferred default time range |
| `default_refresh_interval` | INTEGER | Default 15 | Preferred refresh interval (5-300 seconds) |
| `timezone` | VARCHAR(64) | Default 'UTC' | User timezone (IANA format: "America/New_York") |
| `favorite_dashboards` | JSONB | Default '[]' | Array of favorited dashboard IDs |
| `recent_dashboards` | JSONB | Default '[]' | Array of recently viewed dashboard IDs (max 10) |
| `theme` | VARCHAR(32) | Default 'light' | UI theme: 'light', 'dark', 'auto' |
| `created_at` | TIMESTAMP | NOT NULL, Default NOW() | Preferences created timestamp |
| `updated_at` | TIMESTAMP | NOT NULL, Default NOW() | Last updated timestamp |

**Validation Rules**:
- Timezone must be valid IANA timezone identifier
- Refresh interval between 5-300 seconds (FR-002)
- favorite_dashboards limited to 50 entries (performance)
- recent_dashboards limited to 10 entries (LRU eviction)

**Example JSON Fields**:
```json
{
  "favorite_dashboards": [
    "dashboard-abc123",
    "dashboard-def456"
  ],
  "recent_dashboards": [
    {"id": "dashboard-xyz789", "viewed_at": "2025-11-14T22:00:00Z"},
    {"id": "dashboard-abc123", "viewed_at": "2025-11-14T21:45:00Z"}
  ]
}
```

**Relationships**:
- One-to-one with OpenShift User (external, not stored in our DB)

---

### 7. Share Token

**Description**: Ephemeral URL-based token for read-only dashboard sharing without authentication. Based on research findings (FR-020).

**Security Model**: Hybrid RBAC + optional tokens (see sharing research)

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `token_id` | UUID | PK, NOT NULL | UUID v4 (not sequential for security) |
| `dashboard_id` | VARCHAR(256) | NOT NULL, FK to Dashboard | Dashboard being shared |
| `created_by` | VARCHAR(256) | NOT NULL | User who generated token (must have Dashboard:share permission) |
| `created_at` | TIMESTAMP | NOT NULL, Default NOW() | Token creation timestamp |
| `expires_at` | TIMESTAMP | NOT NULL | Token expiration timestamp (24h default, 7d max) |
| `permissions` | VARCHAR(32)[] | NOT NULL, Default '{"read"}' | Granted permissions (MVP: read-only) |
| `ip_restrictions` | CIDR[] | Optional | IP address whitelist (CIDR notation) |
| `revoked` | BOOLEAN | NOT NULL, Default false | Token revocation status |
| `revoked_at` | TIMESTAMP | Nullable | Revocation timestamp |
| `revoked_by` | VARCHAR(256) | Nullable | User who revoked token |
| `usage_count` | INTEGER | NOT NULL, Default 0 | Number of times token was used |
| `last_used_at` | TIMESTAMP | Nullable | Last usage timestamp |
| `last_used_ip` | INET | Nullable | IP address of last usage |

**Validation Rules**:
- FR-020: Share tokens enable read-only access without authentication
- SC-010: Zero security violations (ephemeral tokens, short TTL, IP restrictions)
- Token TTL: 24h default, 7d maximum (configurable)
- Permissions: MVP limited to ["read"] only
- IP restrictions: Optional CIDR whitelist (e.g., "10.0.0.0/8", "203.0.113.0/24")

**Security Controls**:
- Token generation requires `Dashboard:share` permission (admin-gated)
- UUID v4 tokens (not sequential, prevents enumeration)
- Rate limiting: 100 validations/min/IP (prevent brute-force)
- Automatic cleanup: Expired tokens deleted after 30 days

**Token Lifecycle**:
```
Created → Active (until expires_at) → Expired
              ↓ (revoked=true)
            Revoked
```

**Relationships**:
- Many-to-one with `Dashboard`
- One-to-many with `AuditLogEntry` (token usage tracked)

---

### 8. Audit Log Entry

**Description**: Immutable audit trail for dashboard operations and access. Required for compliance (SOC2, ISO 27001, HIPAA) and security (SC-010).

**Retention**: 90 days default (configurable per compliance requirements)

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | BIGSERIAL | PK, NOT NULL | Auto-incrementing audit log ID |
| `timestamp` | TIMESTAMP | NOT NULL, Default NOW() | Event timestamp (UTC, millisecond precision) |
| `user_id` | VARCHAR(256) | NOT NULL | User identity (OpenShift username/email, "anonymous" for tokens) |
| `action` | VARCHAR(64) | NOT NULL | Action type (see Action Types below) |
| `resource_type` | VARCHAR(64) | NOT NULL | Resource type: "Dashboard", "DashboardTemplate", "ShareToken" |
| `resource_id` | VARCHAR(256) | NOT NULL | Resource identifier (dashboard ID, template ID, token ID) |
| `project_id` | VARCHAR(128) | NOT NULL | Namespace/project context |
| `result` | VARCHAR(32) | NOT NULL | Result: "success", "denied", "error" |
| `client_ip` | INET | NOT NULL | Client IP address |
| `user_agent` | TEXT | Optional | Client user agent string |
| `access_method` | VARCHAR(32) | NOT NULL | Access method: "rbac" (authenticated), "share_token" (anonymous) |
| `token_id` | UUID | FK to ShareToken, Nullable | If accessed via share token, token reference |
| `metadata` | JSONB | Optional | Additional context (error details, changed fields, etc.) |

**Action Types**:
```yaml
# Dashboard Actions
dashboard.create:        User creates new dashboard
dashboard.read:          User views dashboard
dashboard.update:        User modifies dashboard
dashboard.delete:        User deletes dashboard
dashboard.export:        User exports dashboard config (FR-012)

# Sharing Actions
dashboard.share:         User generates share token
dashboard.share.revoke:  User/admin revokes share token
dashboard.access.token:  Anonymous user accesses via share token

# Template Actions
template.create:         User creates custom template
template.instantiate:    User creates dashboard from template
```

**Validation Rules**:
- All dashboard CRUD operations logged (FR-013: audit logging)
- SC-010: Zero security violations (all access tracked)
- Immutable: No UPDATE or DELETE operations allowed (append-only)
- Indexed for fast queries (see Indexes section)

**Compliance Requirements**:
```yaml
SOC2 (TSC CC7.2):
  - Who: user_id
  - What: action
  - When: timestamp
  - Where: resource_id, project_id
  - Result: result (success/denied/error)

ISO 27001:
  - Access logging: dashboard.read events
  - Change logging: dashboard.update events
  - Retention: 90 days minimum

HIPAA:
  - PHI access: All dashboard access logged
  - Retention: 7 years (if PHI data displayed)
```

**Example Audit Events**:
```json
{
  "id": 12345,
  "timestamp": "2025-11-14T22:00:00.123Z",
  "user_id": "stella@redhat.com",
  "action": "dashboard.share",
  "resource_type": "Dashboard",
  "resource_id": "dashboard-abc123",
  "project_id": "openshift-ai",
  "result": "success",
  "client_ip": "10.0.1.42",
  "user_agent": "Mozilla/5.0...",
  "access_method": "rbac",
  "token_id": null,
  "metadata": {
    "token_ttl": "24h",
    "ip_restrictions": ["10.0.0.0/8"]
  }
}
```

**Relationships**:
- Many-to-one with `Dashboard`
- Many-to-one with `ShareToken` (optional)

---

## Database Schema

### PostgreSQL CREATE TABLE Statements

```sql
-- ============================================================================
-- Perses Observability Dashboard - PostgreSQL Schema
-- Version: 1.0.0
-- Database: PostgreSQL 17+
-- Author: Stella (Staff Engineer)
-- Date: 2025-11-14
-- ============================================================================

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- Full-text search (dashboard names)
CREATE EXTENSION IF NOT EXISTS "btree_gin";      -- GIN indexes for JSONB

-- ============================================================================
-- 1. Dashboard
-- ============================================================================

CREATE TABLE dashboard (
    id VARCHAR(256) NOT NULL,
    name VARCHAR(128) NOT NULL,
    project VARCHAR(128) NOT NULL,
    doc JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(256) NOT NULL,
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_by VARCHAR(256) NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    state VARCHAR(32) NOT NULL DEFAULT 'active',
    template_id VARCHAR(256),
    refresh_interval INTEGER DEFAULT 15,
    time_range_default VARCHAR(64) DEFAULT 'last_1h',

    -- Primary Key
    CONSTRAINT pk_dashboard PRIMARY KEY (id),

    -- Unique Constraints
    CONSTRAINT uq_dashboard_project_name UNIQUE (project, name),

    -- Check Constraints
    CONSTRAINT ck_dashboard_state CHECK (state IN ('draft', 'active', 'archived')),
    CONSTRAINT ck_dashboard_refresh_interval CHECK (refresh_interval BETWEEN 5 AND 300),
    CONSTRAINT ck_dashboard_time_range CHECK (
        time_range_default IN ('last_5m', 'last_15m', 'last_1h', 'last_6h',
                                'last_24h', 'last_7d', 'last_30d')
    ),
    CONSTRAINT ck_dashboard_version CHECK (version > 0)
);

-- Indexes (performance-critical for scalability research findings)
CREATE INDEX idx_dashboard_project_name ON dashboard(project, name);
CREATE INDEX idx_dashboard_created_at ON dashboard(created_at DESC);
CREATE INDEX idx_dashboard_updated_at ON dashboard(updated_at DESC);
CREATE INDEX idx_dashboard_state ON dashboard(state) WHERE state = 'active';
CREATE INDEX idx_dashboard_template_id ON dashboard(template_id) WHERE template_id IS NOT NULL;

-- Full-text search on dashboard names (trigram index)
CREATE INDEX idx_dashboard_name_trgm ON dashboard USING gin(name gin_trgm_ops);

-- JSONB indexes for metadata queries
CREATE INDEX idx_dashboard_doc_metadata ON dashboard USING gin((doc->'metadata') jsonb_path_ops);

-- Comments (documentation)
COMMENT ON TABLE dashboard IS 'Core entity representing a collection of visualization panels (Perses Dashboard)';
COMMENT ON COLUMN dashboard.id IS 'UUID v4 format dashboard identifier';
COMMENT ON COLUMN dashboard.doc IS 'Full Perses dashboard definition (JSON) including panels, layout, datasources';
COMMENT ON COLUMN dashboard.version IS 'Version number for optimistic locking (increment on each update)';
COMMENT ON COLUMN dashboard.state IS 'Lifecycle state: draft (editable), active (published), archived (read-only)';
COMMENT ON COLUMN dashboard.refresh_interval IS 'Metric refresh interval in seconds (FR-002: 5-300s range)';

-- ============================================================================
-- 2. Dashboard Template
-- ============================================================================

CREATE TABLE dashboard_template (
    id VARCHAR(256) NOT NULL,
    name VARCHAR(128) NOT NULL,
    description TEXT NOT NULL,
    category VARCHAR(64) NOT NULL,
    doc JSONB NOT NULL,
    variables JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(256) NOT NULL,
    version VARCHAR(32) NOT NULL,
    is_system BOOLEAN NOT NULL DEFAULT false,
    usage_count INTEGER NOT NULL DEFAULT 0,

    -- Primary Key
    CONSTRAINT pk_dashboard_template PRIMARY KEY (id),

    -- Unique Constraints
    CONSTRAINT uq_template_name UNIQUE (name),

    -- Check Constraints
    CONSTRAINT ck_template_category CHECK (
        category IN ('model_serving', 'training_job', 'notebook', 'data_pipeline', 'custom')
    ),
    CONSTRAINT ck_template_version CHECK (version ~ '^[0-9]+\.[0-9]+\.[0-9]+$'), -- Semver format
    CONSTRAINT ck_template_usage_count CHECK (usage_count >= 0)
);

-- Indexes
CREATE INDEX idx_template_category ON dashboard_template(category);
CREATE INDEX idx_template_is_system ON dashboard_template(is_system) WHERE is_system = true;
CREATE INDEX idx_template_usage_count ON dashboard_template(usage_count DESC);

-- Comments
COMMENT ON TABLE dashboard_template IS 'Pre-configured dashboard templates for common OpenShift AI workload types (FR-003)';
COMMENT ON COLUMN dashboard_template.variables IS 'Template variables for instantiation (e.g., namespace, workload_name)';
COMMENT ON COLUMN dashboard_template.is_system IS 'System-provided template (cannot be deleted by users)';
COMMENT ON COLUMN dashboard_template.usage_count IS 'Number of dashboards created from this template (incremented on instantiation)';

-- ============================================================================
-- 3. User Preferences
-- ============================================================================

CREATE TABLE user_preferences (
    user_id VARCHAR(256) NOT NULL,
    default_time_range VARCHAR(64) DEFAULT 'last_1h',
    default_refresh_interval INTEGER DEFAULT 15,
    timezone VARCHAR(64) DEFAULT 'UTC',
    favorite_dashboards JSONB DEFAULT '[]',
    recent_dashboards JSONB DEFAULT '[]',
    theme VARCHAR(32) DEFAULT 'light',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Primary Key
    CONSTRAINT pk_user_preferences PRIMARY KEY (user_id),

    -- Check Constraints
    CONSTRAINT ck_user_pref_refresh_interval CHECK (default_refresh_interval BETWEEN 5 AND 300),
    CONSTRAINT ck_user_pref_theme CHECK (theme IN ('light', 'dark', 'auto'))
);

-- Indexes
CREATE INDEX idx_user_pref_updated_at ON user_preferences(updated_at DESC);

-- Comments
COMMENT ON TABLE user_preferences IS 'User-specific settings for dashboard interaction and personalization';
COMMENT ON COLUMN user_preferences.user_id IS 'OpenShift user identity (username or email)';
COMMENT ON COLUMN user_preferences.timezone IS 'IANA timezone format (e.g., America/New_York, UTC)';
COMMENT ON COLUMN user_preferences.favorite_dashboards IS 'Array of favorited dashboard IDs (max 50 entries)';
COMMENT ON COLUMN user_preferences.recent_dashboards IS 'Array of recently viewed dashboards with timestamps (max 10, LRU)';

-- ============================================================================
-- 4. Share Token
-- ============================================================================

CREATE TABLE share_token (
    token_id UUID NOT NULL DEFAULT uuid_generate_v4(),
    dashboard_id VARCHAR(256) NOT NULL,
    created_by VARCHAR(256) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    permissions VARCHAR(32)[] NOT NULL DEFAULT ARRAY['read'],
    ip_restrictions CIDR[],
    revoked BOOLEAN NOT NULL DEFAULT false,
    revoked_at TIMESTAMP,
    revoked_by VARCHAR(256),
    usage_count INTEGER NOT NULL DEFAULT 0,
    last_used_at TIMESTAMP,
    last_used_ip INET,

    -- Primary Key
    CONSTRAINT pk_share_token PRIMARY KEY (token_id),

    -- Foreign Keys
    CONSTRAINT fk_share_token_dashboard FOREIGN KEY (dashboard_id)
        REFERENCES dashboard(id) ON DELETE CASCADE,

    -- Check Constraints
    CONSTRAINT ck_share_token_expires_at CHECK (expires_at > created_at),
    CONSTRAINT ck_share_token_ttl CHECK (expires_at <= created_at + INTERVAL '7 days'), -- Max 7d TTL
    CONSTRAINT ck_share_token_revoked_at CHECK (
        (revoked = false AND revoked_at IS NULL) OR
        (revoked = true AND revoked_at IS NOT NULL)
    ),
    CONSTRAINT ck_share_token_usage_count CHECK (usage_count >= 0)
);

-- Indexes (security-critical for token validation performance)
CREATE INDEX idx_share_token_dashboard ON share_token(dashboard_id);
CREATE INDEX idx_share_token_expires_at ON share_token(expires_at) WHERE revoked = false;
CREATE INDEX idx_share_token_created_by ON share_token(created_by);
CREATE INDEX idx_share_token_active ON share_token(dashboard_id, expires_at)
    WHERE revoked = false AND expires_at > NOW();

-- Comments
COMMENT ON TABLE share_token IS 'Ephemeral URL-based tokens for read-only dashboard sharing (FR-020, SC-010)';
COMMENT ON COLUMN share_token.token_id IS 'UUID v4 token (not sequential for security, prevents enumeration)';
COMMENT ON COLUMN share_token.permissions IS 'Granted permissions (MVP: read-only, future: edit)';
COMMENT ON COLUMN share_token.ip_restrictions IS 'Optional IP whitelist in CIDR notation (e.g., 10.0.0.0/8)';
COMMENT ON COLUMN share_token.usage_count IS 'Number of times token was used (incremented on each access)';

-- ============================================================================
-- 5. Audit Log Entry
-- ============================================================================

CREATE TABLE audit_log (
    id BIGSERIAL NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    user_id VARCHAR(256) NOT NULL,
    action VARCHAR(64) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    resource_id VARCHAR(256) NOT NULL,
    project_id VARCHAR(128) NOT NULL,
    result VARCHAR(32) NOT NULL,
    client_ip INET NOT NULL,
    user_agent TEXT,
    access_method VARCHAR(32) NOT NULL,
    token_id UUID,
    metadata JSONB,

    -- Primary Key
    CONSTRAINT pk_audit_log PRIMARY KEY (id),

    -- Foreign Keys (optional, for referential integrity)
    CONSTRAINT fk_audit_log_token FOREIGN KEY (token_id)
        REFERENCES share_token(token_id) ON DELETE SET NULL,

    -- Check Constraints
    CONSTRAINT ck_audit_result CHECK (result IN ('success', 'denied', 'error')),
    CONSTRAINT ck_audit_access_method CHECK (access_method IN ('rbac', 'share_token')),
    CONSTRAINT ck_audit_resource_type CHECK (
        resource_type IN ('Dashboard', 'DashboardTemplate', 'ShareToken')
    )
);

-- Indexes (query-optimized for audit reporting and compliance)
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp DESC);
CREATE INDEX idx_audit_user_id ON audit_log(user_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_project ON audit_log(project_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_token ON audit_log(token_id) WHERE token_id IS NOT NULL;

-- Composite index for common query patterns (user activity report)
CREATE INDEX idx_audit_user_timestamp ON audit_log(user_id, timestamp DESC);

-- Composite index for resource audit trail
CREATE INDEX idx_audit_resource_timestamp ON audit_log(resource_type, resource_id, timestamp DESC);

-- Comments
COMMENT ON TABLE audit_log IS 'Immutable audit trail for compliance (SOC2, ISO 27001, HIPAA) and security (SC-010)';
COMMENT ON COLUMN audit_log.id IS 'Auto-incrementing audit log ID (append-only, no updates/deletes)';
COMMENT ON COLUMN audit_log.timestamp IS 'Event timestamp in UTC with millisecond precision';
COMMENT ON COLUMN audit_log.user_id IS 'User identity (OpenShift username/email, or "anonymous" for token access)';
COMMENT ON COLUMN audit_log.action IS 'Action type (dashboard.create, dashboard.read, dashboard.share, etc.)';
COMMENT ON COLUMN audit_log.access_method IS 'Access method: rbac (authenticated) or share_token (anonymous)';
COMMENT ON COLUMN audit_log.metadata IS 'Additional context (error details, changed fields, request parameters)';

-- ============================================================================
-- 6. Schema Version Tracking
-- ============================================================================

CREATE TABLE schema_version (
    version VARCHAR(32) NOT NULL,
    description TEXT NOT NULL,
    applied_at TIMESTAMP NOT NULL DEFAULT NOW(),
    applied_by VARCHAR(256) NOT NULL,

    -- Primary Key
    CONSTRAINT pk_schema_version PRIMARY KEY (version)
);

-- Insert initial version
INSERT INTO schema_version (version, description, applied_by)
VALUES ('1.0.0', 'Initial Perses Observability Dashboard schema', 'system');

-- Comments
COMMENT ON TABLE schema_version IS 'Database schema version tracking for migrations';
COMMENT ON COLUMN schema_version.version IS 'Semver version (e.g., 1.0.0, 1.1.0, 2.0.0)';

-- ============================================================================
-- Foreign Key Constraints (applied after all tables created)
-- ============================================================================

ALTER TABLE dashboard
    ADD CONSTRAINT fk_dashboard_template
    FOREIGN KEY (template_id) REFERENCES dashboard_template(id) ON DELETE SET NULL;

-- ============================================================================
-- Triggers (automatic timestamp updates)
-- ============================================================================

-- Function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger for dashboard.updated_at
CREATE TRIGGER trigger_dashboard_updated_at
    BEFORE UPDATE ON dashboard
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Trigger for user_preferences.updated_at
CREATE TRIGGER trigger_user_pref_updated_at
    BEFORE UPDATE ON user_preferences
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- Row-Level Security (RLS) Policies (optional, for multi-tenancy)
-- ============================================================================

-- Enable RLS on dashboard table
ALTER TABLE dashboard ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only view dashboards in their authorized projects
-- NOTE: This requires integration with OpenShift RBAC at application layer
-- Actual enforcement happens in Perses authorization middleware
CREATE POLICY dashboard_project_isolation ON dashboard
    FOR ALL
    TO PUBLIC
    USING (true); -- Placeholder (actual RBAC in application layer)

-- Comments
COMMENT ON POLICY dashboard_project_isolation ON dashboard IS
    'Row-level security placeholder (actual authorization via Perses RBAC middleware)';

-- ============================================================================
-- Sample Data (for development/testing)
-- ============================================================================

-- Insert system dashboard templates (FR-003)
INSERT INTO dashboard_template (id, name, description, category, doc, variables, created_by, version, is_system)
VALUES
(
    'template-model-serving',
    'Model Serving Dashboard',
    'Real-time metrics for deployed model serving endpoints including inference latency, request rate, and resource utilization',
    'model_serving',
    '{"kind": "Dashboard", "spec": {"panels": {}}}'::jsonb,
    '{"namespace": {"type": "string", "required": true}, "model_name": {"type": "string", "required": true}}'::jsonb,
    'system',
    '1.0.0',
    true
),
(
    'template-training-job',
    'Training Job Dashboard',
    'Monitor training job progress with metrics for loss/accuracy, GPU utilization, and data throughput',
    'training_job',
    '{"kind": "Dashboard", "spec": {"panels": {}}}'::jsonb,
    '{"namespace": {"type": "string", "required": true}, "job_name": {"type": "string", "required": true}}'::jsonb,
    'system',
    '1.0.0',
    true
),
(
    'template-notebook',
    'Notebook Server Dashboard',
    'Notebook server resource monitoring including CPU, memory, and kernel status',
    'notebook',
    '{"kind": "Dashboard", "spec": {"panels": {}}}'::jsonb,
    '{"namespace": {"type": "string", "required": true}, "notebook_name": {"type": "string", "required": true}}'::jsonb,
    'system',
    '1.0.0',
    true
);

-- ============================================================================
-- Performance Tuning (PostgreSQL configuration recommendations)
-- ============================================================================

-- These should be set in postgresql.conf (not via SQL for production)
-- Documented here for reference from scalability research

/*
# Connection Settings
max_connections = 100
superuser_reserved_connections = 5

# Memory Settings (for 4GB RAM instance)
shared_buffers = 1GB
effective_cache_size = 3GB
work_mem = 20MB
maintenance_work_mem = 256MB

# Query Planner
random_page_cost = 1.1          # SSD storage
effective_io_concurrency = 200  # SSD concurrent I/O

# Write-Ahead Log
wal_buffers = 16MB
checkpoint_completion_target = 0.9

# Replication (for read replicas)
wal_level = replica
max_wal_senders = 5
hot_standby = on
*/

-- ============================================================================
-- Maintenance Procedures
-- ============================================================================

-- Procedure: Clean up expired share tokens (run daily)
CREATE OR REPLACE FUNCTION cleanup_expired_tokens()
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    DELETE FROM share_token
    WHERE expires_at < NOW() - INTERVAL '30 days'  -- Delete 30 days after expiration
    RETURNING COUNT(*) INTO deleted_count;

    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;

-- Procedure: Archive old audit logs (run monthly)
CREATE OR REPLACE FUNCTION archive_old_audit_logs(retention_days INTEGER DEFAULT 90)
RETURNS INTEGER AS $$
DECLARE
    archived_count INTEGER;
BEGIN
    -- In production, this would move to archive table or external storage
    -- For now, just count (actual archival depends on compliance requirements)
    SELECT COUNT(*) INTO archived_count
    FROM audit_log
    WHERE timestamp < NOW() - (retention_days || ' days')::INTERVAL;

    RETURN archived_count;
END;
$$ LANGUAGE plpgsql;

-- Procedure: Update dashboard usage statistics (run hourly via cron)
CREATE OR REPLACE FUNCTION update_template_usage_stats()
RETURNS VOID AS $$
BEGIN
    -- Update template usage counts based on dashboards created
    UPDATE dashboard_template t
    SET usage_count = (
        SELECT COUNT(*)
        FROM dashboard d
        WHERE d.template_id = t.id
    );
END;
$$ LANGUAGE plpgsql;

-- ============================================================================
-- End of Schema
-- ============================================================================
```

---

## Entity Relationships

### Entity Relationship Diagram

```
┌────────────────────────┐
│   DashboardTemplate    │
│  (Pre-configured)      │
└───────┬────────────────┘
        │ 1
        │
        │ creates from (optional)
        │
        │ *
┌───────▼────────────────┐       ┌─────────────────────┐
│      Dashboard         │───────│   UserPreferences   │
│   (Core Entity)        │ *   1 │  (Per-user config)  │
└───────┬────────────────┘       └─────────────────────┘
        │ 1                              user_id (external)
        │
        │ owns
        │
        │ *
┌───────▼────────────────┐
│     ShareToken         │
│  (Ephemeral sharing)   │
└───────┬────────────────┘
        │
        │ references
        │
        │ *
┌───────▼────────────────┐
│    AuditLogEntry       │
│  (Immutable audit)     │
└────────────────────────┘


┌─────────────────────────────────────────────────────────┐
│                    Dashboard.doc (JSONB)                │
│ ┌─────────────────────────────────────────────────────┐ │
│ │  spec.panels (map)                                  │ │
│ │  ├── panel-1 (ChartPanel)                           │ │
│ │  │   ├── kind: TimeSeriesChart                      │ │
│ │  │   ├── spec.queries[] (MetricQuery)               │ │
│ │  │   └── spec.visual (colors, thresholds)           │ │
│ │  ├── panel-2 (ChartPanel)                           │ │
│ │  │   ├── kind: GaugeChart                           │ │
│ │  │   └── spec.queries[]                             │ │
│ │  └── ...                                            │ │
│ └─────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────┐ │
│ │  spec.datasources                                   │ │
│ │  ├── prometheus (PrometheusDatasource)              │ │
│ │  └── thanos (PrometheusDatasource)                  │ │
│ └─────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────┐ │
│ │  spec.layouts                                       │ │
│ │  └── Grid layout (responsive canvas)                │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Relationship Summary**:

| Relationship | Cardinality | Description |
|--------------|-------------|-------------|
| DashboardTemplate → Dashboard | 1:N | Template can create multiple dashboards |
| Dashboard → ShareToken | 1:N | Dashboard can have multiple share tokens |
| Dashboard → AuditLogEntry | 1:N | Dashboard operations create audit log entries |
| ShareToken → AuditLogEntry | 1:N | Token usage creates audit log entries |
| User → UserPreferences | 1:1 | Each user has one preferences record |
| User → Dashboard | N:N | Users access dashboards via RBAC (implicit, not FK) |

**Foreign Key Constraints**:
- `dashboard.template_id` → `dashboard_template.id` (ON DELETE SET NULL)
- `share_token.dashboard_id` → `dashboard.id` (ON DELETE CASCADE)
- `audit_log.token_id` → `share_token.token_id` (ON DELETE SET NULL)

---

## Validation Rules

### Functional Requirements Mapping

| Rule ID | Functional Requirement | Validation | Enforcement Layer |
|---------|------------------------|------------|-------------------|
| **VR-001** | FR-001: Integrate Perses framework | Dashboard.doc must conform to Perses Dashboard schema | Application (JSON validation) |
| **VR-002** | FR-002: Real-time metric refresh (5-300s) | dashboard.refresh_interval BETWEEN 5 AND 300 | Database CHECK constraint |
| **VR-003** | FR-003: Pre-built templates (4 workload types) | dashboard_template.category IN (model_serving, training_job, notebook, data_pipeline) | Database CHECK constraint |
| **VR-004** | FR-004: Chart types (time series, area, bar, stat, gauge, table) | Dashboard.doc.spec.panels[*].spec.plugin.kind validation | Application (chart type registry) |
| **VR-005** | FR-005: User CRUD without admin privileges | User must have Dashboard:create permission in project | Application (Perses RBAC middleware) |
| **VR-006** | FR-006: Dashboard persistence (99.9% reliability) | Database-backed storage with backups | Infrastructure (PostgreSQL replication) |
| **VR-007** | FR-007: Metric filtering by labels | Metric queries must include namespace filter | Application (query validation) |
| **VR-008** | FR-008: Time range selector (15m to 7d, absolute) | dashboard.time_range_default enum validation | Database CHECK constraint |
| **VR-009** | FR-009: Metric metadata display | Metric metadata stored in Prometheus/Thanos | External (Prometheus API) |
| **VR-010** | FR-010: Graceful handling of missing data | Dashboard displays "No Data" indicator | Application (UI rendering) |
| **VR-011** | FR-011: Aggregation support (sum, avg, min, max, percentiles) | PromQL query validation | Application (query parser) |
| **VR-012** | FR-012: Export dashboard configurations | Export endpoint returns dashboard.doc JSON | Application (API endpoint) |
| **VR-013** | FR-013: OpenShift OAuth/RBAC integration | User authentication via OpenShift OAuth, RBAC via Perses | Application (auth middleware) |
| **VR-014** | FR-014: Drill-down navigation | Dashboard links stored in dashboard.doc metadata | Application (UI navigation) |
| **VR-015** | FR-015: Responsive chart rendering | Dashboard layouts use responsive grid | Application (Perses UI components) |
| **VR-016** | FR-016: Data freshness indicators | Timestamps tracked in chart state | Application (UI state management) |
| **VR-017** | FR-017: Event annotations | Annotations stored in dashboard.doc.spec.annotations | Application (Perses annotation plugin) |
| **VR-018** | FR-018: Visualization properties (colors, axes, thresholds) | Properties stored in dashboard.doc.spec.panels[*].spec.visual | Application (Perses chart config) |
| **VR-019** | FR-019: Connect to Prometheus/Thanos | Datasource configured in dashboard.doc.spec.datasources | Application (Perses datasource plugin) |
| **VR-020** | FR-020: Dashboard sharing (RBAC + tokens) | ShareToken table with RBAC permissions + ephemeral tokens | Database + Application |

### Success Criteria Validation

| Criteria ID | Success Criteria | Data Model Support |
|-------------|------------------|---------------------|
| **SC-001** | Create dashboard with 3 charts in <5min | Template instantiation (dashboard_template) |
| **SC-002** | Metric latency <30s | Enforced at application layer (caching, query optimization) |
| **SC-003** | 50+ concurrent users, <3s p95 load | Database indexes, caching (see scalability research) |
| **SC-004** | 85% user satisfaction | Tracked externally (user surveys) |
| **SC-005** | Dashboard load <3s for 8 charts | Database indexes, query optimization, Redis caching |
| **SC-006** | Historical data accessible (retention period) | Prometheus/Thanos retention policy (external) |
| **SC-007** | 99.9% persistence reliability | PostgreSQL replication, backups, schema version tracking |
| **SC-008** | 40% faster time-to-insight | Tracked in audit_log (time between dashboard.create and dashboard.read) |
| **SC-009** | 90% template usage on first attempt | Template usage tracking (dashboard_template.usage_count) |
| **SC-010** | Zero security violations | Audit logging (audit_log), RBAC enforcement, token security |

### Application-Layer Validations

**Dashboard Creation**:
```go
func ValidateDashboardCreate(dashboard *Dashboard, user User, project string) error {
    // FR-005: Check user has Dashboard:create permission
    if !user.HasPermission("Dashboard:create", project) {
        return errors.New("permission denied")
    }

    // VR-001: Validate Perses schema
    if err := validatePersesSchema(dashboard.Doc); err != nil {
        return fmt.Errorf("invalid dashboard schema: %w", err)
    }

    // VR-004: Validate chart types
    for panelID, panel := range dashboard.Doc.Spec.Panels {
        if !isValidChartType(panel.Spec.Plugin.Kind) {
            return fmt.Errorf("invalid chart type in panel %s: %s", panelID, panel.Spec.Plugin.Kind)
        }
    }

    // VR-007: Validate namespace filter in queries
    for _, panel := range dashboard.Doc.Spec.Panels {
        for _, query := range panel.Spec.Plugin.Spec.Queries {
            if !hasNamespaceFilter(query.Spec.Query, project) {
                return errors.New("metric query must include namespace filter for security")
            }
        }
    }

    return nil
}
```

**Share Token Generation**:
```go
func ValidateTokenGeneration(dashboardID string, user User, ttl time.Duration) error {
    // FR-020: Check user has Dashboard:share permission
    if !user.HasPermission("Dashboard:share", dashboard.Project) {
        return errors.New("permission denied: requires Dashboard:share permission")
    }

    // SC-010: Enforce max TTL (7 days)
    if ttl > 7*24*time.Hour {
        return errors.New("token TTL cannot exceed 7 days")
    }

    return nil
}
```

---

## Indexes and Query Optimization

### Index Strategy (from Scalability Research)

**Design Principles**:
1. **Read-heavy workload**: 95% reads, 5% writes (dashboards change infrequently)
2. **Query patterns**: Dashboard listing by project, recent dashboards, template usage
3. **Performance targets**: <100ms p95 for dashboard list queries (SC-005 supports <3s dashboard load)

**Index Definitions** (already included in schema above):

```sql
-- Dashboard Indexes (performance-critical)
CREATE INDEX idx_dashboard_project_name ON dashboard(project, name);
  -- Use case: List dashboards in project (most common query)
  -- Query: SELECT * FROM dashboard WHERE project = 'openshift-ai' ORDER BY name

CREATE INDEX idx_dashboard_created_at ON dashboard(created_at DESC);
  -- Use case: Recently created dashboards
  -- Query: SELECT * FROM dashboard ORDER BY created_at DESC LIMIT 10

CREATE INDEX idx_dashboard_updated_at ON dashboard(updated_at DESC);
  -- Use case: Recently modified dashboards
  -- Query: SELECT * FROM dashboard ORDER BY updated_at DESC LIMIT 10

CREATE INDEX idx_dashboard_state ON dashboard(state) WHERE state = 'active';
  -- Use case: List only active dashboards (partial index)
  -- Query: SELECT * FROM dashboard WHERE state = 'active'

CREATE INDEX idx_dashboard_template_id ON dashboard(template_id) WHERE template_id IS NOT NULL;
  -- Use case: Find dashboards created from specific template
  -- Query: SELECT * FROM dashboard WHERE template_id = 'template-model-serving'

CREATE INDEX idx_dashboard_name_trgm ON dashboard USING gin(name gin_trgm_ops);
  -- Use case: Full-text search on dashboard names (autocomplete)
  -- Query: SELECT * FROM dashboard WHERE name ILIKE '%serving%'

CREATE INDEX idx_dashboard_doc_metadata ON dashboard USING gin((doc->'metadata') jsonb_path_ops);
  -- Use case: Query dashboard metadata (tags, labels)
  -- Query: SELECT * FROM dashboard WHERE doc->'metadata' @> '{"tags": ["production"]}'

-- Share Token Indexes (security-critical)
CREATE INDEX idx_share_token_active ON share_token(dashboard_id, expires_at)
    WHERE revoked = false AND expires_at > NOW();
  -- Use case: Validate active token for dashboard access
  -- Query: SELECT * FROM share_token WHERE token_id = ? AND revoked = false AND expires_at > NOW()

-- Audit Log Indexes (compliance-critical)
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp DESC);
  -- Use case: Recent audit events
  -- Query: SELECT * FROM audit_log ORDER BY timestamp DESC LIMIT 100

CREATE INDEX idx_audit_user_timestamp ON audit_log(user_id, timestamp DESC);
  -- Use case: User activity report
  -- Query: SELECT * FROM audit_log WHERE user_id = 'stella@redhat.com' ORDER BY timestamp DESC

CREATE INDEX idx_audit_resource_timestamp ON audit_log(resource_type, resource_id, timestamp DESC);
  -- Use case: Resource audit trail
  -- Query: SELECT * FROM audit_log WHERE resource_id = 'dashboard-abc123' ORDER BY timestamp DESC
```

### Query Performance Benchmarks

| Query Pattern | Index Used | Expected Performance | Notes |
|---------------|------------|----------------------|-------|
| List dashboards by project | idx_dashboard_project_name | <10ms | Most common query, optimized composite index |
| Recent dashboards (global) | idx_dashboard_created_at | <20ms | Sorted index, DESC scan |
| Search dashboard by name | idx_dashboard_name_trgm | <50ms | Trigram similarity search (ILIKE) |
| Validate share token | idx_share_token_active | <5ms | Partial index (only active tokens) |
| User audit trail | idx_audit_user_timestamp | <30ms | Composite index, filtered by user |
| Resource audit trail | idx_audit_resource_timestamp | <30ms | Composite index, filtered by resource |

**Index Maintenance**:
```sql
-- Weekly maintenance (automated via cron)
REINDEX TABLE dashboard;        -- Rebuild indexes for optimal performance
ANALYZE dashboard;               -- Update query planner statistics

-- Monthly maintenance
VACUUM FULL dashboard;           -- Reclaim disk space from updated/deleted rows
```

---

## Migration Strategy

### Schema Version Control

**Pattern**: Follow Perses migration approach with version tracking

**Migration Files Location**: `migrations/postgresql/`

**Naming Convention**: `VXXX_description.up.sql` and `VXXX_description.down.sql`

**Example Migration**:
```sql
-- migrations/postgresql/V001_initial_schema.up.sql
-- See full schema above

-- migrations/postgresql/V001_initial_schema.down.sql
DROP TABLE IF EXISTS audit_log CASCADE;
DROP TABLE IF EXISTS share_token CASCADE;
DROP TABLE IF EXISTS user_preferences CASCADE;
DROP TABLE IF EXISTS dashboard CASCADE;
DROP TABLE IF EXISTS dashboard_template CASCADE;
DROP TABLE IF EXISTS schema_version CASCADE;
DROP FUNCTION IF EXISTS update_updated_at_column() CASCADE;
DROP FUNCTION IF EXISTS cleanup_expired_tokens() CASCADE;
DROP FUNCTION IF EXISTS archive_old_audit_logs(INTEGER) CASCADE;
```

### Migration Execution

**Tool**: Flyway or golang-migrate

**Example (golang-migrate)**:
```bash
# Apply migrations
migrate -database "postgres://user:pass@localhost:5432/perses?sslmode=disable" \
        -path migrations/postgresql up

# Rollback last migration
migrate -database "postgres://user:pass@localhost:5432/perses?sslmode=disable" \
        -path migrations/postgresql down 1

# Check current version
migrate -database "postgres://user:pass@localhost:5432/perses?sslmode=disable" \
        -path migrations/postgresql version
```

### Zero-Downtime Migration Strategy

**For Production** (99.9% availability requirement from SC-007):

1. **Backward-Compatible Migrations**:
   - Add new columns as NULL or with defaults
   - Never drop columns in same release as migration
   - Use two-phase migrations: Add column → Deploy code → Drop old column

2. **Example Two-Phase Migration**:
   ```sql
   -- Phase 1 (Version 1.1.0): Add new column
   ALTER TABLE dashboard ADD COLUMN new_field VARCHAR(256);

   -- Phase 2 (Version 1.2.0, after code deployment): Drop old column
   ALTER TABLE dashboard DROP COLUMN old_field;
   ```

3. **Rolling Deployment**:
   - Apply schema migration first (backward-compatible)
   - Deploy new API pods gradually (rolling update)
   - Verify schema compatibility with both old and new code

### Future Schema Changes

**Planned Migrations** (based on research findings):

**Version 1.1.0** (Phase 2 - Cross-project sharing):
```sql
-- Add cross-project sharing support
ALTER TABLE dashboard ADD COLUMN visibility VARCHAR(32) DEFAULT 'project';
ALTER TABLE dashboard ADD CONSTRAINT ck_dashboard_visibility
    CHECK (visibility IN ('project', 'cluster', 'custom'));

CREATE TABLE dashboard_access_grant (
    dashboard_id VARCHAR(256) NOT NULL,
    grantee_type VARCHAR(32) NOT NULL,  -- 'user', 'group', 'project'
    grantee_id VARCHAR(256) NOT NULL,
    permissions VARCHAR(32)[] NOT NULL,
    granted_by VARCHAR(256) NOT NULL,
    granted_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    CONSTRAINT pk_dashboard_access_grant PRIMARY KEY (dashboard_id, grantee_type, grantee_id),
    CONSTRAINT fk_access_grant_dashboard FOREIGN KEY (dashboard_id)
        REFERENCES dashboard(id) ON DELETE CASCADE
);
```

**Version 1.2.0** (Phase 3 - Advanced features):
```sql
-- Add dashboard versioning (snapshot history)
CREATE TABLE dashboard_version (
    dashboard_id VARCHAR(256) NOT NULL,
    version_number INTEGER NOT NULL,
    doc JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(256) NOT NULL,
    change_summary TEXT,
    CONSTRAINT pk_dashboard_version PRIMARY KEY (dashboard_id, version_number),
    CONSTRAINT fk_dashboard_version FOREIGN KEY (dashboard_id)
        REFERENCES dashboard(id) ON DELETE CASCADE
);

-- Add annotation support (FR-017)
CREATE TABLE dashboard_annotation (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    dashboard_id VARCHAR(256) NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    title VARCHAR(256) NOT NULL,
    description TEXT,
    tags VARCHAR(64)[],
    created_by VARCHAR(256) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CONSTRAINT fk_annotation_dashboard FOREIGN KEY (dashboard_id)
        REFERENCES dashboard(id) ON DELETE CASCADE
);
```

---

## Data Retention and Archival

### Retention Policies

| Table | Retention Policy | Archival Strategy | Compliance |
|-------|------------------|-------------------|------------|
| `dashboard` | Indefinite (until user deletes) | Soft delete (state='archived') | N/A |
| `dashboard_template` | Indefinite (system templates) | N/A | N/A |
| `user_preferences` | 90 days after last login | Delete after 90 days inactivity | GDPR right to erasure |
| `share_token` | 30 days after expiration | Delete expired tokens (cleanup_expired_tokens()) | Security best practice |
| `audit_log` | 90 days (default), 7 years (HIPAA) | Archive to S3/GCS after 90 days | SOC2, ISO 27001, HIPAA |

### Audit Log Archival

**Strategy**: Partition by time range (PostgreSQL 17+ declarative partitioning)

```sql
-- Convert audit_log to partitioned table (future migration)
CREATE TABLE audit_log_partitioned (
    LIKE audit_log INCLUDING ALL
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions
CREATE TABLE audit_log_2025_11 PARTITION OF audit_log_partitioned
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE audit_log_2025_12 PARTITION OF audit_log_partitioned
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');

-- Automated partition management (pg_partman extension)
-- Automatically creates new partitions and archives old ones
```

**Archival Process** (cron job):
```bash
#!/bin/bash
# Archive audit logs older than 90 days to S3

DATE_CUTOFF=$(date -d "90 days ago" +%Y-%m-%d)

# Export partition to CSV
psql -c "COPY (SELECT * FROM audit_log WHERE timestamp < '$DATE_CUTOFF') TO STDOUT WITH CSV HEADER" \
    | gzip > audit_log_${DATE_CUTOFF}.csv.gz

# Upload to S3
aws s3 cp audit_log_${DATE_CUTOFF}.csv.gz s3://perses-audit-archive/

# Drop archived partition (after verification)
psql -c "DROP TABLE audit_log_${DATE_CUTOFF}"
```

### GDPR Compliance

**Right to Erasure** (user_preferences):
```sql
-- Procedure: Delete user data (GDPR request)
CREATE OR REPLACE FUNCTION delete_user_data(p_user_id VARCHAR)
RETURNS VOID AS $$
BEGIN
    -- Delete user preferences
    DELETE FROM user_preferences WHERE user_id = p_user_id;

    -- Anonymize audit logs (retain for compliance but remove PII)
    UPDATE audit_log
    SET user_id = 'anonymized_' || MD5(user_id)
    WHERE user_id = p_user_id;

    -- Note: Dashboards owned by user are NOT deleted (organizational data)
    -- Instead, ownership transferred to admin or marked as system-owned
END;
$$ LANGUAGE plpgsql;
```

---

## Appendix: Data Model Implementation Checklist

**Phase 1 (Data Model Implementation - Current)**:
- [x] Define entity models (Dashboard, ChartPanel, MetricQuery, etc.)
- [x] Design PostgreSQL schema with constraints and indexes
- [x] Document validation rules mapped to functional requirements
- [x] Create migration scripts (V001_initial_schema.up.sql)
- [x] Add audit logging schema for compliance
- [x] Document retention and archival policies

**Phase 2 (Application Integration - Next)**:
- [ ] Implement Go DAOs for each entity (internal/api/database/sql/)
- [ ] Add application-layer validations (FR-005, FR-007, etc.)
- [ ] Integrate with Perses RBAC middleware
- [ ] Implement caching layer (Redis) for dashboard configs
- [ ] Create API endpoints for CRUD operations
- [ ] Add audit logging middleware

**Phase 3 (Testing - Concurrent with Phase 2)**:
- [ ] Unit tests for DAO layer (70% coverage target)
- [ ] Integration tests for database operations
- [ ] Load testing with 500 dashboards, 50 concurrent users
- [ ] Migration testing (up/down migrations)
- [ ] Data validation testing (constraint violations, edge cases)

**Phase 4 (Production Readiness)**:
- [ ] Set up PostgreSQL replication (primary + 2 replicas)
- [ ] Configure PgBouncer for connection pooling
- [ ] Deploy Redis for caching
- [ ] Set up automated backups (daily snapshots + PITR)
- [ ] Create monitoring dashboards for database metrics
- [ ] Document runbooks for common operations

---

## References

**Specification Documents**:
- Feature Spec: `/workspace/sessions/agentic-session-1763159109/workspace/workflows/ootb-ambient-workflows/specs/001-perses-dashboard/spec.md`
- Implementation Plan: `/workspace/sessions/agentic-session-1763159109/workspace/workflows/ootb-ambient-workflows/specs/001-perses-dashboard/plan.md`

**Research Documents**:
- Metric Backend Research: `/workspace/sessions/agentic-session-1763159109/workspace/workflows/ootb-ambient-workflows/specs/001-perses-dashboard/research.md`
- Sharing & Access Control: `/workspace/sessions/agentic-session-1763159109/workspace/workflows/ootb-ambient-workflows/specs/001-perses-dashboard/research/dashboard-sharing-access-control.md`
- Scalability Architecture: `/workspace/sessions/agentic-session-1763159109/workspace/workflows/ootb-ambient-workflows/specs/001-perses-dashboard/research/scalability-architecture.md`

**Perses Codebase References**:
- Perses Schema: `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/database/sql/sql.go`
- Perses Models: `/workspace/sessions/agentic-session-1763159109/workspace/perses/pkg/model/api/v1/dashboard/`
- Perses RBAC: `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/authorization/`

**Database Best Practices**:
- PostgreSQL Performance Tuning: https://www.postgresql.org/docs/17/performance-tips.html
- PostgreSQL Partitioning: https://www.postgresql.org/docs/17/ddl-partitioning.html
- Index Strategies: https://www.postgresql.org/docs/17/indexes.html

---

**Document Version**: 1.0.0
**Last Updated**: 2025-11-14
**Status**: Ready for Review
**Next Phase**: API Contract Design (Phase 1.2)
**Owner**: Stella (Staff Engineer) - OpenShift AI Platform Team
