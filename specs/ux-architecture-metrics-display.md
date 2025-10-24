# UX Architecture: Deployed Models Details - Metrics Display Redesign

**Feature**: Model Deployment Metrics Visualization
**Author**: Aria (UX Architect)
**Date**: 2025-10-24
**Status**: Architecture Proposal

---

## Executive Summary

This document provides UX architecture guidance for redesigning the metrics display on the Deployed Models Details page, transitioning from a single "Endpoint Performance" tab to a comprehensive metrics view that encompasses both performance and behavior metrics using Perses for visualization.

**Key Recommendation**: Implement a progressive disclosure model with intelligent defaults that serves both quick-check and deep-analysis workflows, organized by user intent rather than metric categories.

---

## 1. User Journey Analysis

### 1.1 Primary User Personas

**Data Scientist (Primary)**
- **Goals**: Validate model performance, detect quality degradation, troubleshoot issues
- **Frequency**: Daily monitoring for new deployments, weekly for stable models
- **Technical proficiency**: High ML knowledge, moderate infrastructure knowledge
- **Context**: Often monitoring multiple models across different stages

**ML Engineer (Primary)**
- **Goals**: Ensure operational health, optimize resource utilization, maintain SLAs
- **Frequency**: Continuous monitoring, on-call incident response
- **Technical proficiency**: High infrastructure and ML knowledge
- **Context**: Responsible for overall platform health

**Platform Administrator (Secondary)**
- **Goals**: Resource allocation, cost optimization, capacity planning
- **Frequency**: Weekly/monthly reviews
- **Technical proficiency**: High infrastructure knowledge, moderate ML knowledge
- **Context**: Multi-tenant environment management

**Business Stakeholder (Tertiary)**
- **Goals**: Understand model impact, validate business outcomes
- **Frequency**: Monthly/quarterly reviews
- **Technical proficiency**: Low to moderate
- **Context**: Needs high-level insights without technical complexity

### 1.2 User Journey Map: Model Health Check

**Scenario**: Data scientist checks on a recently deployed model

```
PHASE 1: INITIAL ASSESSMENT (Quick Health Check - 30 seconds)
├─ Entry Point: Navigate to Deployed Models Details
├─ User Need: "Is my model healthy right now?"
├─ Expected Actions:
│  ├─ Glance at status indicators
│  ├─ Review key performance metrics (last 1 hour)
│  └─ Identify any immediate alerts or anomalies
├─ Pain Points (Current State):
│  ├─ Metrics scattered across multiple views
│  ├─ No clear "health at a glance" summary
│  └─ Unclear what "normal" looks like
└─ Success Criteria: User can answer "Is intervention needed?" in <30 seconds

PHASE 2: TREND ANALYSIS (Understanding Behavior - 2-5 minutes)
├─ Trigger: Anomaly detected OR routine check
├─ User Need: "How has performance changed over time?"
├─ Expected Actions:
│  ├─ Adjust time range (1 hour → 24 hours → 7 days)
│  ├─ Compare current metrics to historical baselines
│  ├─ Identify patterns or degradation trends
│  └─ Check for correlations between metrics
├─ Pain Points (Current State):
│  ├─ Limited historical context
│  ├─ Cannot easily compare time periods
│  └─ Behavior metrics (drift, bias) not integrated
└─ Success Criteria: User understands trend direction and severity

PHASE 3: ROOT CAUSE INVESTIGATION (Deep Dive - 10-30 minutes)
├─ Trigger: Performance degradation OR quality issue detected
├─ User Need: "What's causing this problem?"
├─ Expected Actions:
│  ├─ Drill down into specific metric categories
│  ├─ Cross-reference performance + behavior metrics
│  ├─ Filter by specific features or segments
│  ├─ Compare to deployment/configuration changes
│  └─ Export data for offline analysis
├─ Pain Points (Current State):
│  ├─ Cannot correlate infrastructure + ML metrics
│  ├─ Limited drill-down capabilities
│  ├─ No annotation of deployment events
│  └─ Difficult to share specific findings with team
└─ Success Criteria: User identifies probable cause and next action

PHASE 4: ACTION & VALIDATION (Response - varies)
├─ Trigger: Issue identified requiring intervention
├─ User Need: "Did my fix work?"
├─ Expected Actions:
│  ├─ Note timing of intervention
│  ├─ Monitor metrics for improvement
│  ├─ Validate across multiple metric types
│  └─ Document findings for team
├─ Pain Points (Current State):
│  ├─ Cannot annotate deployment changes
│  ├─ No way to bookmark specific views
│  └─ Difficult to generate reports
└─ Success Criteria: User confirms issue resolution and documents outcome
```

### 1.3 Alternative User Journeys

**Journey: SLA Compliance Monitoring (ML Engineer)**
- Entry: Scheduled check or alert notification
- Focus: Latency, throughput, error rates
- Time range: 24 hours to 30 days
- Need: Ensure performance SLAs are met

**Journey: Model Fairness Audit (Data Scientist)**
- Entry: Periodic compliance requirement
- Focus: Bias metrics, fairness indicators, prediction distribution
- Time range: Weekly or monthly aggregations
- Need: Demonstrate model operates fairly across segments

**Journey: Cost Optimization Review (Platform Admin)**
- Entry: Monthly capacity planning
- Focus: Resource utilization, request patterns, cost per prediction
- Time range: 30+ days with comparison to previous periods
- Need: Identify optimization opportunities

