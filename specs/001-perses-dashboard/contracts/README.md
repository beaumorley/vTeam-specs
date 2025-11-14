# API Contracts: Perses Observability Dashboard

**Version**: 1.0.0
**Last Updated**: 2025-11-14
**Author**: Stella (Staff Engineer)
**Status**: Ready for Implementation

---

## Overview

This directory contains comprehensive API contract specifications for the OpenShift AI Perses Observability Dashboard feature. These contracts define the interface between the frontend UI and backend API, enabling:

- **Contract Testing**: Pact-based consumer-driven contract testing (see `testing-strategy-summary.md`)
- **API Documentation**: OpenAPI 3.0 specification for REST endpoints
- **Schema Validation**: JSON Schema definitions for request/response validation
- **Implementation Guidance**: Detailed examples for all P0 endpoints

---

## Directory Structure

```
contracts/
├── README.md                           # This file
├── openapi.yaml                        # Complete OpenAPI 3.0 specification
├── schemas/                            # JSON Schema definitions
│   ├── dashboard-schema.json           # Dashboard entity schema
│   ├── metric-query-schema.json        # Metric query request/response
│   ├── share-token-schema.json         # Share token management
│   └── error-response-schema.json      # Standard error format
└── examples/                           # Sample requests/responses
    ├── model-serving-dashboard.json    # Complete model serving dashboard
    ├── training-job-dashboard.json     # Training job dashboard example
    ├── metric-query-examples.json      # Metric query examples (10+ queries)
    └── api-response-examples.json      # API response examples (success/errors)
```

---

## Quick Start

### 1. View the OpenAPI Specification

The `openapi.yaml` file is the single source of truth for all REST API endpoints. You can:

**View in Swagger UI**:
```bash
# Using Docker
docker run -p 8080:8080 -e SWAGGER_JSON=/contracts/openapi.yaml \
  -v $(pwd):/contracts swaggerapi/swagger-ui

# Or use Redoc
docker run -p 8080:80 -e SPEC_URL=openapi.yaml \
  -v $(pwd):/usr/share/nginx/html/openapi.yaml redocly/redoc
```

**View in VSCode**:
- Install "OpenAPI (Swagger) Editor" extension
- Open `openapi.yaml` and use preview (Cmd+Shift+P → "OpenAPI: Preview")

### 2. Validate API Contracts

**Validate OpenAPI Specification**:
```bash
# Using openapi-generator-cli
docker run --rm -v $(pwd):/local openapitools/openapi-generator-cli validate \
  -i /local/openapi.yaml

# Or use Spectral (recommended)
npm install -g @stoplight/spectral-cli
spectral lint openapi.yaml
```

**Validate JSON Schemas**:
```bash
# Using ajv-cli
npm install -g ajv-cli
ajv validate -s schemas/dashboard-schema.json \
  -d examples/model-serving-dashboard.json
```

### 3. Generate Client SDKs

**TypeScript/JavaScript Client**:
```bash
docker run --rm -v $(pwd):/local openapitools/openapi-generator-cli generate \
  -i /local/openapi.yaml \
  -g typescript-axios \
  -o /local/generated/typescript-client
```

**Go Client**:
```bash
docker run --rm -v $(pwd):/local openapitools/openapi-generator-cli generate \
  -i /local/openapi.yaml \
  -g go \
  -o /local/generated/go-client
```

---

## P0 Endpoints for Pact Contract Testing

These 6 endpoints are critical path and MUST have Pact contract tests (see `testing-strategy-summary.md`):

| # | Endpoint | Method | Description | Contract Priority |
|---|----------|--------|-------------|-------------------|
| 1 | `/api/v1/projects/{project}/dashboards` | GET | List dashboards in project | P0 |
| 2 | `/api/v1/projects/{project}/dashboards/{id}` | GET | Get dashboard by ID | P0 |
| 3 | `/api/v1/projects/{project}/dashboards` | POST | Create new dashboard | P0 |
| 4 | `/api/v1/projects/{project}/dashboards/{id}` | PUT | Update dashboard | P0 |
| 5 | `/api/v1/projects/{project}/dashboards/{id}/share` | POST | Generate share token | P0 |
| 6 | `/api/v1/metrics/query` | POST | Query metrics | P0 |

