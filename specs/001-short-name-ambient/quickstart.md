# Quickstart Guide: Enhanced Model Metrics Display

**Feature**: Enhanced Model Metrics Display
**Branch**: 001-short-name-ambient
**Target**: New developers joining the project

## Purpose

This guide gets you from zero to running the metrics visualization feature in 30 minutes. It covers environment setup, running the development server, and executing the user acceptance scenarios from the spec.

---

## Prerequisites

Before you begin, ensure you have:

- ✅ Node.js 18.x or higher (`node --version`)
- ✅ npm 8.x or higher (`npm --version`)
- ✅ Access to OpenShift AI Console repository (odh-dashboard)
- ✅ OpenShift cluster with deployed models (or test environment)
- ✅ kubectl configured to access the cluster
- ✅ Git configured with your credentials

---

## Environment Setup

### 1. Clone the Repository

```bash
# Clone OpenShift AI Dashboard repository
git clone https://github.com/opendatahub-io/odh-dashboard.git
cd odh-dashboard

# Checkout the metrics feature branch
git checkout 001-short-name-ambient
```

### 2. Install Dependencies

```bash
# Install all dependencies (frontend and backend)
npm install

# Install frontend-specific dependencies
cd frontend
npm install
cd ..
```

### 3. Configure Environment Variables

Create `.env.local` file in the `frontend/` directory:

```bash
# frontend/.env.local

# Prometheus endpoint
PROMETHEUS_API_URL=http://localhost:9090/api/v1

# TrustyAI endpoint (optional)
TRUSTYAI_API_URL=http://localhost:8080/api/trustyai/v1

# Development mode
NODE_ENV=development

# Feature flags
ENABLE_BEHAVIOR_METRICS=true

# Mock data for offline development
USE_MOCK_PROMETHEUS=false
USE_MOCK_TRUSTYAI=true
```

### 4. Set Up Port Forwarding (if using remote cluster)

```bash
# Forward Prometheus port
kubectl port-forward -n openshift-monitoring \
  svc/prometheus-k8s 9090:9090 &

# Forward TrustyAI port (if available)
kubectl port-forward -n trustyai \
  svc/trustyai-service 8080:8080 &

# Keep terminals open or run in background
```

---

## Running the Application

### Development Server

```bash
# Start frontend development server (port 3000)
cd frontend
npm start
```

Visit http://localhost:3000 to see the console.

### Development Server with Backend

If you need full RBAC and authentication:

```bash
# Terminal 1: Start backend (Go server, port 8080)
cd backend
go run main.go

# Terminal 2: Start frontend (port 3000, proxies to backend)
cd frontend
REACT_APP_BACKEND_URL=http://localhost:8080 npm start
```

### Using Mock Data (Offline Development)

If Prometheus/TrustyAI unavailable:

```bash
# Set environment variable
export USE_MOCK_PROMETHEUS=true
export USE_MOCK_TRUSTYAI=true

# Start frontend
cd frontend
npm start
```

Mock data is located in `frontend/src/__mocks__/prometheus.ts` and `frontend/src/__mocks__/trustyai.ts`.

---

## User Acceptance Scenarios

These scenarios correspond to the acceptance criteria in `spec.md`.

### Scenario 1: View Overview of Model Health

**Given** I am viewing the Deployed Models Details page
**When** I navigate to the metrics section
**Then** I see an overview of model health including both performance metrics (latency, throughput, errors) and behavior metrics (request patterns, prediction distributions)

#### Steps:

1. Navigate to **Model Serving** → **Deployed Models**
2. Click on a deployed model (e.g., "fraud-detection-v2")
3. Select the **Metrics** tab (default view should be Overview)
4. Verify you see:
   - Health summary card with status indicator
   - Key performance metrics: Request rate, error rate, latency P99
   - Key behavior metrics (if TrustyAI enabled): Drift score, prediction distribution
   - All metrics showing data for the last 1 hour (default time range)

**Expected Result**: Overview tab loads in <2 seconds (p90), showing all available metrics.

**Troubleshooting**:
- If no data appears, check Prometheus connection: `curl http://localhost:9090/api/v1/query?query=up`
- If TrustyAI metrics missing, check `USE_MOCK_TRUSTYAI=true` or verify TrustyAI service: `kubectl get svc -n trustyai`

