# Data Model: Enhanced Model Metrics Display

**Feature**: Enhanced Model Metrics Display
**Branch**: 001-short-name-ambient
**Date**: 2025-10-24

## Purpose

This document defines the data structures, entities, relationships, and validation rules for the metrics visualization feature. Since this is a read-only visualization feature, these are primarily TypeScript interfaces for frontend state management and API responses.

---

## Core Entities

### 1. ModelDeployment

Represents a deployed machine learning model with associated metadata.

```typescript
interface ModelDeployment {
  // Identity
  id: string;                    // Unique identifier
  name: string;                  // Display name
  version: string;               // Model version

  // Deployment context
  namespace: string;             // Kubernetes namespace
  project: string;               // Data science project name
  servingRuntime: string;        // KServe, ModelMesh, etc.

  // Metadata
  createdAt: Date;               // Deployment timestamp
  updatedAt: Date;               // Last update timestamp
  status: DeploymentStatus;      // Current deployment status

  // References
  endpointUrl: string;           // Inference endpoint URL
  trustyaiEnabled: boolean;      // Whether TrustyAI is available
}

enum DeploymentStatus {
  RUNNING = 'running',
  STARTING = 'starting',
  FAILED = 'failed',
  STOPPED = 'stopped',
}
```

**Relationships**:
- One ModelDeployment has many PerformanceMetrics (time-series)
- One ModelDeployment has many BehaviorMetrics (time-series, optional)
- One ModelDeployment belongs to one Namespace (RBAC boundary)

**Validation Rules**:
- `id`, `name`, `namespace` are required
- `name` must match Kubernetes naming conventions: `^[a-z0-9]([-a-z0-9]*[a-z0-9])?$`
- `version` format: semver recommended but not enforced
- `endpointUrl` must be valid HTTP/HTTPS URL

**State Transitions**:
```
STARTING → RUNNING
STARTING → FAILED
RUNNING → STOPPED
STOPPED → STARTING
FAILED → STARTING (retry)
```

---

### 2. MetricQuery

Represents a Prometheus query definition with scope and filters.

```typescript
interface MetricQuery {
  // Query definition
  promql: string;                // PromQL query expression
  displayName: string;           // Human-readable name
  description?: string;          // Tooltip text

  // Scope
  namespace: string;             // RBAC-filtered namespace
  modelId?: string;              // Optional model filter

  // Metadata
  category: MetricCategory;      // Query classification
  aggregation: AggregationType;  // How data is aggregated
  unit: string;                  // Display unit (requests/sec, ms, etc.)
  format: MetricFormat;          // Number, percentage, duration, etc.
}

enum MetricCategory {
  OPERATIONAL_HEALTH = 'operational-health',
  MODEL_QUALITY = 'model-quality',
  RESOURCE_UTILIZATION = 'resource-utilization',
}

enum AggregationType {
  SUM = 'sum',
  AVERAGE = 'average',
  RATE = 'rate',
  PERCENTILE = 'percentile',
  COUNT = 'count',
}

enum MetricFormat {
  NUMBER = 'number',               // 1234
  PERCENTAGE = 'percentage',       // 12.34%
  DURATION_MS = 'duration-ms',     // 123ms
  DURATION_S = 'duration-s',       // 1.23s
  BYTES = 'bytes',                 // 1.2 KB
  REQUESTS_PER_SEC = 'req/s',      // 100 req/s
}
```

**Relationships**:
- One MetricQuery produces many MetricDataPoint instances (time-series results)
- Many MetricQuery instances belong to one TimeRange

**Validation Rules**:
- `promql` must be valid PromQL syntax
- `namespace` must be non-empty (enforced by RBAC)
- `displayName` is required for UI display
- `unit` and `format` must align (e.g., DURATION_MS → "ms" unit)

**Query Rewriting**:
All queries are rewritten by backend middleware to inject namespace filter:
```
Original: sum(rate(model_requests_total[5m]))
Rewritten: sum(rate(model_requests_total{namespace="my-project"}[5m]))
```

---

### 3. TimeRange

User-selected time window for metric visualization.

```typescript
interface TimeRange {
  // Preset or custom range
  preset: TimeRangePreset | null;  // Preset option
  start: Date | null;               // Custom start (if preset is null)
  end: Date | null;                 // Custom end (if preset is null)

  // Resolution
  step: number;                     // Query step in seconds
  resolution: 'high' | 'low';       // Affects step parameter

  // Refresh configuration
  refreshInterval: number;          // Auto-refresh interval in ms
  isPaused: boolean;                // User paused auto-refresh
  lastRefreshed: Date;              // Last refresh timestamp
}

enum TimeRangePreset {
  LAST_1_HOUR = '1h',
  LAST_24_HOURS = '24h',
  LAST_7_DAYS = '7d',
  LAST_30_DAYS = '30d',
}
```

**Relationships**:
- One TimeRange applies to many MetricQuery instances
- TimeRange state persisted in URL for sharing

**Validation Rules**:
- Either `preset` is set OR both `start` and `end` are set (mutually exclusive)
- `start` must be before `end` for custom ranges
- `refreshInterval` must be ≥ 30000ms (30 seconds minimum)
- `step` must be ≥ 15s (Prometheus scrape interval)

