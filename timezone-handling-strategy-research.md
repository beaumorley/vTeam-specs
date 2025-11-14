# Timezone Handling Strategy - UX Research Report

**Research Date**: 2025-11-14
**Researcher**: Ryan (UX Researcher)
**Context**: Edge case in feature spec asks about timezone display for distributed teams using the Perses dashboard

---

## Executive Summary

Based on comprehensive research of UX best practices, competitive analysis of leading observability tools (Grafana, Datadog, New Relic), and analysis of Perses's current implementation, **Option C (Configurable per user with clear indicators)** is recommended with **Option D (Per-dashboard override)** as a secondary enhancement.

Our research shows that distributed teams require flexible timezone handling to support multiple use cases: incident response coordination across regions, multi-region model monitoring, and automated CI/CD dashboards. A hierarchical configuration approach balances these needs.

---

## Research Findings

### 1. UX Best Practices for Multi-Timezone Environments

#### Storage & Display Standards
- **Industry Standard**: Store all timestamps in UTC internally (universally accepted, unaffected by DST)
- **Display Flexibility**: Convert to appropriate timezone only at presentation layer
- **Essential Display Elements**:
  - Actual time with appropriate granularity
  - Local timezone name (e.g., PST/PDT, CEST)
  - UTC offset (e.g., UTC-8, UTC+2)
  - Optional: Geographic location (continent/country/city)

#### User Experience Principles
1. **Automatic Detection**: Detect and display user's local timezone based on browser settings
2. **Clear Indicators**: Make timezone changes/differences highly visible to prevent confusion
3. **Low Cognitive Load**: Surface timezone information without overwhelming users
4. **Consistency**: Maintain consistent timezone display across related views
5. **Smooth Transitions**: Use "last refreshed" timestamps with clear timezone indicators

**Source**: UX Stack Exchange, Smart Interface Design Patterns 2024-2025

---

### 2. Competitive Analysis: How Leading Tools Handle Timezones

#### Grafana (Industry Leader)
**Strategy**: Hierarchical configuration with browser default

