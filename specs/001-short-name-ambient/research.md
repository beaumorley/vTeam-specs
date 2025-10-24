# Research Findings: Enhanced Model Metrics Display

**Feature**: Enhanced Model Metrics Display
**Branch**: 001-short-name-ambient
**Date**: 2025-10-24

## Purpose

This document resolves all "NEEDS CLARIFICATION" items from the Technical Context and provides technical decisions for implementation.

---

## Research Questions & Decisions

### 1. OpenShift AI Console SDK Integration

**Question**: What is the OpenShift AI Console SDK version and integration patterns?

**Decision**: Use OpenShift Dynamic Plugin SDK with React 18 + PatternFly 6

**Rationale**:
- OpenShift AI Console (odh-dashboard) uses **React 18.2.0** + **PatternFly 6.4.0** + **React Router 7.6.2**
- Dynamic Plugin SDK (`@openshift/dynamic-plugin-sdk-utils` v5.0.1) provides 75+ extension points
- Webpack Module Federation enables micro-frontend architecture with shared modules
- Console follows extension-based integration pattern with OdhApplication CRs

**Integration Approach**:
- Feature integrates as a new page under `/modelServing/projects/:namespace/models/:modelName/metrics`
- Use PatternFly 6 components for all UI chrome (cards, toolbars, tabs, breadcrumbs)
- Leverage existing RBAC patterns via `useAccessReview` hook with SelfSubjectAccessReview checks
- Follow established tab navigation pattern (Overview, Metrics, Other tabs)

**Alternatives Considered**:
- Standalone React app: Rejected due to need for console integration and RBAC
- PatternFly 5: Rejected as console migrated to PatternFly 6.4.0

---

### 2. Testing Framework Standards

**Question**: What testing frameworks and standards does OpenShift AI Console use?

**Decision**: Jest + React Testing Library + Cypress

**Rationale**:
- **Jest** is the established unit/integration testing framework with existing configuration
- **React Testing Library** aligns with PatternFly component testing patterns
- **Cypress** is the primary E2E framework with active test suite (login, data connections, workbenches, model serving)
- Coverage tracking via CodeCov integration

**Testing Standards**:
- Unit tests: Jest + React Testing Library for component behavior
- Integration tests: Mocked Prometheus API responses, RBAC scenarios
- E2E tests: Cypress for page load, time range selection, auto-refresh, error handling
- Accessibility tests: Automated WCAG 2.2 Level AA checks in test suite
- Test-first approach: Write tests before implementation per Definition of Ready

**Alternatives Considered**:
- Playwright: Offers better performance but console uses Cypress; consistency preferred
- Enzyme: Deprecated and incompatible with React 18

---

### 3. Perses Visualization Framework Production Readiness

**Question**: Is Perses mature enough for production embedding in React/PatternFly applications?

**Decision**: PROCEED with Perses, pending 2-3 day POC validation (Sprint 1, Week 1)

**Rationale**:
- **Red Hat precedent**: OpenShift monitoring plugin successfully integrated Perses in production
- **CNCF backing**: Sandbox project (accepted August 2024) with active development
- **Native Prometheus integration**: Built-in PromQL support, ideal for metrics backend
- **React support**: Official npm packages (`@perses-dev/components`, `@perses-dev/dashboards`)
- **Feature richness**: Dashboard-as-code, PromQL debugger, advanced visualizations

**Production Readiness Assessment**:
- ✅ **Proven**: Red Hat OpenShift deployed in production
- ✅ **Active development**: 105+ releases, 70+ contributors, regular cadence
- ⚠️ **CNCF Sandbox**: Not production-ready by CNCF standards; expect breaking changes
- ⚠️ **Material-UI dependency**: Adds bundle size in PatternFly environment
- ⚠️ **React 18 only**: No React 19 support yet
- ⚠️ **Memory leak history**: Fixed in v0.50.0+ (use this version minimum)

