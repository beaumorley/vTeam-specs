# Testing Strategy Summary: Contract & Resilience Testing

**Feature**: 001-perses-dashboard
**Date**: 2025-11-14
**Author**: Neil (Principal QA Engineer)

---

## Contract Testing Strategy

### Framework Decision: **Pact**

**Rationale**: Consumer-driven contract testing with mature Go and TypeScript support

**Why Pact over alternatives?**
- ✅ `pact-go` v2+ and `@pact-foundation/pact` v12+ actively maintained (2025)
- ✅ Integrates with existing Perses test infrastructure (`httpexpect/v2`)
- ✅ Consumer-driven approach prevents backend breaking changes
- ✅ Pact Broker enables contract versioning and CI/CD integration
- ❌ Spring Cloud Contract: Java-only, incompatible with Go
- ❌ Dredd: Requires maintaining separate OpenAPI specs

### Contracts to Test

**P0 API Endpoints** (6 endpoints):
1. **Dashboard CRUD**: `/api/v1/projects/{project}/dashboards` (POST, GET, PUT, DELETE)
2. **Dashboard Templates**: `/api/v1/templates/dashboards` (GET, POST instantiate)
3. **Metric Queries**: `/api/v1/proxy/prometheus/query` (POST PromQL)
4. **Authentication**: `/api/v1/auth/login`, `/api/v1/auth/verify` (OAuth)
5. **Projects**: `/api/v1/projects` (GET, POST)
6. **Dashboard Export**: `/api/v1/projects/{project}/dashboards/{name}/export`

**Data Schemas** (100% coverage required):
- Dashboard entity (metadata, spec, panels, variables)
- Panel entity (chart type, queries, display config)
- MetricQuery response (Prometheus result format)
- Error responses (4xx/5xx standardized error schema)

**WebSocket Messages** (deferred to Phase 2):
- Metric updates (server → client)
- Dashboard change notifications
- Connection heartbeat

### CI/CD Integration

**Workflow**:
```
Frontend PR → Consumer tests → Publish pacts → Pact Broker

Backend PR → Fetch pacts → Provider verification → Publish results

Merge Gate → Check can-i-deploy → Block if incompatible
```

**When Tests Run**:
- Every frontend PR: Consumer contract tests
- Every backend PR: Provider verification tests
- Nightly: Full contract verification (all versions)

**Infrastructure**: Pact Broker (OSS self-hosted or PactFlow free tier)

### Coverage Target

**MVP** (before Phase 1 design):
- 100% of P0 endpoints (6 dashboard + metrics APIs)
- 100% of Dashboard, Panel, MetricQuery schemas
- All 4xx/5xx error codes

**GA Release**:
- 80% of all REST endpoints
- 100% of WebSocket messages
- N-1 version backward compatibility

---

## Resilience Testing Strategy

### Failure Scenarios (8 Critical Modes)

**P0 Scenarios** (must test before MVP):

1. **PostgreSQL Database Unavailable**
   - Test: Chaos Mesh `PodChaos` (pod-kill)
   - Expected: 503 error, auto-retry with backoff, no data loss
   - Validation: `dashboard_save_failures_total` metric increments

2. **Prometheus/Thanos Query Unreachable**
   - Test: Chaos Mesh `NetworkChaos` (partition)
   - Expected: "Data unavailable" overlay, last known data visible
   - Validation: No dashboard crash, timeout after 10s

3. **OpenShift OAuth Service Failure**
   - Test: Manual OAuth misconfiguration
   - Expected: Fail-closed (401), no unauthenticated access
   - Validation: SC-010 zero security violations

**P1 Scenarios**:

4. **High Memory Pressure** (dashboard rendering)
   - Test: Chaos Mesh `StressChaos`
   - Expected: Progressive loading, backend <2GB memory

5. **Network Latency to Prometheus** (500ms-2s)
   - Test: Toxiproxy latency injection
   - Expected: Adaptive backoff (15s → 30s → 60s refresh)

**P2 Scenarios**:

6. **Kubernetes API Server Slow**
   - Test: Chaos Mesh `NetworkChaos` on kube-apiserver
   - Expected: Fallback to cached workload metadata (5min TTL)

7. **Disk I/O Contention** (PostgreSQL)
   - Test: Chaos Mesh `IOChaos`
   - Expected: Slower saves (<3s p95), no data corruption

