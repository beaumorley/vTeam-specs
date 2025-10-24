# vTeam-specs Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-10-24

## Active Technologies
- **Frontend**: React 18.2.0, TypeScript 4.x+, PatternFly 6.4.0, React Router 7.6.2
- **State Management**: Redux 4.2.0 (global), React Query (server state)
- **Visualization**: Perses (CNCF) with ECharts, fallback to PatternFly Charts
- **Testing**: Jest, React Testing Library, Cypress
- **Backend**: OpenShift AI Console (odh-dashboard) with Dynamic Plugin SDK v5.0.1

## Project Type
This is a **specification repository** for OpenShift AI Console features. The actual implementation lives in the odh-dashboard repository.

## Project Structure
```
specs/
├── 001-short-name-ambient/          # Enhanced Model Metrics Display feature
│   ├── spec.md                      # Feature specification
│   ├── plan.md                      # Implementation plan
│   ├── research.md                  # Technology decisions
│   ├── data-model.md                # TypeScript interfaces
│   ├── quickstart.md                # Developer onboarding
│   └── contracts/                   # OpenAPI specs
│       ├── prometheus-api.yaml      # Prometheus query API
│       └── trustyai-api.yaml        # TrustyAI metrics API
├── ux-architecture-metrics-display.md  # UX architecture decisions
└── [other features]
```

## Feature: Enhanced Model Metrics Display (001-short-name-ambient)

**Goal**: Display comprehensive metrics (performance + behavior) for deployed ML models

**Key Components**:
- Prometheus integration for performance metrics (latency, throughput, errors, resources)
- TrustyAI integration for behavior metrics (drift, fairness, predictions) - optional
- 1-minute auto-refresh with React Query caching
- WCAG 2.2 Level AA accessibility compliance
- RBAC-filtered queries (namespace-scoped)

**Technology Stack**:
- React 18 + TypeScript + PatternFly 6 (OpenShift console style)
- React Query for server state with 90s staleTime, 5min cacheTime
- Perses for visualization (pending POC in Sprint 1)
- Prometheus + PromQL for metrics backend
- Recording rules for query optimization (1m evaluation interval)

**Performance Targets**:
- Initial page load: <2s p90
- Panel render: <500ms p95
- Auto-refresh: 60s interval with staggered loading (2s offset per panel)

**RBAC Pattern**:
- Frontend: `useAccessReview` hook checks InferenceService permissions
- Backend: Middleware injects namespace filter into PromQL queries
- Multi-tenant: Namespace-scoped queries prevent cross-tenant data leakage

## Commands

```bash
# Specification workflow
./.specify/scripts/bash/setup-plan.sh --json      # Setup feature planning
SPECIFY_FEATURE="001-short-name-ambient" bash ... # Override current feature

# OpenShift AI Console development (in odh-dashboard repo)
npm install                                         # Install dependencies
npm start                                          # Start dev server (port 3000)
npm run test                                       # Run Jest tests
npm run test:e2e                                   # Run Cypress tests
npm run test:a11y                                  # Accessibility audit

# Prometheus query testing
kubectl port-forward -n openshift-monitoring svc/prometheus-k8s 9090:9090
curl "http://localhost:9090/api/v1/query?query=up"
```

## Code Style

**TypeScript**:
- Strict mode enabled
- Functional components with hooks (no class components)
- Props interfaces exported from component file
- React Query hooks for server state, Redux for global state

**Component Pattern**:
```typescript
interface MetricPanelProps {
  query: MetricQuery;
  timeRange: TimeRange;
}

export const MetricPanel: React.FC<MetricPanelProps> = ({ query, timeRange }) => {
  const { data, isLoading, isError } = usePrometheusMetric({ query });
  // Component logic
};
```

**Accessibility**:
- All interactive elements must have ARIA labels
- Focus indicators: 3px outline with 4.5:1 contrast
- Color + icons for status (never color alone)
- Alternative text for charts (data table view)

## Recent Changes
- 001-short-name-ambient: Enhanced Model Metrics Display feature added
  - Research completed: Perses, OpenShift AI Console SDK, Prometheus best practices
  - Data model defined: TypeScript interfaces for metrics entities
  - API contracts: OpenAPI specs for Prometheus and TrustyAI
  - Quickstart guide: 30-minute developer onboarding

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->