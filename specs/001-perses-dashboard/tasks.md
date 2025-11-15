# Tasks: Perses Observability Dashboard

**Feature Branch**: `001-perses-dashboard`
**Input**: Design documents from `/workspace/sessions/agentic-session-1763159109/workspace/workflows/ootb-ambient-workflows/specs/001-perses-dashboard/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml, quickstart.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure) ✅ COMPLETE

**Purpose**: Project initialization and basic structure

- [X] T001 Create project directory structure at `/workspace/perses/openshift-ai-dashboard/` with backend/, frontend/, tests/, operator/ subdirectories
- [X] T002 [P] Initialize Go module with `go mod init github.com/openshift/openshift-ai-dashboard` in `/workspace/perses/openshift-ai-dashboard/backend/go.mod`
- [X] T003 [P] Initialize npm project with TypeScript/React in `/workspace/perses/openshift-ai-dashboard/frontend/package.json`
- [X] T004 [P] Add Perses SDK dependencies to `/workspace/perses/openshift-ai-dashboard/backend/go.mod` (github.com/perses/perses@v0.53.0+)
- [X] T005 [P] Add Perses UI dependencies to `/workspace/perses/openshift-ai-dashboard/frontend/package.json` (@perses-dev/components, @perses-dev/dashboards)
- [X] T006 [P] Configure ESLint, Prettier for TypeScript in `/workspace/perses/openshift-ai-dashboard/frontend/.eslintrc.json`
- [X] T007 [P] Configure golangci-lint for Go in `/workspace/perses/openshift-ai-dashboard/backend/.golangci.yml`
- [X] T008 Create Dockerfile for backend in `/workspace/perses/openshift-ai-dashboard/backend/Dockerfile` (distroless/UBI base)
- [X] T009 Create Dockerfile for frontend in `/workspace/perses/openshift-ai-dashboard/frontend/Dockerfile` (nginx runtime)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Database Setup

- [X] T010 Create PostgreSQL schema migration V001_initial_schema.up.sql in `/workspace/openshift-ai-dashboard/backend/db/migrations/` (dashboard, dashboard_template, user_preferences, share_token, audit_log tables per data-model.md)
- [X] T011 Create PostgreSQL schema migration V001_initial_schema.down.sql in `/workspace/openshift-ai-dashboard/backend/db/migrations/`
- [X] T012 Implement database connection manager in `/workspace/openshift-ai-dashboard/backend/pkg/database/pool.go` (connection pooling, health checks, 50 max connections)
- [X] T013 Create database health check endpoint in `/workspace/openshift-ai-dashboard/backend/pkg/database/health.go` (/api/v1/health, /ready, /live endpoints)

### Authentication & Authorization

- [X] T014 [P] Implement OpenShift OAuth client in `/workspace/openshift-ai-dashboard/backend/pkg/auth/openshift_oauth.go` (token validation, user info extraction)
- [X] T015 [P] Implement RBAC middleware in `/workspace/openshift-ai-dashboard/backend/pkg/auth/rbac_middleware.go` (namespace-scoped permissions: Dashboard:create, Dashboard:read, Dashboard:update, Dashboard:delete, Dashboard:share)
- [X] T016 [P] Create User model in `/workspace/openshift-ai-dashboard/backend/pkg/auth/user.go` (email, permissions map)
- [X] T017 [P] Implement JWT token validation in `/workspace/openshift-ai-dashboard/backend/pkg/auth/jwt_validator.go` (parse and validate OpenShift service account tokens)

### API Infrastructure

- [X] T018 [P] Setup Echo REST framework in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go` (router, middleware chain, health endpoint)
- [X] T019 [P] Implement error handling middleware in `/workspace/openshift-ai-dashboard/backend/pkg/middleware/error.go` (structured error responses per openapi.yaml)
- [X] T020 [P] Implement logging middleware in `/workspace/openshift-ai-dashboard/backend/pkg/middleware/logger.go` (structured logs with request ID, user, duration)
- [X] T021 [P] Implement CORS middleware in `/workspace/openshift-ai-dashboard/backend/pkg/middleware/cors.go` (frontend integration)
- [X] T022 [P] Create API router with versioned routes in `/workspace/openshift-ai-dashboard/backend/pkg/api/router.go` (all routes under /api/v1/)

### Prometheus/Thanos Integration