**Integration Strategy**:
```typescript
// Provider hierarchy required
<ThemeProvider theme={createTheme()}>
  <ChartsThemeProvider chartsTheme={ECHARTS_THEME_OVERRIDES}>
    <QueryClientProvider client={queryClient}>
      <PluginRegistry>
        <DatasourceStoreProvider>
          <TimeRangeProvider>
            <VariableProvider>
              <DataQueriesProvider>
                <SnackbarProvider>
                  {/* Perses panels wrapped in PatternFly cards */}
                  <Card>
                    <CardBody>
                      <PersesPanel />
                    </CardBody>
                  </Card>
                </SnackbarProvider>
              </DataQueriesProvider>
            </VariableProvider>
          </TimeRangeProvider>
        </DatasourceStoreProvider>
      </PluginRegistry>
    </QueryClientProvider>
  </ChartsThemeProvider>
</ThemeProvider>
```

**POC Success Criteria** (must pass to proceed):
- [ ] Perses panel renders within PatternFly card with acceptable visual consistency
- [ ] Provider setup complexity is manageable (< 1 day to implement)
- [ ] Initial page load < 2 seconds with 4-6 panels (p90)
- [ ] Individual panel render < 500ms (p95)
- [ ] Bundle size increase < 500KB gzipped
- [ ] Custom ECharts theme achieves visual alignment with PatternFly
- [ ] No critical accessibility issues in automated tests

**Fallback Plan** (if POC fails):
- **Alternative**: PatternFly Charts (built on Victory.js)
- **Impact**: +4-6 weeks development time
- **Trade-off**: Lighter bundle, full visual control, but less feature-rich

**Alternatives Considered**:
- PatternFly Charts: Simpler but lacks advanced features (dashboard-as-code, PromQL debugger)
- Grafana embedding: Licensing complexity and heavyweight integration
- Custom D3.js charts: Excessive development time (8-10 weeks)

---

### 4. Prometheus Query Performance & Caching

**Question**: How to achieve <500ms query performance with 1-minute auto-refresh?

**Decision**: Two-tier caching (React Query + Recording Rules) with optimized PromQL

**Rationale**:
- **Client-side caching** (React Query) reduces network requests and provides instant UI updates from cache
- **Recording rules** pre-compute expensive aggregations on Prometheus side
- **Optimized PromQL** reduces cardinality and query complexity
- **Staggered refresh** prevents thundering herd (10-20 concurrent panels)

**Implementation**:

**1. Recording Rules** (Prometheus server, 1-minute evaluation):
```yaml
groups:
  - name: model_serving_rules
    interval: 1m
    rules:
      - record: tenant:model_requests:rate1m
        expr: sum(rate(model_requests_total{namespace=~".+"}[5m])) by (namespace, model_id)

      - record: tenant:model_latency:p99
        expr: histogram_quantile(0.99, sum(rate(model_latency_bucket[5m])) by (namespace, model_id, le))

      - record: tenant:model_errors:rate1m
        expr: sum(rate(model_errors_total{namespace=~".+"}[5m])) by (namespace, model_id)
```

**2. React Query Configuration**:
```typescript
const usePrometheusMetric = (query: string) => {
  return useQuery({
    queryKey: ['prometheus', query],
    queryFn: () => fetchPrometheusData(query),

    // Refresh every 60 seconds
    refetchInterval: 60000,

    // Cache for 90 seconds (1.5x refresh)
    staleTime: 90000,

    // Keep in memory for 5 minutes
    cacheTime: 300000,

    // Stop polling when page hidden
    refetchOnWindowFocus: false,

    // Show stale data during errors
    keepPreviousData: true,

    // Smart retry with exponential backoff
    retry: 3,
    retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
  });
};
```

**3. Query Optimization Techniques**:
- Use recording rules for repeated queries: `tenant:model_requests:rate1m` instead of `sum(rate(model_requests_total[5m]))`
- Minimal lookback windows: 5m for rate functions (not 1h)
- Specific label selectors first: `{namespace="my-project",model_id="fraud-v2"}`
- Avoid regex on high-cardinality labels
- Step parameter aligned with refresh interval: `step=60s`

