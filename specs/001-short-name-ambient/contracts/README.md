# API Contracts: Enhanced Model Metrics Display

This directory contains OpenAPI 3.0 specifications for all external API interactions.

## Contracts

### 1. `prometheus-api.yaml`

**Purpose**: Query Prometheus for performance metrics (latency, throughput, errors, resource utilization)

**Operations**:
- `GET /query` - Instant query at single point in time
- `GET /query_range` - Range query for time-series visualization
- `GET /health` - Service health check

**Key Features**:
- Automatic namespace filtering by backend (RBAC enforcement)
- 30-second frontend timeout, 2-minute Prometheus timeout
- Error handling for bad queries, timeouts, and service unavailability
- Response format matches Prometheus HTTP API v1

**Usage Example**:
```typescript
// Query request rate for a model
GET /api/prometheus/v1/query_range?
  query=sum(rate(model_requests_total[5m])) by (model_id)&
  start=2025-10-24T09:00:00Z&
  end=2025-10-24T10:00:00Z&
  step=1m

// Backend automatically rewrites query to:
// sum(rate(model_requests_total{namespace="my-project"}[5m])) by (model_id)
```

**Dependencies**:
- Prometheus 2.x+ backend
- OpenShift OAuth for authentication
- Backend middleware for query rewriting

---

### 2. `trustyai-api.yaml`

**Purpose**: Query TrustyAI for behavior metrics (data drift, fairness, bias detection, prediction patterns)

**Status**: OPTIONAL - Feature gracefully degrades if unavailable

**Operations**:
- `GET /health` - Check TrustyAI availability for namespace
- `GET /metrics/{namespace}/{model_id}` - Get comprehensive behavior metrics
- `GET /metrics/{namespace}/{model_id}/drift` - Get drift time-series

**Key Features**:
- Returns `null` for unavailable metrics (partial data support)
- Namespace-scoped queries with RBAC enforcement
- Time-range filtering consistent with Prometheus API
- Graceful degradation when service unavailable

**Usage Example**:
```typescript
// Check if TrustyAI available
GET /api/trustyai/v1/health?namespace=my-project
// Response: { status: "available", version: "0.3.0", supportedMetrics: ["drift", "fairness"] }

// Get behavior metrics
GET /api/trustyai/v1/metrics/my-project/fraud-detection-v2?
  start=2025-10-24T09:00:00Z&
  end=2025-10-24T10:00:00Z&
  metrics=drift,fairness,predictions

// Response includes null for unavailable metrics
```

**Dependencies**:
- TrustyAI service 0.3.0+
- OpenShift OAuth for authentication
- Backend proxy with RBAC filtering

---

## Contract Testing

Contract tests validate that API responses match these specifications.

### Test Files

```
frontend/__tests__/contracts/
├── prometheus-api.test.ts
└── trustyai-api.test.ts
```

### Test Approach

1. **Schema Validation**: Assert response matches OpenAPI schema
2. **Error Scenarios**: Test all documented error responses
3. **RBAC Enforcement**: Verify namespace filtering works
4. **Graceful Degradation**: Test unavailability handling

### Running Tests

```bash
# Unit tests with mocked responses
npm run test:contracts

# Integration tests against test environment
npm run test:integration -- --grep "API contracts"
```

---

## Authentication

All API endpoints require OpenShift OAuth bearer token:

```http
Authorization: Bearer <oauth_token>
```

Token must have permissions:
- `get` verb on `InferenceService` resource in target namespace
- Backend validates permissions before executing queries

---

## Rate Limiting

**Prometheus API**:
- Frontend: 1 request per query per 60 seconds (auto-refresh interval)
- Staggered panel loading: 2-second offset between panels
- Exponential backoff on 429 responses

**TrustyAI API**:
- Frontend: 1 request per query per 60 seconds
- Health check: Maximum 1 request per 5 minutes per namespace

---

## Error Handling

All APIs follow consistent error response format:

```json
{
  "status": "error",
  "errorType": "timeout",
  "error": "human-readable message",
  "warnings": ["optional warning"]
}
```

### Error Types

| Type | HTTP Status | Retry? | Frontend Action |
|------|-------------|--------|-----------------|
| `bad_data` | 400 | No | Show error, don't retry |
| `forbidden` | 403 | No | Show permission denied message |
| `timeout` | 408 | Yes (3x) | Retry with exponential backoff |
| `unavailable` | 503 | Yes (3x) | Show service unavailable, retry |
| `execution` | 422 | No | Show query error |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-10-24 | Initial API contracts for metrics visualization |

---

## References

- [Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)
- [TrustyAI Documentation](https://trustyai.io/)
- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