- [X] T023 [P] Implement Prometheus proxy client in `/workspace/perses/openshift-ai-dashboard/backend/pkg/datasource/prometheus_client.go` (query client with ServiceAccount auth, connection pooling)
- [X] T024 [P] Create datasource configuration loader in `/workspace/perses/openshift-ai-dashboard/backend/pkg/datasource/config.go` (loads Thanos Querier endpoint from env, service discovery)

### Base Models & DAOs

- [X] T025 [P] Create Dashboard model in `/workspace/openshift-ai-dashboard/backend/pkg/models/dashboard.go` (maps to dashboard table per data-model.md)
- [X] T026 [P] Create DashboardTemplate model in `/workspace/openshift-ai-dashboard/backend/pkg/models/dashboard_template.go`
- [X] T027 [P] Create ShareToken model in `/workspace/openshift-ai-dashboard/backend/pkg/models/share_token.go`
- [X] T028 [P] Create AuditLogEntry model in `/workspace/openshift-ai-dashboard/backend/pkg/models/audit_log.go`
- [X] T029 [P] Implement Dashboard DAO in `/workspace/openshift-ai-dashboard/backend/pkg/dao/dashboard_dao.go` (CRUD operations with optimistic locking)
- [X] T030 [P] Implement DashboardTemplate DAO in `/workspace/openshift-ai-dashboard/backend/pkg/dao/dashboard_template_dao.go`
- [X] T031 [P] Implement ShareToken DAO in `/workspace/openshift-ai-dashboard/backend/pkg/dao/share_token_dao.go`
- [X] T032 [P] Implement AuditLog DAO in `/workspace/openshift-ai-dashboard/backend/pkg/dao/audit_log_dao.go`

### System Templates (FR-003)

- [ ] T033 Create Model Serving template in `/workspace/openshift-ai-dashboard/backend/pkg/templates/model_serving_dashboard.go` (6 panels: inference latency p50/p95/p99, request rate, error rate, CPU/memory)
- [ ] T034 Create Training Job template in `/workspace/openshift-ai-dashboard/backend/pkg/templates/training_dashboard.go` (8 panels: loss, accuracy, GPU utilization, data throughput)
- [ ] T035 Create Notebook Server template in `/workspace/openshift-ai-dashboard/backend/pkg/templates/notebook_dashboard.go` (5 panels: CPU, memory, kernel status, I/O)
- [ ] T036 Create Data Pipeline template in `/workspace/openshift-ai-dashboard/backend/pkg/templates/data_pipeline_dashboard.go` (7 panels: throughput, latency, error rate, volume)
- [ ] T037 Implement template registry in `/workspace/openshift-ai-dashboard/backend/pkg/templates/registry.go` (loads and seeds system templates on startup)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Real-time Metrics Monitoring (Priority: P1) 🎯 MVP

**Goal**: Enable users to monitor real-time metrics from deployed models and training jobs with <15s latency (FR-002, SC-002)

**Independent Test**: Deploy a single model to OpenShift AI, configure dashboard to display CPU/memory/latency metrics, verify metrics update within 15 seconds of emission

### Implementation for User Story 1

#### Backend - Dashboard CRUD (openapi.yaml paths: /projects/{project}/dashboards)

- [ ] T038 [P] [US1] Implement GET /projects/{project}/dashboards handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/dashboard_handler.go` (list dashboards with pagination, filtering by state/template_id)
- [ ] T039 [P] [US1] Implement POST /projects/{project}/dashboards handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/dashboard_handler.go` (create dashboard with Perses schema validation, namespace filter validation)
- [ ] T040 [P] [US1] Implement GET /projects/{project}/dashboards/{id} handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/dashboard_handler.go` (retrieve dashboard with ETag support)
- [ ] T041 [P] [US1] Implement PUT /projects/{project}/dashboards/{id} handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/dashboard_handler.go` (update with optimistic locking, version conflict detection)
- [ ] T042 [P] [US1] Implement DELETE /projects/{project}/dashboards/{id} handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/dashboard_handler.go` (soft delete - sets state='archived')
- [ ] T043 [US1] Register dashboard routes in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go` (apply auth + RBAC middleware)

#### Backend - Metrics Query (openapi.yaml path: /metrics/query)

- [ ] T044 [P] [US1] Implement POST /metrics/query handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/metrics_handler.go` (proxy to Thanos, inject namespace filter, validate PromQL)
- [ ] T045 [US1] Implement metric query validator in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/query_validator.go` (ensure namespace filter present, max 500 time series)
- [ ] T046 [US1] Implement query result caching in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/query_cache.go` (Redis-backed, 15s TTL per query hash)
- [ ] T047 [US1] Register metrics routes in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go`