**Coverage Target**: 100% of P0 endpoints by Phase 1 completion

---

## Schema Validation Rules

### Dashboard Schema (`schemas/dashboard-schema.json`)

**Key Validations**:
- `id`: Must match pattern `^dashboard-[a-z0-9]{6,}$`
- `name`: 1-128 characters, unique within project
- `project`: Kubernetes DNS-1123 subdomain format
- `doc`: Must conform to Perses Dashboard schema (kind: Dashboard)
- `refresh_interval`: 5-300 seconds (FR-002)
- `time_range_default`: Enum of valid time ranges (FR-008)
- `version`: Minimum 1, used for optimistic locking

**Example Validation**:
```bash
ajv validate -s schemas/dashboard-schema.json \
  -d examples/model-serving-dashboard.json
```

### Metric Query Schema (`schemas/metric-query-schema.json`)

**Request Validations**:
- `query`: 1-1000 characters (PromQL syntax)
- `start`/`end`: ISO 8601 date-time (UTC)
- `step`: Duration format (`^\d+[smhd]$`)

**Response Types**:
- `matrix`: Range vector (time series over time)
- `vector`: Instant vector (single point in time)
- `scalar`: Single numeric value

**Security Note**: All queries MUST include namespace filter (enforced at API layer, not schema)

### Share Token Schema (`schemas/share-token-schema.json`)

**Key Validations**:
- `token_id`: UUID v4 format (non-sequential for security)
- `expires_in`: 60-604800 seconds (1 minute to 7 days)
- `ip_restrictions`: Valid CIDR notation array (optional)
- `permissions`: Currently only `["read"]` supported (MVP)

**Security Controls**:
- Token TTL: 24h default, 7d maximum (SC-010)
- IP restrictions: Optional CIDR whitelist
- Rate limiting: 10 tokens/hour per dashboard

### Error Response Schema (`schemas/error-response-schema.json`)

**Standard Format**:
```json
{
  "error": {
    "code": "SCREAMING_SNAKE_CASE",
    "message": "Human-readable error message",
    "details": {
      "field": "optional context"
    }
  }
}
```

**Error Codes**:
- `INVALID_REQUEST`: Bad request parameters (400)
- `UNAUTHORIZED`: Authentication required (401)
- `FORBIDDEN`: Insufficient permissions (403)
- `NOT_FOUND`: Resource not found (404)
- `VERSION_CONFLICT`: Optimistic locking failure (409)
- `DASHBOARD_NAME_CONFLICT`: Duplicate name (409)
- `RATE_LIMIT_EXCEEDED`: Rate limit exceeded (429)
- `INVALID_SHARE_TOKEN`: Invalid/expired token (401)
- `IP_RESTRICTION_VIOLATION`: IP not whitelisted (403)
- `INTERNAL_ERROR`: Unexpected server error (500)
- `SERVICE_UNAVAILABLE`: Service temporarily unavailable (503)

---

## Example Usage

### Example 1: Create a Model Serving Dashboard

**Request**:
```bash
curl -X POST https://dashboard.openshift-ai.svc/api/v1/projects/openshift-ai/dashboards \
  -H "Authorization: Bearer $OPENSHIFT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "name": "My Model Dashboard",
  "doc": {
    "kind": "Dashboard",
    "spec": {
      "display": {
        "name": "My Model Dashboard"
      },
      "datasources": {
        "prometheus": {
          "kind": "PrometheusDatasource",
          "spec": {
            "directUrl": "https://thanos-querier.openshift-monitoring.svc:9091"
          }
        }
      },
      "panels": {
        "panel-1": {
          "kind": "Panel",
          "spec": {
            "display": {
              "name": "Inference Latency"
            },
            "plugin": {
              "kind": "TimeSeriesChart",
              "spec": {
                "queries": [
                  {
                    "kind": "PrometheusTimeSeriesQuery",
                    "spec": {
                      "query": "histogram_quantile(0.95, rate(model_inference_duration_seconds_bucket{namespace=\"openshift-ai\"}[5m]))"
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
                "x": 0,
                "y": 0,
                "width": 12,
                "height": 6,
                "content": {
                  "$ref": "#/spec/panels/panel-1"
                }
              }
            ]
          }
        }
      ]
    }
  },
  "refresh_interval": 15,
  "time_range_default": "last_1h"
}
EOF
```

