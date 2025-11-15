# Phase 3 Task Group 6: Chart Components and Perses Integration
## Implementation Summary

**Date**: 2025-11-15
**Implemented By**: Steve (UX Designer)
**Tasks Completed**: T059-T064

---

## Overview

Successfully implemented comprehensive Perses plugin integration for the OpenShift AI Observability Dashboard, including datasource configuration, chart plugins, workload discovery UI, and template instantiation workflow.

---

## 1. Files Created with Line Counts

### Datasource Plugin (T059)
**File**: `/workspace/sessions/agentic-session-1763159109/workspace/perses/openshift-ai-dashboard/frontend/src/plugins/datasource/PrometheusDataSource.ts`
**Lines**: 306
**Purpose**: Prometheus datasource plugin with backend API proxy configuration

### Chart Plugins (T060)

1. **TimeSeriesChart.ts**
   **Lines**: 167
   **Purpose**: Time series line/area chart configurations with preset functions

2. **GaugeChart.ts**
   **Lines**: 163
   **Purpose**: Gauge visualizations with threshold configurations

3. **StatPanel.ts**
   **Lines**: 264
   **Purpose**: Single value stat displays with trend indicators

4. **TableView.ts**
   **Lines**: 357
   **Purpose**: Table visualizations with sorting, filtering, pagination

5. **BarChart.ts**
   **Lines**: 353
   **Purpose**: Bar/column chart configurations for categorical data

### Plugin Registry (T061)
**File**: `/workspace/sessions/agentic-session-1763159109/workspace/perses/openshift-ai-dashboard/frontend/src/plugins/index.ts`
**Lines**: 164
**Purpose**: Centralized plugin exports and registration utilities

### Supporting Types and APIs (T063)

1. **workload.ts**
   **Lines**: 30
   **Purpose**: TypeScript types for workload discovery

2. **workloads.ts**
   **Lines**: 58
   **Purpose**: API client for workload operations

3. **WorkloadSelector.tsx**
   **Lines**: 437
   **Purpose**: Workload discovery UI component

### Updated Components (T062, T064)

1. **PersesDashboard.tsx** (Updated)
   **Changes**: Added plugin initialization and datasource configuration integration

2. **DashboardList.tsx** (Updated)
   **Changes**: Added Quick Start tab with workload template instantiation

**Total New Lines**: ~2,299
**Total Files Created**: 9
**Total Files Updated**: 2

---

## 2. Plugin Configuration Details

### Datasource Plugin Architecture

```typescript
// Prometheus DataSource with Backend Proxy
{
  kind: 'PrometheusDatasource',
  spec: {
    proxy: {
      kind: 'HTTPProxy',
      spec: {
        url: '/api/v1/metrics',
        allowedEndpoints: [
          '/query', '/query_range', '/query_instant',
          '/series', '/labels', '/label/.+/values'
        ],
        headers: {
          'Authorization': 'Bearer {token}',
          'Content-Type': 'application/json'
        }
      }
    },
    scrapeInterval: '15s'
  }
}
```

**Key Features**:
- All queries proxied through backend API (no direct Prometheus access)
- Automatic namespace filtering at backend (security requirement)
- Bearer token authentication from session storage
- Query validation and transformation utilities
- Pre-defined OpenShift AI query patterns

### Chart Plugin Configurations

#### 1. Time Series Chart
**Variants**:
- Request rate (ops/sec)
- Latency (milliseconds with thresholds)
- Error rate (percentage with warnings)
- Resource utilization (CPU/Memory/GPU)
- Training metrics (loss/accuracy)

**Features**:
- Multiple series support
- Legend positioning (bottom/right/top)
- Area opacity control
- Threshold overlays
- OpenShift AI color palette

#### 2. Gauge Chart
**Variants**:
- CPU/Memory/GPU utilization
- Success rate
- Storage capacity
- Model accuracy score

**Features**:
- Min/max range configuration
- Multi-level thresholds with colors
- Value formatting (%, decimals)
- Last/mean/max calculation modes

#### 3. Stat Panel
**Variants**:
- Request count
- Requests per second
- Error rate
- Latency
- Active models
- Memory usage
- Training epochs
- Accuracy score
- Uptime percentage