#### Backend - Real-time Updates (FR-002: 15s refresh)

- [ ] T048 [US1] Implement Server-Sent Events endpoint in `/workspace/openshift-ai-dashboard/backend/pkg/api/stream_handler.go` (pushes metric updates every 15s)
- [ ] T049 [US1] Implement metric polling service in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/metric_poller.go` (background goroutine, queries Thanos at refresh interval)
- [ ] T050 [US1] Register SSE route in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go`

#### Frontend - Dashboard List & View

- [ ] T051 [P] [US1] Create DashboardList component in `/workspace/openshift-ai-dashboard/frontend/src/pages/DashboardList/DashboardList.tsx` (displays dashboards with filter/sort, pagination)
- [ ] T052 [P] [US1] Create DashboardView component in `/workspace/openshift-ai-dashboard/frontend/src/pages/DashboardView/DashboardView.tsx` (renders Perses dashboard with real-time updates)
- [ ] T053 [P] [US1] Implement useDashboard hook in `/workspace/openshift-ai-dashboard/frontend/src/hooks/useDashboard.ts` (TanStack Query for dashboard CRUD)
- [ ] T054 [P] [US1] Implement useMetrics hook in `/workspace/openshift-ai-dashboard/frontend/src/hooks/useMetrics.ts` (queries metrics via /metrics/query endpoint)
- [ ] T055 [US1] Integrate PersesDashboard renderer in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/PersesDashboard.tsx` (wraps @perses-dev/dashboards component)
- [ ] T056 [US1] Implement SSE client in `/workspace/openshift-ai-dashboard/frontend/src/services/websocket.ts` (subscribes to /stream endpoint, auto-reconnect)
- [ ] T057 [US1] Connect SSE to chart updates in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/PersesDashboard.tsx` (update chart data on SSE events)

#### Frontend - Time Range Selector (FR-008)

- [ ] T058 [P] [US1] Create TimeRangeSelector component in `/workspace/openshift-ai-dashboard/frontend/src/components/TimeRangeSelector/TimeRangeSelector.tsx` (dropdown: last_15m, last_1h, last_6h, last_24h, last_7d + date picker)
- [ ] T059 [US1] Integrate TimeRangeSelector in DashboardView (triggers metric re-query on change)

#### Workload Discovery (FR-014: Drill-down)

- [ ] T060 [P] [US1] Implement workload discovery service in `/workspace/openshift-ai-dashboard/backend/pkg/workload/model_serving.go` (queries Kubernetes API for InferenceService CRs)
- [ ] T061 [P] [US1] Implement training job discovery in `/workspace/openshift-ai-dashboard/backend/pkg/workload/training_job.go` (queries for PyTorchJob, TFJob CRs)
- [ ] T062 [P] [US1] Implement notebook discovery in `/workspace/openshift-ai-dashboard/backend/pkg/workload/notebook.go` (queries for StatefulSets with label app=jupyter)
- [ ] T063 [US1] Create WorkloadSelector component in `/workspace/openshift-ai-dashboard/frontend/src/components/WorkloadSelector/WorkloadSelector.tsx` (autocomplete dropdown for workload selection)
- [ ] T064 [US1] Integrate WorkloadSelector in DashboardView (filters metrics by selected workload)

**Checkpoint**: User Story 1 complete - users can view real-time metrics for deployed models with <15s latency

---

## Phase 4: User Story 2 - Custom Dashboard Creation (Priority: P2)

**Goal**: Enable users to create custom dashboards from scratch or templates with 3-5 chart types (FR-004, SC-001: <5min creation time)

**Independent Test**: Create new dashboard from Model Serving template, add 3 charts (time series, gauge, stat panel), save, verify persistence across browser sessions

### Implementation for User Story 2

#### Backend - Template Management (openapi.yaml paths: /templates)

- [ ] T065 [P] [US2] Implement GET /templates handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/template_handler.go` (list templates with category filter)
- [ ] T066 [P] [US2] Implement GET /templates/{id} handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/template_handler.go` (retrieve template details)
- [ ] T067 [P] [US2] Implement POST /projects/{project}/dashboards/from-template handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/template_handler.go` (instantiate template with variable substitution)
- [ ] T068 [US2] Implement template variable substitution engine in `/workspace/openshift-ai-dashboard/backend/pkg/templates/variable_substitution.go` (replaces {{.namespace}}, {{.model_name}} in template JSON)
- [ ] T069 [US2] Register template routes in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go`

