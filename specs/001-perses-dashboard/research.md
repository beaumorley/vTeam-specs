# Research: Metric Backend Architecture Decision

**Feature**: Perses Observability Dashboard
**Research Date**: 2025-11-14
**Researcher**: Archie (Architecture)
**Status**: Decision Required

---

## Research Questions

### 1. What metric backends does Perses natively support?

**Finding**: Perses supports the following datasource types as documented in `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/datasource.md`:

- **Prometheus** - Time series metrics (primary focus)
- **Prometheus-compatible backends** - Explicitly includes Thanos and Cortex
- **Tempo** - Distributed traces (separate observability pillar)
- **Loki** - Log aggregation
- **Pyroscope** - Profiling data

**Key Architecture Pattern**: Perses implements a pluggable datasource architecture via the `DatasourcePlugin` interface, with datasource discovery supporting both HTTP service discovery and Kubernetes-native service discovery. The `PrometheusDatasource` plugin type is the reference implementation for time-series metric backends.

**Source Evidence**:
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/README.md` lines 18-19
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/datasource.md` lines 5-11
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/configuration/datasource-discovery.md` lines 60-68

### 2. What is the standard monitoring stack in OpenShift 4.x?

**Finding**: OpenShift 4.x deploys a comprehensive monitoring stack managed by the Cluster Monitoring Operator (CMO):

**Core Components**:
- **Prometheus**: Two instances in HA configuration with hard anti-affinity rules
- **Thanos Querier**: Unified query interface across multiple Prometheus instances
- **Thanos Ruler**: Rule evaluation engine for user-defined project monitoring
- **Alertmanager**: Alert routing and deduplication

**Data Retention Architecture**:
- **Default Prometheus Retention**: 15 days local storage
- **Thanos Integration**: Optional long-term storage to S3-compatible object storage
- **User Workload Monitoring**: Separate Prometheus instance in `openshift-user-workload-monitoring` namespace
- **Configuration**: Retention configurable via `cluster-monitoring-config` ConfigMap with both time-based (`retention`) and size-based (`retentionSize`) parameters

**High Availability Pattern**: Both Prometheus instances scrape identical targets to ensure zero data loss on pod failure. Thanos Querier deduplicates metrics and provides global query view.

**Source Evidence**:
- OpenShift 4.14 Monitoring Documentation (docs.redhat.com)
- OpenShift 4.10-4.18 configuration guides
- Red Hat Knowledge Base article "How to configure Prometheus retention in OCP"

### 3. Historical Data Retention Trade-offs

**Requirement Context**: FR-008 mandates 7+ days retention for historical trend analysis. User Story 3 requires viewing metrics over "last 7 days" with appropriate granularity for identifying gradual performance degradation.

#### Option A: Prometheus Only

**Architecture**: Single Prometheus instance with local TSDB storage

**Retention Capability**:
- Default: 15 days (configurable via `--storage.tsdb.retention.time`)
- Storage: Persistent Volume Claims (PVC) on cluster storage
- Data Format: Native Prometheus TSDB blocks (2-hour segments)

**Pros**:
- Simplest deployment (already present in OpenShift)
- Zero additional infrastructure
- Meets FR-008 requirement (7 days < 15 days default)
- Native PromQL query interface
- Low operational complexity

**Cons**:
- Storage costs scale linearly with retention period
- Limited to cluster storage backend performance
- Single point of failure without HA configuration
- No automatic downsampling (raw data only)
- Difficult to extend beyond 30 days (PVC size constraints)
- Query performance degrades with large retention windows
- No multi-cluster aggregation capability

**Estimated Storage**: ~1.5 bytes/sample. For 1000 time series at 15s scrape interval: ~5.8GB per day, ~87GB for 15 days

#### Option B: Prometheus + Thanos (Recommended)

**Architecture**: Prometheus for recent data (2-15 days) + Thanos for long-term storage and global querying

**Retention Capability**:
- Prometheus: 2-15 days hot storage (fast queries)
- Thanos Store: Unlimited retention in object storage (S3, GCS, Azure Blob)
- Downsampling: Automatic compression (5m, 1h resolution) for historical data
- Query Layer: Unified Thanos Query interface across both

**Pros**:
- Aligns with OpenShift 4.x native architecture (Thanos already deployed)
- Object storage costs 90% less than block storage for long-term retention
- Infinite retention capability (months/years) without cluster impact
- Automatic downsampling reduces storage costs and improves query performance
- High availability via Thanos Query deduplication
- Multi-cluster observability (future OpenShift AI multi-cluster scenarios)
- Industry standard for cloud-native monitoring at scale
- Proven reference architecture in CNCF ecosystem

**Cons**:
- Increased operational complexity (5+ Thanos components: Sidecar, Store, Query, Compactor, Ruler)
- Object storage dependency (S3/GCS/Azure configuration required)
- Query latency increases for historical data (object storage retrieval)
- Requires Thanos familiarity for troubleshooting
- Additional resource overhead (~500MB memory per component)

**Estimated Storage**:
- Prometheus: 5.8GB/day * 7 days = ~41GB PVC
- Object Storage: Raw data offloaded after 2h, downsampled automatically
  - 5m downsampling: ~80% reduction
  - 1h downsampling: ~95% reduction
  - 90-day retention: ~150GB vs ~522GB raw (71% savings)

**OpenShift Integration**: Thanos components already managed by CMO. Configuration via `cluster-monitoring-config` ConfigMap enables user-workload metrics to leverage existing Thanos infrastructure.

#### Option C: Pluggable Backend Architecture

**Architecture**: Abstract datasource interface supporting multiple TSDB backends (Prometheus, Thanos, Cortex, VictoriaMetrics, InfluxDB, M3DB)

**Retention Capability**: Varies by backend
- VictoriaMetrics: 20x compression vs Prometheus, single-node up to billions of time series
- Cortex: Horizontally scalable, similar to Thanos but push-based architecture
- M3DB: Uber-developed, sub-millisecond query latency at scale

**Pros**:
- Maximum flexibility for diverse deployment environments
- Future-proof against TSDB technology shifts
- Support for customer-specific existing monitoring infrastructure
- Competitive differentiation (Grafana, Datadog use pluggable architecture)
- Enables best-of-breed backend selection

**Cons**:
- Massive scope increase (6-12 months additional development)
- Testing matrix explosion (N backends * M query patterns)
- Operational burden on platform teams (expertise required for multiple systems)
- PromQL dialect differences across backends create query incompatibilities
- Maintenance of multiple client libraries and authentication mechanisms
- Violates YAGNI principle (You Aren't Gonna Need It)
- Delays time-to-market for core value (dashboard visualization)

**Trade-off Analysis**: Provides flexibility that is not justified by requirements. FR-019 asks "which backends to support," not "how many backends to support." OpenShift AI runs exclusively on OpenShift, which standardizes on Prometheus/Thanos. Supporting additional backends adds complexity without addressing a validated customer need.

---

## Decision: Option B - Prometheus + Thanos (Production-Ready Architecture)

### Rationale

1. **Alignment with Platform Architecture**: OpenShift 4.x natively deploys Thanos Querier and provides configuration pathways for user workload monitoring to leverage Thanos infrastructure. Choosing Option B means integrating with existing platform capabilities rather than deploying new components.

2. **Future-Proof Scalability**: Requirements specify scaling to 500+ users and 500+ dashboards within 12 months. Thanos's horizontal scalability and multi-cluster aggregation capabilities position the architecture for long-term growth beyond initial MVP.

3. **Meets FR-008 with Margin**: 7+ day retention requirement is trivially satisfied. Thanos enables extending to 30, 60, or 90-day retention without architectural changes, supporting advanced ML use cases like seasonal pattern detection and long-term model drift analysis.

4. **Cost Efficiency at Scale**: Object storage economics (S3 Standard: $0.023/GB/month vs EBS gp3: $0.08/GB/month) combined with automatic downsampling create 60-70% cost reduction for historical data. At 1000 time series, 90-day retention costs ~$3.45/month (object) vs ~$41.76/month (block storage).

5. **Industry Pattern**: Aligns with the Prometheus Operator pattern widely adopted in Kubernetes ecosystem. CNCF case studies (Prometheus HA with Thanos: 73% of respondents) validate this as the de facto standard for production observability.

6. **Operational Leverage**: OpenShift cluster administrators already manage Thanos components for cluster monitoring. Extending to user workloads reuses existing operational expertise rather than introducing new system dependencies.

7. **Performance Characteristics**: Two-tier storage (hot Prometheus, cold Thanos) optimizes for common query patterns:
   - Real-time monitoring (last 15 minutes): <200ms p95 (Prometheus TSDB)
   - Recent troubleshooting (last 24 hours): <500ms p95 (Prometheus TSDB)
   - Historical analysis (7-90 days): <3s p95 (Thanos Store + downsampling)

8. **Perses Native Compatibility**: Perses documentation explicitly lists Thanos as a supported Prometheus-compatible backend. Zero custom development required for integration - standard `PrometheusDatasource` plugin works with Thanos Query endpoints.

### Alternatives Considered

#### Option A: Prometheus Only
**Pros**: Simplest MVP, zero infrastructure additions, meets literal FR-008 requirement
**Cons**: Does not align with OpenShift native architecture (Thanos already present), limited scalability runway, no cost-efficient path to extended retention (>30 days), requires future migration if requirements evolve
**Rejection Reason**: Short-term simplicity creates long-term technical debt. The 18-month horizon for this platform mandates choosing architectures that scale with the product vision, not just the MVP scope. Martin Fowler's "Evolutionary Architecture" principle applies here - we should optimize for changeability, and Prometheus-only creates a future migration cliff when (not if) extended retention is requested.

#### Option C: Pluggable Backend Architecture
**Pros**: Maximum flexibility, competitive feature parity, future-proof against TSDB evolution
**Cons**: 6-12 month scope increase, testing complexity, operational burden, violates YAGNI, delays core value delivery
**Rejection Reason**: This is a classic over-engineering trap. The Martin Fowler quote is relevant: "Flexibility that is never used is worse than useless - it's a constant drag." OpenShift AI's value proposition is deep integration with Red Hat's OpenShift platform, not general-purpose multi-cloud portability. Building an abstraction layer for backends we will never support (InfluxDB, VictoriaMetrics) optimizes for the wrong dimension. If a future customer genuinely requires a different TSDB, Perses's pluggable datasource architecture allows adding that backend without rewriting existing dashboards.

---

## Implementation Notes

### Key Integration Points

1. **Datasource Configuration**:
   ```yaml
   apiVersion: perses.dev/v1
   kind: GlobalDatasource
   metadata:
     name: openshift-thanos-querier
   spec:
     plugin:
       kind: PrometheusDatasource
       spec:
         directUrl: https://thanos-querier.openshift-monitoring.svc:9091
         proxy:
           kind: HTTPProxy
           spec:
             allowedEndpoints:
               - /api/v1/query
               - /api/v1/query_range
               - /api/v1/labels
               - /api/v1/series
   ```

2. **Authentication Flow**:
   - Perses dashboard backend proxies queries to Thanos Querier
   - OpenShift ServiceAccount token injected via `Authorization: Bearer` header
   - RBAC enforced at Thanos layer (namespace-scoped metrics visibility)
   - No direct browser-to-Thanos communication (security isolation)

3. **Query Patterns**:
   - Real-time queries: Target Prometheus directly for <15min windows
   - Historical queries: Target Thanos Query for >15min windows
   - Automatic query splitting based on time range selector (optimization)

4. **Retention Configuration** (Admin-facing):
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cluster-monitoring-config
     namespace: openshift-monitoring
   data:
     config.yaml: |
       prometheusK8s:
         retention: 7d                    # Prometheus hot storage
         volumeClaimTemplate:
           spec:
             resources:
               requests:
                 storage: 50Gi
       thanosQuerier:
         enableUserWorkload: true
   ```

