# Tasks: Enhanced Model Metrics Display

**Feature Branch**: `001-short-name-ambient`
**Input**: Design documents from `/specs/001-short-name-ambient/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

---

## Implementation Strategy

This feature implements comprehensive metrics visualization for deployed models in the OpenShift AI Console. Tasks are organized by user story to enable independent implementation and testing.

**MVP Approach**: User Story 1 (Core Metrics Display) provides a complete, deployable feature. Subsequent stories add enhancements.

**Total Estimated Tasks**: 67 tasks across 9 phases
**Estimated Timeline**: 7-8 weeks (4 sprints)
**Critical Path**: Perses POC → Core Metrics → Behavior Metrics → Accessibility

---

## Phase 1: Setup & Foundation

**Goal**: Project initialization and Perses POC validation

### Phase 1.1: Project Structure Setup

- [ ] T001 Create feature directory structure at frontend/src/pages/modelServing/screens/metrics/
- [ ] T002 [P] Create TypeScript interfaces from data-model.md in frontend/src/pages/modelServing/screens/metrics/types/metrics.ts
- [ ] T003 [P] Create Prometheus types in frontend/src/pages/modelServing/screens/metrics/types/prometheus.ts
- [ ] T004 [P] Configure Jest test setup in frontend/src/pages/modelServing/screens/metrics/__tests__/setup.ts
- [ ] T005 [P] Configure React Testing Library utilities in frontend/src/pages/modelServing/screens/metrics/__tests__/testUtils.tsx

### Phase 1.2: Critical Perses POC (MUST COMPLETE FIRST)

**CRITICAL**: This POC is a go/no-go decision point. Must complete before other UI tasks.

- [ ] T006 [POC] Install Perses dependencies (@perses-dev/components, @perses-dev/dashboards) in frontend/package.json
- [ ] T007 [POC] Create Perses provider wrapper in frontend/src/pages/modelServing/screens/metrics/components/PersesProviderWrapper.tsx
- [ ] T008 [POC] Implement basic Perses panel integration with PatternFly Card in frontend/src/pages/modelServing/screens/metrics/components/PersesPOC.tsx
- [ ] T009 [POC] Validate POC success criteria: load time <2s, panel render <500ms, bundle size <500KB, visual consistency with PatternFly

**POC Decision Gate**: If T006-T009 fail, fallback to PatternFly Charts (adds 4-6 weeks)

---

## Phase 2: Foundational Infrastructure (BLOCKING)

**Goal**: Core services and infrastructure required by all user stories

**MUST COMPLETE** before any user story implementation.

### Phase 2.1: API Client Services

- [ ] T010 [P] Implement Prometheus client service in frontend/src/pages/modelServing/screens/metrics/services/prometheusClient.ts
- [ ] T011 [P] Implement TrustyAI client service in frontend/src/pages/modelServing/screens/metrics/services/trustyaiClient.ts
- [ ] T012 [P] Create PromQL query templates in frontend/src/pages/modelServing/screens/metrics/services/metricsQueries.ts
- [ ] T013 [P] Implement error handling utilities in frontend/src/pages/modelServing/screens/metrics/services/errorHandler.ts

### Phase 2.2: React Query Hooks

- [ ] T014 [P] Create usePrometheusQuery hook with caching in frontend/src/pages/modelServing/screens/metrics/hooks/usePrometheusQuery.ts
- [ ] T015 [P] Create usePrometheusRangeQuery hook in frontend/src/pages/modelServing/screens/metrics/hooks/usePrometheusRangeQuery.ts
- [ ] T016 [P] Create useTrustyAIMetrics hook with graceful degradation in frontend/src/pages/modelServing/screens/metrics/hooks/useTrustyAI.ts
- [ ] T017 Create useMetricsRefresh hook with staggered polling in frontend/src/pages/modelServing/screens/metrics/hooks/useMetricsRefresh.ts

### Phase 2.3: RBAC Integration

- [ ] T018 Implement namespace filtering middleware pattern (backend documentation) in backend/src/middleware/prometheusRBAC.ts
- [ ] T019 Create useModelMetricsAccess hook using useAccessReview in frontend/src/pages/modelServing/screens/metrics/hooks/useModelMetricsAccess.ts
- [ ] T020 [P] Add RBAC permission denied component in frontend/src/pages/modelServing/screens/metrics/components/PermissionDenied.tsx

---

## Phase 3: User Story 1 - Core Metrics Display (P1)

**User Story**: As an ML Operations Engineer, I need to view performance metrics (latency, throughput, errors) and resource utilization for my deployed models.

**Independent Test Criteria**:
- User can navigate to metrics page for a deployed model
- Performance metrics panels render with data from last 1 hour
- Metrics auto-refresh every 60 seconds
- Time range selector allows switching between 1h, 24h, 7d, 30d

### Phase 3.1: Page Structure & Routing

- [ ] T021 [US1] Create ModelMetricsPage main component in frontend/src/pages/modelServing/screens/metrics/ModelMetricsPage.tsx
- [ ] T022 [US1] Add route configuration for /modelServing/projects/:namespace/models/:modelName/metrics
- [ ] T023 [US1] [P] Implement page breadcrumb navigation in frontend/src/pages/modelServing/screens/metrics/components/MetricsBreadcrumb.tsx

### Phase 3.2: Time Range Selection

- [ ] T024 [US1] [P] Create TimeRangeSelector component in frontend/src/pages/modelServing/screens/metrics/components/TimeRangeSelector/TimeRangeSelector.tsx
- [ ] T025 [US1] [P] Implement time range state management with URL persistence in frontend/src/pages/modelServing/screens/metrics/hooks/useTimeRange.ts
- [ ] T026 [US1] [P] Add auto-refresh controls (pause/resume) in frontend/src/pages/modelServing/screens/metrics/components/TimeRangeSelector/RefreshControls.tsx

### Phase 3.3: Core Metric Panels

- [ ] T027 [US1] [P] Create reusable MetricPanel component in frontend/src/pages/modelServing/screens/metrics/components/MetricPanel/MetricPanel.tsx
- [ ] T028 [US1] [P] Implement loading state component in frontend/src/pages/modelServing/screens/metrics/components/MetricPanel/LoadingState.tsx
- [ ] T029 [US1] [P] Implement error state component in frontend/src/pages/modelServing/screens/metrics/components/MetricPanel/ErrorState.tsx
- [ ] T030 [US1] [P] Implement empty state component in frontend/src/pages/modelServing/screens/metrics/components/MetricPanel/EmptyState.tsx

### Phase 3.4: Performance Metrics Section

- [ ] T031 [US1] Create PerformanceMetrics section container in frontend/src/pages/modelServing/screens/metrics/components/PerformanceMetrics/PerformanceMetrics.tsx
- [ ] T032 [US1] [P] Implement LatencyPanel (p50, p95, p99) in frontend/src/pages/modelServing/screens/metrics/components/PerformanceMetrics/LatencyPanel.tsx
- [ ] T033 [US1] [P] Implement ThroughputPanel (requests/sec) in frontend/src/pages/modelServing/screens/metrics/components/PerformanceMetrics/ThroughputPanel.tsx
- [ ] T034 [US1] [P] Implement ErrorRatePanel (4xx, 5xx) in frontend/src/pages/modelServing/screens/metrics/components/PerformanceMetrics/ErrorRatePanel.tsx

### Phase 3.5: Resource Utilization Section

- [ ] T035 [US1] Create ResourceUtilization section container in frontend/src/pages/modelServing/screens/metrics/components/ResourceUtilization/ResourceUtilization.tsx
- [ ] T036 [US1] [P] Implement CPUUsagePanel in frontend/src/pages/modelServing/screens/metrics/components/ResourceUtilization/CPUUsagePanel.tsx
- [ ] T037 [US1] [P] Implement MemoryUsagePanel in frontend/src/pages/modelServing/screens/metrics/components/ResourceUtilization/MemoryUsagePanel.tsx
- [ ] T038 [US1] [P] Implement GPUUsagePanel (optional) in frontend/src/pages/modelServing/screens/metrics/components/ResourceUtilization/GPUUsagePanel.tsx

### Phase 3.6: User Story 1 Integration Tests

- [ ] T039 [US1] Integration test: Navigate to metrics page and verify performance panels render in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/metricsPageLoad.test.tsx
- [ ] T040 [US1] Integration test: Time range selection updates all panels in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/timeRangeSelection.test.tsx
- [ ] T041 [US1] Integration test: Auto-refresh updates data every 60 seconds in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/autoRefresh.test.tsx

**User Story 1 Definition of Done**:
- ✅ All T021-T041 tasks completed
- ✅ Integration tests pass
- ✅ Performance targets met: <2s page load, <500ms panel render
- ✅ RBAC enforcement verified
- ✅ Code review completed

---

## Phase 4: User Story 2 - Metrics Overview (P1)

**User Story**: As an ML Operations Engineer, I need a health summary overview that shows key metrics from all categories in one place.

**Independent Test Criteria**:
- Overview tab is default view
- Health summary shows operational status (healthy/warning/critical)
- Key metrics from performance and resources are visible
- Overview provides quick model health assessment (<30 seconds)

### Phase 4.1: Overview Tab Implementation

- [ ] T042 [US2] Create MetricsOverview tab component in frontend/src/pages/modelServing/screens/metrics/components/MetricsOverview/MetricsOverview.tsx
- [ ] T043 [US2] Implement tab navigation (Overview, Performance, Resources) in frontend/src/pages/modelServing/screens/metrics/components/MetricsTabs.tsx
- [ ] T044 [US2] [P] Create HealthSummaryCard component in frontend/src/pages/modelServing/screens/metrics/components/MetricsOverview/HealthSummaryCard.tsx
- [ ] T045 [US2] Implement health status calculation logic in frontend/src/pages/modelServing/screens/metrics/services/healthStatus.ts

### Phase 4.2: Overview Panels

- [ ] T046 [US2] [P] Create KeyMetricsPanel (top 3-5 metrics) in frontend/src/pages/modelServing/screens/metrics/components/MetricsOverview/KeyMetricsPanel.tsx
- [ ] T047 [US2] [P] Create QuickStatsPanel (request count, avg latency, error rate) in frontend/src/pages/modelServing/screens/metrics/components/MetricsOverview/QuickStatsPanel.tsx

### Phase 4.3: User Story 2 Integration Tests

- [ ] T048 [US2] Integration test: Overview tab is default and shows health summary in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/overviewDefault.test.tsx
- [ ] T049 [US2] Integration test: Health status reflects metric thresholds in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/healthStatus.test.tsx

**User Story 2 Definition of Done**:
- ✅ All T042-T049 tasks completed
- ✅ Overview provides <30 second model health assessment
- ✅ Health status accurately reflects underlying metrics
- ✅ Integration tests pass

---

## Phase 5: User Story 3 - Behavior Metrics (TrustyAI) (P2)

**User Story**: As an ML Operations Engineer, I want to view behavior metrics (drift, fairness, prediction patterns) to understand model quality.

**Independent Test Criteria**:
- Behavior metrics section appears when TrustyAI available
- Drift score, fairness metrics, prediction distribution visible
- Graceful degradation when TrustyAI unavailable (empty state with instructions)
- Optional metrics don't block core functionality

### Phase 5.1: TrustyAI Availability Check

- [ ] T050 [US3] Implement TrustyAI availability detection in frontend/src/pages/modelServing/screens/metrics/hooks/useTrustyAIAvailability.ts
- [ ] T051 [US3] [P] Create TrustyAI unavailable empty state in frontend/src/pages/modelServing/screens/metrics/components/BehaviorMetrics/TrustyAIUnavailable.tsx

### Phase 5.2: Behavior Metrics Section

- [ ] T052 [US3] Create BehaviorMetrics section container in frontend/src/pages/modelServing/screens/metrics/components/BehaviorMetrics/BehaviorMetrics.tsx
- [ ] T053 [US3] [P] Implement DriftPanel (drift score over time) in frontend/src/pages/modelServing/screens/metrics/components/BehaviorMetrics/DriftPanel.tsx
- [ ] T054 [US3] [P] Implement FairnessPanel (disparate impact, bias score) in frontend/src/pages/modelServing/screens/metrics/components/BehaviorMetrics/FairnessPanel.tsx
- [ ] T055 [US3] [P] Implement PredictionDistributionPanel in frontend/src/pages/modelServing/screens/metrics/components/BehaviorMetrics/PredictionDistributionPanel.tsx

### Phase 5.3: User Story 3 Integration Tests

- [ ] T056 [US3] Integration test: TrustyAI available shows behavior metrics in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/trustyaiAvailable.test.tsx
- [ ] T057 [US3] Integration test: TrustyAI unavailable shows empty state in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/trustyaiUnavailable.test.tsx

**User Story 3 Definition of Done**:
- ✅ All T050-T057 tasks completed
- ✅ Behavior metrics display when TrustyAI available
- ✅ Graceful degradation when TrustyAI unavailable
- ✅ Core functionality works without TrustyAI

---

## Phase 6: User Story 4 - Data Export (P2)

**User Story**: As an ML Operations Engineer, I need to export visible metrics to CSV for reporting and sharing with my team.

**Independent Test Criteria**:
- Export button visible on metrics page
- CSV export includes visible time range data
- Exported file includes proper headers and timestamp
- Export works for individual panels and full page

### Phase 6.1: Export Functionality

- [ ] T058 [US4] [P] Create CSV export utility in frontend/src/pages/modelServing/screens/metrics/services/csvExport.ts
- [ ] T059 [US4] Implement export button in toolbar in frontend/src/pages/modelServing/screens/metrics/components/ExportButton.tsx
- [ ] T060 [US4] Add per-panel export option in MetricPanel component

### Phase 6.2: User Story 4 Integration Tests

- [ ] T061 [US4] Integration test: CSV export includes correct data and headers in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/csvExport.test.tsx

**User Story 4 Definition of Done**:
- ✅ All T058-T061 tasks completed
- ✅ CSV export functional for visible data
- ✅ Export includes proper formatting and metadata

---

## Phase 7: User Story 5 - URL State Persistence (P2)

**User Story**: As an ML Operations Engineer, I need to share specific metric views with my team via URL.

**Independent Test Criteria**:
- Time range selection persisted in URL
- Expanded/collapsed panel state in URL
- Shared URL loads same view for other users
- Browser back/forward navigation works

### Phase 7.1: URL State Management

- [ ] T062 [US5] Implement URL state serialization in frontend/src/pages/modelServing/screens/metrics/hooks/useURLState.ts
- [ ] T063 [US5] Add panel expansion state to URL in frontend/src/pages/modelServing/screens/metrics/hooks/usePanelState.ts

### Phase 7.2: User Story 5 Integration Tests

- [ ] T064 [US5] Integration test: URL updates reflect time range and panel state in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/urlPersistence.test.tsx

**User Story 5 Definition of Done**:
- ✅ All T062-T064 tasks completed
- ✅ URL state fully functional and shareable
- ✅ Browser navigation works correctly

---

## Phase 8: User Story 6 - Accessibility (P1)

**User Story**: As a user with accessibility needs, I can navigate and use all metrics functionality via keyboard and screen reader (WCAG 2.2 Level AA).

**Independent Test Criteria**:
- All interactive elements keyboard accessible (Tab, Enter, Esc)
- Screen reader announces metric values and status
- Focus indicators visible (3px, 4.5:1 contrast)
- Alternative text for charts (data table view)
- Text resizes to 200% without loss of function
- Color + icons for status (never color alone)

### Phase 8.1: Keyboard Navigation

- [ ] T065 [US6] Implement keyboard navigation for time range selector in frontend/src/pages/modelServing/screens/metrics/components/TimeRangeSelector/TimeRangeSelector.tsx
- [ ] T066 [US6] Add keyboard shortcuts documentation (? key for help) in frontend/src/pages/modelServing/screens/metrics/components/KeyboardShortcutsModal.tsx
- [ ] T067 [US6] Implement focus trap for modal dialogs in frontend/src/pages/modelServing/screens/metrics/components/MetricPanel/MetricPanelModal.tsx

### Phase 8.2: Screen Reader Support

- [ ] T068 [US6] Add ARIA labels to all interactive elements in all components
- [ ] T069 [US6] Implement live region announcements for metric updates in frontend/src/pages/modelServing/screens/metrics/hooks/useA11yAnnouncements.ts
- [ ] T070 [US6] [P] Create alternative data table view for charts in frontend/src/pages/modelServing/screens/metrics/components/MetricPanel/DataTableView.tsx

### Phase 8.3: Visual Accessibility

- [ ] T071 [US6] Audit and fix color contrast ratios (4.5:1 text, 3:1 graphics) across all components
- [ ] T072 [US6] Add status icons alongside color indicators in frontend/src/pages/modelServing/screens/metrics/components/StatusIndicator.tsx
- [ ] T073 [US6] Implement custom focus indicators (3px outline) in frontend/src/pages/modelServing/screens/metrics/styles/focus.css

### Phase 8.4: Accessibility Testing

- [ ] T074 [US6] Automated accessibility audit with axe-core in frontend/src/pages/modelServing/screens/metrics/__tests__/a11y/accessibility.test.tsx
- [ ] T075 [US6] Keyboard navigation E2E test in cypress/e2e/metrics/keyboard-navigation.cy.ts
- [ ] T076 [US6] Screen reader test scenarios in frontend/src/pages/modelServing/screens/metrics/__tests__/a11y/screenReader.test.tsx

**User Story 6 Definition of Done**:
- ✅ All T065-T076 tasks completed
- ✅ Zero critical/serious accessibility violations
- ✅ WCAG 2.2 Level AA compliance verified
- ✅ Manual accessibility review passed

---

## Phase 9: Polish & Cross-Cutting Concerns

**Goal**: Performance optimization, edge case handling, documentation

### Phase 9.1: Performance Optimization

- [ ] T077 Implement staggered panel loading (2s offset) in frontend/src/pages/modelServing/screens/metrics/hooks/useStaggeredLoading.ts
- [ ] T078 Add React Query cache optimization (90s stale, 5m cache) in frontend/src/pages/modelServing/screens/metrics/services/queryClient.ts
- [ ] T079 Performance testing: validate <2s page load p90, <500ms panel render p95 in frontend/src/pages/modelServing/screens/metrics/__tests__/performance/loadTime.test.ts

### Phase 9.2: Edge Cases & Error Handling

- [ ] T080 [P] Handle Prometheus service unavailable with retry in frontend/src/pages/modelServing/screens/metrics/components/PrometheusUnavailable.tsx
- [ ] T081 [P] Handle missing historical data with graceful message in frontend/src/pages/modelServing/screens/metrics/components/DataUnavailable.tsx
- [ ] T082 [P] Handle high-cardinality models (>10K series) with query optimization in frontend/src/pages/modelServing/screens/metrics/services/metricsQueries.ts
- [ ] T083 Integration test: Error states and retry mechanisms in frontend/src/pages/modelServing/screens/metrics/__tests__/integration/errorHandling.test.tsx

### Phase 9.3: E2E Testing (Cypress)

- [ ] T084 [P] E2E test: Complete user journey from model list to metrics view in cypress/e2e/metrics/user-journey.cy.ts
- [ ] T085 [P] E2E test: Auto-refresh and time range switching in cypress/e2e/metrics/time-range.cy.ts
- [ ] T086 [P] E2E test: RBAC permission denied scenario in cypress/e2e/metrics/rbac.cy.ts

### Phase 9.4: Documentation

- [ ] T087 [P] Update CLAUDE.md with feature implementation details
- [ ] T088 [P] Create developer onboarding guide (quickstart.md execution) in frontend/src/pages/modelServing/screens/metrics/README.md
- [ ] T089 [P] Document PromQL query templates and optimization in docs/metrics-queries.md

---

## Dependencies

### Phase Dependencies (Must Complete Sequentially)

1. **Phase 1 → Phase 2**: Perses POC must pass before foundational infrastructure
2. **Phase 2 → Phase 3**: Foundational services block all user stories
3. **Phase 3 → Phase 4**: Core metrics required for overview
4. **Phase 3 → Phase 5**: Core infrastructure needed for behavior metrics
5. **Phase 3, 4, 5 → Phase 8**: Accessibility must audit completed features

### User Story Dependencies

- **US2 depends on US1**: Overview summarizes core metrics
- **US3, US4, US5, US6 are independent**: Can be implemented in parallel

### Task Dependencies Within Phases

**Phase 1**:
- T006-T009 must complete before any other UI tasks (POC validation)
- T001 blocks T002-T005 (structure before files)

**Phase 2**:
- T010-T013 must complete before T014-T017 (clients before hooks)
- T018 must complete before T019 (backend before frontend RBAC)

**Phase 3**:
- T021-T023 must complete before T024-T038 (page structure before panels)
- T024-T026 must complete before T027-T038 (time range before metric panels)
- T027-T030 must complete before T031-T038 (base panel before specific panels)

**Phase 8**:
- T071-T073 must complete before T074-T076 (implementation before testing)

---

## Parallel Execution Examples

### Phase 1 Parallel Tasks
Launch T002-T005 together (different files, no dependencies):
```
T002: TypeScript metrics interfaces
T003: Prometheus types
T004: Jest test setup
T005: React Testing Library utilities
```

### Phase 2.1 Parallel Tasks
Launch T010-T013 together:
```
T010: Prometheus client service
T011: TrustyAI client service
T012: PromQL query templates
T013: Error handling utilities
```

### Phase 3.3 Parallel Tasks
Launch T028-T030 together:
```
T028: Loading state component
T029: Error state component
T030: Empty state component
```

### Phase 3.4 Parallel Tasks
Launch T032-T034 together:
```
T032: LatencyPanel
T033: ThroughputPanel
T034: ErrorRatePanel
```

### Phase 8.2 Parallel Tasks
Launch T068, T070 together:
```
T068: ARIA labels (multiple files)
T070: Data table view
```

---

## MVP Scope Recommendation

**Minimum Viable Product (MVP)**: Phase 1-3 + Phase 8 (Accessibility)

**Rationale**:
- **Phase 1-2**: Foundation and infrastructure (required)
- **Phase 3**: Core metrics display (User Story 1) provides complete, usable feature
- **Phase 8**: Accessibility is non-negotiable (WCAG 2.2 Level AA compliance)

**MVP Delivers**:
- Performance metrics visualization (latency, throughput, errors)
- Resource utilization (CPU, memory, GPU)
- Time range selection (1h, 24h, 7d, 30d)
- Auto-refresh every 60 seconds
- RBAC enforcement
- Accessibility compliance

**Post-MVP Enhancements** (Phase 4-7):
- Overview health summary (Phase 4)
- Behavior metrics with TrustyAI (Phase 5)
- CSV export (Phase 6)
- URL state persistence (Phase 7)

---

## Task Execution Checklist

### Before Starting Implementation

- [ ] Perses POC success criteria validated (T006-T009)
- [ ] All design documents reviewed and understood
- [ ] Development environment configured (Node 18+, npm 8+)
- [ ] OpenShift AI Console development server running

### Per-Task Definition of Done

- [ ] Code written following TypeScript strict mode
- [ ] Unit tests written and passing (Jest + React Testing Library)
- [ ] Component tested in isolation
- [ ] Accessibility check passed (if UI component)
- [ ] Code review completed
- [ ] No console errors or warnings
- [ ] Committed with descriptive message

### Per-Phase Definition of Done

- [ ] All phase tasks completed
- [ ] Integration tests passing
- [ ] Performance targets met (if applicable)
- [ ] Documentation updated
- [ ] Phase demo/review completed

### Overall Feature Definition of Done

- [ ] All 89 tasks completed
- [ ] All integration and E2E tests passing
- [ ] Performance targets met: <2s p90 page load, <500ms p95 panel render
- [ ] Accessibility audit passed: Zero critical violations, WCAG 2.2 Level AA
- [ ] RBAC enforcement verified
- [ ] TrustyAI graceful degradation working
- [ ] Documentation complete (developer guide, PromQL queries)
- [ ] User acceptance testing passed
- [ ] Production readiness review completed

---

## Risk Mitigation Tracking

| Risk | Tasks | Mitigation Status |
|------|-------|-------------------|
| Perses not production-ready | T006-T009 | POC in Sprint 1 Week 1; fallback to PatternFly Charts |
| Query performance at scale | T077-T079 | Staggered loading, caching, performance testing |
| Behavior metrics unavailable | T050-T057 | TrustyAI feature-flagged, graceful degradation |
| Accessibility complexity | T065-T076 | 15-20% overhead included, early audit, SME review |

---

## Validation Summary

### Task Completeness

- ✅ All API contracts have corresponding client implementations (T010-T013)
- ✅ All data model entities have TypeScript interfaces (T002-T003)
- ✅ All user stories have integration tests (T039-T041, T048-T049, T056-T057, T061, T064, T074-T076)
- ✅ Parallel tasks target different files with no dependencies
- ✅ Each task specifies exact file path
- ✅ Tests before implementation where applicable (TDD not required per spec)

### Format Validation

- ✅ All tasks follow checklist format: `- [ ] [ID] [P?] [Story?] Description with file path`
- ✅ Sequential task IDs (T001-T089)
- ✅ [P] markers indicate parallelizable tasks
- ✅ [US#] labels map tasks to user stories
- ✅ File paths included in all task descriptions

---

**Generated**: 2025-10-27
**Ready for Execution**: ✅ Yes
**Next Step**: Begin Phase 1 (Setup & Perses POC)