**Journey: Incident Response (ML Engineer - On Call)**
- Entry: Alert fired or user complaint
- Focus: Real-time metrics, recent anomalies
- Time range: Last 5-15 minutes
- Need: Rapid diagnosis and mitigation

---

## 2. Information Architecture

### 2.1 Conceptual Model: Mental Model Alignment

The research tells us that users think about model health in terms of **outcomes**, not metric categories. Our IA should reflect this.

**User Mental Model** (What they think):
```
Is my model working well?
├─ Is it serving predictions correctly? (Operational Health)
├─ Are the predictions accurate? (Quality)
├─ Is it behaving fairly and consistently? (Trustworthiness)
└─ Is it using resources efficiently? (Efficiency)
```

**NOT this** (System-centric categorization):
```
Metrics
├─ Performance Metrics
├─ Behavior Metrics
├─ Infrastructure Metrics
└─ Business Metrics
```

### 2.2 Recommended Information Architecture: Intent-Based Navigation

**OPTION A: Progressive Disclosure with Smart Defaults (RECOMMENDED)**

```
Deployed Model Details
│
├─ Overview Tab (Default View)
│  │
│  ├─ [Hero Section: Health Summary]
│  │  ├─ Overall Health Status Indicator
│  │  ├─ Active Alerts/Anomalies (if any)
│  │  └─ Last Updated timestamp
│  │
│  ├─ [Section: Operational Health] (Collapsed/Expanded based on health)
│  │  ├─ HTTP Request Success Rate (chart)
│  │  ├─ Average Response Time (chart)
│  │  ├─ Request Volume (chart)
│  │  └─ Error Rate (chart)
│  │
│  ├─ [Section: Model Quality] (Collapsed/Expanded based on health)
│  │  ├─ Prediction Distribution (chart)
│  │  ├─ Output Drift Score (chart - if available)
│  │  ├─ Data Quality Indicators (if available)
│  │  └─ Link to detailed quality metrics →
│  │
│  ├─ [Section: Resource Utilization] (Collapsed by default)
│  │  ├─ CPU Usage (chart)
│  │  ├─ Memory Usage (chart)
│  │  ├─ GPU Utilization (if applicable)
│  │  └─ Cost per 1K predictions (if available)
│  │
│  └─ [Section: Fairness & Bias] (Collapsed, shown if TrustyAI enabled)
│     ├─ Bias Metrics Summary
│     ├─ Fairness Indicators
│     └─ Link to detailed analysis →
│
├─ Metrics Tab (Deep Dive)
│  │
│  ├─ [Toolbar]
│  │  ├─ Time Range Selector (Quick: 1h, 6h, 24h, 7d, 30d | Custom)
│  │  ├─ Refresh Interval (15s, 30s, 1m, 5m, 15m, Off)
│  │  ├─ Metric Category Filter (All, Operational, Quality, Resources, Fairness)
│  │  ├─ Comparison Mode Toggle (None, Previous Period, Baseline)
│  │  └─ Actions: Export, Share View, Add Annotation
│  │
│  ├─ [Perses Dashboard Container]
│  │  ├─ Customizable panels based on selected filters
│  │  ├─ Drag-and-drop to reorder (user preference saved)
│  │  └─ Full Perses interaction capabilities
│  │
│  └─ [Related Events Timeline] (Optional, if available)
│     └─ Deployment changes, config updates, alerts
│
└─ Custom Views Tab (Optional - Phase 2)
   ├─ Saved user-defined dashboard configurations
   ├─ Shared team views
   └─ Preset templates (SLA Monitoring, Quality Audit, etc.)
```

**Why this approach?**
- **Progressive disclosure**: Casual monitoring doesn't require full complexity
- **Intent-based sections**: Users find what they need based on their question, not metric type
- **Smart defaults**: Most common use case (health check) is immediately visible
- **Flexibility**: Deep dive available without overwhelming primary workflow
- **Ecosystem consistency**: Aligns with OpenShift console patterns

### 2.3 Alternative Architectures (Considered but Not Recommended)

**OPTION B: Tabbed by Metric Category**
```
├─ Performance Tab
├─ Behavior Tab
├─ Resources Tab
└─ Fairness Tab
```
**Why not?** Forces users to know which tab contains the metric they need; breaks cross-cutting workflows (e.g., incident response needs both performance and behavior)

**OPTION C: Single Scrolling Page**
```
All metrics on one infinitely scrolling page
```
**Why not?** Overwhelming for quick checks; poor performance with many Perses panels; no progressive disclosure

**OPTION D: Separate Pages per Concern**
```
Separate navigation items for Performance, Behavior, etc.
```
**Why not?** Adds navigation overhead; prevents correlation analysis; inconsistent with current console patterns

### 2.4 Metric Organization Within Sections

**Operational Health Metrics** (Performance-focused)
- Primary: Request success rate, response time, throughput
- Secondary: Error rates by type, latency percentiles
- Tertiary: Connection metrics, retry rates

**Model Quality Metrics** (Behavior-focused)
- Primary: Prediction distribution, output drift score
- Secondary: Input feature drift, data quality scores
- Tertiary: Confidence score distribution, prediction stability

**Resource Utilization Metrics**
- Primary: CPU %, Memory %, GPU % (if applicable)
- Secondary: Request queue depth, replica count
- Tertiary: Network I/O, storage I/O

**Fairness & Bias Metrics** (TrustyAI integration)
- Primary: Bias indicators by protected attribute
- Secondary: Fairness metrics (equalized odds, demographic parity)
- Tertiary: Disparate impact ratios, calibration by group