---

### Scenario 2: Select Time Range

**Given** I need to investigate a model performance issue
**When** I select a time range (1 hour, 24 hours, 7 days, or 30 days)
**Then** All metric visualizations update to show data for that time period

#### Steps:

1. From the Metrics tab, locate the **Time Range Selector** in the toolbar
2. Click the dropdown (default: "Last 1 hour")
3. Select **"Last 24 hours"**
4. Observe all metric panels refresh with new data
5. Verify:
   - Loading indicators appear briefly
   - All charts update to show 24-hour data
   - Time axis labels reflect the new range
   - URL updates to `?timeRange=24h`

**Expected Result**: All panels update within 500ms (p95), URL state persists for sharing.

**Test All Ranges**:
```bash
# Test each time range option
- Last 1 hour  → URL: ?timeRange=1h
- Last 24 hours → URL: ?timeRange=24h
- Last 7 days   → URL: ?timeRange=7d
- Last 30 days  → URL: ?timeRange=30d
```

---

### Scenario 3: Auto-Refresh Metrics

**Given** I am monitoring model health
**When** The page is open
**Then** Metrics automatically refresh every 1 minute to show current data

#### Steps:

1. Open the Metrics page
2. Note the current timestamp in the "Last Updated" indicator
3. Wait 60 seconds without interacting with the page
4. Observe:
   - "Refreshing..." indicator appears briefly
   - Panels update with new data
   - "Last Updated" timestamp changes to current time

**Expected Result**: Metrics refresh automatically every 60 seconds.

**Test Pause/Resume**:
1. Click the **Pause** button in the toolbar
2. Wait 60 seconds → metrics should NOT refresh
3. Click the **Resume** button
4. Wait 60 seconds → metrics refresh as expected

**Test Page Visibility**:
1. Switch to another browser tab
2. Wait 60 seconds
3. Return to the Metrics tab → refresh happens immediately upon focus

---

### Scenario 4: View Model Behavior Trends

**Given** I want to understand model behavior trends
**When** I view the metrics dashboard
**Then** I can see prediction patterns, request volume trends, and data quality indicators alongside performance metrics

#### Steps:

1. Navigate to Metrics tab → **Model Quality** section (if TrustyAI enabled)
2. Verify you see:
   - **Prediction Distribution** chart (pie or bar chart showing class distribution)
   - **Request Volume Trends** line chart over time
   - **Data Drift Score** gauge or trend line
   - **Confidence Score Distribution** histogram

**Expected Result**: Behavior metrics display alongside performance metrics in organized sections.

**If TrustyAI Unavailable**:
- Empty state appears: "Enable TrustyAI to view behavior metrics"
- Link to documentation or setup instructions
- Performance metrics still fully functional

---

### Scenario 5: Export Metrics Data

**Given** I need to share metrics with my team
**When** I view specific metrics
**Then** I can export the visible data to CSV format for reporting

#### Steps:

1. From the Metrics tab, locate any metric panel (e.g., Request Rate)
2. Click the **Export** button (or three-dot menu → Export)
3. Select **"Export as CSV"**
4. Browser downloads a CSV file: `model-fraud-detection-v2-request-rate-2025-10-24.csv`
5. Open the CSV file and verify:
   - Column headers: Timestamp, Value, Model ID, Namespace
   - Data rows correspond to visible time range
   - Timestamps in ISO 8601 format

**Expected Result**: CSV export completes in <3 seconds for 1440 data points.

**Test Export Formats** (if implemented):
- CSV (default)
- JSON (advanced)
- PNG (chart image)

---

### Scenario 6: Accessibility Compliance

**Given** I am a user with accessibility needs
**When** I navigate the metrics page
**Then** I can access all functionality via keyboard and screen reader with WCAG 2.2 Level AA compliance

#### Steps:

**Keyboard Navigation**:
1. Tab through the page using **Tab** key
2. Verify:
   - All interactive elements receive visible focus indicator (3px outline)
   - Focus order is logical (top to bottom, left to right)
   - Toolbar controls reachable by keyboard
   - Time range selector operable with arrow keys

