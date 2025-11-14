# Implementation Plan: Perses Observability Dashboard

**Branch**: `001-perses-dashboard` | **Date**: 2025-11-14 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-perses-dashboard/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

This feature implements a comprehensive observability dashboard for OpenShift AI using the Perses visualization framework. The primary requirement is to provide real-time and historical monitoring capabilities for ML workloads (model serving, training jobs, notebook servers) through an integrated, customizable dashboard interface. The technical approach leverages the existing Perses open-source project as the foundation, extending it with OpenShift AI-specific integrations for authentication, metric collection, and workload-aware templates. The implementation will deliver real-time metric visualization (<30s latency), custom dashboard creation capabilities, and historical trend analysis while meeting strict performance targets (50+ concurrent users, <3s load times) and 99.9% configuration persistence reliability.

## Technical Context

**Language/Version**:
- Backend: Go 1.21+ (Perses core requirement)
- Frontend: TypeScript 5.4+ with React 18.2, Node.js v22+

**Primary Dependencies**:
- `github.com/perses/perses` (v0.53.0+) - Core dashboard framework
- `github.com/labstack/echo/v4` - REST API framework
- `k8s.io/client-go` - Kubernetes/OpenShift integration
- `github.com/prometheus/client_golang` - Prometheus metrics client
- `@perses-dev/components`, `@perses-dev/dashboards` - UI component library
- `@mui/material` v6+ - Material-UI for consistent OpenShift design
- `@tanstack/react-query` v4 - API state management

**Storage**:
- PostgreSQL (recommended) or MySQL for dashboard configuration persistence (FR-006: 99.9% reliability requirement)
- Prometheus/Thanos TSDB for metric data with 7+ day retention (FR-008)
- **NEEDS CLARIFICATION**: Backup and disaster recovery requirements for dashboard configurations

**Testing**:
- Backend: Go standard `testing` package + `github.com/gavv/httpexpect/v2` for API integration tests (70% coverage target)
- Frontend: Jest v29.5 + React Testing Library + Playwright v1.54 for E2E (60% UI, 80% critical path coverage)
- Performance: Load testing for 50+ concurrent users with <3s p95 load time (SC-003, SC-005)

**Target Platform**:
- OpenShift 4.x (Kubernetes 1.28+) with Operator pattern deployment
- Containerized (OCI images, distroless/UBI base)
- Integration: OpenShift OAuth authentication, RBAC authorization (FR-013)

**Project Type**: Web application (hybrid - embedded Perses components in OpenShift AI console)
- Backend: RESTful API + WebSocket for real-time updates (FR-002: 15s default refresh)
- Frontend: React SPA with Perses dashboard renderer
- Integration: Prometheus/Thanos query proxy

**Performance Goals**:
- Metric latency: <30s end-to-end (SC-002)
- Dashboard load: <3s p95 for 8-chart dashboards (SC-005)
- Concurrent users: 50+ without degradation (SC-003)
- Dashboard creation: <5min for 3-chart dashboard (SC-001)
- Historical queries: <5s p95 for 7-day range (SC-006)
- Persistence SLA: 99.9% (SC-007)

**Constraints**:
- Must use Perses v0.53.0+ API contract (chart types limited to time series, area, bar, stat, gauge, table per FR-004)
- OpenShift SCCs: no privileged containers or root filesystem access
- Memory: <2GB per pod; API response: <500ms p95
- Browser support: Chrome/Edge 90+, Firefox 88+, Safari 14+ (no IE11)
- Authentication: OpenShift OAuth only (no custom auth)
- Authorization: Namespace-scoped via OpenShift RBAC
- **NEEDS CLARIFICATION**: Timezone handling strategy (user local time vs UTC vs configurable)
- **NEEDS CLARIFICATION**: Dashboard migration strategy for schema changes during upgrades

**Scale/Scope**:
- Users: 50-100 concurrent (initial), scale to 500+ within 12 months
- Dashboards: Support 500+ custom dashboards cluster-wide, 100+ per namespace
- Metrics: 1000+ unique time series per workload
- Workloads: 100+ concurrent ML workloads
- MVP scope: P1 user stories (real-time monitoring + custom dashboards)
- Phase 2: P2 (historical trends), Phase 3: P3 (alerts + correlation)
- **NEEDS CLARIFICATION (FR-019)**: Metric backend support - Prometheus only (MVP) vs Prometheus + Thanos (recommended for historical data) vs additional backends
- **NEEDS CLARIFICATION (FR-020)**: Dashboard sharing mechanism - URL-based tokens, RBAC permissions, or hybrid; scope (project-level vs cluster-wide)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### 1. Modularity & Separation of Concerns

**Status**: ✅ PASS

**Assessment**: Architecture properly separated across backend API (Go/REST/WebSocket), frontend UI (React/TypeScript/Perses), and integration layer (k8s client-go, Prometheus client). Clear delineation allows independent evolution and metric backend substitution.