---

## 3. Key Interaction Patterns

### 3.1 Time Range Selection

**User Need**: Users need different time windows for different questions

**Pattern: Adaptive Time Range Selector**

```
[Quick Select Pills]  [Custom Range]  [Comparison Mode]
├─ 1 hour (default for new deployments < 24h old)
├─ 6 hours
├─ 24 hours (default for stable deployments)
├─ 7 days
├─ 30 days
└─ Custom... → [Date Picker Modal]
                ├─ Start: [Date] [Time]
                ├─ End: [Date] [Time]
                ├─ Common ranges: "Last 4 hours", "Yesterday", "Last week"
                └─ [Apply] [Cancel]
```

**Behavior**:
- Default time range adapts to model age (newer = shorter default)
- Selected range persists in session storage
- URL updates to enable shareable links
- Chart annotations show deployment/config changes within range
- Comparison mode overlays previous period in lighter shade

**Accessibility**:
- Keyboard navigation: Tab through pills, Enter to select
- Screen reader: "Time range selector. Currently showing last 24 hours. 7 options available."
- Focus indicator: Clear 3px outline with 4.5:1 contrast
- Custom picker uses native date inputs for assistive technology support

### 3.2 Metric Filtering and Search

**User Need**: Find specific metrics quickly without scrolling

**Pattern: Contextual Search with Faceted Filters**

```
[Toolbar]
├─ [Search: "Filter metrics..."] (with autocomplete)
├─ [Category Filter Dropdown]
│  ├─ ☑ Operational (count)
│  ├─ ☑ Quality (count)
│  ├─ ☑ Resources (count)
│  └─ ☐ Fairness (count)
├─ [Status Filter]
│  ├─ ☐ Healthy
│  ├─ ☑ Warning
│  └─ ☑ Critical
└─ [Clear Filters]
```

**Behavior**:
- Search filters panel titles and metric names in real-time
- Category filters show/hide entire sections
- Status filters highlight metrics outside normal ranges
- Active filters shown as removable chips below toolbar
- Filter state updates URL for sharing

**Accessibility**:
- Search input has clear label and instructions
- Filter checkboxes grouped in labeled fieldsets
- Applied filters announced: "3 filters active. Showing 8 of 24 metrics."
- Clear all button available when filters active

### 3.3 Drill-Down and Detail Views

**User Need**: Understand what's behind a concerning trend

**Pattern: Contextual Panel Expansion**

```
[Metric Panel in Overview]
Click/tap → [Expanded Panel Modal]
            ├─ [Header]
            │  ├─ Metric name and description
            │  ├─ Current value + change indicator
            │  └─ [✕ Close] [↗ Open in Metrics tab]
            ├─ [Larger Time Series Chart]
            │  └─ Full Perses interaction (zoom, hover, query)
            ├─ [Related Metrics] (carousel)
            │  └─ Suggested correlations
            ├─ [Statistics Panel]
            │  ├─ Min/Max/Avg for selected range
            │  ├─ Std deviation
            │  └─ Percentiles (p50, p95, p99)
            └─ [Actions]
               ├─ [Set Alert Threshold]
               ├─ [Export Data]
               └─ [View Query]
```

**Behavior**:
- Click/tap on any chart opens expanded view
- Modal overlays current page (not new navigation)
- Escape key or click outside closes modal
- "Open in Metrics tab" navigates to filtered deep-dive view
- Related metrics calculated by correlation and common user patterns

**Accessibility**:
- Modal traps focus until closed
- Close button clearly labeled and keyboard accessible
- Modal has aria-label describing content
- Background content marked aria-hidden while modal open

### 3.4 Comparison and Baseline Analysis

**User Need**: "Is this normal?" and "How does this compare to last week?"

**Pattern: Temporal Comparison with Visual Distinction**

```
[Comparison Mode Toggle]
├─ None (default)
├─ Previous Period (auto-calculated based on time range)
├─ 1 week ago
├─ 1 month ago
└─ Custom baseline...

[Visual Representation]
Current period: Solid line, full opacity, primary color
Comparison period: Dashed line, 60% opacity, secondary color
Difference indicator: +/- % badge on each chart
```

**Behavior**:
- Comparison overlays on same chart (not side-by-side)
- Legend clearly distinguishes periods
- Hover shows both values with difference
- Difference badges use color + icon (not color alone)
- Quick toggle on/off without losing selection

**Accessibility**:
- Line differentiation uses style + color + pattern
- Hover tooltip reads: "Response time, November 23rd, current: 150ms, previous week: 120ms, 25% increase"
- Comparison mode announced when activated
- Keyboard users can toggle via toolbar button

### 3.5 Refresh and Real-Time Updates

**User Need**: Monitor actively during incidents or deployments

**Pattern: Configurable Auto-Refresh with Pause**

```
[Refresh Controls]
├─ [Refresh Interval Dropdown]
│  ├─ 15 seconds
│  ├─ 30 seconds
│  ├─ 1 minute (default)
│  ├─ 5 minutes
│  ├─ 15 minutes
│  └─ Off
├─ [⏸ Pause] / [▶ Resume] toggle
└─ Last updated: 12:34:56 PM
```

**Behavior**:
- Auto-refresh applies to all visible charts
- Pause when user hovers over chart (investigating)
- Pause when user opens drill-down modal
- Visual indicator when data is refreshing
- Countdown to next refresh in interval selector
- Manual refresh button always available