#### Backend - Dashboard Export (FR-012)

- [ ] T070 [US2] Implement GET /projects/{project}/dashboards/{id}/export handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/dashboard_handler.go` (exports dashboard as standalone Perses JSON)

#### Frontend - Dashboard Builder

- [ ] T071 [P] [US2] Create DashboardBuilder component in `/workspace/openshift-ai-dashboard/frontend/src/components/DashboardBuilder/DashboardBuilder.tsx` (drag-drop canvas for chart layout)
- [ ] T072 [P] [US2] Create ChartTypeSelector component in `/workspace/openshift-ai-dashboard/frontend/src/components/DashboardBuilder/ChartTypeSelector.tsx` (gallery: TimeSeriesChart, GaugeChart, StatPanel, TableView, BarChart per FR-004)
- [ ] T073 [P] [US2] Create MetricQueryBuilder component in `/workspace/openshift-ai-dashboard/frontend/src/components/DashboardBuilder/MetricQueryBuilder.tsx` (PromQL editor with autocomplete)
- [ ] T074 [P] [US2] Create ChartConfigPanel component in `/workspace/openshift-ai-dashboard/frontend/src/components/DashboardBuilder/ChartConfigPanel.tsx` (configure colors, thresholds, axes per FR-018)
- [ ] T075 [US2] Create DashboardEditor page in `/workspace/openshift-ai-dashboard/frontend/src/pages/DashboardEditor/DashboardEditor.tsx` (integrates DashboardBuilder with save/cancel actions)
- [ ] T076 [US2] Implement dashboard save logic in `/workspace/openshift-ai-dashboard/frontend/src/hooks/useDashboard.ts` (calls POST /dashboards or PUT /dashboards/{id})

#### Frontend - Template Gallery

- [ ] T077 [P] [US2] Create TemplateGallery component in `/workspace/openshift-ai-dashboard/frontend/src/components/TemplateGallery/TemplateGallery.tsx` (grid view of templates with preview cards)
- [ ] T078 [P] [US2] Create TemplateVariableForm component in `/workspace/openshift-ai-dashboard/frontend/src/components/TemplateGallery/TemplateVariableForm.tsx` (input form for namespace, workload_name, etc.)
- [ ] T079 [US2] Implement template instantiation flow in TemplateGallery (select template → fill variables → create dashboard)

#### Chart Type Implementations (FR-004)

- [ ] T080 [P] [US2] Configure TimeSeriesChart plugin in `/workspace/openshift-ai-dashboard/frontend/src/plugins/TimeSeriesChart.tsx` (uses @perses-dev/components)
- [ ] T081 [P] [US2] Configure GaugeChart plugin in `/workspace/openshift-ai-dashboard/frontend/src/plugins/GaugeChart.tsx`
- [ ] T082 [P] [US2] Configure StatPanel plugin in `/workspace/openshift-ai-dashboard/frontend/src/plugins/StatPanel.tsx`
- [ ] T083 [P] [US2] Configure TableView plugin in `/workspace/openshift-ai-dashboard/frontend/src/plugins/TableView.tsx`
- [ ] T084 [P] [US2] Configure BarChart plugin in `/workspace/openshift-ai-dashboard/frontend/src/plugins/BarChart.tsx`
- [ ] T085 [US2] Register all plugins in `/workspace/openshift-ai-dashboard/frontend/src/plugins/index.ts`

**Checkpoint**: User Story 2 complete - users can create custom dashboards in <5 minutes using templates or from scratch

---

## Phase 5: User Story 3 - Historical Trend Analysis (Priority: P2)

**Goal**: Enable users to view historical metrics over 7+ days with appropriate downsampling (FR-008, SC-006)

**Independent Test**: Select workload running for 24+ hours, use time range picker to view last 7 days, verify chart displays trend with appropriate granularity

### Implementation for User Story 3

#### Backend - Historical Query Optimization

- [ ] T086 [US3] Implement query router in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/query_router.go` (routes to Prometheus for <15min, Thanos for >15min)
- [ ] T087 [US3] Implement downsampling logic in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/downsampler.go` (auto-select 5m or 1h resolution based on time range)
- [ ] T088 [US3] Add caching strategy for historical queries in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/query_cache.go` (longer TTL for historical data: 5min)

#### Frontend - Historical Analysis Features

