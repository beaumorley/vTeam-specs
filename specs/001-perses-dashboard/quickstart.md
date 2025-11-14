# Developer Quickstart Guide: Perses Observability Dashboard

**Feature**: OpenShift AI Perses Observability Dashboard
**Version**: 1.0.0
**Last Updated**: 2025-11-14
**Target Audience**: Backend Go developers, Frontend TypeScript developers, QA engineers

---

## Overview

The Perses Observability Dashboard provides real-time and historical monitoring for OpenShift AI workloads (model serving, training jobs, notebooks) through customizable dashboards. This feature integrates the open-source [Perses visualization framework](https://github.com/perses/perses) with OpenShift AI, enabling data scientists and ML engineers to:

- Monitor model serving endpoints (inference latency, request rates, error rates)
- Track training job progress (loss/accuracy, GPU utilization, throughput)
- Visualize notebook server resource usage (CPU, memory, kernel status)
- Create custom dashboards with real-time metric updates (<30s latency)
- Share dashboards via ephemeral tokens or RBAC permissions

### Architecture

The implementation extends the upstream Perses project with OpenShift AI-specific integrations:

```
┌──────────────────────────────────────────────────────────────┐
│                    Frontend (React + TypeScript)             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Perses Dashboard Renderer (@perses-dev/dashboards)    │  │
│  │  + OpenShift AI Workload Selectors + Templates         │  │
│  └────────────────────────────────────────────────────────┘  │
└────────────────────┬─────────────────────────────────────────┘
                     │ REST API + WebSocket (real-time updates)
┌────────────────────▼─────────────────────────────────────────┐
│              Backend (Go 1.21+)                              │
│  ┌──────────────────┐  ┌──────────────────────────────────┐  │
│  │ Perses Core SDK  │  │ OpenShift AI Extension           │  │
│  │ - Dashboard CRUD │  │ - OAuth Authentication           │  │
│  │ - Authorization  │  │ - Namespace-scoped RBAC          │  │
│  │ - Template Mgmt  │  │ - Workload Templates (4 types)   │  │
│  └──────────────────┘  └──────────────────────────────────┘  │
└────────────────────┬─────────────────────────────────────────┘
                     │
         ┌───────────┴────────────┐
         ▼                        ▼
  ┌──────────────┐       ┌───────────────────┐
  │ PostgreSQL   │       │ Prometheus/Thanos │
  │ (Dashboards) │       │ (Metrics TSDB)    │
  └──────────────┘       └───────────────────┘
```

### Key Technologies

- **Backend**: Go 1.21+, Echo v4 (REST framework), Perses Go SDK, Kubernetes client-go
- **Frontend**: React 18.2, TypeScript 5.4+, Perses UI components, Material-UI v6, TanStack Query
- **Storage**: PostgreSQL 14+ (dashboard configs), Prometheus/Thanos (metric data)
- **Auth**: OpenShift OAuth 2.0, Perses RBAC middleware
- **Testing**: Go testing, Pact (contract tests), Jest + React Testing Library, Playwright (E2E)

---

## Prerequisites

### Required Software

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| Go | 1.21+ | Backend development | [go.dev/doc/install](https://go.dev/doc/install) |
| Node.js | v22+ | Frontend development | [nodejs.org](https://nodejs.org) (use nvm recommended) |
| TypeScript | 5.4+ | Frontend type safety | `npm install -g typescript@5.4` |
| Docker | 24+ | Database and testing | [docker.com/get-docker](https://docker.com/get-docker) |
| kubectl/oc CLI | 1.28+ | OpenShift interaction | [docs.openshift.com](https://docs.openshift.com/container-platform/4.14/cli_reference/openshift_cli/getting-started-cli.html) |
| PostgreSQL Client | 14+ | Database migrations | `brew install postgresql@14` or `apt install postgresql-client` |
| git | 2.40+ | Version control | [git-scm.com](https://git-scm.com) |

### Access Requirements

- **OpenShift Cluster**: Access to OpenShift 4.x cluster (dev/staging environment)
- **Namespace**: Permissions to create resources in a test namespace
- **OAuth Token**: OpenShift authentication token for API testing

> **Tip**: Use `oc login` to authenticate, then `oc whoami -t` to get your token

### Environment Setup

Create a `.env` file in the project root:

```bash
# Database
DATABASE_URL=postgresql://perses:password@localhost:5432/perses_dev?sslmode=disable

# OpenShift
OPENSHIFT_API_URL=https://api.your-cluster.openshift.com:6443
OPENSHIFT_OAUTH_CLIENT_ID=perses-dashboard
OPENSHIFT_OAUTH_CLIENT_SECRET=your-secret-here

# Prometheus
PROMETHEUS_URL=https://thanos-querier.openshift-monitoring.svc:9091
PROMETHEUS_BEARER_TOKEN=your-prometheus-token

# Application
API_PORT=8080
UI_PORT=3000
LOG_LEVEL=debug
ENABLE_AUDIT_LOGGING=true
```

---

## Quick Start (5 Minutes)

### 1. Clone Repository

```bash
# Clone the Perses upstream repository (contains core framework)
git clone https://github.com/perses/perses.git /workspace/perses

# Clone the OpenShift AI extension repository
git clone <your-repo-url> /workspace/openshift-ai-dashboard

cd /workspace/openshift-ai-dashboard
```

### 2. Install Dependencies

**Backend**:
```bash
cd backend/
go mod download
go mod verify
```

**Frontend**:
```bash
cd frontend/
npm install
```

### 3. Start Database

```bash
# Start PostgreSQL with Docker
docker run -d \
  --name perses-postgres \
  -e POSTGRES_USER=perses \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=perses_dev \
  -p 5432:5432 \
  postgres:17-alpine

# Wait for database to be ready
docker exec perses-postgres pg_isready -U perses
```

### 4. Run Database Migrations

```bash
cd backend/

# Install golang-migrate if not already installed
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Apply migrations
migrate -database "${DATABASE_URL}" \
  -path db/migrations up

# Verify schema version
migrate -database "${DATABASE_URL}" \
  -path db/migrations version
# Expected: 001 (initial schema)
```

### 5. Start Backend API Server

```bash
cd backend/

# Build and run
go run cmd/server/main.go

# Expected output:
# [INFO] Starting Perses Dashboard API v1.0.0
# [INFO] Database: Connected to PostgreSQL
# [INFO] Loading system templates...
# [INFO] API server listening on :8080
# [INFO] WebSocket endpoint ready at ws://localhost:8080/ws
```

**Verify API is running**:
```bash
curl http://localhost:8080/health
# Expected: {"status":"healthy","database":"connected","version":"1.0.0"}
```

### 6. Start Frontend Dev Server

```bash
cd frontend/

# Start development server
npm run dev

# Expected output:
# VITE v5.0.0  ready in 500 ms
# ➜  Local:   http://localhost:3000/
# ➜  Network: use --host to expose
```

### 7. Access Dashboard

Open your browser to **http://localhost:3000**

You should see the dashboard list page. For local development without OpenShift OAuth:

```bash
# Set development mode (bypasses OAuth)
export DEVELOPMENT_MODE=true

# Restart backend server
cd backend/ && go run cmd/server/main.go
```

> **What happens if I see authentication errors?**
> In development mode, OAuth is bypassed. If you see 401 errors, verify `DEVELOPMENT_MODE=true` is set in your environment.

---

## Project Structure

Understanding the directory layout helps you navigate the codebase:

```text
openshift-ai-dashboard/
├── backend/                          # Go backend (API server)
│   ├── cmd/
│   │   └── server/
│   │       └── main.go               # Server entrypoint
│   ├── pkg/
│   │   ├── auth/                     # OpenShift OAuth + RBAC
│   │   │   ├── openshift_oauth.go    # OAuth client implementation
│   │   │   └── rbac_middleware.go    # Namespace-scoped authorization
│   │   ├── workload/                 # Workload metadata integration
│   │   │   ├── model_serving.go      # Model serving workload discovery
│   │   │   ├── training_job.go       # Training job metadata
│   │   │   └── notebook.go           # Notebook server detection
│   │   ├── templates/                # Dashboard templates
│   │   │   ├── model_serving_dashboard.go  # Template: Model serving
│   │   │   ├── training_dashboard.go       # Template: Training jobs
│   │   │   ├── notebook_dashboard.go       # Template: Notebook servers
│   │   │   └── data_pipeline_dashboard.go  # Template: Data pipelines
│   │   ├── proxy/                    # Prometheus proxy layer
│   │   │   ├── prometheus_proxy.go   # Query proxy with auth
│   │   │   └── metric_aggregator.go  # Metric aggregation utilities
│   │   └── api/                      # REST API handlers
│   │       ├── dashboard_handler.go  # Dashboard CRUD endpoints
│   │       ├── template_handler.go   # Template management
│   │       ├── metrics_handler.go    # Metric query proxy
│   │       └── share_handler.go      # Share token management
│   ├── internal/
│   │   ├── config/                   # Configuration loading
│   │   ├── database/                 # Database access layer (DAOs)
│   │   └── middleware/               # HTTP middleware (auth, logging)
│   ├── db/
│   │   └── migrations/               # PostgreSQL migrations
│   │       ├── 001_initial_schema.up.sql
│   │       └── 001_initial_schema.down.sql
│   ├── go.mod                        # Go dependencies
│   └── Dockerfile                    # Backend container image
├── frontend/                         # React + TypeScript frontend
│   ├── src/
│   │   ├── components/
│   │   │   ├── DashboardBuilder/     # Custom dashboard creation UI
│   │   │   ├── WorkloadSelector/     # OpenShift AI workload picker
│   │   │   ├── TemplateGallery/      # Template selection gallery
│   │   │   └── PersesDashboard/      # Wrapper around Perses renderer
│   │   ├── pages/
│   │   │   ├── DashboardList/        # Dashboard listing page
│   │   │   ├── DashboardView/        # Dashboard rendering page
│   │   │   └── DashboardEditor/      # Dashboard edit mode
│   │   ├── services/
│   │   │   ├── api-client.ts         # Backend API client
│   │   │   ├── auth.ts               # OAuth token management
│   │   │   └── websocket.ts          # Real-time update handler
│   │   ├── hooks/
│   │   │   ├── useDashboard.ts       # Dashboard state management
│   │   │   ├── useMetrics.ts         # Metric query hook
│   │   │   └── useWorkloads.ts       # Workload metadata hook
│   │   └── App.tsx                   # Application root
│   ├── package.json                  # NPM dependencies
│   ├── tsconfig.json                 # TypeScript configuration
│   └── vite.config.ts                # Vite build configuration
├── tests/                            # Test suites
│   ├── contract/                     # Pact contract tests
│   │   ├── api_consumer_test.go      # Backend provider verification
│   │   └── dashboard_consumer.spec.ts # Frontend consumer tests
│   ├── integration/                  # Integration tests
│   │   ├── dashboard_crud_test.go    # Dashboard CRUD operations
│   │   ├── auth_integration_test.go  # Authentication flow
│   │   └── metric_query_test.go      # Metric query validation
│   ├── e2e/                          # End-to-end tests (Playwright)
│   │   ├── dashboard_creation.spec.ts
│   │   ├── real_time_updates.spec.ts
│   │   └── template_instantiation.spec.ts
│   └── performance/                  # Load tests
│       ├── concurrent_users_test.go
│       └── dashboard_rendering_test.ts
└── specs/                            # Feature specifications
    └── 001-perses-dashboard/
        ├── spec.md                   # Feature requirements
        ├── data-model.md             # Database schema
        ├── plan.md                   # Implementation plan
        ├── quickstart.md             # This file
        └── contracts/                # API contracts
            ├── openapi.yaml          # OpenAPI 3.0 specification
            └── schemas/              # JSON Schema definitions
```

---

## Development Workflow

### Creating a New Dashboard Template

Dashboard templates provide pre-configured layouts for common workload types. Here's how to add a new template:

**Step 1**: Define the template structure in `backend/pkg/templates/`

```go
// backend/pkg/templates/data_pipeline_dashboard.go
package templates

import (
    persesv1 "github.com/perses/perses/pkg/model/api/v1"
)

func DataPipelineDashboard() *DashboardTemplate {
    return &DashboardTemplate{
        ID:          "template-data-pipeline",
        Name:        "Data Pipeline Dashboard",
        Description: "Monitor data pipeline throughput, latency, and error rates",
        Category:    "data_pipeline",
        Variables: map[string]VariableDefinition{
            "namespace": {
                Type:     "string",
                Required: true,
                Label:    "Namespace",
            },
            "pipeline_name": {
                Type:     "string",
                Required: true,
                Label:    "Pipeline Name",
            },
        },
        Doc: persesv1.Dashboard{
            Kind: "Dashboard",
            Spec: persesv1.DashboardSpec{
                Display: &persesv1.Display{
                    Name:        "{{.pipeline_name}} Pipeline Metrics",
                    Description: "Data pipeline performance metrics",
                },
                Datasources: map[string]*persesv1.DatasourceSpec{
                    "prometheus": {
                        Kind: "PrometheusDatasource",
                        Spec: map[string]interface{}{
                            "directUrl": "https://thanos-querier.openshift-monitoring.svc:9091",
                        },
                    },
                },
                Panels: map[string]*persesv1.Panel{
                    "throughput": createThroughputPanel(),
                    "latency":    createLatencyPanel(),
                    "errors":     createErrorPanel(),
                },
                Layouts: []persesv1.Layout{
                    createGridLayout(),
                },
            },
        },
    }
}

func createThroughputPanel() *persesv1.Panel {
    return &persesv1.Panel{
        Kind: "Panel",
        Spec: persesv1.PanelSpec{
            Display: &persesv1.Display{
                Name: "Data Throughput (Records/sec)",
            },
            Plugin: &persesv1.Plugin{
                Kind: "TimeSeriesChart",
                Spec: map[string]interface{}{
                    "queries": []map[string]interface{}{
                        {
                            "kind": "PrometheusTimeSeriesQuery",
                            "spec": map[string]interface{}{
                                "query": `rate(pipeline_records_processed_total{namespace="{{.namespace}}",pipeline="{{.pipeline_name}}"}[5m])`,
                            },
                        },
                    },
                },
            },
        },
    }
}
```

**Step 2**: Register the template in the template registry

```go
// backend/pkg/templates/registry.go
package templates

func LoadSystemTemplates() []*DashboardTemplate {
    return []*DashboardTemplate{
        ModelServingDashboard(),
        TrainingJobDashboard(),
        NotebookDashboard(),
        DataPipelineDashboard(), // Add new template
    }
}
```

**Step 3**: Test the template

```go
// backend/pkg/templates/data_pipeline_dashboard_test.go
package templates

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestDataPipelineDashboard(t *testing.T) {
    template := DataPipelineDashboard()

    // Validate template structure
    assert.Equal(t, "template-data-pipeline", template.ID)
    assert.Equal(t, "data_pipeline", template.Category)
    assert.NotNil(t, template.Doc.Spec.Panels["throughput"])

    // Validate variable substitution
    dashboard, err := template.Instantiate(map[string]string{
        "namespace":     "openshift-ai",
        "pipeline_name": "training-data-pipeline",
    })
    assert.NoError(t, err)
    assert.Contains(t, dashboard.Spec.Display.Name, "training-data-pipeline")
}
```

### Adding a New API Endpoint

Follow these steps to add a new REST API endpoint:

**Step 1**: Define the API contract in `contracts/openapi.yaml`

```yaml
paths:
  /api/v1/projects/{project}/dashboards/{id}/clone:
    post:
      summary: Clone an existing dashboard
      operationId: cloneDashboard
      parameters:
        - name: project
          in: path
          required: true
          schema:
            type: string
        - name: id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                  minLength: 1
                  maxLength: 128
      responses:
        '201':
          description: Dashboard cloned successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Dashboard'
```

**Step 2**: Implement the handler in `backend/pkg/api/`

```go
// backend/pkg/api/dashboard_handler.go
package api

import (
    "net/http"
    "github.com/labstack/echo/v4"
)

func (h *DashboardHandler) CloneDashboard(c echo.Context) error {
    project := c.Param("project")
    dashboardID := c.Param("id")

    // Parse request body
    var req struct {
        Name string `json:"name" validate:"required,min=1,max=128"`
    }
    if err := c.Bind(&req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "invalid request body")
    }

    // Validate request
    if err := c.Validate(&req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }

    // Check RBAC permissions
    user := c.Get("user").(*auth.User)
    if !user.HasPermission("Dashboard:create", project) {
        return echo.NewHTTPError(http.StatusForbidden, "insufficient permissions")
    }

    // Get source dashboard
    source, err := h.dashboardDAO.Get(project, dashboardID)
    if err != nil {
        return echo.NewHTTPError(http.StatusNotFound, "dashboard not found")
    }

    // Clone dashboard
    clone := &model.Dashboard{
        Name:    req.Name,
        Project: project,
        Doc:     source.Doc, // Copy dashboard definition
        CreatedBy: user.Email,
        UpdatedBy: user.Email,
        RefreshInterval: source.RefreshInterval,
        TimeRangeDefault: source.TimeRangeDefault,
    }

    // Save to database
    created, err := h.dashboardDAO.Create(clone)
    if err != nil {
        return echo.NewHTTPError(http.StatusInternalServerError, "failed to clone dashboard")
    }

    // Audit log
    h.auditLog.Log(audit.Entry{
        Action:       "dashboard.clone",
        UserID:       user.Email,
        ResourceType: "Dashboard",
        ResourceID:   created.ID,
        ProjectID:    project,
        Result:       "success",
    })

    return c.JSON(http.StatusCreated, created)
}
```

**Step 3**: Register the route in `backend/cmd/server/main.go`

```go
// Register API routes
api := e.Group("/api/v1", authMiddleware, rbacMiddleware)
api.POST("/projects/:project/dashboards/:id/clone", dashboardHandler.CloneDashboard)
```

**Step 4**: Write contract tests

```typescript
// tests/contract/dashboard_consumer.spec.ts
import { PactV3 } from '@pact-foundation/pact';

describe('Dashboard Clone API', () => {
  it('should clone an existing dashboard', async () => {
    await provider
      .given('dashboard exists with ID dashboard-abc123')
      .uponReceiving('a request to clone dashboard')
      .withRequest({
        method: 'POST',
        path: '/api/v1/projects/openshift-ai/dashboards/dashboard-abc123/clone',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer token'
        },
        body: {
          name: 'Cloned Dashboard'
        }
      })
      .willRespondWith({
        status: 201,
        body: {
          id: like('dashboard-def456'),
          name: 'Cloned Dashboard',
          project: 'openshift-ai',
          state: 'active'
        }
      });
  });
});
```

### Writing Contract Tests (Pact)

Contract tests ensure the frontend and backend APIs stay synchronized.

**Frontend Consumer Test**:

```typescript
// tests/contract/dashboard_consumer.spec.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
const { like, eachLike } = MatchersV3;

const provider = new PactV3({
  consumer: 'perses-dashboard-ui',
  provider: 'perses-dashboard-api'
});

describe('Dashboard API Contract', () => {
  it('should list dashboards in a project', async () => {
    await provider
      .given('dashboards exist in project openshift-ai')
      .uponReceiving('a request to list dashboards')
      .withRequest({
        method: 'GET',
        path: '/api/v1/projects/openshift-ai/dashboards',
        headers: {
          'Authorization': 'Bearer token'
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
            state: like('active'),
            created_at: like('2025-11-14T23:00:00Z')
          })
        }
      });

    await provider.executeTest(async (mockServer) => {
      const client = new DashboardClient(mockServer.url);
      const result = await client.listDashboards('openshift-ai');
      expect(result.dashboards.length).toBeGreaterThan(0);
    });
  });
});
```

**Backend Provider Verification**:

```go
// tests/contract/api_consumer_test.go
package contract

import (
    "testing"
    "github.com/pact-foundation/pact-go/v2/provider"
)

func TestPactProvider(t *testing.T) {
    verifier := provider.NewVerifier()

    err := verifier.VerifyProvider(t, provider.VerifyRequest{
        ProviderBaseURL: "http://localhost:8080",
        BrokerURL:       "https://pact-broker.example.com",
        PublishVerificationResults: true,
        ProviderVersion: "1.0.0",
        StateHandlers: provider.StateHandlers{
            "dashboards exist in project openshift-ai": func(setup bool, state provider.State) (provider.StateResponse, error) {
                if setup {
                    // Seed test data
                    seedDashboards("openshift-ai")
                } else {
                    // Cleanup
                    cleanupDashboards()
                }
                return provider.StateResponse{}, nil
            },
        },
    })

    if err != nil {
        t.Fatalf("Pact verification failed: %v", err)
    }
}
```

### Running E2E Tests (Playwright)

End-to-end tests validate the complete user workflow.

**Example E2E Test**:

```typescript
// tests/e2e/dashboard_creation.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Dashboard Creation', () => {
  test.beforeEach(async ({ page }) => {
    // Login with test credentials
    await page.goto('http://localhost:3000/login');
    await page.fill('[data-testid="username"]', 'testuser');
    await page.fill('[data-testid="password"]', 'password');
    await page.click('[data-testid="login-button"]');
    await expect(page).toHaveURL(/.*dashboards/);
  });

  test('should create dashboard from template', async ({ page }) => {
    // Navigate to template gallery
    await page.click('[data-testid="create-dashboard"]');
    await page.click('[data-testid="use-template"]');

    // Select model serving template
    await page.click('[data-testid="template-model-serving"]');

    // Fill template variables
    await page.fill('[name="namespace"]', 'openshift-ai');
    await page.fill('[name="model_name"]', 'sentiment-analysis');
    await page.fill('[name="dashboard_name"]', 'My Model Dashboard');

    // Submit
    await page.click('[data-testid="create-button"]');

    // Verify dashboard created
    await expect(page).toHaveURL(/.*dashboards\/dashboard-[a-z0-9]+/);
    await expect(page.locator('h1')).toContainText('My Model Dashboard');

    // Verify panels loaded
    await expect(page.locator('[data-testid="panel-inference-latency"]')).toBeVisible();
  });

  test('should display real-time metric updates', async ({ page }) => {
    await page.goto('http://localhost:3000/dashboards/dashboard-test123');

    // Wait for initial data load
    await page.waitForSelector('[data-testid="chart-loaded"]');

    // Get initial value
    const initialValue = await page.locator('[data-testid="latency-value"]').textContent();

    // Wait for refresh interval (15 seconds)
    await page.waitForTimeout(16000);

    // Verify value updated
    const updatedValue = await page.locator('[data-testid="latency-value"]').textContent();
    expect(updatedValue).not.toBe(initialValue);
  });
});
```

**Run E2E tests**:

```bash
cd tests/e2e/

# Install Playwright
npm install -D @playwright/test

# Run tests headless
npx playwright test

# Run tests with UI (for debugging)
npx playwright test --ui

# Run specific test file
npx playwright test dashboard_creation.spec.ts
```

---

## Key Concepts

### Dashboard Model (Perses JSON Schema)

Dashboards are stored as JSON documents conforming to the Perses schema:

```json
{
  "kind": "Dashboard",
  "metadata": {
    "name": "model-serving-dashboard",
    "project": "openshift-ai",
    "version": 1
  },
  "spec": {
    "display": {
      "name": "Model Serving Overview",
      "description": "Real-time model serving metrics"
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
          "display": { "name": "Inference Latency (p95)" },
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
              "x": 0, "y": 0, "width": 12, "height": 6,
              "content": { "$ref": "#/spec/panels/panel-1" }
            }
          ]
        }
      }
    ]
  }
}
```

**Key Fields**:
- `kind`: Always "Dashboard" (Perses resource type)
- `spec.panels`: Map of panel definitions (charts)
- `spec.layouts`: Grid layout positioning
- `spec.datasources`: Prometheus/Thanos connection

### Metric Queries (PromQL)

All metric queries use Prometheus Query Language (PromQL):

```promql
# Real-time inference latency (p95)
histogram_quantile(0.95, rate(model_inference_duration_seconds_bucket{namespace="openshift-ai"}[5m]))

# CPU utilization aggregated by workload
sum by (workload) (rate(container_cpu_usage_seconds_total{namespace="openshift-ai"}[5m]))

# Request rate with status codes
sum(rate(http_requests_total{namespace="openshift-ai",workload="model-serving"}[5m])) by (status)

# GPU utilization (if available)
avg(nvidia_gpu_duty_cycle{namespace="openshift-ai"}) by (pod)
```

**Security**: All queries MUST include a namespace filter (`namespace="..."`) to enforce tenant isolation.

### RBAC + Share Tokens

**OpenShift RBAC**: Namespace-scoped permissions

```yaml
# Dashboard creator role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dashboard-creator
  namespace: openshift-ai
rules:
- apiGroups: ["perses.openshift.io"]
  resources: ["dashboards"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: ["perses.openshift.io"]
  resources: ["dashboards/share"]
  verbs: ["create", "delete"]
```

**Share Tokens**: Ephemeral URL-based access

```bash
# Generate share token (24h expiration, IP restricted)
curl -X POST http://localhost:8080/api/v1/projects/openshift-ai/dashboards/dashboard-abc123/share \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "expires_in": 86400,
    "ip_restrictions": ["10.0.0.0/8"]
  }'

# Response
{
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "share_url": "http://localhost:3000/shared/550e8400-e29b-41d4-a716-446655440000",
  "expires_at": "2025-11-15T23:00:00Z"
}
```

**Security Features**:
- UUID v4 tokens (non-sequential, prevents enumeration)
- Maximum 7-day TTL
- Optional IP whitelist (CIDR notation)
- Rate limiting: 10 tokens/hour per dashboard
- Automatic revocation on expiration

### Real-time Updates (Server-Sent Events)

Dashboards receive real-time metric updates via Server-Sent Events (SSE):

```typescript
// Frontend: Subscribe to real-time updates
const eventSource = new EventSource(
  `http://localhost:8080/api/v1/dashboards/${dashboardId}/stream`,
  { headers: { 'Authorization': `Bearer ${token}` } }
);

eventSource.onmessage = (event) => {
  const update = JSON.parse(event.data);
  // Update chart data
  updateChart(update.panelId, update.metrics);
};

eventSource.onerror = (error) => {
  console.error('SSE connection failed:', error);
  eventSource.close();
  // Implement reconnection logic
};
```

**Backend**: Emit updates on refresh interval

```go
// backend/pkg/api/stream_handler.go
func (h *StreamHandler) StreamDashboard(c echo.Context) error {
    dashboardID := c.Param("id")

    // Set SSE headers
    c.Response().Header().Set("Content-Type", "text/event-stream")
    c.Response().Header().Set("Cache-Control", "no-cache")
    c.Response().Header().Set("Connection", "keep-alive")

    ticker := time.NewTicker(15 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // Query metrics and send update
            metrics, err := h.queryMetrics(dashboardID)
            if err != nil {
                continue
            }
            fmt.Fprintf(c.Response(), "data: %s\n\n", metrics)
            c.Response().Flush()
        case <-c.Request().Context().Done():
            return nil
        }
    }
}
```

---

## Common Tasks

### Adding a New Chart Type

Perses supports extensible chart plugins. To add a custom chart type:

**Step 1**: Define the plugin spec

```typescript
// frontend/src/plugins/HeatmapChart.tsx
import { Plugin } from '@perses-dev/plugin-system';

export const HeatmapChartPlugin: Plugin = {
  kind: 'HeatmapChart',
  component: HeatmapChart,
  optionsSchema: {
    type: 'object',
    properties: {
      colorScheme: {
        type: 'string',
        enum: ['viridis', 'plasma', 'inferno'],
        default: 'viridis'
      },
      bucketSize: {
        type: 'number',
        default: 60,
        minimum: 10
      }
    }
  }
};

function HeatmapChart({ queries, options }: ChartProps) {
  const { data } = useMetrics(queries);
  // Render heatmap using data
  return <div>...</div>;
}
```

**Step 2**: Register the plugin

```typescript
// frontend/src/plugins/index.ts
import { PluginRegistry } from '@perses-dev/plugin-system';
import { HeatmapChartPlugin } from './HeatmapChart';

PluginRegistry.register(HeatmapChartPlugin);
```

### Implementing a New Metric Query

Add custom metric aggregations or transformations:

```go
// backend/pkg/proxy/metric_aggregator.go
package proxy

type MetricAggregator struct {
    promClient prometheus.Client
}

// CalculateP95Latency computes p95 latency with fallback
func (m *MetricAggregator) CalculateP95Latency(namespace, workload string, timeRange time.Duration) (float64, error) {
    query := fmt.Sprintf(
        `histogram_quantile(0.95, rate(model_inference_duration_seconds_bucket{namespace="%s",workload="%s"}[%s]))`,
        namespace, workload, timeRange,
    )

    result, err := m.promClient.Query(query)
    if err != nil {
        // Fallback: Try simplified query
        return m.fallbackLatencyQuery(namespace, workload, timeRange)
    }

    return extractScalarValue(result), nil
}
```

### Creating a Dashboard Template

See "Creating a New Dashboard Template" section above for complete workflow.

### Testing Authentication/Authorization

**Unit Test for RBAC Middleware**:

```go
// backend/pkg/auth/rbac_middleware_test.go
package auth

import (
    "testing"
    "github.com/labstack/echo/v4"
    "github.com/stretchr/testify/assert"
)

func TestRBACMiddleware_AllowsAuthorizedUser(t *testing.T) {
    e := echo.New()
    req := httptest.NewRequest("GET", "/api/v1/projects/openshift-ai/dashboards", nil)
    rec := httptest.NewRecorder()
    c := e.NewContext(req, rec)

    // Set user with Dashboard:read permission
    user := &User{
        Email: "test@example.com",
        Permissions: map[string][]string{
            "openshift-ai": {"Dashboard:read"},
        },
    }
    c.Set("user", user)

    middleware := RBACMiddleware("Dashboard:read")
    handler := middleware(func(c echo.Context) error {
        return c.String(200, "OK")
    })

    err := handler(c)
    assert.NoError(t, err)
    assert.Equal(t, 200, rec.Code)
}

func TestRBACMiddleware_DeniesUnauthorizedUser(t *testing.T) {
    // Similar setup but user lacks permission
    user := &User{
        Email: "test@example.com",
        Permissions: map[string][]string{}, // No permissions
    }
    c.Set("user", user)

    middleware := RBACMiddleware("Dashboard:create")
    handler := middleware(func(c echo.Context) error {
        return c.String(200, "OK")
    })

    err := handler(c)
    assert.Error(t, err)
    httpErr := err.(*echo.HTTPError)
    assert.Equal(t, 403, httpErr.Code)
}
```

---

## Testing

### Unit Tests

**Backend (Go)**:

```bash
cd backend/

# Run all tests
go test ./...

# Run with coverage
go test -v -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Run specific package
go test -v ./pkg/auth/

# Run specific test
go test -v -run TestRBACMiddleware ./pkg/auth/
```

**Frontend (TypeScript)**:

```bash
cd frontend/

# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run in watch mode
npm test -- --watch

# Run specific test file
npm test -- DashboardBuilder.test.tsx
```

### Contract Tests (Pact)

**Setup Pact Broker** (local development):

```bash
docker run -d --name pact-broker \
  -p 9292:9292 \
  -e PACT_BROKER_DATABASE_URL=sqlite:///pact_broker.sqlite \
  pactfoundation/pact-broker:latest
```

**Run Consumer Tests** (Frontend):

```bash
cd tests/contract/

# Run consumer tests and publish pacts
npm run test:pact

# Pacts are published to http://localhost:9292
```

**Run Provider Verification** (Backend):

```bash
cd tests/contract/

# Verify provider against published pacts
go test -v ./api_consumer_test.go

# Expected output:
# Verifying pact between perses-dashboard-ui and perses-dashboard-api
# ✓ Dashboard list endpoint
# ✓ Dashboard get endpoint
# ✓ Dashboard create endpoint
```

### E2E Tests (Playwright)

```bash
cd tests/e2e/

# Run all E2E tests
npx playwright test

# Run with UI (for debugging)
npx playwright test --ui

# Run specific browser
npx playwright test --project=chromium

# Generate test report
npx playwright show-report
```

### Resilience Tests (Chaos Mesh)

Test system behavior under failure conditions:

```bash
# Install Chaos Mesh on OpenShift
kubectl apply -f https://mirrors.chaos-mesh.org/v2.6.0/install.yaml

# Apply database failure scenario
kubectl apply -f tests/chaos/db-network-delay.yaml
```

**Example Chaos Experiment**:

```yaml
# tests/chaos/db-network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: db-network-delay
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - openshift-ai
    labelSelectors:
      app: postgresql
  delay:
    latency: "500ms"
    correlation: "50"
  duration: "5m"
```

---

## Troubleshooting

### Common Setup Issues

#### Database Connection Failed

**Symptom**: `ERROR: failed to connect to database: connection refused`

**Solution**:
```bash
# Check PostgreSQL is running
docker ps | grep postgres

# Verify connection details
psql -h localhost -U perses -d perses_dev

# Check DATABASE_URL format
echo $DATABASE_URL
# Expected: postgresql://perses:password@localhost:5432/perses_dev?sslmode=disable
```

#### Migration Version Mismatch

**Symptom**: `ERROR: Dirty database version 1. Fix and force version`

**Solution**:
```bash
# Force version reset (development only!)
migrate -database "${DATABASE_URL}" -path db/migrations force 1

# Re-run migrations
migrate -database "${DATABASE_URL}" -path db/migrations up
```

#### OpenShift OAuth Authentication Failed

**Symptom**: `401 Unauthorized: invalid token`

**Solution**:
```bash
# Verify OpenShift login
oc whoami

# Get fresh token
export OPENSHIFT_TOKEN=$(oc whoami -t)

# Test token validity
curl -H "Authorization: Bearer $OPENSHIFT_TOKEN" \
  https://api.your-cluster.openshift.com:6443/apis/user.openshift.io/v1/users/~
```

#### Frontend Can't Connect to Backend

**Symptom**: `Network Error: Failed to fetch`

**Solution**:
```bash
# Check backend is running
curl http://localhost:8080/health

# Verify CORS settings in backend
# backend/cmd/server/main.go should include:
e.Use(middleware.CORS())

# Check frontend API URL configuration
# frontend/src/services/api-client.ts
const API_BASE_URL = process.env.VITE_API_URL || 'http://localhost:8080';
```

#### Prometheus Metrics Not Loading

**Symptom**: `No data available` in charts

**Solution**:
```bash
# Verify Prometheus is accessible
curl -k https://thanos-querier.openshift-monitoring.svc:9091/api/v1/query?query=up

# Check namespace filter in query
# All queries MUST include: namespace="openshift-ai"

# Verify RBAC permissions for Prometheus
oc adm policy add-cluster-role-to-user cluster-monitoring-view testuser
```

#### Hot Reload Not Working

**Symptom**: Frontend changes not reflected in browser

**Solution**:
```bash
# Clear Vite cache
cd frontend/
rm -rf node_modules/.vite

# Restart dev server
npm run dev
```

---

## Next Steps

Now that you have the development environment running, explore:

1. **Full API Documentation**: Review `/specs/001-perses-dashboard/contracts/openapi.yaml` in Swagger UI
2. **Data Model**: Read `/specs/001-perses-dashboard/data-model.md` for database schema details
3. **Architecture Deep Dive**: See `/specs/001-perses-dashboard/plan.md` for scalability and security architecture
4. **Perses Documentation**: Visit [perses.dev/docs](https://perses.dev/docs) for upstream framework documentation
5. **OpenShift AI Platform**: Explore [OpenShift AI Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed)

### Hands-On Tutorials

**Tutorial 1**: Create your first custom dashboard from scratch

```bash
# Follow the interactive tutorial
npm run tutorial:dashboard-creation
```

**Tutorial 2**: Implement a new metric aggregation

```bash
# Step-by-step guide for backend metric queries
go run tutorials/metric_aggregation/main.go
```

**Tutorial 3**: Build a custom chart plugin

```bash
# Frontend plugin development tutorial
cd frontend/
npm run tutorial:custom-chart
```

### Join the Community

- **Slack**: #openshift-ai-platform
- **GitHub**: [openshift/openshift-ai-dashboard](https://github.com/openshift/openshift-ai-dashboard)
- **Weekly Sync**: Thursdays 10am ET (calendar invite on Slack)
- **Office Hours**: Tuesdays 2pm ET for developer questions

### Contributing

Ready to contribute? See `CONTRIBUTING.md` for:
- Code style guidelines
- Pull request process
- Commit message conventions
- Review checklist

---

**Questions?** Contact the OpenShift AI Platform Team at openshift-ai-dev@redhat.com

**Found a bug?** File an issue at [github.com/openshift/openshift-ai-dashboard/issues](https://github.com/openshift/openshift-ai-dashboard/issues)

---

**Document Version**: 1.0.0
**Last Updated**: 2025-11-14
**Maintained By**: OpenShift AI Platform Team