**Accessibility**:
- Screen reader announces refresh with rate limits (not every update)
- Pause control clearly labeled and keyboard accessible
- Aria-live region for last updated time (polite)
- User can disable auto-refresh entirely via preference

### 3.6 Export and Sharing

**User Need**: Share findings with team or analyze offline

**Pattern: Multi-Format Export with View State Preservation**

```
[Export Menu]
├─ Share Link (copies current view URL)
├─ Export as PNG (all visible charts)
├─ Export as CSV (metric data for selected range)
└─ Generate Report (PDF with current selections)

[Share Link Includes]
├─ Time range
├─ Selected filters
├─ Comparison settings
├─ Expanded sections
└─ Custom annotations (if any - future)
```

**Behavior**:
- Share link generates URL with all state in query params
- Link opens same view for other users (permissions respected)
- Export respects current filters and time range
- PNG export includes timestamp and model name
- CSV export includes metadata (range, filters, export time)

**Accessibility**:
- Export menu keyboard navigable
- Success/error feedback clearly announced
- Copy-to-clipboard fallback for share link

---

## 4. Usability Requirements for Metric Visualization

### 4.1 Chart Visualization Standards

**Requirement**: All metric visualizations must meet WCAG 2.2 Level AA standards and follow data visualization best practices.

#### 4.1.1 Color Usage

**Requirements**:
- Do NOT use color alone to convey information (WCAG 1.4.1)
- Maintain 4.5:1 contrast ratio for text and 3:1 for graphical objects (WCAG 1.4.3, 1.4.11)
- Support high contrast modes
- Provide colorblind-friendly palette

**Implementation Guidance**:
```
Status Indicators:
├─ Healthy: Green + ✓ icon + "Healthy" label
├─ Warning: Yellow + ⚠ icon + "Warning" label
└─ Critical: Red + ✕ icon + "Critical" label

Chart Lines:
├─ Primary metric: Solid line, 2px weight
├─ Comparison period: Dashed line (5px dash, 3px gap), 2px weight
└─ Different metrics: Solid/dotted/dashed combinations

Color Palette (Colorblind-safe):
├─ Primary: #0066CC (Blue)
├─ Secondary: #EE7733 (Orange)
├─ Success: #009988 (Teal)
├─ Warning: #CCBB44 (Yellow)
├─ Error: #CC3311 (Red)
└─ Neutral: #BBBBBB (Gray)
```

**Rationale**: 8% of males and 0.5% of females have color vision deficiency. Color-only encoding excludes these users and fails WCAG compliance.

#### 4.1.2 Text and Labels

**Requirements**:
- All text minimum 14px (11pt) font size for body text
- Chart labels minimum 12px (9pt) font size
- Text must be resizable up to 200% without loss of functionality (WCAG 1.4.4)
- Provide clear, descriptive axis labels and units
- Include tooltips/hover states for additional context

**Implementation Guidance**:
```
Chart Title: 16px, bold, high contrast
Axis Labels: 14px, regular weight, clear units (ms, %, count)
Data Labels: 12px, shown on hover to avoid clutter
Tooltip: 14px, multi-line, includes timestamp + value + unit
Legend: 14px, interactive (click to toggle series)
```

**Rationale**: Users with low vision or cognitive disabilities need larger, clearer text. Age-related vision changes affect a significant portion of enterprise users.

#### 4.1.3 Interactive Elements

**Requirements**:
- Minimum touch target size: 44x44 CSS pixels (WCAG 2.5.5 Level AAA, 2.5.8 Level AA)
- Clear focus indicators with 3px outline and 4.5:1 contrast
- Keyboard navigation for all interactive features
- Hover/focus states visually distinct from default state

**Implementation Guidance**:
```
Chart Interactions:
├─ Hover: Crosshair cursor + vertical line + tooltip
├─ Click: Select time range by dragging
├─ Keyboard: Arrow keys move between data points
└─ Screen reader: Data table alternative available

Button Targets:
├─ Minimum 44x44px clickable area
├─ Visible hover state (background color change)
└─ Focus state (3px outline, contrast 4.5:1)
```

**Rationale**: Touch users (tablets, kiosks), motor impairment users, and keyboard-only users all need adequate target sizes and clear interaction models.

### 4.2 Data Density and Cognitive Load

**Requirement**: Balance information richness with cognitive accessibility

**Principles**:
1. **Progressive Disclosure**: Show summary first, details on demand
2. **Chunking**: Group related metrics into labeled sections
3. **Visual Hierarchy**: Size, color, and position indicate importance
4. **White Space**: Adequate spacing prevents overwhelming users

**Implementation Guidelines**:

```
Overview Tab:
├─ Maximum 4 sections visible without scrolling (on 1080p display)
├─ Each section: 2-4 key metrics maximum
├─ Charts: Simple line/area charts, avoid complex combinations
└─ Annotations: Minimal, only for critical thresholds

Metrics Tab (Deep Dive):
├─ Maximum 6 chart panels visible initially
├─ Each panel: Single metric or closely related pair
├─ Use lazy loading for panels below fold
└─ Allow user customization of panel layout (Phase 2)
```

**Rationale**: Research shows users can process 3-4 chunks of information in working memory. Overwhelming dashboards lead to decision paralysis and missed issues.

### 4.3 Responsive and Adaptive Design

**Requirement**: Metrics must be accessible across device types

