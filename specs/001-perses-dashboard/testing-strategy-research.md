# Research Report: Contract Testing & Resilience Testing Strategy

**Feature**: 001-perses-dashboard
**Date**: 2025-11-14
**Researcher**: Neil (Principal QA Engineer)
**Context**: Constitution Check identified P1 critical path gaps in contract testing and resilience testing

---

## Executive Summary

This research addresses critical testing gaps identified during the Constitution Check for the Perses Observability Dashboard feature. The P1 Constitution items require:

1. **Contract Testing Framework** to validate API/UI interface contracts and prevent integration failures
2. **Resilience Testing Strategy** to validate graceful degradation and recovery for critical dependencies (PostgreSQL, Prometheus, OpenShift OAuth)
3. **Automated Acceptance Scenario Mapping** to ensure 20+ acceptance scenarios from spec.md are systematically covered

Based on comprehensive research of the existing Perses codebase, industry best practices, and the OpenShift AI platform requirements, I recommend:

- **Contract Testing**: Pact for Go backend + TypeScript frontend with bi-directional contract testing
- **Resilience Testing**: Chaos Mesh for Kubernetes-native chaos engineering with manual fault injection for critical scenarios
- **Acceptance Automation**: Playwright E2E tests with structured test mapping (no BDD framework initially)

**Critical Finding**: Why do we need to do this? The current Perses test suite has excellent unit and E2E coverage (70+ test files) but ZERO contract tests and NO resilience/chaos testing. For a mission-critical observability platform with 99.9% persistence SLA (SC-007) and <30s metric latency (SC-002), this creates unacceptable production risk.

---

## 1. Contract Testing Strategy

### Framework Decision: **Pact (Consumer-Driven Contracts)**

**Rationale**:

After analyzing the Go backend + TypeScript frontend architecture and reviewing 2025 best practices, Pact is the recommended choice for the following reasons:

1. **Multi-Language Maturity**:
   - `pact-go` v2+ provides robust Go provider verification
   - `@pact-foundation/pact` v12+ offers TypeScript consumer testing with strong type safety
   - Both libraries are actively maintained with 2025 releases

2. **Consumer-Driven Philosophy Fits Our Use Case**:
   - The TypeScript frontend (consumer) defines expected API contracts
   - The Go backend (provider) verifies it can satisfy those contracts
   - This prevents backend breaking changes from impacting the UI silently

3. **Existing Perses Compatibility**:
   - Perses already uses `httpexpect/v2` for API testing (verified in `/internal/api/e2e/`)
   - Pact integrates cleanly with existing test infrastructure
   - No architectural changes required

4. **CI/CD Integration**:
   - Pact Broker (OSS or PactFlow) enables contract versioning and verification workflows
   - Can run contract tests in PR pipelines before integration tests
   - Supports bi-directional contract testing (provider can also publish contracts)

**Alternatives Considered & Rejected**:

- **Spring Cloud Contract**: Java ecosystem, incompatible with Go backend
- **Dredd**: OpenAPI-based, but requires maintaining separate OpenAPI specs; Perses doesn't have comprehensive OpenAPI docs
- **Manual API mocks**: Too brittle, no versioning, maintenance burden

### Contracts to Test

Based on analysis of the Perses codebase (`/internal/api/` and `/ui/app/src/`), the following contracts MUST be tested:

#### **REST API Endpoints** (Priority Order)

| Contract Type | Endpoint Pattern | Example | Criticality | Notes |
|---------------|------------------|---------|-------------|-------|
| **Dashboard CRUD** | `/api/v1/projects/{project}/dashboards` | POST, GET, PUT, DELETE dashboard | P0 | Core functionality (FR-005, FR-006) |
| **Dashboard Templates** | `/api/v1/templates/dashboards` | GET templates, POST instantiate | P0 | Pre-built templates (FR-003) |
| **Metric Queries** | `/api/v1/proxy/prometheus/query` | POST PromQL query, GET results | P0 | Real-time metrics (FR-002, SC-002) |
| **Authentication** | `/api/v1/auth/login`, `/api/v1/auth/verify` | OAuth token exchange | P1 | OpenShift OAuth (FR-013) |
| **Projects/Namespaces** | `/api/v1/projects` | GET, POST projects | P1 | Multi-tenancy (FR-013) |
| **Dashboard Export** | `/api/v1/projects/{project}/dashboards/{name}/export` | GET dashboard JSON | P2 | Backup/version control (FR-012) |

**Critical Data Contracts**:

