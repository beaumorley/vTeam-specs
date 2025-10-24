
# Implementation Plan: Enhanced Model Metrics Display

**Branch**: `001-short-name-ambient` | **Date**: 2025-10-24 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-short-name-ambient/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → If not found: ERROR "No feature spec at {path}"
2. Fill Technical Context (scan for NEEDS CLARIFICATION)
   → Detect Project Type from file system structure or context (web=frontend+backend, mobile=app+api)
   → Set Structure Decision based on project type
3. Fill the Constitution Check section based on the content of the constitution document.
4. Evaluate Constitution Check section below
   → If violations exist: Document in Complexity Tracking
   → If no justification possible: ERROR "Simplify approach first"
   → Update Progress Tracking: Initial Constitution Check
5. Execute Phase 0 → research.md
   → If NEEDS CLARIFICATION remain: ERROR "Resolve unknowns"
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, agent-specific template file (e.g., `CLAUDE.md` for Claude Code, `.github/copilot-instructions.md` for GitHub Copilot, `GEMINI.md` for Gemini CLI, `QWEN.md` for Qwen Code or `AGENTS.md` for opencode).
7. Re-evaluate Constitution Check section
   → If new violations: Refactor design, return to Phase 1
   → Update Progress Tracking: Post-Design Constitution Check