**4. Staggered Panel Refresh**:
```typescript
// Offset each panel by 2 seconds to spread load
const useStaggeredPolling = (query: string, panelIndex: number) => {
  const [initialized, setInitialized] = useState(false);

  useEffect(() => {
    const timer = setTimeout(() => setInitialized(true), panelIndex * 2000);
    return () => clearTimeout(timer);
  }, [panelIndex]);

  return useQuery({
    queryKey: ['prometheus', query],
    queryFn: () => fetchPrometheusData(query),
    refetchInterval: initialized ? 60000 : false,
    enabled: initialized,
  });
};
```

**Performance Targets**:
- Initial page load: < 2s p90 (spec requirement)
- Individual panel render: < 500ms p95 (spec requirement)
- Recording rule evaluation: 1-minute interval (aligned with auto-refresh)
- Client cache hit rate: > 90% for repeated views
- Query timeout: 30s frontend, 2m Prometheus backend

**Alternatives Considered**:
- Server-side caching only: Higher latency, more backend load
- No caching: Unacceptable performance with 1-minute refresh
- Real-time streaming (WebSockets): Excessive complexity for 1-minute refresh cadence

---

### 5. High-Cardinality Metric Handling

**Question**: How to handle model serving metrics without overwhelming Prometheus?

**Decision**: Label-based cardinality reduction + topk() queries + sample limits

**Rationale**:
- Individual models can generate 1000s requests/sec, creating high-cardinality series
- Prometheus performance degrades with >100K active series
- Model serving often has dynamic endpoints and per-user metrics (unbounded cardinality)

**Cardinality Guidelines**:
- Keep individual metrics under **100 unique label combinations**
- Use **route templates** instead of full paths: `/predictions/:user_id/:model_id`
- Track top N models explicitly, sample or aggregate others
- Set `sample_limit: 10000` per scrape target in Prometheus config

**Anti-Patterns to Avoid**:
```promql
# BAD: Per-endpoint with dynamic paths
http_requests_total{endpoint="/api/predictions/user123/model456"}

# BAD: Request IDs in labels
model_inference_duration{request_id="abc-123-def"}

# BAD: User emails
requests_total{user_email="user@example.com"}
```

**Best Practices**:
```promql
# GOOD: Route templates with bounded labels
http_requests_total{endpoint="/predictions",namespace="my-project"}

# GOOD: Model family aggregation
model_requests_total{model_family="fraud-detection",status="success"}

# GOOD: Use topk() to focus on high-volume models
topk(10, sum(rate(model_requests_total[5m])) by (model_id))
```

**Dashboard Query Patterns**:
- Overview: Top 10 models by request volume
- Model Details: Specific model with namespace filter
- Comparison: Side-by-side for 2-3 selected models only

**Alternatives Considered**:
- Track all models individually: Unacceptable cardinality explosion
- Use logs for everything: Loses aggregation and alerting capabilities
- Separate Prometheus per model: Excessive infrastructure cost

---

### 6. Multi-Tenant Query Filtering & RBAC

**Question**: How to enforce namespace/project scope and RBAC permissions?

**Decision**: Backend middleware injection with reserved `namespace` label

**Rationale**:
- **Security**: Frontend filtering is insufficient; must enforce on backend
- **Prometheus pattern**: All KServe/ModelMesh metrics include `namespace` label
- **RBAC integration**: Leverage existing `useAccessReview` pattern from console
- **Query isolation**: Middleware rewrites PromQL to inject namespace filter

**Implementation Pattern**:

**1. Frontend RBAC Check**:
```typescript
// Check user access before rendering metrics page
const canViewMetrics = useAccessReview({
  group: 'serving.kserve.io',
  resource: 'inferenceservices',
  namespace: projectNamespace,
  verb: 'get',
});

if (!canViewMetrics) {
  return <PermissionDenied />;
}
```