**P3 Scenarios**:

8. **Cascading Failures** (all dependencies down)
   - Test: Manual orchestration or Chaos Mesh `Workflow`
   - Expected: Comprehensive error page, no infinite retry loops

### Testing Approach: **Chaos Mesh + Manual Fault Injection**

**Why Chaos Mesh over Litmus?**
- ✅ Better beginner experience (Constitution feedback: team new to chaos engineering)
- ✅ 17+ attack types (network, disk I/O, memory stress, kernel panic)
- ✅ Grafana dashboard integration (existing observability stack)
- ✅ Kubernetes-only focus (perfect fit for OpenShift)
- ⚠️ Litmus: Broader scope (bare metal, VMs) but steeper learning curve

**Manual Fault Injection Use Cases**:
- OAuth failures (security-sensitive, need precise control)
- Network latency (Toxiproxy for deterministic latency)
- Cascading failures (complex multi-service orchestration)

### Validation Criteria (What "Passing" Looks Like)

**SLA Targets**:

| Scenario | Normal | Degraded | Failure Mode |
|----------|--------|----------|--------------|
| Dashboard Load | <3s p95 | <5s p95 | Show cached/stale |
| Metric Latency | <30s | <60s | Display "unavailable" |
| Save Operation | <1s p95 | <3s p95 | Retry with backoff |
| Authentication | <500ms | <2s | Fail-closed (401) |

**Passing Criteria**:
1. Correctness: Matches expected degradation (no crashes, no data loss)
2. Observability: Metrics/logs capture failure (alerts fire)
3. Recovery: Auto-recovers within SLA (no manual intervention)
4. UX: Error messages actionable, no user data lost

### Frequency

| Test Type | Schedule | Environment |
|-----------|----------|-------------|
| **CI Manual Fault Injection** | Every PR | Scenarios 1, 3, 5 |
| **Nightly Chaos Mesh** | 2am-4am daily | Staging cluster (Scenarios 1-7) |
| **Weekly Game Day** | Friday 10am | Production-like cluster (All scenarios) |
| **Pre-Release Drill** | Before GA | All scenarios sequentially |

---

## Automated Acceptance Scenario Mapping

### Framework: **Playwright E2E** (No BDD)

**Why Playwright over Cucumber BDD?**
- ✅ Perses already has 16+ Playwright E2E tests (`/ui/e2e/`)
- ✅ Faster development (no Gherkin + step definitions)
- ✅ Full TypeScript type safety and IntelliSense
- ✅ Lower maintenance (single codebase, not 3 layers)
- ❌ BDD: 2-3 week setup, team training, duplicate effort

**Decision**: Use Playwright with structured test mapping. Defer BDD to Phase 3 if stakeholders demand.

### Mapping Strategy: User Stories → Test Cases

**Traceability**:
```
User Story (spec.md)
  └─> Acceptance Scenarios (Given/When/Then)
      └─> Playwright Test File (.spec.ts)
          └─> Test Cases (test() blocks)
              └─> Assertions (expect() statements)
```

### Test Coverage (Which User Stories Automated in MVP)

| User Story | Priority | Scenarios | Automated? | Test File |
|------------|----------|-----------|------------|-----------|
| **US-1: Real-time Metrics** | P1 | 4/4 | ✅ 100% | `real-time-monitoring.spec.ts` |
| **US-2: Custom Dashboards** | P2 | 5/5 | ✅ 100% | `custom-dashboard-creation.spec.ts` |
| **US-3: Historical Trends** | P2 | 3/4 | ⚠️ 75% | `historical-trends.spec.ts` |
| **US-4: Alert Thresholds** | P3 | 0/4 | ❌ 0% | Phase 2 (alert integration) |
| **US-5: Metric Correlation** | P3 | 0/3 | ❌ 0% | Phase 2 (advanced feature) |

**Total**: 17/20 scenarios automated in MVP (85% coverage)

**Deferred**:
- US-3 Scenario 4: Time-shift comparison (needs Perses feature clarification)
- US-4: All alert scenarios (not in MVP scope)
- US-5: All correlation scenarios (not in MVP scope)

### Test Data Management

**3-Tier Strategy**:

1. **Tier 1: Mock Prometheus Data** (unit/contract tests)
   - TypeScript fixtures with predefined metric values
   - Fast, deterministic, no infrastructure