**Business Rules**:
- Preset ranges map to relative times: `1h` = now - 1 hour to now
- `step` automatically calculated based on range:
  - 1h → 15s step (240 points)
  - 24h → 1m step (1440 points)
  - 7d → 5m step (2016 points)
  - 30d → 30m step (1440 points)

**URL State Format**:
```
?timeRange=24h&refresh=60000&paused=false
?timeRange=custom&start=2025-10-24T00:00:00Z&end=2025-10-24T23:59:59Z
```

---

### 4. PerformanceMetrics

Time-series data representing operational performance.

```typescript
interface PerformanceMetrics {
  // Context
  modelId: string;
  namespace: string;
  timestamp: Date;

  // Latency metrics (milliseconds)
  latencyP50: number | null;      // 50th percentile
  latencyP95: number | null;      // 95th percentile
  latencyP99: number | null;      // 99th percentile
  latencyAvg: number | null;      // Average latency

  // Throughput metrics
  requestRate: number | null;     // Requests per second
  successRate: number | null;     // Successful requests per second
  errorRate: number | null;       // Failed requests per second

  // Error breakdown
  error4xxRate: number | null;    // Client errors per second
  error5xxRate: number | null;    // Server errors per second
  timeoutRate: number | null;     // Timeout errors per second

  // Resource utilization
  cpuUsage: number | null;        // CPU usage (cores)
  memoryUsage: number | null;     // Memory usage (bytes)
  gpuUsage: number | null;        // GPU usage (percentage, 0-100)
}
```

**Relationships**:
- Many PerformanceMetrics belong to one ModelDeployment
- PerformanceMetrics filtered by one TimeRange

**Validation Rules**:
- All rate fields must be ≥ 0
- Percentile values must be ≥ 0 (latency cannot be negative)
- `cpuUsage` ≥ 0 (measured in cores, e.g., 0.5 = 500 millicores)
- `memoryUsage` ≥ 0 (measured in bytes)
- `gpuUsage` must be 0-100 (percentage)
- `null` values indicate data unavailable (not zero)

**Data Source**: Prometheus via PromQL queries:
```promql
# Latency P99
histogram_quantile(0.99, sum(rate(model_latency_bucket{namespace="$namespace"}[5m])) by (model_id, le))

# Request rate
sum(rate(model_requests_total{namespace="$namespace"}[5m])) by (model_id)

# Error rate
sum(rate(model_errors_total{namespace="$namespace"}[5m])) by (model_id)
```

---

### 5. BehaviorMetrics

Time-series data representing model usage and prediction patterns (TrustyAI).

```typescript
interface BehaviorMetrics {
  // Context
  modelId: string;
  namespace: string;
  timestamp: Date;

  // Request patterns
  requestVolume: number | null;         // Total requests in window
  uniqueUsers: number | null;           // Distinct users/clients
  batchRequestRatio: number | null;     // Batch vs single requests (0-1)

  // Prediction patterns
  predictionDistribution: Record<string, number> | null;  // Label → count
  confidenceScoreAvg: number | null;                      // Average confidence (0-1)
  confidenceScoreP50: number | null;                      // Median confidence

  // Data quality indicators
  driftScore: number | null;            // Data drift indicator (0-1, 1 = high drift)
  missingFeatureRate: number | null;    // Rate of missing input features (0-1)
  outlierRate: number | null;           // Rate of outlier requests (0-1)

  // Fairness metrics (optional TrustyAI)
  biasScore: number | null;             // Bias detection score (0-1)
  disparateImpact: number | null;       // Disparate impact ratio
}
```

**Relationships**:
- Many BehaviorMetrics belong to one ModelDeployment (when TrustyAI enabled)
- BehaviorMetrics filtered by one TimeRange

**Validation Rules**:
- Ratio/percentage fields must be 0-1 or null
- `confidenceScoreAvg`, `confidenceScoreP50` must be 0-1 or null
- `driftScore`, `biasScore` must be 0-1 or null (higher = more concerning)
- `predictionDistribution` keys are model-specific class labels
- All fields nullable (TrustyAI may not be available)

**Data Source**: TrustyAI service REST API or Prometheus (if TrustyAI exports metrics):
```typescript
// TrustyAI API endpoint (conceptual)
GET /api/trustyai/v1/metrics/{namespace}/{model_id}?start={iso_date}&end={iso_date}
```

**Graceful Degradation**:
- If TrustyAI service unavailable: All fields return `null`
- UI displays empty state with "Enable TrustyAI" instructions (FR-019)

---

### 6. TrustyAIService

Optional external service providing behavior metrics.

```typescript
interface TrustyAIService {
  // Service status
  available: boolean;             // Is TrustyAI service reachable
  version: string | null;         // TrustyAI service version
  lastChecked: Date;              // Last availability check

  // Configuration
  endpoint: string;               // TrustyAI API base URL
  namespace: string;              // Service namespace

  // Capabilities
  supportedMetrics: string[];     // Available metric types
}
```