**Features**:
- Sparkline trends
- Value abbreviation (K/M/B)
- Color modes (value/background/none)
- Calculation modes (last/first/mean/sum/min/max)

#### 4. Table View
**Preset Tables**:
- Model serving instances (requests, latency, errors)
- Training jobs (status, epochs, GPU usage)
- Notebook servers (CPU, memory, disk)
- Resource quotas (used/limit/utilization)
- Pod metrics (CPU, memory, restarts)

**Features**:
- Column sorting
- Row filtering
- Pagination (5/10/25/50/100 rows)
- Cell styling with thresholds
- Format units (bytes, cores, percentage)

#### 5. Bar Chart
**Variants**:
- Model request volume comparison
- Training duration comparison
- Resource utilization by workload type
- Errors by status code
- Top N resource consumers
- GPU allocation across jobs
- Latency distribution
- Dataset sizes

**Features**:
- Vertical/horizontal orientation
- Stacking modes (none/normal/percent)
- Data labels
- Threshold lines
- Auto-sorting by value or name

### OpenShift AI Branding

**Color Palette**:
```typescript
{
  primary: ['#06c', '#004080', '#0088ce', '#00659c', '#004d75'],
  secondary: ['#3e8635', '#2d7623', '#1e6823', '#0f5323', '#006400'],
  warning: ['#f0ab00', '#ec7a08', '#c58c00', '#f4c145', '#f9d67a'],
  danger: ['#c9190b', '#a30000', '#7d1007', '#470000', '#3c0000'],
  neutral: ['#6a6e73', '#4f5255', '#72767b', '#8a8d90', '#d2d2d2']
}
```

Applied consistently across all chart types for visual cohesion.

---

## 3. Integration with Backend API

### Query Flow Architecture

```
Frontend (Perses Plugin)
    ↓ PromQL Query
Backend Proxy (/api/v1/metrics/query)
    ↓ Inject namespace filter
Prometheus/Thanos
    ↓ Metrics data
Backend (namespace validation)
    ↓ Filtered results
Frontend (Chart rendering)
```

### Security Model

1. **No Direct Prometheus Access**: All queries route through backend
2. **Automatic Namespace Filtering**: Backend injects `namespace="project-name"` to all queries
3. **Authentication**: Bearer tokens on all requests
4. **Query Validation**: Frontend validates PromQL syntax before sending

### API Endpoints Used

**Metrics**:
- `POST /api/v1/metrics/query` - Range queries
- `POST /api/v1/metrics/query_instant` - Instant queries
- `POST /api/v1/metrics/query_range` - Explicit range queries

**Workloads**:
- `GET /api/v1/projects/{project}/workloads` - List discovered workloads

**Templates**:
- `POST /api/v1/projects/{project}/dashboards/from-template` - Instantiate template

### Query Optimization

**Automatic Step Calculation**:
- Last hour: 15s steps (~240 data points)
- Last 6 hours: 1m steps
- Last 24 hours: 5m steps
- Last 7 days: 30m steps
- Last 30 days: 2h steps

Balances data resolution with query performance.

---

## 4. Workload Discovery UI Design

### Visual Hierarchy

```
┌─────────────────────────────────────────────────┐
│ Quick Start Tab                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Info: Create dashboards from workloads     │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ Search: [__________]  Status: [All ▼]          │
│                                                 │
│ ┌─ Model Serving (3) ─────────────────────────┐│
│ │ ┌───────┐  ┌───────┐  ┌───────┐            ││
│ │ │Model A│  │Model B│  │Model C│            ││
│ │ │Running│  │Running│  │Pending│            ││
│ │ │[Create]│  │[Create]│  │[Create]│          ││
│ │ └───────┘  └───────┘  └───────┘            ││
│ └─────────────────────────────────────────────┘│
│                                                 │
│ ┌─ Training Jobs (2) ─────────────────────────┐│
│ │ ┌───────┐  ┌───────┐                        ││
│ │ │Job 1  │  │Job 2  │                        ││
│ │ └───────┘  └───────┘                        ││
│ └─────────────────────────────────────────────┘│
└─────────────────────────────────────────────────┘
```

### Workload Cards