- [ ] T089 [P] [US3] Extend TimeRangeSelector with absolute date picker in `/workspace/openshift-ai-dashboard/frontend/src/components/TimeRangeSelector/TimeRangeSelector.tsx` (calendar picker for custom ranges)
- [ ] T090 [P] [US3] Implement zoom/pan controls in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/PersesDashboard.tsx` (zoom into specific time window on chart)
- [ ] T091 [P] [US3] Create TooltipDisplay component in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/TooltipDisplay.tsx` (displays exact metric values on hover)
- [ ] T092 [US3] Implement time-shift comparison mode in `/workspace/openshift-ai-dashboard/frontend/src/components/TimeRangeSelector/TimeShiftSelector.tsx` (overlay this week vs last week)

**Checkpoint**: User Story 3 complete - users can analyze historical trends over 7+ days with <5s p95 query latency

---

## Phase 6: User Story 4 - Alert Threshold Visualization (Priority: P3)

**Goal**: Display alerting thresholds and alert states on metric charts (FR-017: annotations)

**Independent Test**: Configure alert rule with CPU > 80% threshold, view dashboard, verify threshold line displayed on chart with alert state indicator

### Implementation for User Story 4

#### Backend - Alert Integration

- [ ] T093 [US4] Implement alert rule fetcher in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/alert_fetcher.go` (queries Thanos Ruler for alert rules matching dashboard metrics)
- [ ] T094 [US4] Implement alert state enricher in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/alert_enricher.go` (fetches current alert state from Alertmanager)
- [ ] T095 [US4] Extend metrics query response to include thresholds in `/workspace/openshift-ai-dashboard/backend/pkg/api/metrics_handler.go`

#### Frontend - Threshold Visualization

- [ ] T096 [P] [US4] Create ThresholdOverlay component in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/ThresholdOverlay.tsx` (renders threshold lines on charts)
- [ ] T097 [P] [US4] Create AlertStateIndicator component in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/AlertStateIndicator.tsx` (color highlight when alert firing)
- [ ] T098 [US4] Integrate threshold overlays in PersesDashboard renderer

**Checkpoint**: User Story 4 complete - users can see alert thresholds and states on charts

---

## Phase 7: User Story 5 - Multi-Metric Correlation (Priority: P3)

**Goal**: Enable synchronized viewing of multiple related metrics for troubleshooting (FR-014: drill-down)

**Independent Test**: Create dashboard with CPU, memory, network I/O, latency charts; verify synchronized crosshair and zoom across all charts

### Implementation for User Story 5

#### Frontend - Synchronized Charts

- [ ] T099 [US5] Implement synchronized crosshair in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/SynchronizedCrosshair.tsx` (hover on one chart highlights same timestamp on all charts)
- [ ] T100 [US5] Implement synchronized zoom in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/SynchronizedZoom.tsx` (zoom on one chart updates all chart time ranges)
- [ ] T101 [US5] Integrate synchronized controls in PersesDashboard renderer

**Checkpoint**: User Story 5 complete - users can correlate metrics across multiple charts during incident analysis

---

## Phase 8: Dashboard Sharing (FR-020)

**Goal**: Enable read-only dashboard sharing via ephemeral tokens (hybrid RBAC + tokens per research.md)

**Independent Test**: Generate 24h share token for dashboard, access share URL without authentication, verify read-only access with IP restrictions

### Implementation

#### Backend - Share Token Management (openapi.yaml paths: /projects/{project}/dashboards/{id}/share)

- [ ] T102 [P] Implement POST /projects/{project}/dashboards/{id}/share handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/share_handler.go` (generate UUID v4 token, validate TTL ≤7 days)
- [ ] T103 [P] Implement DELETE /projects/{project}/dashboards/{id}/share/{token_id} handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/share_handler.go` (revoke token)
- [ ] T104 [P] Implement GET /shared/{token} handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/share_handler.go` (validate token, check expiration, enforce IP restrictions)
- [ ] T105 Implement token cleanup job in `/workspace/openshift-ai-dashboard/backend/pkg/jobs/token_cleanup.go` (cron job deletes expired tokens after 30 days)
- [ ] T106 Register share routes in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go`

#### Frontend - Share Dialog