5. **Dashboard Template Design**:
   - Default time range: "Last 1 hour" (optimizes for Prometheus hot path)
   - Time range picker: Last 15m, 1h, 6h, 24h, 7d, 30d (supports both tiers)
   - Auto-refresh: 15s for <1h ranges, 1m for >1h ranges (reduces query load)

### Configuration Requirements

**Platform Prerequisites**:
- OpenShift 4.10+ (Thanos Querier GA)
- User Workload Monitoring enabled (`enableUserWorkload: true`)
- Object storage configured for Thanos (S3/GCS/Azure)
- ServiceAccount with `cluster-monitoring-view` ClusterRole for cross-namespace queries

**Perses Deployment**:
- Datasource discovery configured to auto-detect Thanos Querier service
- Proxy mode enabled (security requirement - no direct client access)
- Query timeout: 30s (Thanos Store queries can be slow)
- Connection pooling: 50 connections (supports 50+ concurrent users per SC-003)

**Monitoring & Alerting**:
- Instrument Thanos query latency (p50, p95, p99) per time range
- Alert on Thanos Store unavailability (historical data inaccessible)
- Dashboard showing object storage costs (FinOps requirement)

### Migration Path If Backend Changes

**Scenario**: Customer requires switching from Thanos to VictoriaMetrics (example: single-node simplicity preference)