**Action Required**: Document interface contracts between layers during Phase 1.

### 2. Test Coverage

**Status**: ⚠️ WARNING

**Assessment**: Testing strategy outlined (70% backend, 60-80% frontend, performance tests) but lacks contract testing, resilience testing, and automated mapping of acceptance scenarios. Gap puts 99.9% persistence SLA (SC-007) at risk.

**Action Required**:
1. Add contract testing framework (Pact or similar) for API/UI interface validation
2. Define resilience testing for dependency failures (DB, TSDB, auth)
3. Create performance test suite for 500+ dashboard scale
4. Map acceptance scenarios to automated E2E tests before Phase 2

### 3. Security

**Status**: ✅ PASS (with clarifications needed)

**Assessment**: OpenShift OAuth auth, namespace-scoped RBAC, SCCs enforced. SC-010 requires zero security violations.

**Outstanding Clarifications**:
- FR-020: Dashboard sharing security model
- Audit logging for dashboard access/modifications (compliance requirement)
- Data encryption (TLS in-transit, at-rest for configs)

**Action Required**:
1. Clarify dashboard sharing security during Phase 0
2. Add audit logging to functional requirements
3. Document encryption requirements (TLS 1.2+, PostgreSQL at-rest)

### 4. Performance

**Status**: ✅ PASS

**Assessment**: Clear, measurable targets: <30s metric latency (SC-002), 50+ concurrent users (SC-003), <3s p95 load (SC-005), <5min dashboard creation (SC-001). Targets differentiate from competitors (SageMaker/Vertex AI: 5-10s load times).

**Action Required**: Validate with realistic load distribution testing.

### 5. Scalability

**Status**: ⚠️ WARNING

**Assessment**: Scale targets defined (500+ users in 12mo, 500+ dashboards, 1000+ time series) but scaling strategy underspecified. Missing: horizontal scaling architecture, PostgreSQL scaling, caching layer, WebSocket connection scaling, high-cardinality query optimization.

**Action Required**:
1. Define horizontal scaling (stateless API pods, load balancer, sticky sessions if needed)
2. Add caching layer (Redis/Memcached) for dashboards and metric queries
3. Document DB connection pooling and read-replica strategy
4. Consider Server-Sent Events as WebSocket alternative for better scale

### 6. Dependency Management

**Status**: ✅ PASS

**Assessment**: Dependencies justified and manageable. Perses (Apache 2.0, active CNCF project) avoids 6-12mo custom dev. Prometheus/Thanos already in platform. PostgreSQL battle-tested. All dependencies stable and widely adopted.

**Action Required**: Document dependency upgrade policy during Phase 0 (quarterly security patches, biannual feature upgrades).

### 7. Open Source Compliance

**Status**: ✅ PASS

**Assessment**: Perses under Apache 2.0 (permissive, Red Hat compatible, includes patent grant). Compliance requires including LICENSE/NOTICE files and marking modifications.

**Action Required**: Create compliance checklist during Phase 0. Ensure build pipeline includes license files in container images.

### 8. Operational Excellence

**Status**: ⚠️ WARNING

**Assessment**: Missing system instrumentation, SLIs/SLOs, alerting strategy, dashboard schema migration plan, backup/DR for configs, and SRE runbooks.

**Action Required**:
1. Add FR for dashboard system instrumentation (Prometheus metrics, structured logs)
2. Define SLIs/SLOs (e.g., 99.9% availability, p95 <3s)
3. Create dashboard migration strategy (backward-compatible schemas, versioned APIs)
4. Document backup/DR for PostgreSQL configs
5. Develop operational runbooks during Phase 2

### Summary

**Overall Status**: ⚠️ CONDITIONAL PASS

Solid architectural foundations with proper separation, justified dependencies, and clear performance targets. Security and licensing well-addressed. However, gaps in operational readiness and scalability planning must be resolved before production.

**Critical Path Items**:
- **P1 - Test Coverage**: Contract + resilience testing
- **P1 - Operational Excellence**: System instrumentation, SLIs/SLOs, DR plan
- **P2 - Scalability**: Horizontal scaling architecture, caching
- **P2 - Security**: Dashboard sharing model, audit logging

**Recommendation**: PROCEED to Phase 0 research with mandatory P1 resolution before Phase 1 design. Re-evaluate after Phase 1.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

This is a web application extending the existing Perses project located at `/workspace/sessions/agentic-session-1763159109/workspace/perses/`