- [ ] T107 [P] Create ShareDialog component in `/workspace/openshift-ai-dashboard/frontend/src/components/ShareDialog/ShareDialog.tsx` (modal with TTL selector, IP restrictions input)
- [ ] T108 [P] Create ShareTokenDisplay component in `/workspace/openshift-ai-dashboard/frontend/src/components/ShareDialog/ShareTokenDisplay.tsx` (copy-to-clipboard URL)
- [ ] T109 Integrate ShareDialog in DashboardView (share button triggers dialog)

**Checkpoint**: Dashboard sharing complete - users can share dashboards via secure tokens with expiration and IP restrictions

---

## Phase 9: User Preferences (FR-016: data freshness)

**Goal**: Persist user-specific settings (default time range, refresh interval, timezone, favorites)

**Independent Test**: Set timezone to America/New_York, favorite 2 dashboards, verify settings persist across browser sessions

### Implementation

#### Backend - Preferences API (openapi.yaml paths: /preferences)

- [ ] T110 [P] Implement GET /preferences handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/preferences_handler.go` (retrieve user preferences)
- [ ] T111 [P] Implement PUT /preferences handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/preferences_handler.go` (update preferences)
- [ ] T112 Implement UserPreferences DAO in `/workspace/openshift-ai-dashboard/backend/internal/database/user_preferences_dao.go`
- [ ] T113 Register preferences routes in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go`

#### Frontend - Preferences UI

- [ ] T114 [P] Create PreferencesDialog component in `/workspace/openshift-ai-dashboard/frontend/src/components/PreferencesDialog/PreferencesDialog.tsx` (timezone, theme, default refresh)
- [ ] T115 [P] Implement favorite dashboard toggle in `/workspace/openshift-ai-dashboard/frontend/src/components/DashboardList/FavoriteToggle.tsx`
- [ ] T116 Integrate preferences in app layout

**Checkpoint**: User preferences complete - settings persist and apply across sessions

---

## Phase 10: Audit Logging (SC-010: zero security violations)

**Goal**: Complete audit trail for compliance (SOC2, ISO 27001, HIPAA per data-model.md)

**Independent Test**: Create dashboard, view audit logs, verify dashboard.create event logged with user, timestamp, IP, result

### Implementation

#### Backend - Audit Query API (openapi.yaml path: /audit/logs)

- [ ] T117 Implement GET /audit/logs handler in `/workspace/openshift-ai-dashboard/backend/pkg/api/audit_handler.go` (query audit logs with filters: user_id, action, resource_id, time range)
- [ ] T118 Register audit routes in `/workspace/openshift-ai-dashboard/backend/cmd/server/main.go` (requires AuditLog:read permission - admin only)

#### Frontend - Audit Log Viewer

- [ ] T119 [P] Create AuditLogViewer component in `/workspace/openshift-ai-dashboard/frontend/src/pages/AuditLogViewer/AuditLogViewer.tsx` (table view with filters, pagination)
- [ ] T120 Add audit log navigation to admin menu

**Checkpoint**: Audit logging complete - all dashboard operations tracked for compliance

---

## Phase 11: Performance Optimization (SC-003, SC-005)

**Goal**: Meet performance targets - 50+ concurrent users, <3s p95 dashboard load, <30s metric latency

### Backend Optimizations

- [ ] T121 [P] Implement Redis caching for dashboard configs in `/workspace/openshift-ai-dashboard/backend/pkg/cache/redis_cache.go` (30s TTL)
- [ ] T122 [P] Implement connection pooling for PostgreSQL in `/workspace/openshift-ai-dashboard/backend/internal/database/postgres.go` (max 100 connections)
- [ ] T123 [P] Add database query indexes per data-model.md in migration V002_add_indexes.up.sql
- [ ] T124 [P] Implement metric query batching in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/query_batcher.go` (combines multiple panel queries into single Thanos request)
- [ ] T125 Implement graceful degradation for Prometheus unavailability in `/workspace/openshift-ai-dashboard/backend/pkg/proxy/prometheus_proxy.go` (fallback to cached data)

### Frontend Optimizations

- [ ] T126 [P] Implement lazy loading for chart components in `/workspace/openshift-ai-dashboard/frontend/src/components/PersesDashboard/LazyChart.tsx`
- [ ] T127 [P] Add service worker for offline support in `/workspace/openshift-ai-dashboard/frontend/src/service-worker.ts`
- [ ] T128 Optimize bundle size with code splitting in `/workspace/openshift-ai-dashboard/frontend/vite.config.ts`

**Checkpoint**: Performance targets met - dashboard loads in <3s, supports 50+ concurrent users

---

## Phase 12: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