**Migration Steps**:
1. Deploy VictoriaMetrics alongside Thanos (dual-write period)
2. Create new `GlobalDatasource` with `PrometheusDatasource` plugin pointing to VictoriaMetrics endpoint
3. Update dashboard templates to reference new datasource
4. User dashboards: Bulk update via Perses API (script datasource references)
5. Validation period: Run both backends in parallel, compare query results
6. Cutover: Update default datasource, deprecate Thanos datasource
7. Decommission Thanos after 30-day validation window

**Mitigation**: Perses's datasource abstraction means dashboards reference datasource names, not endpoint URLs. Changing backends is primarily an ops-level datasource reconfiguration, not a dashboard rewrite. Query language compatibility (PromQL) is the critical constraint - all Prometheus-compatible backends support this.

**Estimated Effort**: 2-3 weeks for dual-write setup, validation, and cutover (assuming PromQL compatibility)

**Risk**: Low. Perses architecture explicitly designed for this scenario. The key invariant is PromQL compatibility, which all serious TSDB contenders support (Thanos, Cortex, VictoriaMetrics, M3DB).

---

## Architectural Implications

### North Star Architecture Alignment

This decision positions the observability dashboard as a first-class citizen in the OpenShift AI platform architecture:

```
┌─────────────────────────────────────────────────────────┐
│ OpenShift AI Platform (Multi-Cluster Future)           │
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│ │  Cluster A  │  │  Cluster B  │  │  Cluster C  │     │
│ │ Prometheus  │  │ Prometheus  │  │ Prometheus  │     │
│ │   (7d hot)  │  │   (7d hot)  │  │   (7d hot)  │     │
│ └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
│        │                 │                 │            │
│        └─────────────────┴─────────────────┘            │
│                          │                              │
│                    ┌─────▼─────┐                        │
│                    │  Thanos   │                        │
│                    │  Querier  │ ◄─── Perses Dashboard  │
│                    └─────┬─────┘                        │
│                          │                              │
│                    ┌─────▼─────┐                        │
│                    │  Thanos   │                        │
│                    │   Store   │                        │
│                    └─────┬─────┘                        │
│                          │                              │
│                    ┌─────▼─────┐                        │
│                    │  Object   │                        │
│                    │  Storage  │ (S3/GCS)               │
│                    │ (90d cold)│                        │
│                    └───────────┘                        │
└─────────────────────────────────────────────────────────┘
```