**Response** (201 Created):
```json
{
  "id": "dashboard-abc123",
  "name": "My Model Dashboard",
  "project": "openshift-ai",
  "created_at": "2025-11-14T23:00:00Z",
  "created_by": "user@example.com",
  "updated_at": "2025-11-14T23:00:00Z",
  "updated_by": "user@example.com",
  "version": 1,
  "state": "active",
  "template_id": null,
  "refresh_interval": 15,
  "time_range_default": "last_1h"
}
```

### Example 2: Generate Share Token

**Request**:
```bash
curl -X POST https://dashboard.openshift-ai.svc/api/v1/projects/openshift-ai/dashboards/dashboard-abc123/share \
  -H "Authorization: Bearer $OPENSHIFT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "expires_in": 86400,
  "ip_restrictions": ["10.0.0.0/8"]
}
EOF
```

**Response** (201 Created):
```json
{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "dashboard_id": "dashboard-abc123",
  "share_url": "https://dashboard.openshift-ai.svc/shared/550e8400-e29b-41d4-a716-446655440000",
  "created_by": "user@example.com",
  "created_at": "2025-11-14T23:00:00Z",
  "expires_at": "2025-11-15T23:00:00Z",
  "permissions": ["read"],
  "ip_restrictions": ["10.0.0.0/8"],
  "revoked": false,
  "usage_count": 0
}
```

### Example 3: Query Metrics

**Request**:
```bash
curl -X POST https://dashboard.openshift-ai.svc/api/v1/metrics/query \
  -H "Authorization: Bearer $OPENSHIFT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "query": "histogram_quantile(0.95, rate(model_inference_duration_seconds_bucket{namespace=\"openshift-ai\"}[5m]))",
  "start": "2025-11-14T22:00:00Z",
  "end": "2025-11-14T23:00:00Z",
  "step": "15s"
}
EOF
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "__name__": "model_inference_duration_seconds",
          "namespace": "openshift-ai",
          "pod": "model-serving-abc123"
        },
        "values": [
          [1731628800, "0.125"],
          [1731628815, "0.130"],
          [1731628830, "0.128"]
        ]
      }
    ]
  }
}
```

---

## Contract Testing with Pact

### Consumer (Frontend) Tests

**Example TypeScript Consumer Test**:
```typescript
import { PactV3 } from '@pact-foundation/pact';

const provider = new PactV3({
  consumer: 'perses-dashboard-ui',
  provider: 'perses-dashboard-api'
});

describe('Dashboard API', () => {
  it('should list dashboards in project', async () => {
    await provider
      .given('dashboards exist in project openshift-ai')
      .uponReceiving('a request to list dashboards')
      .withRequest({
        method: 'GET',
        path: '/api/v1/projects/openshift-ai/dashboards',
        headers: {
          Authorization: 'Bearer token'
        }
      })
      .willRespondWith({
        status: 200,
        headers: {
          'Content-Type': 'application/json'
        },
        body: {
          dashboards: eachLike({
            id: like('dashboard-abc123'),
            name: like('Model Serving Dashboard'),
            project: 'openshift-ai',
            state: 'active'
          })
        }
      });

    await provider.executeTest(async (mockServer) => {
      const client = new DashboardClient(mockServer.url);
      const dashboards = await client.listDashboards('openshift-ai');
      expect(dashboards.length).toBeGreaterThan(0);
    });
  });
});
```

### Provider (Backend) Verification

**Example Go Provider Test**:
```go
import (
    "github.com/pact-foundation/pact-go/v2/provider"
)

func TestPactProvider(t *testing.T) {
    verifier := provider.NewVerifier()

    err := verifier.VerifyProvider(t, provider.VerifyRequest{
        ProviderBaseURL:            "http://localhost:8080",
        BrokerURL:                  os.Getenv("PACT_BROKER_URL"),
        PublishVerificationResults: true,
        ProviderVersion:            "1.0.0",
        StateHandlers: provider.StateHandlers{
            "dashboards exist in project openshift-ai": func(setup bool, state provider.State) (provider.StateResponse, error) {
                // Setup test data in database
                seedDashboards()
                return provider.StateResponse{}, nil
            },
        },
    })

    if err != nil {
        t.Fatal(err)
    }
}
```