### Documentation

- [ ] T129 [P] Create API documentation with Swagger UI in `/workspace/openshift-ai-dashboard/docs/api/` (serve openapi.yaml)
- [ ] T130 [P] Create deployment guide in `/workspace/openshift-ai-dashboard/docs/deployment.md` (Operator deployment, environment variables, RBAC setup)
- [ ] T131 [P] Create troubleshooting guide in `/workspace/openshift-ai-dashboard/docs/troubleshooting.md` (common errors, debugging steps)
- [ ] T132 [P] Create developer onboarding guide in `/workspace/openshift-ai-dashboard/docs/contributing.md` (setup, architecture, PR process)

### Security Hardening

- [ ] T133 [P] Add input validation for all API endpoints in `/workspace/openshift-ai-dashboard/backend/pkg/api/validators.go` (max lengths, regex patterns)
- [ ] T134 [P] Implement SQL injection protection via prepared statements in all DAOs
- [ ] T135 [P] Add CSRF protection middleware in `/workspace/openshift-ai-dashboard/backend/internal/middleware/csrf.go`
- [ ] T136 [P] Implement rate limiting per IP for share token validation in `/workspace/openshift-ai-dashboard/backend/pkg/api/share_handler.go` (100 validations/min)
- [ ] T137 Add security headers middleware in `/workspace/openshift-ai-dashboard/backend/internal/middleware/security_headers.go` (CSP, X-Frame-Options, HSTS)

### Monitoring & Observability

- [ ] T138 [P] Add Prometheus metrics instrumentation in `/workspace/openshift-ai-dashboard/backend/pkg/metrics/instrumentation.go` (request latency, error rates, DB connection pool)
- [ ] T139 [P] Add structured logging with context in `/workspace/openshift-ai-dashboard/backend/internal/logging/logger.go` (request ID, user ID, operation)
- [ ] T140 Create dashboard for monitoring the dashboard system in `/workspace/openshift-ai-dashboard/monitoring/dashboard-metrics.json` (uses meta-observability pattern)

### Deployment

- [ ] T141 Create Kubernetes manifests in `/workspace/openshift-ai-dashboard/deploy/kubernetes/` (Deployment, Service, ConfigMap, RBAC)
- [ ] T142 Create Helm chart in `/workspace/openshift-ai-dashboard/deploy/helm/` (values.yaml for customization)
- [ ] T143 Create OpenShift Operator in `/workspace/openshift-ai-dashboard/operator/` (CRD: Dashboard CR, reconciler)
- [ ] T144 Create CI/CD pipeline in `/workspace/openshift-ai-dashboard/.github/workflows/ci.yml` (lint, test, build, push images)

### Quickstart Validation

- [ ] T145 Run quickstart.md validation - verify all steps work for new developer setup

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - **BLOCKS all user stories**
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - US1 (Phase 3): Can start immediately after Phase 2
  - US2 (Phase 4): Depends on US1 completion (uses dashboard CRUD from US1)
  - US3 (Phase 5): Depends on US1 completion (extends time range features)
  - US4 (Phase 6): Depends on US1 completion (adds overlays to existing charts)
  - US5 (Phase 7): Depends on US1 completion (adds synchronization to existing charts)
- **Sharing (Phase 8)**: Depends on US1 completion (shares dashboards created in US1)
- **Preferences (Phase 9)**: Can run in parallel with user stories (independent feature)
- **Audit (Phase 10)**: Can run in parallel with user stories (middleware already logs events)
- **Performance (Phase 11)**: Depends on all desired user stories being complete
- **Polish (Phase 12)**: Depends on all desired user stories being complete

### Critical Path for MVP (US1 Only)

```
Setup (Phase 1) → Foundational (Phase 2) → US1 (Phase 3) → MVP Ready
Estimated: 2-3 weeks for experienced team
```

### Critical Path for Full Feature (US1-5)

```
Setup → Foundational → US1 → US2 → US3 → US4 → US5 → Polish
Estimated: 8-12 weeks for experienced team
```

### Parallelization Opportunities

**Within Foundational Phase**:
- T014-T016 (auth) can run in parallel
- T018-T021 (API infrastructure) can run in parallel
- T025-T028 (models) can run in parallel
- T029-T032 (DAOs) can run in parallel
- T033-T036 (templates) can run in parallel