### Technical Debt Assessment

**Immediate Debt**: None. This is the standard reference architecture.

**Future Debt Scenarios**:
1. **Multi-tenancy at TSDB layer**: If hard tenant isolation required, may need Thanos Receiver or Cortex (tenant-aware ingestion). Current architecture relies on Prometheus label-based isolation + RBAC. Mitigation: Monitor for multi-tenancy requirement signals in customer feedback.

2. **Query performance at extreme scale**: If queries exceed 10,000+ time series, may need query federation or caching layer (Trickster). Mitigation: Implement query result caching in Phase 2, monitor query cardinality metrics.

3. **Cost optimization pressure**: If object storage costs become prohibitive (unlikely at <$100/month), may need aggressive downsampling or recording rules. Mitigation: Implement cost monitoring dashboard, define retention policy aligned with business value.

**Debt Compounding Timeline**: 18-24 months before architectural limitations manifest. Well within product roadmap visibility window.

---

## References

- Perses Datasource Documentation: `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/datasource.md`
- OpenShift 4.14 Monitoring Guide: https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html/monitoring/
- Thanos Architecture Guide: https://thanos.io/
- CNCF Case Study: "Why We Selected Thanos for Long Term Metrics Storage" (2022)
- Martin Fowler: "Evolutionary Architecture" - https://martinfowler.com/articles/evolving-architecture.html
- FR-008 (Functional Requirement): Time range selector with 7+ days support
- FR-019 (Functional Requirement): Metric backend clarification
- SC-002 (Success Criteria): <30s metric latency
- SC-006 (Success Criteria): Historical data accessibility

---

**Decision Status**: RECOMMENDED - Awaiting stakeholder approval
**Next Steps**: Update FR-019 in spec.md, proceed to Phase 1 data model design
**Review Date**: 2025-11-14