**Information Display**:
- Workload name
- Type icon with color coding
- Status chip (Running/Pending/Failed/Succeeded)
- Resource allocations (CPU, Memory, GPU)

**Interaction**:
- Hover effect (lift + shadow)
- "Create Dashboard" button
- One-click template instantiation

### Grouping Strategy

**By Workload Type**:
1. **Model Serving** (Blue) - ModelTraining icon
2. **Training Jobs** (Green) - Psychology icon
3. **Notebooks** (Yellow) - Code icon
4. **Data Pipelines** (Red) - DataObject icon

**Accordion Behavior**:
- All groups expanded by default
- Toggle to collapse/expand
- Count badge shows workload count
- Empty groups hidden

### Status Indicators

| Status | Color | Icon | Description |
|--------|-------|------|-------------|
| Running | Green | CheckCircle | Workload active |
| Pending | Yellow | Schedule | Starting up |
| Failed | Red | Error | Error state |
| Succeeded | Blue | CheckCircleOutline | Completed |
| Unknown | Gray | HelpOutline | Status unavailable |

### Empty State

**Display When**:
- No workloads discovered in project
- Shows large icon + message
- Suggests creating workloads first

**Message**:
> "No workloads discovered
> Create model serving deployments, training jobs, or notebooks to see them here"

---

## 5. User Flow for Template Instantiation

### Step-by-Step Flow

```
1. User navigates to Dashboards page
   ↓
2. Clicks "Quick Start" tab
   ↓
3. Views discovered workloads grouped by type
   ↓
4. Filters/searches for specific workload (optional)
   ↓
5. Clicks "Create Dashboard" on workload card
   ↓
6. Loading indicator shows "Creating dashboard from template..."
   ↓
7. Backend instantiates template with workload variables
   ↓
8. Success message: "Dashboard created successfully!"
   ↓
9. Auto-redirect to new dashboard (1 second delay)
   ↓
10. Dashboard loads with pre-configured panels
```

### Template Variable Mapping

**Workload → Template Variables**:
```typescript
{
  template_id: 'model-serving-v1', // Based on workload type
  dashboard_name: `${workload.name} Dashboard`,
  variables: {
    workload_name: workload.name,
    workload_type: workload.type,
    namespace: workload.namespace
  }
}
```

**Template Suggestions**:
- Model Serving → `model-serving-v1`
- Training Job → `training-job-v1`
- Notebook → `notebook-v1`
- Data Pipeline → `data-pipeline-v1`

### Error Handling

**Scenarios**:
1. **API Error**: Display error alert with message
2. **Network Timeout**: Show retry option
3. **Invalid Template**: Graceful fallback message
4. **Insufficient Permissions**: Permission denied message

**Recovery**:
- Error alert dismissible
- User remains on Quick Start page
- Can retry with different workload
- Error details logged to console

### Loading States

**Visual Feedback**:
1. **Creating**: CircularProgress + "Creating dashboard..."
2. **Success**: Green checkmark + "Dashboard created!"
3. **Redirecting**: "Redirecting..." message
4. **Error**: Red X + error details

### Navigation Patterns

**Entry Points**:
- DashboardList page → "Quick Start" tab
- Direct link from workload details page (future)
- Onboarding tutorial (future)

**Exit Points**:
- Auto-redirect to new dashboard on success
- Manual navigation back to "My Dashboards" tab
- Breadcrumb navigation to project overview

---

## Testing Recommendations

### Unit Tests

1. **Plugin Configuration**:
   - Test datasource creation with/without auth token
   - Verify query transformation logic
   - Validate PromQL query validation

2. **Chart Presets**:
   - Test all preset configurations return valid specs
   - Verify color palette consistency
   - Check threshold configurations

3. **Workload Selector**:
   - Test filtering by name
   - Test filtering by status
   - Test grouping logic
   - Verify empty states

### Integration Tests

1. **Plugin → Backend**:
   - Test query proxying through backend
   - Verify authentication headers
   - Test error handling for failed queries

2. **Template Instantiation**:
   - Test workload → template mapping
   - Verify variable substitution
   - Test dashboard creation flow

### E2E Tests