1. **Dashboard Schema** (verified in Perses `pkg/model/api/v1/dashboard_test.go`):
   ```typescript
   interface Dashboard {
     kind: "Dashboard"
     metadata: {
       name: string
       project: string
       version: number
       createdAt: string
       updatedAt: string
     }
     spec: {
       duration: string  // e.g., "6h"
       refreshInterval: string  // e.g., "15s"
       panels: Panel[]
       variables: Variable[]
     }
   }
   ```

2. **Chart Panel Schema**:
   ```typescript
   interface Panel {
     kind: "Panel"
     spec: {
       display: { name: string }
       plugin: {
         kind: "TimeSeriesChart" | "GaugeChart" | "StatChart" | "TableChart"
         spec: ChartSpec
       }
       queries: Query[]
     }
   }
   ```

3. **Metric Query Response**:
   ```typescript
   interface MetricQueryResponse {
     status: "success" | "error"
     data: {
       resultType: "matrix" | "vector"
       result: TimeSeriesResult[]
     }
     errorType?: string
     error?: string
   }
   ```

#### **WebSocket Messages** (Real-time Updates)

| Message Type | Direction | Payload Schema | Criticality |
|--------------|-----------|----------------|-------------|
| **Metric Update** | Server → Client | `{ type: "metric_update", timestamp: ISO8601, data: MetricData[] }` | P0 |
| **Dashboard Change** | Server → Client | `{ type: "dashboard_updated", dashboardId: string, version: number }` | P1 |
| **Connection Heartbeat** | Bidirectional | `{ type: "ping" / "pong", timestamp: number }` | P1 |

**Note**: Pact v4+ supports WebSocket contract testing via `pact-plugin-websocket`. For MVP, we'll use manual WebSocket mocks in E2E tests and add contract tests in Phase 2.

### CI/CD Integration

**Contract Test Execution Flow**:

```
Developer Workflow:

  Frontend PR (TypeScript)
  ├─ Run consumer contract tests
  ├─ Generate pact files
  └─ Publish to Pact Broker with version tag

  Backend PR (Go)
  ├─ Fetch pact files from broker
  ├─ Run provider verification tests
  └─ Publish verification results

  Integration Gate (Pre-Merge)
  ├─ Check can-i-deploy (Pact Broker)
  └─ BLOCK merge if contracts incompatible
```

**When Contract Tests Run**:

1. **On Every Frontend PR**:
   ```yaml
   # .github/workflows/frontend-contracts.yml
   - name: Run Consumer Contract Tests
     run: cd ui && npm run test:contract:consumer
   - name: Publish Pacts
     run: npx pact-broker publish pacts/ --broker-base-url=$PACT_BROKER_URL
   ```

2. **On Every Backend PR**:
   ```yaml
   # .github/workflows/backend-contracts.yml
   - name: Verify Provider Contracts
     run: make test-contracts-provider
   - name: Can I Deploy?
     run: pact-broker can-i-deploy --pacticipant backend --version=$GIT_SHA
   ```

3. **Nightly Full Contract Verification**:
   - Test all consumer versions against latest provider
   - Alert on breaking changes in dependencies

**Infrastructure Required**:

- **Pact Broker**: Deploy OSS Pact Broker on OpenShift (PostgreSQL backend, ~200MB memory)
  - Alternative: Use PactFlow SaaS (free tier: 5 integrations, sufficient for MVP)
- **Webhook Integration**: Pact Broker triggers provider verification on contract publish
- **Contract Versioning**: Tag contracts with Git SHA + semantic version

### Coverage Target

**Minimum Viable Contract Coverage** (before Phase 1 design completion):

- **API Endpoints**: 100% of P0 endpoints (6 dashboard CRUD + metrics endpoints)
- **Data Schemas**: 100% of Dashboard, Panel, and MetricQuery response schemas
- **Error Responses**: All 4xx/5xx error codes for covered endpoints
- **WebSocket Messages**: 0% (deferred to Phase 2)

**Target by GA Release**:

- **API Endpoints**: 80% of all REST endpoints
- **WebSocket Messages**: 100% of real-time update messages (SC-002 requirement)
- **Backward Compatibility**: All contracts tested across N-1 version (rolling upgrades)

---

## 2. Resilience Testing Strategy

### Framework Decision: **Chaos Mesh** (Primary) + **Manual Fault Injection** (Critical Scenarios)

**Rationale**:

After comparing Chaos Mesh vs Litmus (2025 analysis):