**Screen Reader** (use NVDA, JAWS, or VoiceOver):
1. Navigate to Metrics tab
2. Verify:
   - Page title announced: "Model Metrics: fraud-detection-v2"
   - Each metric panel has descriptive label: "Request rate: 150 requests per second"
   - Chart data accessible via data table alternative view
   - Status indicators announced: "Operational health: Healthy"

**Visual Accessibility**:
1. Check color contrast using browser DevTools (Lighthouse audit)
2. Verify:
   - Text contrast ratio ≥ 4.5:1
   - Graphical object contrast ≥ 3:1
   - Status indicators use icons + text (not color alone)

**Text Resizing**:
1. Zoom browser to 200% (Ctrl/Cmd + +)
2. Verify:
   - All text readable
   - No content overflow or cutoff
   - Layout adapts responsively

**Expected Result**: Zero critical or serious accessibility violations in automated tests.

---

## Edge Cases Testing

### Edge Case 1: Historical Metrics Data Unavailable

**Scenario**: What happens when historical metrics data is unavailable?

**Steps**:
1. Select **"Last 30 days"** time range
2. If Prometheus retention < 30 days, some data will be missing
3. Verify:
   - Message displays: "Data available for last 15 days only (retention limit)"
   - Charts show available data (gaps allowed)
   - No application errors

### Edge Case 2: Behavior Metrics Not Instrumented

**Scenario**: How does the system handle models without behavior metrics instrumentation?

**Steps**:
1. Set `USE_MOCK_TRUSTYAI=false` (disable TrustyAI)
2. Navigate to Metrics tab
3. Verify:
   - **Operational Health** and **Resource Utilization** sections load normally
   - **Model Quality** section shows empty state: "TrustyAI not available"
   - Instructions link to TrustyAI setup documentation

### Edge Case 3: Prometheus Temporarily Unavailable

**Scenario**: What happens when Prometheus is temporarily unavailable?

**Steps**:
1. Stop Prometheus port-forward: `pkill -f "port-forward.*prometheus"`
2. Wait for next auto-refresh (60 seconds)
3. Verify:
   - Error state displays: "Unable to fetch metrics. Retrying..."
   - Retry button available for manual refresh
   - Auto-refresh pauses until service recovers
4. Restart port-forward: `kubectl port-forward -n openshift-monitoring svc/prometheus-k8s 9090:9090`
5. Verify:
   - Metrics automatically resume after 3 retry attempts
   - Success notification: "Metrics updated"

### Edge Case 4: High-Volume Model Performance

**Scenario**: How does the page perform with high-volume models?

**Setup**:
```bash
# Simulate high cardinality with mock data
export MOCK_CARDINALITY=10000  # 10K series
npm start
```

**Steps**:
1. Navigate to Metrics tab for high-volume model
2. Verify:
   - Initial load completes within 5 seconds (degraded but acceptable)
   - Loading indicators show progress
   - Panels load progressively (staggered 2s intervals)
   - No browser freeze or unresponsive UI

**Expected Result**: Query caching and progressive loading maintain responsiveness.

### Edge Case 5: Concurrent Users

**Scenario**: What happens when multiple users view the same model metrics simultaneously?

**Setup**: Open metrics page in 3 browser tabs (simulates 3 users)

**Steps**:
1. Open same model metrics in 3 tabs
2. Verify:
   - Each tab receives independent data views
   - Auto-refresh in each tab works independently
   - No race conditions or data corruption
   - Prometheus backend handles concurrent queries

---

## Running Tests

### Unit Tests

```bash
cd frontend
npm run test

# Run specific test suite
npm run test -- MetricsPage.test.tsx

# Watch mode for development
npm run test -- --watch
```

### Integration Tests

```bash
# Requires backend and Prometheus running
npm run test:integration

# Specific test file
npm run test:integration -- --grep "Metrics API"
```

### Contract Tests

```bash
# Validate API responses match OpenAPI specs
npm run test:contracts
```

### E2E Tests (Cypress)

```bash
cd frontend

# Headless mode
npm run test:e2e

# Interactive mode
npm run cypress:open

# Specific spec file
npm run test:e2e -- --spec cypress/e2e/metrics.cy.ts
```