**2. Backend Query Filter Injection**:
```typescript
// Express middleware
app.use('/api/prometheus/*', authenticateJWT, (req, res, next) => {
  const allowedNamespaces = req.user.namespaces; // From JWT
  const query = req.query.query;

  // Inject namespace filter into PromQL
  const namespaceFilter = allowedNamespaces.length === 1
    ? `{namespace="${allowedNamespaces[0]}"}`
    : `{namespace=~"${allowedNamespaces.join('|')}"}`;

  // Rewrite query to include filter
  const modifiedQuery = injectLabelFilter(query, 'namespace', namespaceFilter);

  req.query.query = modifiedQuery;
  next();
});
```

**3. PromQL Query Pattern**:
```promql
# Frontend sends:
sum(rate(model_requests_total[5m])) by (model_id)

# Backend rewrites to:
sum(rate(model_requests_total{namespace="my-project"}[5m])) by (model_id)

# Multi-namespace (admin):
sum(rate(model_requests_total{namespace=~"project-a|project-b"}[5m])) by (namespace, model_id)
```

**Recording Rules with Namespace Scope**:
```yaml
groups:
  - name: tenant_metrics
    interval: 1m
    rules:
      # Per-namespace aggregations
      - record: namespace:model_requests:rate1m
        expr: sum(rate(model_requests_total[5m])) by (namespace, model_id)
```

**Security Considerations**:
- ✅ Always enforce namespace filtering on backend (primary security boundary)
- ✅ Validate user has access to requested namespaces before proxying query
- ✅ Use SelfSubjectAccessReview (SSAR) for real-time permission checks
- ⚠️ Never rely on frontend filtering alone (defense in depth only)
- ⚠️ Log unauthorized access attempts for security audit

**Alternatives Considered**:
- Separate Prometheus per tenant: Excessive infrastructure cost
- Prometheus per-tenant RBAC: Limited support, complex configuration
- Frontend-only filtering: Insecure, rejected

---

## Technology Stack Summary

Based on all research findings, the complete technology stack is:

**Frontend**:
- **Language**: TypeScript 4.x+
- **Framework**: React 18.2.0
- **UI Library**: PatternFly 6.4.0
- **Routing**: React Router 7.6.2 (with v5-compat)
- **State Management**: Redux 4.2.0 (global), React Query (server state)
- **Visualization**: Perses (CNCF, pending POC) with fallback to PatternFly Charts
- **Testing**: Jest + React Testing Library + Cypress

**Backend**:
- **Integration**: OpenShift Dynamic Plugin SDK v5.0.1
- **Metrics Backend**: Prometheus with PromQL queries
- **Optional**: TrustyAI service for behavior metrics (feature-flagged)

**Performance**:
- **Caching**: React Query (client, 90s TTL) + Recording Rules (server, 1m eval)
- **Refresh**: 1-minute auto-refresh with staggered panel loading
- **Query Timeout**: 30s frontend, 2m Prometheus backend

**Security**:
- **RBAC**: `useAccessReview` pattern with SelfSubjectAccessReview
- **Multi-tenancy**: Backend middleware injection of namespace filters
- **Isolation**: Namespace-scoped queries, no cross-tenant data leakage

---

## Implementation Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Perses not production-ready | HIGH | 2-3 day POC in Sprint 1 Week 1; fallback to PatternFly Charts (+4-6 weeks) |
| Query performance at scale | MEDIUM | Recording rules + client caching + staggered refresh + load testing |
| Behavior metrics unavailable | MEDIUM | TrustyAI feature-flagged; graceful degradation with empty states |
| PatternFly + Perses visual mismatch | MEDIUM | Custom ECharts theme + wrap in PatternFly cards + early UX review |
| Accessibility complexity | MEDIUM | 15-20% overhead in estimates + accessibility SME review + early audit |
| High cardinality explosion | MEDIUM | Cardinality limits + topk() queries + sample_limit=10K + monitoring |

---

## Next Steps

1. ✅ Research complete (all NEEDS CLARIFICATION resolved)
2. → Proceed to Phase 1: Data model design
3. → Generate API contracts for Prometheus queries
4. → Create quickstart.md for developer onboarding
5. → Update agent context with technology decisions

---

*Research completed: 2025-10-24*
*Research agents: Perses Integration, Console SDK, Prometheus Best Practices*