**Relationships**:
- One TrustyAIService provides BehaviorMetrics for many ModelDeployments
- TrustyAIService availability checked per namespace

**Validation Rules**:
- `endpoint` must be valid HTTP/HTTPS URL
- `available` determined by health check API call
- `supportedMetrics` list determines which behavior metrics are shown

**Health Check Pattern**:
```typescript
async function checkTrustyAIAvailability(namespace: string): Promise<boolean> {
  try {
    const response = await fetch(`/api/trustyai/v1/health?namespace=${namespace}`, {
      timeout: 5000,
    });
    return response.ok;
  } catch {
    return false;
  }
}
```

---

## Derived Entities

### MetricDataPoint

Individual data point from Prometheus query result.

```typescript
interface MetricDataPoint {
  timestamp: number;              // Unix timestamp (seconds)
  value: number;                  // Metric value
  labels: Record<string, string>; // Prometheus labels
}

interface PrometheusQueryResult {
  query: MetricQuery;
  data: MetricDataPoint[];
  status: 'success' | 'error' | 'timeout';
  error?: string;
}
```

---

### MetricPanel

UI component state for a metric visualization.

```typescript
interface MetricPanel {
  // Identity
  id: string;                     // Unique panel ID
  title: string;                  // Panel title
  query: MetricQuery;             // Associated query

  // Visualization
  chartType: ChartType;           // Line, bar, gauge, etc.
  options: ChartOptions;          // Chart-specific config

  // State
  isLoading: boolean;             // Fetching data
  isError: boolean;               // Query failed
  isExpanded: boolean;            // Panel expanded/collapsed
  lastUpdated: Date | null;       // Last successful update
}

enum ChartType {
  LINE = 'line',
  AREA = 'area',
  BAR = 'bar',
  GAUGE = 'gauge',
  STAT = 'stat',
  TABLE = 'table',
}
```

---

## Entity Relationships Diagram

```
ModelDeployment (1) ──── (N) PerformanceMetrics
                   └──── (N) BehaviorMetrics (optional)

TimeRange (1) ──── (N) MetricQuery
                └──── (N) MetricPanel

MetricQuery (1) ──── (N) MetricDataPoint

TrustyAIService (1) ──── (N) BehaviorMetrics

User
  │
  ├─ hasAccessTo → Namespace (RBAC)
  │                   │
  │                   └─ contains → ModelDeployment
  │
  └─ selects → TimeRange
               └─ filters → MetricQuery
```

---

## State Transitions

### Panel Loading State Machine

```
IDLE → LOADING → SUCCESS → IDLE (after refresh interval)
  └──→ LOADING → ERROR → RETRYING → SUCCESS
                      └─→ RETRYING → ERROR (after 3 attempts)
```

```typescript
enum PanelState {
  IDLE = 'idle',           // Not loaded yet
  LOADING = 'loading',     // Fetching data
  SUCCESS = 'success',     // Data loaded
  ERROR = 'error',         // Query failed
  RETRYING = 'retrying',   // Retry after error
  STALE = 'stale',         // Cached data, refresh failed
}
```

---

## Data Validation Summary

### Critical Validations (must enforce):

1. **Namespace scoping**: All queries must include namespace filter (enforced backend)
2. **RBAC checks**: User must have `get` permission on InferenceService in namespace
3. **Query timeout**: Queries must complete within 30s (frontend) / 2m (Prometheus)
4. **Time range bounds**: Start < end, future dates rejected
5. **Metric value ranges**: Percentages 0-100, ratios 0-1, counts ≥ 0

### Graceful Degradation Rules:

1. **TrustyAI unavailable**: Show empty state with enablement instructions (FR-019)
2. **Prometheus unavailable**: Show error state with retry button (FR-020)
3. **Missing data points**: Interpolate or show gaps in chart (FR-021)
4. **Partial query failures**: Show available panels, mark others as error
5. **Stale cache**: Display cached data with staleness indicator

---

## Performance Considerations

### Query Optimization:

- **Recording rules**: Pre-compute aggregations on Prometheus (1m evaluation interval)
- **Client caching**: React Query with 90s staleTime, 5m cacheTime
- **Staggered refresh**: Offset panel queries by 2s to prevent thundering herd
- **Cardinality limits**: Use topk(10) for multi-model views

### Memory Management:

- **Time-series pruning**: Keep only visible time range in memory
- **Component unmount**: Cancel in-flight queries when panels unmounted
- **Cache eviction**: React Query automatically evicts stale data after 5 minutes

---

## TypeScript Type Exports

All types exported from:
```typescript
// src/pages/modelServing/screens/metrics/types/metrics.ts
export type {
  ModelDeployment,
  DeploymentStatus,
  MetricQuery,
  MetricCategory,
  AggregationType,
  MetricFormat,
  TimeRange,
  TimeRangePreset,
  PerformanceMetrics,
  BehaviorMetrics,
  TrustyAIService,
  MetricDataPoint,
  PrometheusQueryResult,
  MetricPanel,
  ChartType,
  PanelState,
};
```

---

*Data model version: 1.0*
*Last updated: 2025-10-24*