1. **Quick Start Flow**:
   - Navigate to Quick Start tab
   - Select workload
   - Create dashboard
   - Verify dashboard appears in list

2. **Dashboard Rendering**:
   - Load dashboard with plugins
   - Verify charts render correctly
   - Test time range selector
   - Verify real-time updates (SSE)

---

## Performance Considerations

### Plugin Loading
- Plugins loaded on demand (not upfront)
- Datasource configuration memoized
- Chart configurations cached

### Query Optimization
- Automatic step calculation reduces data points
- 30-second refresh interval for workload discovery
- Debounced search filtering

### Rendering
- React virtualization for large tables
- Lazy loading of chart components
- Memoized dashboard configurations

---

## Accessibility Features

1. **Keyboard Navigation**: All interactive elements keyboard accessible
2. **ARIA Labels**: Proper labeling on icons and buttons
3. **Color Contrast**: All text meets WCAG AA standards
4. **Screen Reader**: Meaningful status announcements
5. **Focus Management**: Visible focus indicators

---

## Future Enhancements

### Phase 4 Candidates

1. **Custom Chart Builder**: Visual editor for chart configuration
2. **Template Editor**: UI for creating custom templates
3. **Workload Health Scores**: Aggregate health metrics
4. **Alert Integration**: Link charts to alert rules
5. **Export/Import**: Dashboard portability across projects
6. **Collaborative Editing**: Multi-user dashboard editing
7. **Version History**: Dashboard change tracking
8. **Annotation Support**: Add notes to time-series data

### Advanced Features

1. **AI-Powered Insights**: Anomaly detection in metrics
2. **Cost Analysis**: Resource cost visualization
3. **Capacity Planning**: Predictive resource recommendations
4. **Custom Datasources**: Support for additional metric sources
5. **Mobile Optimization**: Responsive dashboard layouts

---

## Dependencies

### NPM Packages
- `@perses-dev/components@^0.53.0`
- `@perses-dev/dashboards@^0.53.0`
- `@perses-dev/plugin-system@^0.53.0`
- `@mui/material@^6.1.9`
- `@tanstack/react-query@^4.36.1`

### Internal Dependencies
- Dashboard API client
- Workload API client
- Type definitions
- Authentication utilities

---

## Security Considerations

1. **No Direct Prometheus Access**: All queries routed through authenticated backend
2. **Namespace Isolation**: Backend enforces project-level data isolation
3. **Token Management**: Tokens stored in session storage (not localStorage)
4. **CSRF Protection**: Backend validates request origins
5. **Input Validation**: PromQL queries validated before sending
6. **XSS Prevention**: All user inputs sanitized

---

## Documentation

### Developer Guide

**Creating Custom Chart Preset**:
```typescript
import { createTimeSeriesChartPlugin } from '@/plugins';

const myCustomChart = createTimeSeriesChartPlugin({
  visual: {
    axis: {
      yAxis: {
        label: 'Custom Metric',
        format: { unit: 'custom', decimals: 2 }
      }
    }
  }
});
```

**Adding New Workload Type**:
```typescript
// 1. Update workload.ts
export type WorkloadType = 'model_serving' | 'training_job' | 'notebook' | 'data_pipeline' | 'my_new_type';

// 2. Add to WORKLOAD_TYPE_CONFIG in WorkloadSelector.tsx
my_new_type: {
  label: 'My New Type',
  icon: <CustomIcon />,
  color: '#custom',
  suggestedTemplate: 'my-template-v1'
}
```

### User Guide

**Quick Start Instructions**:
1. Navigate to project dashboards page
2. Click "Quick Start" tab
3. Browse discovered workloads
4. Click "Create Dashboard" on desired workload
5. Wait for creation (usually < 5 seconds)
6. Automatically redirected to new dashboard

---

## Conclusion

Phase 3 Task Group 6 successfully delivers:

- Complete Perses plugin architecture
- 5 chart plugin types with 30+ preset configurations
- Secure Prometheus integration via backend proxy
- Intuitive workload discovery UI
- Streamlined template instantiation workflow
- OpenShift AI visual branding
- Production-ready error handling
- Comprehensive type safety

**Next Steps**: Phase 4 (Dashboard Editor) will build on this foundation to enable visual dashboard creation and editing.