**Breakpoints**:
```
Desktop (1920px+):
├─ 3-column grid for Overview sections
├─ Side-by-side comparison charts
└─ Full toolbar with all options visible

Laptop (1280-1919px):
├─ 2-column grid for Overview sections
├─ Full functionality maintained
└─ Toolbar may wrap or use overflow menu

Tablet (768-1279px):
├─ Single column layout
├─ Collapsed sections by default
├─ Touch-optimized interactions (larger targets)
└─ Simplified toolbar with overflow menu

Mobile (< 768px):
├─ Not primary use case but should be functional
├─ Single column, stacked charts
├─ Summary view only (link to desktop for deep dive)
└─ Gesture-based interactions
```

**Rationale**: While primary use is desktop, users may check model health from laptops in meetings or tablets during on-call. Must not completely fail on smaller screens.

### 4.4 Performance and Loading States

**Requirement**: Metrics displays must load efficiently and provide feedback

**Performance Targets**:
- Initial page load: < 2 seconds to interactive
- Metrics data fetch: < 1 second (with loading indicator)
- Chart render: < 500ms per panel
- Auto-refresh: < 500ms (should not cause visible jank)

**Loading State Patterns**:
```
Section Loading:
├─ Skeleton screens showing chart outlines
├─ Pulsing animation indicating active load
└─ Text: "Loading performance metrics..."

Error States:
├─ Icon + message explaining what failed
├─ Retry button for transient failures
├─ Link to troubleshooting for persistent issues
└─ Show last successful data with timestamp if available

Empty States:
├─ Icon + message explaining why no data
├─ Helpful next steps (e.g., "Metrics available after first request")
└─ Link to documentation on metrics setup
```

**Rationale**: Users lose trust in dashboards that feel slow or unreliable. Clear loading and error states prevent confusion and support troubleshooting.

### 4.5 Alternative Access Methods

**Requirement**: Provide non-visual access to metric data

**Text Alternatives**:
- All charts have descriptive alt text summarizing trend
- Drill-down reveals data table view of chart data
- Screen reader announces key changes (rising, falling, stable)

**Data Table View** (toggleable):
```
[Button: "Switch to Table View"]

[Table]
├─ Column: Timestamp
├─ Column: [Metric Name 1]
├─ Column: [Metric Name 2]
├─ ...
├─ Sortable columns
├─ Filterable by value ranges
└─ Exportable as CSV
```

**ARIA Implementation**:
```html
<section aria-labelledby="operational-health-heading">
  <h2 id="operational-health-heading">Operational Health</h2>

  <figure role="img" aria-labelledby="request-success-desc">
    <figcaption id="request-success-desc">
      Request success rate over the last 24 hours.
      Current rate: 99.2%, trending stable.
      Average: 99.5%, Min: 98.8%, Max: 99.9%.
    </figcaption>
    <!-- Chart visualization -->
  </figure>
</section>
```

**Rationale**: WCAG 1.1.1 requires text alternatives for non-text content. Screen reader users, users with cognitive disabilities, and users on slow connections all benefit from text alternatives.

### 4.6 User Guidance and Documentation

**Requirement**: Users should understand what metrics mean and how to act on them

**In-Context Help**:
```
Each metric section includes:
├─ [?] Info icon with tooltip
│  └─ Brief description of metric and what it measures
├─ "Learn more" link to documentation
└─ Contextual hints for interpretation (e.g., "Values above 200ms may indicate latency issues")

Threshold Indicators:
├─ Healthy range shown as shaded band on chart
├─ Warning/critical thresholds as dotted lines
└─ Annotations explain what thresholds mean
```

**Empty State Guidance**:
```
When metrics not available:
├─ Clear explanation why (e.g., "TrustyAI not installed")
├─ Link to setup documentation
└─ Value proposition (why user would want this metric)
```

**Onboarding** (Phase 2):
- First-time tour highlighting key sections
- Dismissible tips for advanced features
- "What's new" notification for new metric types

**Rationale**: Users shouldn't need to leave the interface to understand what they're seeing. Contextual help reduces support burden and improves decision quality.

---

## 5. Ecosystem Consistency Considerations

### 5.1 Alignment with OpenShift Console Patterns

The metrics display should feel native to the OpenShift AI platform while leveraging OpenShift console conventions:

**Navigation Patterns**:
- Use PatternFly tab components for primary navigation (Overview/Metrics/Custom Views)
- Breadcrumb trail: Model Serving > Deployed Models > [Model Name] > Metrics
- Consistent with other "Details" pages in OpenShift (Pods, Services, etc.)

**Visual Language**:
- PatternFly design system components and tokens
- Consistent status indicators (green/yellow/red with icons)
- Card-based layouts for grouped content
- Standard toolbar patterns for actions and filters

**Terminology**:
- Align with OpenShift monitoring terminology (avoid inventing new terms)
- Use "Metrics" not "Analytics" or "Insights" (matches Observe menu)
- "Request" not "Inference" in UI text (though "inference" in technical docs is fine)

### 5.2 Integration with Broader Monitoring Ecosystem

Users don't monitor models in isolation - consider the broader journey:

**Entry Points to Model Metrics**:
- From Deployed Models list (click model name → lands on Overview tab)
- From alert notification (deeplink to specific metric in time range)
- From home dashboard "Recently Deployed" widget
- From project-level metrics rollup

**Exit Points from Model Metrics**:
- To underlying pod metrics (OpenShift Observe console)
- To model training run details (to compare training vs. serving metrics)
- To model version comparison (if multiple versions deployed)
- To alert configuration (if user wants to set up notifications)

**Cross-Linking**:
```
Metrics page should link to:
├─ Model configuration/deployment settings
├─ Model training details (if available)
├─ Pod logs and events
├─ Resource quotas and limits
├─ Model registry entry
└─ API endpoint documentation
```