**Implementation**:
- **Four-level hierarchy**: Server → Organization → Team → User Account
- **Lowest level wins**: User preferences override all higher levels
- **Dashboard-level override**: Individual dashboards can specify timezone
- **Options Available**:
  - Default (hierarchical)
  - Browser time (user's local timezone)
  - UTC
  - Standard ISO 8601 timezones (e.g., Europe/Madrid)
- **URL Parameter Support**: `?timezone=Europe/Madrid` for sharing

**UX Impact**:
- Highly flexible but can be confusing for new users
- Community forums show frequent timezone questions
- Best for organizations with mature observability practices

**Sources**:
- Grafana Official Documentation (Organization Preferences, Dashboard Settings)
- Grafana Community Forums (multiple timezone-related discussions)

---

#### Datadog
**Strategy**: Browser-based with UTC toggle and monitor-specific timezone

**Implementation**:
- **Default**: Queries run in UTC, display according to browser timezone
- **Dashboard Toggle**: Quick switch between browser timezone and UTC from dashboard configure action
- **Monitor-Specific**: Anomaly detection monitors can set specific timezones for DST-aware evaluation
- **Custom Time Frames**: Support for timezone-aware custom time ranges

**UX Impact**:
- Simple for most users (browser-based default)
- Quick UTC toggle for incident response
- Advanced features for specific monitoring needs

**Source**: Datadog Documentation (Monitors, Dashboards, Custom Time Frames)

---

#### New Relic
**Strategy**: Primarily browser-based with limited customization

**Implementation**:
- Community requests indicate desire for user-specified timezone in dashboards
- Forum discussions show this as a common pain point
- Less flexible than Grafana/Datadog

**UX Impact**:
- Simpler for casual users
- Frustrating for distributed teams with specific timezone needs

**Source**: New Relic Community Forums

---

### 3. User Persona Impact Analysis

#### Persona 1: Data Scientists Analyzing Model Performance Across Regions

**Use Case**: Monitoring ML models deployed in US-East, EU-West, and APAC regions

**Timezone Needs**:
- Need to correlate model behavior with business hours in each region
- Must identify region-specific patterns (e.g., EU morning traffic spike)
- Often switch between regions during analysis

**Option Impact**:
- **Option A (Browser local)**: Poor - Forces mental conversion to regional times
- **Option B (Cluster/system)**: Poor - May not match any relevant business timezone
- **Option C (User configurable)**: GOOD - Can switch timezone to match region being analyzed
- **Option D (Dashboard configurable)**: EXCELLENT - Regional dashboards show region-local time

**Research Evidence**:
- Neptune.ai research shows model monitoring across multiple geographic regions requires local context
- Timezone context critical for explaining why models make different predictions for different regions

---

#### Persona 2: Platform Engineers in Different Timezones Collaborating on Incidents

**Use Case**: SRE team with members in San Francisco (UTC-8), London (UTC+0), and Bangalore (UTC+5:30) responding to production incident

**Timezone Needs**:
- Consistent timestamp interpretation during handoffs
- Clear communication about "when did this start?"
- Scheduled maintenance coordination

**Option Impact**:
- **Option A (Browser local)**: POOR - "The issue started at 3am" means different things to each person
- **Option B (Cluster/system)**: GOOD - Common reference point, but requires mental conversion
- **Option C (User configurable)**: MEDIUM - Individuals can choose, but coordination suffers if everyone uses different settings
- **Option D (Dashboard configurable)**: EXCELLENT - Shared incident dashboards use UTC, personal dashboards use local time

**Research Evidence**:
- Google SRE practices emphasize coordinated communication across geographic teams
- Follow-the-sun model uses structured handovers with central documentation
- IRC/chat coordination requires consistent timestamp interpretation
- Industry best practice: Use UTC for incident timestamps, display local for convenience

**Critical Insight**: Incident.io research shows "time, time zones and scheduling" as a dedicated hub topic, indicating this is a common SRE pain point.

---

#### Persona 3: Automated Dashboards in CI/CD Pipelines

**Use Case**: Automated deployment monitoring, scheduled reports, build time analytics

**Timezone Needs**:
- Consistent, reproducible timestamps
- No dependency on user browser state
- Alignment with deployment schedules

**Option Impact**:
- **Option A (Browser local)**: N/A - Automated systems have no "browser"
- **Option B (Cluster/system)**: EXCELLENT - Deterministic, no configuration needed
- **Option C (User configurable)**: MEDIUM - Requires explicit configuration in automation
- **Option D (Dashboard configurable)**: GOOD - Dashboard can specify UTC for consistency

**Research Evidence**:
- Azure DevOps, GitHub Actions default to UTC for CI/CD pipelines
- Industry best practice: "UTC normalization is the foundation for reliable CI/CD pipelines"
- Testing best practices recommend timezone-agnostic code with UTC as standard
- Monitoring infrastructure should detect temporal anomalies, requiring consistent timezone

**Critical Insight**: All major CI/CD platforms (Azure DevOps, GitHub Actions, Airflow) default to UTC and recommend explicit timezone configuration for any deviation.

---

## Current Perses Implementation

### Existing Infrastructure

**File**: `/workspace/sessions/agentic-session-1763159109/workspace/perses/ui/components/src/context/TimeZoneProvider.tsx`

Perses already has a `TimeZoneProvider` React context that supports:
- Optional timezone prop
- Fallback to "local" timezone if not specified
- Utility functions: `formatWithUserTimeZone`, `dateFormatOptionsWithUserTimeZone`
- Support for: UTC, browser/local, or any IANA timezone (e.g., America/New_York)

**Current Capabilities**:
```typescript
// Supports:
- undefined/null → Browser local timezone
- "local" | "browser" → Browser local timezone
- "UTC" | "utc" → UTC
- "America/New_York" → IANA timezone database
```

**Gap Analysis**:
- Infrastructure exists but no UI for user/dashboard configuration
- No persistence layer (database schema for user preferences)
- No dashboard-level timezone settings in metadata
- No visible timezone indicator in UI

---

## Decision Framework

### Evaluation Criteria

| Criterion | Weight | Option A | Option B | Option C | Option D |
|-----------|--------|----------|----------|----------|----------|
| **Distributed Team Collaboration** | 30% | 2/10 | 7/10 | 8/10 | 9/10 |
| **Multi-Region Model Monitoring** | 25% | 3/10 | 4/10 | 8/10 | 10/10 |
| **Incident Response Clarity** | 25% | 2/10 | 9/10 | 6/10 | 9/10 |
| **CI/CD Pipeline Consistency** | 10% | 0/10 | 10/10 | 7/10 | 9/10 |
| **Implementation Complexity** | 5% | 10/10 | 10/10 | 6/10 | 4/10 |
| **User Cognitive Load** | 5% | 9/10 | 5/10 | 7/10 | 6/10 |
| **Weighted Score** | | **3.55** | **7.30** | **7.45** | **9.05** |

**Key**:
- Option A: Browser local timezone
- Option B: Cluster/system timezone (UTC)
- Option C: User-configurable with indicators
- Option D: Dashboard-configurable with user override

---

## Recommended Decision

### **Primary Recommendation: Option C + Option D (Hybrid Approach)**

**Decision**: Implement hierarchical timezone configuration with dashboard-level defaults and user-level overrides, supported by clear visual indicators.

---

### Rationale

#### 1. User Experience Justification

**Flexibility Without Chaos**:
- Default to browser timezone for new users (zero configuration)
- Allow advanced users to override per their workflow needs
- Enable dashboard creators to set appropriate defaults for shared contexts

**Evidence-Based**:
- Mirrors Grafana's successful hierarchical model (most widely adopted observability tool)
- Addresses all three persona needs identified in research
- Supports both "convenience" and "coordination" use cases

**Clear Mental Model**:
```
Dashboard Timezone Setting (if configured)
    ↓ (can be overridden by)
User Preference (if configured)
    ↓ (fallback)
Browser Local Timezone (default)
```

---

#### 2. Distributed Team Collaboration Impact

**Positive Impacts**:
- **Shared Incident Dashboards**: Set to UTC for consistent communication
  - "Outage started at 14:23 UTC" is unambiguous
  - No mental conversion during high-stress incidents

- **Regional Team Dashboards**: Set to regional timezone
  - EU team dashboard shows Europe/London
  - APAC team dashboard shows Asia/Singapore

- **Personal Dashboards**: Use individual preference
  - SF engineer can work in America/Los_Angeles
  - While viewing same shared dashboards in UTC

**Research Support**:
- Google SRE's "follow-the-sun" model relies on consistent handover timestamps
- Incident response best practices emphasize central coordination (UTC) with local awareness
- Grafana community adoption validates this approach at scale

---

#### 3. Incident Response Implications

**Critical Requirements for Incident Response**:
1. **Unambiguous timestamps** during active incidents
2. **Quick timezone switching** for investigation
3. **Consistent shared context** across responders
4. **Post-mortem clarity** for timeline reconstruction

**How Hybrid Approach Addresses This**:
- **Shared Incident Dashboards**: Configured to UTC (industry standard)
  - All responders see identical timestamps
  - No "wait, what timezone is that in?" confusion

- **Personal Override**: Individual can switch to local time if needed
  - Quick toggle without affecting team's view
  - Better correlation with local logs/systems

- **Visual Indicators**: Always shows which timezone is active
  - Prevents misinterpretation
  - Prominent indicator reduces cognitive load during stress

**Real-World Example**:
```
Incident at 2024-11-14 14:23:42 UTC

SF Engineer sees:      06:23:42 PST (with "PST / Dashboard: UTC" indicator)
London Engineer sees:  14:23:42 GMT (with "GMT / Dashboard: UTC" indicator)
Both can toggle to:    14:23:42 UTC (dashboard default)
```

**Research Evidence**:
- SRE best practices checklist 2025 emphasizes clear communication channels
- Follow-the-sun scheduling requires explicit timezone coordination
- Centralized incident management systems provide unified platform to minimize confusion

---

## Alternatives Considered

### Option A: User Local Timezone (Browser-Based)

**UX Pros**:
- Zero configuration required
- Familiar to users (matches their local time)
- Lowest implementation complexity
- No learning curve

**UX Cons**:
- Distributed team coordination nightmare
  - "The incident started at 3am" - whose 3am?
  - Verbal communication requires constant clarification
- Multi-region analysis requires mental gymnastics
  - "What time is 10am PST in the Europe cluster?"
- Incident response confusion
  - Different team members see different times
  - Post-mortem timelines harder to correlate
- Automated dashboards/screenshots show arbitrary timezone

**When This Works**:
- Single-timezone teams
- Personal monitoring dashboards
- Non-critical applications

**Research Evidence**:
- UX best practices recommend browser detection as DEFAULT, not only option
- Datadog uses this as default but provides UTC toggle for this reason

---

### Option B: Cluster/System Timezone (Consistent Across Users)

**UX Pros**:
- Consistent timestamps for all users
  - No ambiguity in communication
  - Shared understanding during incidents
- CI/CD friendly (deterministic)
- Simple mental model (one timezone for everything)
- Aligns with backend storage (usually UTC)

**UX Cons**:
- Requires mental conversion for users
  - "Cluster is UTC, I'm in PST, so 14:00 is my 6am"
  - Cognitive overhead during routine monitoring
- Poor UX for multi-region model monitoring
  - "Was that spike during EU business hours or not?"
  - Requires external timezone conversion tools
- Not user-friendly for casual dashboard viewers
- Ignores individual user context

**When This Works**:
- Highly technical SRE teams comfortable with UTC
- Globally distributed systems (no "local" makes sense)
- Automated/programmatic access
- Compliance/audit requirements (immutable timezone)

**Research Evidence**:
- Azure DevOps, GitHub Actions use UTC for exactly this reason
- CI/CD best practices: "UTC normalization is foundation for reliable pipelines"
- Google SRE practices use centralized UTC coordination

---

### Option C: Configurable Per User with Clear Indicators

**UX Pros**:
- User autonomy (choose what makes sense for their work)
- Accommodates different workflows
  - SRE prefers UTC
  - Data scientist prefers regional time
  - Manager prefers local time
- Clear indicators prevent misunderstanding
- Personalized experience

**UX Cons**:
- Team coordination challenge if everyone uses different settings
  - "Look at the spike at 10am" - which timezone?
  - Requires verbal clarification in discussions
- Configuration burden
  - Users must discover and configure the setting
  - Defaults matter significantly
- Shared screenshots/reports may be confusing
  - No guarantee of consistent timezone in shared media
- More complex implementation (user preferences storage)

**When This Works**:
- Individual contributor dashboards
- Personal monitoring workflows
- Teams with mature timezone communication practices

**Research Evidence**:
- Grafana offers user-level configuration (4-level hierarchy)
- Common in productivity tools (Slack, Asana, etc.)
- Requires strong visual indicators to prevent confusion

---

### Option D: Configurable Per Dashboard (For Shared Dashboards)

**UX Pros**:
- Dashboard creator sets appropriate context
  - "EU Cluster Dashboard" → Europe/London
  - "Incident Response" → UTC
  - "APAC Model Monitoring" → Asia/Singapore
- Shared understanding for team dashboards
  - Everyone viewing dashboard sees same times
  - Screenshots/sharing maintains context
- Flexible across different use cases
  - Automated CI/CD dashboards can specify UTC
  - Regional monitoring can specify local time
- User can still override if needed (with D+C hybrid)

**UX Cons**:
- Dashboard proliferation if people want different timezones
  - "EU Dashboard (UTC version)" vs "EU Dashboard (London version)"
- Creator burden (must choose appropriate timezone)
- Users may forget which dashboard uses which timezone
  - Requires prominent indicators
- More complex implementation (dashboard metadata + UI)

**When This Works**:
- Shared team dashboards
- Use-case-specific dashboards (incident vs. monitoring vs. reporting)
- Organizations with dashboard curators/owners

**Research Evidence**:
- Grafana supports per-dashboard timezone (most flexible option)
- Datadog allows custom time frames per dashboard
- Dashboard design best practices: "convey the story behind the data"

---

## Implementation Guidelines

### 1. Display Format

**Timestamp Format**:
```
MMM DD, YYYY HH:mm:ss ZZZ
Nov 14, 2025 14:23:42 UTC
Nov 14, 2025 06:23:42 PST
```

**Compact Format** (for space-constrained UI):
```
YYYY-MM-DD HH:mm ZZZ
2025-11-14 14:23 UTC
```

**Tooltip/Detailed Format**:
```
Wednesday, November 14, 2025
14:23:42 UTC (06:23:42 your local time)
```

**Research Evidence**:
- Intl.DateTimeFormat best practices from UX patterns
- ISO 8601 standard for unambiguous date representation

---

### 2. Timezone Indicator

**Required Elements**:
1. **Always visible timezone badge** in dashboard header
   ```
   [🌍 UTC] [🌍 PST] [🌍 Browser Local]
   ```

2. **Override indicator** when user overrides dashboard default
   ```
   [🌍 Your timezone: PST | Dashboard: UTC]
   ```

3. **Tooltip on hover**: Full timezone name and current offset
   ```
   Pacific Standard Time (UTC-8)
   Coordinated Universal Time
   Europe/London (GMT, UTC+0)
   ```

**Color Coding**:
- Green: Browser local / No override
- Blue: Dashboard-specified timezone
- Orange: User override active (different from dashboard)
- Purple: UTC (special case, always obvious)

**Research Evidence**:
- UX patterns: "clear indicators when timezone change occurs"
- Dashboard design principles: "smooth transitions with 'last refreshed' timestamps"
- Avoid confusion: make timezone differences highly visible

---

### 3. Switching Mechanism

**User-Level Configuration**:
- **Location**: User profile settings
- **UI**: Dropdown with common timezones + search
- **Categories**:
  1. Browser Local (default)
  2. UTC
  3. Common: PST/PDT, EST/EDT, GMT, CET/CEST, JST, etc.
  4. Search: Full IANA timezone database
- **Live Preview**: Shows current time in selected timezone
- **Save Scope**: Applies to all dashboards (unless overridden)

**Dashboard-Level Configuration**:
- **Location**: Dashboard settings (edit mode)
- **UI**: Same dropdown as user settings
- **Default**: "Inherit from user preference" (null/undefined)
- **Common Choices**:
  - Inherit from user (default)
  - UTC (for shared incident dashboards)
  - Specific timezone (for regional monitoring)
- **Visibility**: Dashboard viewers see indicator showing dashboard timezone

**Quick Toggle** (Advanced Feature):
- **Location**: Dashboard header, next to timezone indicator
- **Action**: Click timezone badge to open quick menu
- **Options**:
  - Switch to UTC (temporary)
  - Switch to browser local (temporary)
  - Switch to dashboard default (if overridden)
  - Open full timezone selector
- **Persistence**: Can be temporary (session) or saved to user preference

**Research Evidence**:
- Grafana URL parameter support: `?timezone=Europe/Madrid`
- Datadog quick toggle between browser and UTC
- Smart Interface Design Patterns: timezone selection with search and categorization

---

### 4. Edge Cases

#### UTC Events (Kubernetes Jobs, Cron, etc.)
**Challenge**: Backend scheduled events often use UTC
**Solution**:
- Show original UTC time in tooltip: "Scheduled: 14:00 UTC (06:00 PST in your timezone)"
- Visual marker for UTC-scheduled events
- Option to "Show all schedules in UTC" toggle in dashboard

#### Scheduled Reports
**Challenge**: When should report run? What timezone should it show?
**Solution**:
- Report configuration specifies timezone explicitly
- Default to dashboard timezone, fallback to UTC
- Email/PDF includes timezone in header: "Generated at 2025-11-14 06:00 PST"
- Recipients see their local time in parentheses if system knows their timezone

#### Alert Timestamps
**Challenge**: Alerts must be unambiguous for incident response
**Solution**:
- Alerts ALWAYS include UTC timestamp
- Also include user's local time if known: "Alert fired at 14:23 UTC (06:23 your time)"
- Alert history view respects dashboard/user timezone preference
- Alert rules editor shows next evaluation time in selected timezone with UTC reference

#### Daylight Saving Time Transitions
**Challenge**: Hour disappears (spring) or repeats (fall)
**Solution**:
- Use IANA timezone database (handles DST automatically via date-fns-tz)
- Show warning in UI during DST transition dates
- Time range selector prevents selecting invalid times (e.g., 2am on spring forward day)
- Historical data: show gaps/overlaps explicitly with annotation

#### Cross-Timezone Data Correlation
**Challenge**: Correlating events from systems in different timezones
**Solution**:
- Backend stores everything in UTC (already done)
- UI can overlay multiple timezone indicators on single timeline
- Advanced feature: "Show multiple timezones" toggle
  ```
  14:00 UTC
  06:00 PST (US-West cluster)
  15:00 CET (EU cluster)
  ```

#### Exported Dashboards/Screenshots
**Challenge**: Shared images lose context of timezone
**Solution**:
- Always include timezone in screenshot watermark
- Export dialog warns: "This dashboard will export in [UTC]. Viewers should be aware."
- CSV/JSON exports include timezone metadata in header
- Shareable URLs include timezone parameter: `?tz=UTC`

#### API/Programmatic Access
**Challenge**: REST API consumers need consistent behavior
**Solution**:
- API always returns UTC timestamps (ISO 8601 with Z)
- Optional query parameter: `?display_timezone=America/Los_Angeles`
- Response includes both UTC and converted time if display_timezone specified
- Documentation clearly states UTC is canonical format

**Research Evidence**:
- Azure DevOps timezone best practices for CI/CD
- Grafana URL parameter support for edge cases
- Airflow timezone issues during CI/CD pipeline setup (common pitfall)
- Datadog anomaly monitor DST handling

---

## Implementation Phases

### Phase 1: Foundation (Minimal Viable Feature)
**Effort**: 2-3 sprints

1. **Backend**:
   - Add `timezone` field to user preferences schema (string, nullable)
   - Add `timezone` field to dashboard metadata (string, nullable)
   - API endpoints: GET/PUT user timezone preference

2. **Frontend**:
   - User settings page: timezone dropdown (browser/UTC/common timezones)
   - Dashboard settings: timezone dropdown (inherit/UTC/specific)
   - Timezone indicator badge in dashboard header
   - Update all timestamp displays to use TimeZoneProvider context

3. **Testing**:
   - Unit tests for timezone conversion edge cases
   - E2E tests for user preference persistence
   - DST transition testing

**Success Metrics**:
- Users can set timezone preference
- Dashboards respect user/dashboard timezone
- No timestamp display regressions

---

### Phase 2: Enhanced UX (User Delight)
**Effort**: 1-2 sprints

1. **Visual Indicators**:
   - Color-coded timezone badges
   - Override indicator (user ≠ dashboard)
   - Tooltip with full timezone info

2. **Quick Toggle**:
   - Click badge to switch UTC ↔ Browser ↔ Dashboard
   - Temporary vs. permanent toggle option

3. **Improved Timezone Selector**:
   - Searchable dropdown with categories
   - Live preview of current time
   - Recent timezones (for quick switching)

**Success Metrics**:
- Reduced timezone-related support tickets
- User feedback on clarity of indicators
- Quick toggle adoption rate

---

### Phase 3: Advanced Features (Power User)
**Effort**: 2 sprints

1. **Multi-Timezone Display**:
   - Overlay multiple timezones on timeline
   - Useful for multi-region correlation

2. **Smart Defaults**:
   - Detect regional dashboards (name contains "EU", "APAC", etc.)
   - Suggest appropriate timezone

3. **Export/Share Improvements**:
   - Timezone watermark on screenshots
   - URL parameter support: `?tz=UTC`
   - CSV/JSON export with timezone metadata

4. **Alert Enhancements**:
   - Dual timestamp in alert notifications
   - DST transition warnings

**Success Metrics**:
- Advanced feature adoption by power users
- Reduction in timezone-related errors during incident response
- Positive feedback from distributed teams

---

## Success Metrics & Validation

### Quantitative Metrics

1. **User Adoption**:
   - % of active users who configure timezone preference
   - Target: >30% within 3 months (indicates awareness and value)

2. **Dashboard Configuration**:
   - % of shared dashboards with explicit timezone setting
   - Target: >50% of incident/team dashboards within 6 months

3. **Support Reduction**:
   - Decrease in timezone-related support tickets/questions
   - Target: -50% within 6 months

4. **Error Reduction**:
   - Decrease in incorrect timestamp interpretation (measured via incident post-mortems)
   - Target: Zero timezone-related miscommunications in incident timelines

### Qualitative Metrics

1. **User Interviews** (3 months post-launch):
   - Interview 10-15 users across personas
   - Questions:
     - "How clear is it which timezone you're viewing?"
     - "Have you experienced any timezone confusion?"
     - "How easy is it to switch timezones when needed?"

2. **Usability Testing**:
   - Task: "What time did this event occur in your local time?"
   - Success: >90% correct answer within 5 seconds
   - Validate: Timezone indicators are clear and discoverable

3. **Incident Response Observation**:
   - Shadow 3-5 incident responses
   - Document any timezone-related communication delays
   - Validate: Shared dashboards reduce coordination overhead

### A/B Testing Opportunities

1. **Default Behavior**:
   - A: Browser local (current implicit behavior)
   - B: UTC (SRE best practice)
   - Measure: User confusion, preference overrides, support tickets

2. **Indicator Design**:
   - A: Badge in header
   - B: Inline with each timestamp
   - Measure: User awareness, comprehension testing

3. **Quick Toggle Placement**:
   - A: Header badge click
   - B: Time range selector integration
   - Measure: Discoverability, usage frequency

---

## Risk Mitigation

### Risk 1: Migration Confusion
**Issue**: Existing users suddenly see timezone indicators

**Mitigation**:
- Default to browser local (no behavior change for most users)
- Launch with in-app announcement/tutorial
- Changelog with clear examples
- Optional: One-time modal explaining new feature

### Risk 2: Team Coordination Breakdown
**Issue**: Team members use different timezones, causing confusion

**Mitigation**:
- Strongly recommend UTC for shared incident dashboards (documentation + in-app tips)
- Dashboard templates set sensible defaults (e.g., "Incident Response Template" defaults to UTC)
- Team admin can set organization-wide default timezone

### Risk 3: Performance Impact
**Issue**: Timezone conversion on every timestamp render

**Mitigation**:
- Memoize timezone conversion results
- Use existing date-fns-tz library (already in use, optimized)
- Benchmark: no >10ms impact on dashboard load time

### Risk 4: DST Edge Cases
**Issue**: Bugs during spring/fall DST transitions

**Mitigation**:
- Use IANA timezone database (handles DST automatically)
- Automated tests for DST boundaries
- Warning messages during DST transition weekends
- Documentation about invalid/ambiguous times

---

## Appendix A: Code Architecture

### Current Implementation Analysis

**File**: `/workspace/sessions/agentic-session-1763159109/workspace/perses/ui/components/src/context/TimeZoneProvider.tsx`

Perses already has solid foundation:
```typescript
interface TimeZoneProviderProps {
  timeZone?: string;  // undefined | "local" | "browser" | "UTC" | IANA timezone
}

// Utility functions:
formatWithUserTimeZone(date, formatString)  // Uses date-fns-tz
dateFormatOptionsWithUserTimeZone(options)  // Intl.DateTimeFormat
```

**Required Additions**:

1. **User Preferences API** (Backend):
```go
type UserPreferences struct {
    Timezone *string `json:"timezone,omitempty"`  // nil = browser, "UTC" | IANA
}
```

2. **Dashboard Metadata** (Backend):
```go
type DashboardSpec struct {
    Timezone *string `json:"timezone,omitempty"`  // nil = inherit from user
}
```

3. **Frontend Context Enhancement**:
```typescript
interface TimeZoneContextValue {
  effectiveTimeZone: string;      // Resolved timezone (dashboard → user → browser)
  dashboardTimeZone?: string;     // Dashboard-configured timezone
  userTimeZone?: string;          // User preference
  browserTimeZone: string;        // Detected from Intl.DateTimeFormat
  isOverridden: boolean;          // User TZ ≠ Dashboard TZ
  setUserTimeZone: (tz: string) => void;
}
```

---

## Appendix B: Competitive Feature Matrix

| Feature | Grafana | Datadog | New Relic | Perses (Proposed) |
|---------|---------|---------|-----------|-------------------|
| Browser local timezone | ✅ | ✅ | ✅ | ✅ |
| UTC display | ✅ | ✅ | ❌ | ✅ |
| User-level config | ✅ | ❌ | ❌ | ✅ |
| Dashboard-level config | ✅ | ⚠️ Toggle only | ❌ | ✅ |
| Hierarchical config | ✅ (4 levels) | ❌ | ❌ | ✅ (2 levels) |
| IANA timezone support | ✅ | ❌ | ❌ | ✅ |
| URL parameter override | ✅ | ❌ | ❌ | 🔮 Phase 3 |
| Visual timezone indicator | ⚠️ Subtle | ⚠️ Subtle | ❌ | ✅ Prominent |
| Quick toggle | ❌ | ✅ | ❌ | ✅ |
| DST-aware monitors | ❌ | ✅ | ❌ | 🔮 Phase 3 |

**Legend**: ✅ Full support | ⚠️ Partial support | ❌ Not supported | 🔮 Planned

---

## Appendix C: Research Sources

### Primary Sources
1. **UX Stack Exchange** - "Displaying Dates and Time with Locally Relevant Time Zone Information" (2024)
2. **Smart Interface Design Patterns** - "Designing A Time Zone Selection UX" (2024-2025)
3. **Grafana Official Documentation** - Organization Preferences, Dashboard Settings (2024)
4. **Datadog Documentation** - Monitors, Dashboards, Custom Time Frames (2024)
5. **Google SRE Resources** - Incident Management Guide, Workbook on Incident Response (2024)
6. **Incident.io** - "Time, time zones and scheduling" Hub (2024)
7. **Microsoft Azure DevOps** - "Timezone settings and usage" Documentation (2024)
8. **Neptune.ai** - "How to Monitor Your Models in Production" (2024)

### Community Forums
1. Grafana Community Forums - Multiple timezone discussions (2023-2024)
2. New Relic Community - Timezone feature requests (2023-2024)
3. Sisense Community - "How do you handle timezone conversions on dashboards?" (2024)

### Industry Best Practices
1. SRE Best Practices Checklist 2025 (Rootly)
2. CI/CD Monitoring Best Practices (Datadog, 2024)
3. Dashboard UX Design Principles 2025 (DesignRush, UXPin)

### Technical Documentation
1. IANA Time Zone Database
2. ISO 8601 Standard
3. Intl.DateTimeFormat API (MDN)
4. date-fns-tz Library Documentation

---

## Conclusion

Based on comprehensive research of industry best practices, competitive analysis, and user persona needs, **Option C + D (Hierarchical configuration with dashboard defaults and user overrides)** provides the optimal balance of flexibility, usability, and team coordination.

This approach:
- **Supports all three user personas** identified in research
- **Aligns with industry leader Grafana's** proven model
- **Addresses incident response needs** with UTC for shared dashboards
- **Enables multi-region monitoring** with timezone context
- **Maintains CI/CD consistency** through explicit configuration
- **Builds on Perses's existing infrastructure** (TimeZoneProvider already exists)

The phased implementation approach allows validation of core assumptions before investing in advanced features, with clear success metrics to measure impact on distributed team collaboration and incident response effectiveness.

---

**Next Steps**:
1. Validate approach with 5-10 users from different personas
2. Create design mockups for timezone indicator and selector UI
3. Define database schema for user preferences and dashboard metadata
4. Estimate implementation effort for Phase 1
5. Add to product roadmap with priority based on user feedback

---

**Report prepared by**: Ryan (UX Researcher)
**Review requested from**: Product Manager, Engineering Lead, Design Lead
**Expected decision date**: [To be scheduled]