```text
perses/  (existing upstream Perses repository)
├── cmd/
│   └── perses/               # Existing Perses server entrypoint
├── pkg/
│   ├── model/
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── dashboard/  # Core dashboard models
│   │   │   │   ├── datasource/ # Datasource definitions
│   │   │   │   └── ...
│   │   └── ...
│   └── client/               # Perses Go client library
├── internal/
│   ├── api/
│   │   ├── core/            # Core API handlers
│   │   ├── crypto/
│   │   ├── database/        # PostgreSQL/MySQL persistence layer
│   │   └── ...
│   └── ...
├── go-sdk/                  # Go SDK for Perses API
└── ui/
    ├── app/                 # Main Perses UI application
    ├── components/          # @perses-dev/components package
    ├── dashboards/          # @perses-dev/dashboards package
    ├── plugin-system/       # Plugin architecture
    └── core/                # Core UI utilities

# OpenShift AI Extension (NEW - to be created)
openshift-ai-dashboard/
├── backend/
│   ├── pkg/
│   │   ├── auth/
│   │   │   ├── openshift_oauth.go     # OpenShift OAuth integration
│   │   │   └── rbac_middleware.go     # Namespace-scoped RBAC
│   │   ├── workload/
│   │   │   ├── model_serving.go       # Model serving workload metadata
│   │   │   ├── training_job.go        # Training job metadata
│   │   │   └── notebook.go            # Notebook server metadata
│   │   ├── templates/
│   │   │   ├── model_serving_dashboard.go  # Pre-built dashboard templates
│   │   │   ├── training_dashboard.go
│   │   │   └── notebook_dashboard.go
│   │   ├── proxy/
│   │   │   ├── prometheus_proxy.go    # Prometheus query proxy with auth
│   │   │   └── metric_aggregator.go   # Metric aggregation logic
│   │   └── api/
│   │       ├── dashboard_handler.go   # Dashboard CRUD endpoints
│   │       ├── template_handler.go    # Template instantiation
│   │       └── metrics_handler.go     # Real-time metric queries
│   ├── cmd/
│   │   └── server/
│   │       └── main.go                # Server entrypoint
│   ├── go.mod                         # Depends on perses/go-sdk
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── DashboardBuilder/      # Custom dashboard creation UI
│   │   │   ├── WorkloadSelector/      # OpenShift AI workload picker
│   │   │   ├── TemplateGallery/       # Template selection UI
│   │   │   └── PersesDashboard/       # Wrapper around @perses-dev/dashboards
│   │   ├── pages/
│   │   │   ├── DashboardList/         # Dashboard listing
│   │   │   ├── DashboardView/         # Dashboard rendering page
│   │   │   └── DashboardEditor/       # Dashboard edit mode
│   │   ├── services/
│   │   │   ├── api-client.ts          # Backend API client
│   │   │   ├── auth.ts                # OpenShift OAuth token handling
│   │   │   └── websocket.ts           # Real-time updates via WebSocket
│   │   ├── hooks/
│   │   │   ├── useDashboard.ts        # Dashboard state management
│   │   │   ├── useMetrics.ts          # Metric query hook
│   │   │   └── useWorkloads.ts        # Workload metadata hook
│   │   └── App.tsx
│   ├── package.json                   # Depends on @perses-dev/* packages
│   └── tsconfig.json
├── operator/                          # Kubernetes Operator (optional Phase 2)
│   ├── config/
│   │   ├── crd/                       # Custom Resource Definitions
│   │   ├── rbac/                      # RBAC manifests
│   │   └── manager/                   # Operator deployment
│   ├── controllers/
│   │   └── dashboard_controller.go    # Dashboard CR reconciler
│   └── go.mod
└── tests/
    ├── contract/                      # Contract tests (Pact)
    │   ├── api_consumer_test.go       # Frontend-Backend contract
    │   └── pacts/
    ├── integration/                   # Integration tests
    │   ├── dashboard_crud_test.go
    │   ├── auth_integration_test.go
    │   └── metric_query_test.go
    ├── e2e/                           # End-to-end tests (Playwright)
    │   ├── dashboard_creation.spec.ts
    │   ├── real_time_updates.spec.ts
    │   └── template_instantiation.spec.ts
    └── performance/                   # Load tests
        ├── concurrent_users_test.go
        └── dashboard_rendering_test.ts
```

**Structure Decision**:

We are extending the existing Perses project (web application architecture) with OpenShift AI-specific components in a separate `openshift-ai-dashboard/` directory. This approach:

1. **Preserves upstream Perses**: The core `/workspace/.../perses/` directory remains unchanged, allowing us to pull upstream updates
2. **Extension pattern**: New OpenShift AI integration code lives in `openshift-ai-dashboard/` with clear separation
3. **Dependency flow**: Our backend depends on `perses/go-sdk` for API models and client; frontend depends on `@perses-dev/*` npm packages
4. **Deployment strategy**: Option to deploy as sidecar to Perses server OR standalone server embedding Perses libraries (to be decided in Phase 0)

This aligns with the web application structure (Option 2) but adapted for an extension/integration project rather than greenfield development.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations requiring justification. All warnings (Test Coverage, Scalability, Security clarifications, Operational Excellence) have remediation actions defined and will be addressed during Phase 0 research and Phase 1 design.