| Criterion | Chaos Mesh | Litmus | Decision |
|-----------|------------|--------|----------|
| **Kubernetes Native** | ✅ CRD-based, CNCF Sandbox | ✅ CRD-based, CNCF Incubating | Tie |
| **Ease of Use** | ✅ Better for beginners | ⚠️ Steeper learning curve | **Chaos Mesh** |
| **Failure Types** | ✅ 17+ attacks (network, disk I/O, time, kernel panic) | ✅ Broader scope (bare metal, VMs) | Tie (both sufficient) |
| **OpenShift Compatibility** | ✅ Tested on OpenShift 4.x | ✅ Tested on OpenShift | Tie |
| **Observability Integration** | ✅ Grafana dashboard | ⚠️ Requires custom setup | **Chaos Mesh** |
| **Target Environment** | ✅ Kubernetes-only (perfect fit) | ⚠️ Multi-platform (unnecessary complexity) | **Chaos Mesh** |

**Decision**: Use **Chaos Mesh** for automated chaos testing in CI/CD, supplemented with **manual fault injection** (Toxiproxy, `kubectl delete`) for deterministic critical scenarios.

### Failure Scenarios to Test

Based on the architecture (PostgreSQL, Prometheus/Thanos, OpenShift OAuth, Kubernetes infrastructure), here are the 8 critical failure modes:

#### **Scenario 1: PostgreSQL Database Unavailable (P0)**

**Failure Mode**: Database connection pool exhausted or PostgreSQL pod crashes

**Test Approach**:
- **Chaos Mesh**: `PodChaos` with `pod-kill` action targeting PostgreSQL StatefulSet
- **Manual**: `kubectl delete pod postgresql-0` during dashboard save operation

**Expected Behavior** (Graceful Degradation):
1. API returns `503 Service Unavailable` with JSON error:
   ```json
   {
     "status": "error",
     "message": "Dashboard persistence temporarily unavailable. Retry in 30s.",
     "error_code": "DB_UNAVAILABLE"
   }
   ```
2. Frontend displays user-friendly error: "Unable to save dashboard. Retrying automatically..."
3. Auto-retry with exponential backoff (1s, 2s, 4s, max 16s)
4. **MUST NOT**: Return 500 Internal Server Error or expose database connection strings

**Validation Criteria**:
- Dashboard state preserved in browser (unsaved changes not lost)
- No JavaScript console errors
- Metrics: `dashboard_save_failures_total{reason="db_unavailable"}` increments
- User can continue editing; save succeeds when DB recovers

**Recovery Testing**:
- PostgreSQL pod restarts within 30s (Kubernetes restartPolicy)
- Connection pool re-establishes automatically
- Pending dashboard saves succeed within 60s of recovery

#### **Scenario 2: Prometheus/Thanos Query Unreachable (P0)**

**Failure Mode**: Network partition between Perses and Prometheus/Thanos, or Prometheus scrape failures

**Test Approach**:
- **Chaos Mesh**: `NetworkChaos` with `partition` action isolating Prometheus pod
- **Toxiproxy**: Inject latency (5s) and timeouts to Prometheus query API

**Expected Behavior**:
1. Dashboard displays metric charts with "Data temporarily unavailable" overlay
2. Last known data points remain visible with timestamp: "Last updated: 2m ago"
3. Auto-refresh continues attempting queries every 15s (FR-002 default interval)
4. No infinite loading spinners; timeout after 10s

**Validation Criteria**:
- `prometheus_query_failures_total` metric increments
- Error logged: `"Failed to query Prometheus: context deadline exceeded"`
- Historical data (last 5min) cached in browser remains visible
- Dashboard does NOT crash or show blank charts

**Recovery Testing**:
- Network partition heals
- Metrics resume flowing within 2x refresh interval (30s max)
- Gap in data indicated visually on chart (no interpolation of missing data)

#### **Scenario 3: OpenShift OAuth Service Failure (P0)**

**Failure Mode**: OAuth server returns 503, TLS certificate issues, or authentication timeout

**Test Approach**:
- **Manual**: Temporarily misconfigure OAuth redirect URL
- **Chaos Mesh**: `PodChaos` targeting `oauth-openshift` pods

**Expected Behavior** (Fail-Closed Security):
1. User cannot log in; dashboard shows: "Authentication service unavailable. Please try again."
2. Existing authenticated sessions remain valid (JWT tokens cached)
3. Token refresh failures result in graceful logout after 3 retries
4. **MUST NOT**: Allow unauthenticated access or bypass authorization