**Within US1**:
- T038-T042 (dashboard handlers) can run in parallel (different endpoints)
- T051-T054 (frontend components) can run in parallel (different files)
- T060-T062 (workload discovery) can run in parallel (different workload types)
- T080-T084 (chart plugins) can run in parallel (different chart types)

**After Foundational Phase**:
- US1 (Phase 3) MUST complete first (provides core dashboard functionality)
- US2-5 can then proceed sequentially OR:
  - Team A: US2 (custom dashboards)
  - Team B: US3 (historical analysis)
  - Team C: US4 (alerts) + US5 (correlation)

**Performance & Polish**:
- T121-T128 (optimizations) can run in parallel
- T129-T132 (documentation) can run in parallel
- T133-T137 (security) can run in parallel
- T138-T140 (monitoring) can run in parallel

---

## Implementation Strategy

### MVP First (Fastest Time to Value)

**Goal**: Deliver real-time monitoring capability in 2-3 weeks

1. Complete Phase 1: Setup (1-2 days)
2. Complete Phase 2: Foundational (1 week)
3. Complete Phase 3: US1 only (1 week)
4. **STOP**: Deploy MVP with real-time monitoring

**Delivers**: Users can monitor models in production with <15s latency

### Incremental Delivery (Recommended)

**Goal**: Add value iteratively, validate each story independently

1. **Sprint 1-2**: Setup + Foundational → Foundation ready
2. **Sprint 3**: US1 → MVP deployed (real-time monitoring)
3. **Sprint 4**: US2 → Custom dashboards deployed
4. **Sprint 5**: US3 → Historical analysis deployed
5. **Sprint 6-7**: US4 + US5 → Alerts + correlation deployed
6. **Sprint 8**: Performance + Polish → Production ready

**Delivers**: Incremental value every 2 weeks, each story independently testable

### Parallel Team Strategy

**Goal**: Maximize throughput with multiple developers

**Team Allocation**:
- **Week 1-2**: All hands on Foundational (critical path)
- **Week 3-4**:
  - Developer A: US1 backend (T038-T050)
  - Developer B: US1 frontend (T051-T064)
  - Developer C: Templates (T033-T037)
- **Week 5-6**:
  - Developer A: US2 backend (T065-T070)
  - Developer B: US2 frontend (T071-T085)
  - Developer C: US3 (T086-T092)
- **Week 7-8**:
  - Developer A: US4 (T093-T098)
  - Developer B: US5 (T099-T101)
  - Developer C: Sharing (T102-T109)

**Delivers**: All user stories in 8 weeks with 3-developer team

---

## Summary

**Total Tasks**: 145

**Tasks by Phase**:
- Phase 1 (Setup): 9 tasks
- Phase 2 (Foundational): 28 tasks (CRITICAL PATH)
- Phase 3 (US1 - Real-time Monitoring): 27 tasks
- Phase 4 (US2 - Custom Dashboards): 21 tasks
- Phase 5 (US3 - Historical Analysis): 7 tasks
- Phase 6 (US4 - Alert Thresholds): 6 tasks
- Phase 7 (US5 - Multi-Metric Correlation): 3 tasks
- Phase 8 (Sharing): 8 tasks
- Phase 9 (Preferences): 7 tasks
- Phase 10 (Audit): 4 tasks
- Phase 11 (Performance): 8 tasks
- Phase 12 (Polish): 17 tasks

**Parallelization Potential**:
- 47 tasks marked [P] (can run in parallel within their phase)
- ~32% of tasks can be parallelized with sufficient team size

**MVP Scope Recommendation** (Fastest time to value):
- Phase 1: Setup (9 tasks)
- Phase 2: Foundational (28 tasks)
- Phase 3: US1 only (27 tasks)
- **Total MVP: 64 tasks** (~40% of full feature)

**Full Feature Scope**:
- All phases including US1-US5 (145 tasks)

**Estimated Timeline** (with 3-developer team):
- MVP (US1): 2-3 weeks
- MVP + US2: 4-5 weeks
- MVP + US2 + US3: 5-6 weeks
- Full feature (US1-5 + Polish): 8-12 weeks

**Next Steps**:
1. Review task list with product owner - confirm user story priorities
2. Assign ownership for Foundational phase (required before stories start)
3. Choose implementation strategy: MVP First vs Incremental vs Parallel
4. Begin Sprint 1: Setup + Database migrations (T001-T013)

---

**Document Version**: 1.0.0
**Last Updated**: 2025-11-14
**Status**: Ready for Implementation
**Owner**: Lee (Team Lead) - OpenShift AI Platform Team