8. Plan Phase 2 → Describe task generation approach (DO NOT create tasks.md)
9. STOP - Ready for /tasks command
```

**IMPORTANT**: The /plan command STOPS at step 7. Phases 2-4 are executed by other commands:
- Phase 2: /tasks command creates tasks.md
- Phase 3-4: Implementation execution (manual or via tools)

## Summary
This feature enhances the Deployed Models Details page in the OpenShift AI Console to display comprehensive metrics including both performance and behavior-related data. The current "Endpoint Performance" tab will be replaced with an improved visualization experience powered by Perses, allowing ML Operations Engineers to diagnose model issues in under 15 minutes instead of 45-60 minutes. The system will retrieve metrics from Prometheus, organize them into intent-based sections (Operational Health, Model Quality, Resource Utilization), and support time ranges of 1 hour, 24 hours, 7 days, and 30 days with automatic 1-minute refresh intervals.

## Technical Context
**Language/Version**: TypeScript 4.x+ / ES6+ (React frontend)
**Primary Dependencies**: React, PatternFly 5 (UI framework), Perses (visualization), Prometheus API client, NEEDS CLARIFICATION (OpenShift AI Console SDK version and integration patterns)
**Storage**: N/A (read-only queries to Prometheus backend; no persistent storage in frontend)
**Testing**: Jest for unit tests, React Testing Library for component tests, Cypress/Playwright for integration tests (NEEDS CLARIFICATION on OpenShift AI Console test framework standards)
**Target Platform**: Modern web browsers (Chrome, Firefox, Safari, Edge current - 1), embedded in OpenShift AI Console web application
**Project Type**: web (frontend component within larger OpenShift AI Console application)
**Performance Goals**: <2s p90 initial page load, <500ms p95 metric panel render, support 50 concurrent users, 1-minute auto-refresh
**Constraints**: Must integrate with PatternFly design system, comply with WCAG 2.2 Level AA, PromQL queries <5s timeout, desktop-first (1280px+ width)
**Scale/Scope**: Single-page view within console, estimated 15-20 React components, 5-10 PromQL query templates, 3-4 metric dashboard sections, integration with existing console navigation and RBAC

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Note**: No project constitution file found. Applying general software engineering best practices as constitutional gates.

| Gate | Status | Justification |
|------|--------|---------------|
| **Simplicity**: Minimize abstractions, avoid over-engineering | ✅ PASS | Feature is well-scoped to metric visualization only; no unnecessary abstractions; leveraging existing Perses library instead of building custom charting |
| **Component Isolation**: Components should be self-contained and testable | ✅ PASS | React component-based architecture naturally supports isolation; each metric panel can be developed and tested independently |
| **Test-First Development**: Write tests before implementation | ✅ PASS | Phase 1 includes contract test generation; integration tests from user scenarios; TDD approach enforced in task ordering |
| **Performance Requirements**: Meet specified performance targets | ⚠️ MONITOR | <2s p90 page load and <500ms p95 panel render targets are aggressive; requires query optimization and caching strategy (addressed in technical risks) |
| **Accessibility**: WCAG 2.2 Level AA compliance | ⚠️ MONITOR | PatternFly provides accessible base components, but Perses integration needs validation; 15-20% overhead included in estimates |
| **Security**: Multi-tenancy and RBAC enforcement | ✅ PASS | FR-016 through FR-018 enforce namespace/project scope filtering and RBAC; query isolation prevents cross-tenant data leakage |
| **Graceful Degradation**: Handle failures gracefully | ✅ PASS | FR-019 through FR-021 define empty states and error handling; TrustyAI is optional with feature flags |
| **Maintainability**: Code should be readable and documented | ✅ PASS | Metric queries documented; component hierarchy clear; Phase 1 includes quickstart.md for onboarding |

**Overall**: PASS with 2 items to monitor during implementation

## Project Structure

### Documentation (this feature)
```
specs/[###-feature]/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output (/plan command)
├── data-model.md        # Phase 1 output (/plan command)
├── quickstart.md        # Phase 1 output (/plan command)
├── contracts/           # Phase 1 output (/plan command)
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (repository root)

**Note**: This is a specification repository. The actual implementation will be in the OpenShift AI Console repository (location TBD). The expected structure there would be:

```
frontend/
├── src/
│   ├── pages/
│   │   └── modelServing/
│   │       └── screens/
│   │           └── metrics/              # New feature location
│   │               ├── ModelMetricsPage.tsx
│   │               ├── components/
│   │               │   ├── MetricsOverview/
│   │               │   ├── PerformanceMetrics/
│   │               │   ├── BehaviorMetrics/
│   │               │   ├── TimeRangeSelector/
│   │               │   └── MetricPanel/
│   │               ├── hooks/
│   │               │   ├── usePrometheusQuery.ts
│   │               │   ├── useMetricsRefresh.ts
│   │               │   └── useTrustyAI.ts
│   │               ├── services/
│   │               │   ├── prometheusClient.ts
│   │               │   ├── metricsQueries.ts
│   │               │   └── trustyaiClient.ts
│   │               └── types/
│   │                   ├── metrics.ts
│   │                   └── prometheus.ts
│   └── __tests__/
│       └── pages/
│           └── modelServing/
│               └── screens/
│                   └── metrics/
│                       ├── integration/
│                       ├── components/
│                       └── services/
└── package.json
```

**Structure Decision**: Frontend-only feature within existing OpenShift AI Console web application. Follows console conventions with feature-based organization under `/pages/modelServing/screens/`. Components organized by domain (overview, performance, behavior), with shared hooks for data fetching and services for external API integration.

## Phase 0: Outline & Research
1. **Extract unknowns from Technical Context** above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. **Generate and dispatch research agents**:
   ```
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all NEEDS CLARIFICATION resolved

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

1. **Extract entities from feature spec** → `data-model.md`:
   - Entity name, fields, relationships
   - Validation rules from requirements
   - State transitions if applicable

2. **Generate API contracts** from functional requirements:
   - For each user action → endpoint
   - Use standard REST/GraphQL patterns
   - Output OpenAPI/GraphQL schema to `/contracts/`

3. **Generate contract tests** from contracts:
   - One test file per endpoint
   - Assert request/response schemas
   - Tests must fail (no implementation yet)

4. **Extract test scenarios** from user stories:
   - Each story → integration test scenario
   - Quickstart test = story validation steps

5. **Update agent file incrementally** (O(1) operation):
   - Run `.specify/scripts/bash/update-agent-context.sh claude`
     **IMPORTANT**: Execute it exactly as specified above. Do not add or remove any arguments.
   - If exists: Add only NEW tech from current plan
   - Preserve manual additions between markers
   - Update recent changes (keep last 3)
   - Keep under 150 lines for token efficiency
   - Output to repository root

**Output**: data-model.md, /contracts/*, failing tests, quickstart.md, agent-specific file

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:

1. **From API Contracts** (`contracts/prometheus-api.yaml`, `contracts/trustyai-api.yaml`):
   - Generate contract test tasks for each endpoint
   - Create API client service implementation tasks
   - Add error handling and retry logic tasks

2. **From Data Model** (`data-model.md`):
   - Create TypeScript type definitions (all interfaces exported)
   - Generate validation function tasks
   - Add state transition logic tasks

3. **From User Scenarios** (`spec.md` Acceptance Scenarios):
   - Each scenario → E2E test task
   - Each edge case → integration test task
   - User story validation via quickstart scenarios

4. **Component Hierarchy** (from project structure):
   - Page-level components (ModelMetricsPage)
   - Feature sections (MetricsOverview, PerformanceMetrics, BehaviorMetrics)
   - Reusable components (MetricPanel, TimeRangeSelector)
   - Hooks (usePrometheusQuery, useMetricsRefresh, useTrustyAI)
   - Services (prometheusClient, metricsQueries, trustyaiClient)

5. **POC Task** (High Priority - Sprint 1 Week 1):
   - **Critical**: Perses embedding proof-of-concept
   - Must complete before other UI tasks
   - Go/No-Go decision point for Perses vs fallback

**Ordering Strategy**:

**Sprint 1 - Foundation & POC (2 weeks)**:
1. [Critical] Perses POC (2-3 days) - MUST complete first
2. [P] TypeScript type definitions (data-model.md → types/)
3. [P] API contract tests (Prometheus, TrustyAI)
4. [P] Prometheus client service
5. [P] TrustyAI client service (with feature flag)
6. React Query hooks for metrics fetching
7. [P] RBAC integration (useAccessReview pattern)
8. Basic page routing and navigation

**Sprint 2 - Core Visualization (2 weeks)**:
9. [P] MetricsOverview component (health summary)
10. [P] TimeRangeSelector component
11. [P] PerformanceMetrics section (latency, throughput, errors)
12. [P] ResourceUtilization section (CPU, memory, GPU)
13. Auto-refresh implementation with pause/resume
14. [P] Error states and empty states
15. Integration tests for Scenarios 1-3

**Sprint 3 - Behavior Metrics & Advanced Features (2 weeks)**:
16. [P] BehaviorMetrics section (TrustyAI integration)
17. Graceful degradation for TrustyAI unavailable
18. [P] CSV export functionality
19. URL state persistence (time range, filters)
20. Integration tests for Scenarios 4-6
21. Performance optimization (staggered loading, caching)

**Sprint 4 - Accessibility & Edge Cases (1-2 weeks)**:
22. Accessibility audit and remediation
23. Keyboard navigation implementation
24. Screen reader support (ARIA labels)
25. Alternative text for charts (data table view)
26. [P] Edge case tests (data unavailable, service errors, high cardinality)
27. Load testing and performance validation
28. Documentation updates

**Parallelization Markers**:
- [P] = Can be executed in parallel (independent files/components)
- Non-[P] = Sequential dependency on previous task

**Estimated Output**:
- 28-32 numbered, dependency-ordered tasks in tasks.md
- 4 sprints total (7-8 weeks)
- Critical path: Perses POC → Core metrics → Behavior metrics → Accessibility

**Definition of Done** (per task):
- Unit tests written and passing
- Component tested with React Testing Library
- Accessibility check passed (if UI component)
- Code review completed
- No console errors or warnings

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)  
**Phase 4**: Implementation (execute tasks.md following constitutional principles)  
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |


## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command)
- [x] Phase 1: Design complete (/plan command)
- [ ] Phase 2: Task planning complete (/plan command - describe approach only)
- [ ] Phase 3: Tasks generated (/tasks command)
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS (with 2 items to monitor)
- [x] Post-Design Constitution Check: PASS (no new violations)
- [x] All NEEDS CLARIFICATION resolved (research.md complete)
- [x] Complexity deviations documented (none required)

---
*Based on Constitution v2.1.1 - See `/memory/constitution.md`*