2. **Tier 2: Synthetic Metrics** (E2E tests)
   - Deploy `prometheus-mock-server` in test cluster
   - Pre-defined metric patterns for scenarios (CPU spike, OOM, etc.)
   - Eliminates test flakiness

3. **Tier 3: Real Workloads** (smoke tests, nightly only)
   - Actual model serving (ONNX ResNet-50)
   - Validates end-to-end with real Prometheus data
   - Too slow for PR pipeline

### CI/CD Integration

**Test Execution Times**:

| Test Suite | Count | Time | Frequency |
|------------|-------|------|-----------|
| P1 Acceptance | 9 tests | <5min | Every PR |
| P2 Acceptance | 8 tests | <8min | Every PR |
| P3 Acceptance | 3 tests | <3min | Nightly |
| Full Regression | 20 tests | <15min | Pre-release |

**Can I test this locally?** ✅ Yes
```bash
cd backend && make run &
docker run -p 9090:9090 prom/prometheus &
cd ui/e2e && npx playwright test --headed
```

---

## Success Metrics

### Contract Testing

| Metric | Target | Measurement |
|--------|--------|-------------|
| Breaking Changes Caught | 100% | Pact verification failures block PR |
| Contract Coverage | 80% by GA | Pact Broker reports |
| Execution Time | <3min per PR | GitHub Actions timing |

### Resilience Testing

| Metric | Target | Measurement |
|--------|--------|-------------|
| Critical Failure Modes Tested | 8/8 by GA | Chaos experiment logs |
| Graceful Degradation | 100% expected behavior | Manual verification |
| MTTR (Mean Time To Recovery) | <5min | Chaos Mesh timeline metrics |

### Acceptance Testing

| Metric | Target | Measurement |
|--------|--------|-------------|
| Scenario Coverage | 17/20 (85%) MVP | Test execution reports |
| Execution Time | <10min P1+P2 | Playwright HTML report |
| Test Stability | <10% flaky rate | Retry count analysis |

---

## Open Questions (Require Answers Before Implementation)

### For Product Team (Parker)

1. **Dashboard Sharing Security** (FR-020): URL tokens, RBAC, or hybrid? (Deadline: Week 3)
2. **Timezone Handling**: Local time, UTC, or configurable? (Deadline: Week 2)

### For Engineering Team

3. **Metric Backend** (FR-019): Prometheus only or Prometheus + Thanos from day 1? (Deadline: Week 1)
4. **Perses Time-Shift**: Does v0.53.0+ support time-shift comparison natively? (Deadline: Week 3)
5. **Real-time Updates**: WebSocket or Server-Sent Events? (Deadline: Week 2)

### For SRE/Platform Team

6. **Chaos Mesh Permissions**: Admin access on staging cluster? (Deadline: Week 2)
7. **Production Chaos Policy**: GameDays only or automated experiments? (Deadline: Week 8)

---

## Recommended Next Steps

### This Week (Immediate)

1. Socialize research with engineering, product, SRE teams
2. Get sign-off on framework decisions (Pact, Chaos Mesh, Playwright)
3. Answer 7 open questions above
4. Provision infrastructure (Pact Broker, Chaos Mesh staging)

### Week 1-2 (Foundation)

5. Implement first contract test (Dashboard GET)
6. Run first chaos experiment (PostgreSQL pod delete)
7. Write first acceptance test (US-1 Scenario 1)
8. Document workflows in `/docs/testing/`

### Week 3-4 (MVP Coverage)

9. Complete P0 contract coverage (6 endpoints)
10. Automate Scenarios 1-3 resilience tests
11. Finish US-1 and US-2 E2E automation (9 tests)
12. Validate Constitution Check P1 items resolved

### Pre-Release (Weeks 5-6)

13. Run Game Day exercise (full failure scenario suite)
14. Validate SLAs under chaos conditions
15. Document runbooks based on observed failure modes

---

**Status**: Research Complete ✅
**Constitution Check Impact**: Resolves P1 critical path items (Contract Testing, Resilience Testing)
**Next Gate**: Get stakeholder approval to proceed with Phase 0 implementation

**Full Research Document**: `/workspace/sessions/agentic-session-1763159109/workspace/artifacts/specs/001-perses-dashboard/testing-strategy-research.md`