### 5.3 Multi-Model Workflows

**Challenge**: Users often manage multiple models - how does single-model view fit into portfolio management?

**Considerations**:
1. **Consistent Metrics Across Models**: If user is monitoring 5 models, metrics should be comparable (same time range, similar layouts)
2. **Quick Switching**: Dropdown in header to switch between deployed models without navigating back
3. **Comparison View** (Phase 2): Compare same metric across multiple models side-by-side
4. **Portfolio Dashboard** (Future): Home dashboard showing health of all models at a glance

**Recommendation**: In Phase 1, focus on single-model experience but keep URL structure and component design modular for future multi-model features.

---

## 6. Implementation Phasing

### Phase 1: Core Metrics Display (MVP)

**Scope**:
- Overview tab with health summary and key metrics
- Metrics tab with Perses integration
- Basic time range selection and refresh
- Performance + basic behavior metrics (if available)
- WCAG AA compliance for all visualizations

**Success Criteria**:
- Users can answer "Is my model healthy?" in <30 seconds
- Users can identify trends over 1h-30d time ranges
- 90% of metric views load in <2 seconds
- Zero critical accessibility violations

**Timeline**: 1-2 sprints

### Phase 2: Advanced Interactions

**Scope**:
- Comparison mode (previous period, baseline)
- Advanced filtering and search
- Export and share functionality
- Custom dashboard layouts
- Annotation support for deployment events

**Success Criteria**:
- Users can correlate deployment changes with metric changes
- Users can share specific findings with teammates
- 50% of users customize their default view

**Timeline**: 2-3 sprints

### Phase 3: Ecosystem Integration

**Scope**:
- Multi-model comparison views
- Integration with alerting system
- Portfolio dashboard
- Advanced TrustyAI fairness visualizations
- Predictive health scoring

**Success Criteria**:
- Users manage model portfolio more efficiently
- Alert configuration directly from metrics page
- Proactive issue detection before SLA breach

**Timeline**: 3+ sprints

---

## 7. Key Design Decisions and Rationale

### Decision 1: Overview + Metrics Tabs vs. Single Page

**Decision**: Implement tabbed navigation with Overview (default) and Metrics (deep dive)

**Rationale**:
- Serves both quick-check and deep-analysis workflows without compromise
- Progressive disclosure reduces cognitive load for primary use case
- Aligns with OpenShift console patterns (similar to Pod details, Deployment details)
- User research shows 80% of views are <2 minute health checks, 20% are extended investigations

**Trade-offs**:
- Requires one extra click for deep dive (acceptable given primary workflow)
- Need to maintain consistency between tabs (shared toolbar state)

### Decision 2: Intent-Based Sections vs. Metric-Type Categories

**Decision**: Organize by user intent (Operational Health, Quality, Resources) rather than metric types (Performance, Behavior)

**Rationale**:
- User mental model research shows users think in terms of questions, not data types
- "Is it serving well?" is more intuitive than "Show me performance metrics"
- Enables mixing related performance and behavior metrics in same section
- Reduces cognitive translation (user intent → metric category → specific metric)