### Accessibility Tests

```bash
# Automated WCAG 2.2 Level AA audit
npm run test:a11y

# Manual testing checklist
# 1. Tab through page (keyboard navigation)
# 2. Test with screen reader (NVDA/JAWS/VoiceOver)
# 3. Zoom to 200% (text resizing)
# 4. Run Lighthouse accessibility audit in Chrome DevTools
```

---

## Common Issues & Troubleshooting

### Issue: "Prometheus connection refused"

**Symptom**: Metrics page shows "Unable to fetch metrics"

**Solution**:
```bash
# Verify Prometheus is running
kubectl get pods -n openshift-monitoring | grep prometheus

# Check port-forward
lsof -i :9090

# Restart port-forward
kubectl port-forward -n openshift-monitoring svc/prometheus-k8s 9090:9090
```

### Issue: "CORS error when querying Prometheus"

**Symptom**: Browser console shows CORS error

**Solution**: Ensure backend proxy is running:
```bash
# Backend handles CORS headers
cd backend
go run main.go
```

Or configure `.env.local`:
```bash
REACT_APP_BACKEND_URL=http://localhost:8080
```

### Issue: "TrustyAI metrics always null"

**Symptom**: Behavior metrics show empty state

**Solution**:
```bash
# Enable mock data for development
export USE_MOCK_TRUSTYAI=true

# Or verify TrustyAI service
kubectl get svc -n trustyai trustyai-service
```

### Issue: "Page loads slowly (>5 seconds)"

**Symptom**: Initial page load exceeds performance target

**Solution**:
1. Check Prometheus query cardinality:
   ```promql
   # Count series
   count(model_requests_total{namespace="my-project"})
   ```
2. Enable query caching (React Query staleTime):
   ```typescript
   staleTime: 90000, // 90 seconds
   ```
3. Use recording rules for expensive queries

### Issue: "Auto-refresh not working"

**Symptom**: Metrics don't update after 60 seconds

**Solution**:
1. Check if page is visible (auto-refresh pauses when hidden)
2. Verify console for errors (F12 → Console)
3. Check if refresh is paused (toolbar button)
4. Clear browser cache and reload

---

## Development Workflow

### Making Changes

1. **Create feature branch**:
   ```bash
   git checkout -b metrics/add-new-panel
   ```

2. **Make changes** (follow TDD approach):
   ```bash
   # Write test first
   vi frontend/src/__tests__/components/MetricPanel.test.tsx

   # Run test (should fail)
   npm run test -- MetricPanel.test.tsx

   # Implement feature
   vi frontend/src/components/MetricPanel.tsx

   # Run test (should pass)
   npm run test -- MetricPanel.test.tsx
   ```

3. **Commit changes**:
   ```bash
   git add .
   git commit -m "feat(metrics): add new latency breakdown panel"
   ```

4. **Run full test suite before push**:
   ```bash
   npm run test
   npm run test:integration
   npm run test:a11y
   ```

### Code Review Checklist

Before submitting PR:

- [ ] All tests passing (`npm run test`)
- [ ] Accessibility audit passing (`npm run test:a11y`)
- [ ] Code follows TypeScript/React best practices
- [ ] New components have unit tests
- [ ] API changes documented in `contracts/`
- [ ] No console errors or warnings
- [ ] Performance targets met (<2s page load, <500ms panel render)

---

## Next Steps

After completing this quickstart:

1. **Read the full spec**: `specs/001-short-name-ambient/spec.md`
2. **Review data model**: `specs/001-short-name-ambient/data-model.md`
3. **Explore API contracts**: `specs/001-short-name-ambient/contracts/`
4. **Review implementation plan**: `specs/001-short-name-ambient/plan.md`
5. **Join team sync**: Ask EM for invite to sprint planning

---

## Support & Resources

- **Slack Channel**: #openshift-ai-console
- **Team Wiki**: [link to internal wiki]
- **API Documentation**: `contracts/README.md`
- **Architecture Decisions**: `specs/ux-architecture-metrics-display.md`

---

*Last updated: 2025-10-24*
*Estimated completion time: 30 minutes*