---

## Implementation Checklist

### Phase 1: API Implementation (Backend)

- [ ] Implement dashboard CRUD endpoints (GET, POST, PUT, DELETE)
- [ ] Add dashboard export endpoint (GET `/dashboards/{id}/export`)
- [ ] Implement template management (list, get, instantiate)
- [ ] Add share token management (create, revoke, validate)
- [ ] Implement metrics query proxy with security enforcement
- [ ] Add user preferences endpoints (get, update)
- [ ] Implement audit log query endpoint
- [ ] Add OpenAPI specification serving endpoint
- [ ] Implement request validation using JSON schemas
- [ ] Add rate limiting middleware (100 req/min per user)
- [ ] Implement RBAC permission checks
- [ ] Add audit logging for all operations

### Phase 2: Contract Testing

- [ ] Set up Pact Broker (OSS or PactFlow)
- [ ] Implement consumer tests for P0 endpoints (frontend)
- [ ] Implement provider verification tests (backend)
- [ ] Add contract tests to CI/CD pipelines
- [ ] Configure can-i-deploy checks before deployment
- [ ] Set up contract versioning and compatibility checks

### Phase 3: API Documentation

- [ ] Deploy Swagger UI for interactive API docs
- [ ] Generate client SDKs (TypeScript, Go)
- [ ] Create API integration guide for developers
- [ ] Add authentication/authorization examples
- [ ] Document error handling best practices

---

## API Design Principles

### 1. RESTful Conventions

- **Resource-based URLs**: `/projects/{project}/dashboards/{id}`
- **HTTP methods**: GET (read), POST (create), PUT (update), DELETE (delete)
- **Plural nouns**: `/dashboards`, `/templates`, `/preferences`
- **Nested resources**: `/dashboards/{id}/share` for related resources

### 2. Status Codes

- `200 OK`: Successful GET/PUT
- `201 Created`: Successful POST with Location header
- `204 No Content`: Successful DELETE
- `400 Bad Request`: Invalid request parameters
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `409 Conflict`: Version conflict or unique constraint violation
- `422 Unprocessable Entity`: Valid request but business logic validation failed
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Unexpected server error
- `503 Service Unavailable`: Temporary unavailability

### 3. Versioning

- **URL path versioning**: `/api/v1/` prefix
- **Backward compatibility**: New fields are optional, old fields deprecated gradually
- **Version negotiation**: Major version in URL, minor versions backward compatible

### 4. Pagination

- **Query parameters**: `limit` (default 20, max 100), `offset` (default 0)
- **Response metadata**: `pagination` object with `total`, `limit`, `offset`, `next_offset`
- **No limit**: Use `limit=0` to disable pagination (admin-only for exports)

### 5. Security

- **Authentication**: OpenShift OAuth bearer tokens in `Authorization` header
- **RBAC**: Permission checks via Perses authorization middleware
- **Namespace filter**: All metric queries MUST include namespace filter (FR-007)
- **Rate limiting**: Per-user rate limits with `X-RateLimit-*` headers
- **Audit logging**: All operations logged for compliance (SC-010)

### 6. Error Handling

- **Consistent format**: All errors use standard `ErrorResponse` schema
- **Machine-readable codes**: `SCREAMING_SNAKE_CASE` error codes
- **User-friendly messages**: Human-readable error messages in English
- **Actionable details**: Additional context in `details` object

---

## Related Documents

- **Feature Spec**: `../spec.md` - User stories and functional requirements
- **Data Model**: `../data-model.md` - Database schema and entity definitions
- **Testing Strategy**: `../testing-strategy-summary.md` - Contract and resilience testing approach
- **Implementation Plan**: `../plan.md` - Phase breakdown and timeline

---

## Support

For questions or issues with API contracts, contact:

- **Stella (Staff Engineer)**: stella@redhat.com
- **OpenShift AI Platform Team**: openshift-ai-dev@redhat.com
- **Slack**: #openshift-ai-platform

---

**Document Version**: 1.0.0
**Last Updated**: 2025-11-14
**Status**: Ready for Implementation