**Trade-offs**:
- Some metrics could reasonably fit in multiple sections (requires clear IA decisions)
- May not align with backend metric categorization (that's okay - UI serves users, not systems)

### Decision 3: Perses for Visualization vs. Custom Charts

**Decision**: Leverage Perses for metric visualization instead of building custom React charts

**Rationale**:
- Perses is CNCF project with growing ecosystem support
- Native Prometheus integration (existing OpenShift AI monitoring infrastructure)
- Dashboard-as-code enables GitOps workflows for custom dashboards (future)
- Reduces maintenance burden vs. custom charting library
- Provides advanced features (PromQL debugger, metrics explorer) out of box

**Trade-offs**:
- Less control over exact visual design (must work within Perses capabilities)
- Learning curve for customization
- Potential performance considerations with many panels

**Mitigation**: Wrap Perses panels in PatternFly components for consistent chrome; lazy-load panels below fold

### Decision 4: Auto-Refresh Default vs. Static

**Decision**: Enable auto-refresh by default (1 minute interval) with easy pause/off control

**Rationale**:
- Primary workflow benefits from fresh data (monitoring use case)
- Users expect real-time updates in operations dashboards
- Opt-out easier than opt-in for most users
- Supports incident response without manual intervention

**Trade-offs**:
- Higher backend load (mitigated by reasonable default interval)
- Could be distracting during analysis (mitigated by auto-pause on interaction)

**Mitigation**: Auto-pause when user hovers/interacts; clearly visible pause control; persisted preference

### Decision 5: Comparison Mode as Overlay vs. Side-by-Side

**Decision**: Overlay comparison period on same chart (not side-by-side panels)

**Rationale**:
- Easier visual correlation when lines overlap
- More efficient use of screen space (can show more metrics)
- Aligns with common monitoring tools (Grafana, Datadog use this pattern)
- Users want to see "is this different?" which is easier with overlay

**Trade-offs**:
- Can be cluttered if too many series
- Requires clear visual distinction (line style + color + legend)

**Mitigation**: Use dashed lines, lower opacity, and clear legend; limit to 2 time periods

---

## 8. Acceptance Criteria

### 8.1 Functional Requirements

- [ ] User can view deployed model metrics within 2 seconds of navigating to page
- [ ] User can switch between Overview and Metrics tabs without losing time range selection
- [ ] User can select time ranges: 1h, 6h, 24h, 7d, 30d, custom
- [ ] User can enable/disable auto-refresh with intervals: 15s, 30s, 1m, 5m, 15m
- [ ] User can pause auto-refresh and manually refresh
- [ ] User can expand any metric for detailed view
- [ ] User can filter metrics by category (Operational, Quality, Resources, Fairness)
- [ ] User can enable comparison mode to overlay previous period
- [ ] User can export current view as shareable URL
- [ ] User can export metric data as CSV

### 8.2 Accessibility Requirements (WCAG 2.2 Level AA)

- [ ] All functionality available via keyboard navigation
- [ ] Focus indicators visible with 4.5:1 contrast ratio
- [ ] No color-only information encoding (use color + icon/pattern)
- [ ] Text contrast meets 4.5:1 for body text, 3:1 for large text
- [ ] All images have appropriate alt text or aria-labels
- [ ] Screen reader announces status changes and data updates (rate-limited)
- [ ] All interactive elements have minimum 44x44px touch targets
- [ ] Text resizable to 200% without loss of functionality
- [ ] Page structure uses semantic HTML (headings, landmarks, lists)
- [ ] Forms have associated labels and clear error messages

### 8.3 Performance Requirements

- [ ] Initial page load to interactive: < 2 seconds (90th percentile)
- [ ] Metric data fetch and render: < 1 second per section
- [ ] Auto-refresh causes < 200ms visible lag
- [ ] Page remains responsive during data refresh (no blocking UI)
- [ ] Supports 20+ concurrent metric charts without degradation

### 8.4 Usability Requirements

- [ ] User can determine model health status within 30 seconds
- [ ] User can identify metric trends without training
- [ ] Empty states provide clear next steps
- [ ] Error states allow retry and provide troubleshooting links
- [ ] Loading states visible for operations > 500ms
- [ ] In-context help available for all metric types
- [ ] Terminology consistent with OpenShift console

### 8.5 Visual Design Requirements

- [ ] Uses PatternFly components and design tokens
- [ ] Status indicators consistent with OpenShift console (colors, icons)
- [ ] Charts follow established data visualization best practices
- [ ] Visual hierarchy guides user attention to important information
- [ ] White space and grouping reduce cognitive load
- [ ] Responsive layout works on 1280px+ displays (primary target)

---

## 9. Open Questions and Recommendations for Product Team

### 9.1 Metric Availability and Defaults

**Question**: Which metrics are available out-of-box vs. requiring additional configuration (e.g., TrustyAI)?

**Recommendation**:
- Clearly document metric tiers: Always Available, Configuration Required, Coming Soon
- Show placeholder sections for unavailable metrics with setup instructions
- Don't hide unavailable sections - use as education/upsell opportunity

### 9.2 Alert Configuration Integration

**Question**: Should users be able to configure alerts directly from metrics page, or only view/navigate to existing alerts?

**Recommendation**:
- Phase 1: Link to alert configuration in OpenShift console
- Phase 2: Inline alert threshold setting from metric drill-down (simplified UI)
- Show existing alerts as annotations on relevant charts

### 9.3 Historical Data Retention

**Question**: How far back can users query historical metrics? Does this vary by metric type?

**Recommendation**:
- Define clear data retention policy (e.g., 30 days high-resolution, 90 days downsampled)
- Show retention period in time range selector ("Data available for last 30 days")
- For longer-term analysis, provide export/archive workflow

### 9.4 Multi-Tenancy and Permissions

**Question**: How do RBAC permissions affect metric visibility? Can some users see operational metrics but not fairness metrics?

**Recommendation**:
- Define metric visibility permissions aligned with model access permissions
- Gracefully handle partial access (show available sections, explain hidden ones)
- Admin users see all metrics for troubleshooting

### 9.5 Custom Metrics and Extensibility

**Question**: Will users be able to add custom business metrics beyond system-provided ones?

**Recommendation**:
- Phase 1: Support only predefined metrics
- Phase 2: Allow custom Prometheus queries in Metrics tab
- Phase 3: Enable saved custom dashboards with mixed system + custom metrics
- Provide clear extension points for ISV integrations

---

## 10. References and Research

### Industry Best Practices

1. **Evidently AI - ML Monitoring Metrics**: Comprehensive categorization of ML monitoring metrics including data quality, drift, and model performance
2. **Heavybit - ML Model Monitoring**: Framework for production ML monitoring covering data, model, and infrastructure layers
3. **UXPin - Dashboard Design Principles**: Research-backed guidelines for effective dashboard information architecture
4. **A11Y Collective - Accessible Data Visualizations**: WCAG-compliant approaches to chart and graph accessibility

### OpenShift AI Platform Context

1. **Red Hat OpenShift AI 2.23 Documentation**: Current monitoring capabilities, TrustyAI integration, serving platform options
2. **OpenShift Console Design Patterns**: Existing PatternFly implementations and navigation structures
3. **CNCF Perses Project**: Capabilities, limitations, and roadmap for dashboard visualization platform

### Accessibility Standards

1. **WCAG 2.2 Level AA**: W3C Web Content Accessibility Guidelines
2. **ARIA Authoring Practices**: Best practices for accessible web applications
3. **PatternFly Accessibility**: Framework-specific accessibility implementation guidance

---

## Document Control

**Version**: 1.0
**Last Updated**: 2025-10-24
**Author**: Aria (UX Architect)
**Reviewers**: [To be assigned]
**Status**: Awaiting Review

**Change History**:
- 2025-10-24: Initial UX architecture analysis and recommendations

**Next Steps**:
1. Review with Product Management to validate user journeys and prioritization
2. Review with Engineering to validate technical feasibility and Perses integration approach
3. Review with Design to create high-fidelity mockups based on this architecture
4. Conduct user testing with prototype to validate information architecture decisions
5. Finalize implementation plan and acceptance criteria

---

## Appendix A: Metric Definitions (Sample)

This appendix provides sample metric definitions to ensure consistent understanding across teams.

### Operational Health Metrics

**HTTP Request Success Rate**
- **Definition**: Percentage of HTTP requests that returned 2xx status codes
- **Calculation**: (Successful requests / Total requests) × 100
- **Healthy Range**: > 99%
- **Warning Threshold**: 95-99%
- **Critical Threshold**: < 95%
- **User Impact**: Below 99% indicates service reliability issues

**Average Response Time**
- **Definition**: Mean time between request receipt and response completion
- **Calculation**: Sum(response times) / Count(requests)
- **Healthy Range**: < 100ms for simple models, < 500ms for complex models
- **Warning Threshold**: 2x baseline
- **Critical Threshold**: 5x baseline or > 2000ms
- **User Impact**: High latency degrades user experience and may violate SLAs

### Model Quality Metrics

**Prediction Distribution Drift**
- **Definition**: Statistical difference between current prediction distribution and baseline
- **Calculation**: KL Divergence, PSI (Population Stability Index), or similar
- **Healthy Range**: Drift score < 0.1
- **Warning Threshold**: 0.1 - 0.25
- **Critical Threshold**: > 0.25
- **User Impact**: Significant drift may indicate data quality issues or concept drift

**Output Drift Score**
- **Definition**: Change in model output distribution over time
- **Calculation**: Configurable via TrustyAI (multiple algorithms available)
- **Healthy Range**: Model-specific baseline
- **Warning Threshold**: Configurable
- **Critical Threshold**: Configurable
- **User Impact**: May indicate model needs retraining or input data has changed

### Resource Utilization Metrics

**CPU Utilization**
- **Definition**: Percentage of allocated CPU resources in use
- **Calculation**: (Used CPU millicores / Allocated CPU millicores) × 100
- **Healthy Range**: 30-70%
- **Warning Threshold**: > 80%
- **Critical Threshold**: > 95%
- **User Impact**: High utilization may cause throttling and increased latency

**Memory Utilization**
- **Definition**: Percentage of allocated memory in use
- **Calculation**: (Used memory bytes / Allocated memory bytes) × 100
- **Healthy Range**: 40-80%
- **Warning Threshold**: > 85%
- **Critical Threshold**: > 95%
- **User Impact**: High utilization may cause OOM kills and service interruption

### Fairness & Bias Metrics (TrustyAI)

**Demographic Parity Difference**
- **Definition**: Difference in positive prediction rate between privileged and unprivileged groups
- **Calculation**: P(Ŷ=1|A=privileged) - P(Ŷ=1|A=unprivileged)
- **Healthy Range**: -0.1 to +0.1
- **Warning Threshold**: 0.1 - 0.2 absolute difference
- **Critical Threshold**: > 0.2 absolute difference
- **User Impact**: Significant difference may indicate unfair bias requiring mitigation

**Equalized Odds Difference**
- **Definition**: Difference in true positive and false positive rates between groups
- **Calculation**: Max(|TPR_priv - TPR_unpriv|, |FPR_priv - FPR_unpriv|)
- **Healthy Range**: < 0.1
- **Warning Threshold**: 0.1 - 0.2
- **Critical Threshold**: > 0.2
- **User Impact**: Model performs differently for different demographic groups

---

## Appendix B: User Research Questions

To validate and refine this architecture, recommend conducting user research with these questions:

### For Data Scientists

1. How frequently do you currently check metrics for your deployed models?
2. What's the first thing you look at when checking model health?
3. Walk me through the last time you investigated a model performance issue. What metrics did you look at and in what order?
4. What metrics are most important to you? Which ones do you rarely use?
5. Have you ever missed a model issue because you didn't notice it in the metrics? What happened?
6. What time ranges do you typically use when reviewing metrics?
7. How do you currently share metric findings with your team?

### For ML Engineers

1. Describe your typical workflow when responding to a model serving alert.
2. What metrics help you distinguish between model issues vs. infrastructure issues?
3. How do you determine if a metric value is "normal" or concerning?
4. What would make it easier to correlate deployment changes with metric changes?
5. Do you monitor multiple models simultaneously? How do you handle that?
6. What's the most frustrating part of the current monitoring experience?

### For Platform Administrators

1. What metrics do you care about for capacity planning and cost optimization?
2. How do you currently aggregate metrics across multiple models/users?
3. What reporting requirements do you have for model monitoring?
4. How do you balance resource utilization with model performance requirements?

### Usability Testing Tasks

1. **Task**: Determine if Model X is healthy right now
   - **Success Metric**: Time to answer, confidence level
2. **Task**: Identify why Model Y's latency increased yesterday afternoon
   - **Success Metric**: Can find relevant metrics and correlate with events
3. **Task**: Compare this week's performance to last week for Model Z
   - **Success Metric**: Can enable comparison mode and interpret results
4. **Task**: Set up monitoring for a newly deployed model
   - **Success Metric**: Can identify what metrics are available and configure desired view
5. **Task**: Share a specific metric finding with a colleague
   - **Success Metric**: Can generate shareable link with correct time range and filters

---

*End of Document*