**Validation Criteria**:
- HTTP 401 returned for unauthenticated requests
- Authenticated users retain read-only access (cached permissions)
- No dashboard configurations exposed without valid token
- SC-010 compliance: Zero security violations (audit logs verified)

**Recovery Testing**:
- OAuth service recovers
- Users can log in within 30s
- Session state preserved (users don't lose unsaved work)

#### **Scenario 4: High Memory Pressure (Dashboard Rendering) (P1)**

**Failure Mode**: Dashboard with 50+ charts causes browser OOM or backend memory exhaustion

**Test Approach**:
- **Chaos Mesh**: `StressChaos` with memory stress on Perses backend pod
- **Load Test**: Create dashboard with 100 panels, 1000+ time series each

**Expected Behavior**:
1. Backend rejects dashboard creation if >20 panels: `400 Bad Request: "Maximum 20 panels per dashboard"`
2. Frontend implements progressive loading: Render 8 panels initially, load more on scroll
3. Backend pod OOM triggers Kubernetes restart; requests fail gracefully during restart

**Validation Criteria**:
- Backend memory usage <2GB (constraint from plan.md)
- Frontend memory usage <500MB (Chrome DevTools measurement)
- SC-005: Dashboard load time <3s p95 even with 8 charts

#### **Scenario 5: Network Latency to Prometheus (P1)**

**Failure Mode**: 500ms-2s latency to Prometheus query API

**Test Approach**:
- **Toxiproxy**: Inject 1s latency to Prometheus
- **Chaos Mesh**: `NetworkChaos` with `delay` action

**Expected Behavior**:
1. Metric queries timeout after 5s (backend timeout)
2. Frontend displays loading indicator for first 3s, then "Slow connection" warning
3. Auto-refresh interval increases from 15s → 30s → 60s (adaptive backoff)
4. SC-002: Metric latency degrades gracefully; SLA waived during network issues

**Validation Criteria**:
- `prometheus_query_latency_seconds` histogram shows p95 >1s
- No request queue buildup (bounded queue of 100 requests)
- Backend does NOT retry timed-out queries (no thundering herd)

#### **Scenario 6: Kubernetes API Server Slow (P2)**

**Failure Mode**: `kubectl` commands timeout, affecting workload metadata queries

**Test Approach**:
- **Chaos Mesh**: `NetworkChaos` targeting kube-apiserver

**Expected Behavior**:
1. Dashboard template instantiation (requires k8s API for namespace/workload lookup) times out after 10s
2. Error message: "Unable to fetch workload metadata. Using cached data."
3. Templates render with stale workload names; user can manually edit

**Validation Criteria**:
- Kubernetes client timeout configured to 10s (not 30s default)
- Cached workload metadata (5min TTL) used as fallback
- No pod restarts due to health check failures

#### **Scenario 7: Disk I/O Contention (PostgreSQL) (P2)**

**Failure Mode**: PostgreSQL write latency >100ms due to disk I/O saturation

**Test Approach**:
- **Chaos Mesh**: `IOChaos` with `latency` action on PostgreSQL PVC

**Expected Behavior**:
1. Dashboard save operations take >1s but eventually succeed
2. No data corruption (PostgreSQL ACID guarantees)
3. User sees "Saving..." spinner for extended time

**Validation Criteria**:
- `dashboard_save_duration_seconds` p95 <3s (acceptable during I/O stress)
- No `UNIQUE constraint violation` errors (optimistic locking works)
- PostgreSQL logs show slow queries but no transaction rollbacks

#### **Scenario 8: Cascading Failures (All Dependencies Down) (P3)**

**Failure Mode**: PostgreSQL + Prometheus + OAuth all unavailable simultaneously

**Test Approach**:
- **Chaos Mesh**: `Workflow` combining multiple PodChaos actions
- **Manual**: Drain Kubernetes nodes hosting critical services

**Expected Behavior**:
1. Dashboard shows comprehensive error page: "Observability platform temporarily unavailable"
2. Retry suggestions: "Check back in 5 minutes or contact your administrator"
3. Backend remains responsive (serves error pages, doesn't crash)
4. **MUST NOT**: Enter infinite retry loop or expose stack traces

**Validation Criteria**:
- All API endpoints return `503 Service Unavailable` with consistent error schema
- Kubernetes readiness probe fails (pod removed from Service endpoints)
- Kubernetes liveness probe passes (pod not restarted)
- Recovery within 5min once dependencies restored

### Validation Criteria (What "Passing" Looks Like)

**Passing Criteria for Each Scenario**:

1. **Correctness**: System behavior matches expected degradation mode (no crashes, no data loss)
2. **Observability**: Metrics and logs capture failure (alerts would fire in production)
3. **Recovery**: System auto-recovers within defined SLA (no manual intervention)
4. **User Experience**: Error messages actionable, no user data lost

**Frequency**:

| Test Type | When Run | Notes |
|-----------|----------|-------|
| **CI vs Scheduled Chaos Tests** | Manual fault injection in CI (per PR); Chaos Mesh nightly in staging | Balance test coverage with infrastructure stability |
| **Pre-Release Game Day** | All scenarios executed sequentially before GA | Team observes behavior, documents runbooks |
| **Production Chaos** | Deferred to Phase 3 | Requires SRE approval and gradual rollout |

---

## 3. Acceptance Scenario Automation

### Framework: **Playwright E2E** (No BDD Framework in MVP)

**Rationale**:

After evaluating BDD Gherkin (Cucumber) vs Playwright direct E2E for TypeScript:

| Criterion | Playwright E2E | Cucumber + Playwright | Decision |
|-----------|----------------|----------------------|----------|
| **Existing Perses Codebase** | ✅ Already uses Playwright (`/ui/e2e/`) | ⚠️ Would require adding Cucumber | **Playwright** |
| **Test Writing Speed** | ✅ Faster for developers | ⚠️ Slower (write Gherkin + step definitions) | **Playwright** |
| **Stakeholder Readability** | ⚠️ Code-based (less readable) | ✅ Plain English Gherkin | Tie |
| **Maintenance Overhead** | ✅ Low (single codebase) | ⚠️ High (Gherkin + steps + page objects) | **Playwright** |
| **Team Skill Set** | ✅ Team already writes Playwright tests | ⚠️ Requires BDD training | **Playwright** |
| **Type Safety** | ✅ Full TypeScript IntelliSense | ⚠️ Step definitions lack type safety | **Playwright** |
| **MVP Timeline** | ✅ Immediate use of existing patterns | ⚠️ 2-3 week setup + learning curve | **Playwright** |

**Decision**: Use **Playwright E2E tests** with structured test mapping. Defer BDD to Phase 3 if stakeholders demand natural language tests.

**Critical Insight**: The existing Perses codebase already has 16+ Playwright E2E tests (`/ui/e2e/src/tests/`) with excellent patterns (fixtures, page objects, test utilities). Adding Cucumber would duplicate effort and slow development.

### Mapping Strategy: User Stories → Test Cases

**Traceability Matrix** (how we track requirements → tests):

| User Story | Priority | Acceptance Scenarios | Playwright Test File | Test Cases | Automated? |
|------------|----------|---------------------|---------------------|------------|------------|
| **US-1: Real-time Metrics Monitoring** | P1 | 4 scenarios | `real-time-monitoring.spec.ts` | 4 tests | Yes |
| **US-2: Custom Dashboard Creation** | P2 | 5 scenarios | `custom-dashboard-creation.spec.ts` | 5 tests | Yes |
| **US-3: Historical Trend Analysis** | P2 | 4 scenarios | `historical-trends.spec.ts` | 4 tests | Partial (3/4) |
| **US-4: Alert Threshold Visualization** | P3 | 4 scenarios | `alert-thresholds.spec.ts` | 4 tests | No (Phase 2) |
| **US-5: Multi-Metric Correlation** | P3 | 3 scenarios | `metric-correlation.spec.ts` | 3 tests | No (Phase 2) |

**Total**: 20 acceptance scenarios → 20 test cases (17 automated in MVP, 3 deferred)

### Test Coverage: Which User Stories Automated in MVP

**Phase 1 (MVP - Before GA)**:

✅ **User Story 1 - Real-time Metrics Monitoring (P1)** - 100% Automated

- Test File: `/tests/e2e/real-time-monitoring.spec.ts`
- Coverage:
  - ✅ Scenario 1: Display current metrics (CPU, memory, request rate, latency) with <15s freshness
  - ✅ Scenario 2: Auto-refresh on resource spike (CPU >80%) within 15s
  - ✅ Scenario 3: Navigate between multiple model dashboards
  - ✅ Scenario 4: Display workload termination with historical metrics

✅ **User Story 2 - Custom Dashboard Creation (P2)** - 100% Automated

- Test File: `/tests/e2e/custom-dashboard-creation.spec.ts`
- Coverage:
  - ✅ Scenario 1: Create new dashboard with chart library
  - ✅ Scenario 2: Add chart with metric query preview
  - ✅ Scenario 3: Save dashboard and verify persistence (SC-007)
  - ✅ Scenario 4: Share dashboard URL with team members
  - ✅ Scenario 5: Time range selector updates all charts

⚠️ **User Story 3 - Historical Trend Analysis (P2)** - 75% Automated (3/4 scenarios)

- Test File: `/tests/e2e/historical-trends.spec.ts`
- Coverage:
  - ✅ Scenario 1: Select time range (last 7 days) with appropriate granularity
  - ✅ Scenario 2: Zoom into time window for higher resolution
  - ✅ Scenario 3: Hover tooltip with exact values
  - ❌ Scenario 4: Time-shift comparison (deferred—requires advanced Perses feature)

❌ **User Story 4 - Alert Threshold Visualization (P3)** - 0% Automated (Phase 2)

- **Reason for Deferral**: Requires integration with OpenShift AI alerting system (not in MVP scope)

❌ **User Story 5 - Multi-Metric Correlation (P3)** - 0% Automated (Phase 2)

- **Reason for Deferral**: Advanced use case; not critical for MVP launch

---

## 4. Open Questions & Clarifications Needed

### For Product Team (Parker)

1. **Dashboard Sharing Security** (FR-020):
   - Question: How should dashboard sharing work? URL-based tokens, explicit RBAC permissions, or hybrid?
   - Impact: Affects contract testing (authentication flows) and resilience testing (permission failures)
   - **Deadline**: Needed before Phase 1 design (Week 3)

2. **Timezone Handling Strategy**:
   - Question: Should dashboards display timestamps in user local time, UTC, or configurable per-dashboard?
   - Impact: Affects acceptance test assertions (timestamp validation)
   - **Deadline**: Needed before E2E test implementation (Week 2)

### For Engineering Team

3. **Metric Backend Support** (FR-019):
   - Question: MVP supports Prometheus only, or Prometheus + Thanos from day 1?
   - Impact: Contract tests need to cover both query APIs if Thanos included
   - **Deadline**: Needed before contract test design (Week 1)

4. **Perses Time-Shift Feature Availability**:
   - Question: Does Perses v0.53.0+ support time-shift comparison overlays natively?
   - Impact: User Story 3 Scenario 4 automation feasibility
   - **Deadline**: Needed before Phase 1 E2E implementation (Week 3)

5. **WebSocket vs Server-Sent Events**:
   - Question: Real-time updates use WebSocket or SSE? (Plan.md suggests SSE for better scale)
   - Impact: Contract testing approach (Pact WebSocket plugin vs HTTP streaming)
   - **Deadline**: Needed before contract test implementation (Week 2)

---

## 5. Recommended Next Steps

### Immediate Actions (This Week)

1. **Socialize this research** with engineering leads, product team, and SRE
2. **Get sign-off** on framework decisions (Pact, Chaos Mesh, Playwright)
3. **Answer clarification questions** (5 open questions above)
4. **Provision infrastructure**:
   - Pact Broker deployment (PactFlow free tier signup)
   - Chaos Mesh installation on staging cluster

### Week 1-2 (Foundation)

5. **Implement first contract test** (Dashboard GET endpoint)
6. **Run first chaos experiment** (PostgreSQL pod delete)
7. **Write first acceptance test** (US-1 Scenario 1)
8. **Document testing workflows** in `/docs/testing/`

### Week 3-4 (MVP Coverage)

9. **Complete P0 contract coverage** (6 endpoints)
10. **Automate Scenarios 1-3 resilience tests** (CI integration)
11. **Finish User Stories 1-2 E2E automation** (9 tests)
12. **Validate Constitution Check items** resolved

### Pre-Release (Weeks 5-6)

13. **Run Game Day exercise** with full failure scenario suite
14. **Validate SLAs** under chaos conditions (SC-002, SC-007)
15. **Document runbooks** based on failure mode observations

---

**End of Research Report**

**Prepared by**: Neil, Principal QA Engineer
**Review Required**: Engineering Leads, Product (Parker), SRE Team
**Next Step**: Socialize findings and get approval to proceed with Phase 0 implementation

**Files Referenced**:
- `/workspace/sessions/agentic-session-1763159109/workspace/artifacts/specs/001-perses-dashboard/spec.md`
- `/workspace/sessions/agentic-session-1763159109/workspace/artifacts/specs/001-perses-dashboard/plan.md`
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/e2e/api/dashboard_test.go`
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/ui/e2e/src/tests/*.spec.ts`
