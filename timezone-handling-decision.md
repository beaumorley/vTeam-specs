### Timezone Handling Strategy

**Decision**: Option C + Option D (Hybrid Hierarchical Configuration)
- Dashboard-level timezone defaults (configurable by dashboard creator)
- User-level timezone overrides (configurable in user preferences)
- Fallback to browser local timezone
- Prominent visual indicators showing active timezone and override status

**Rationale**:

**User Experience Justification**:
- Supports zero-configuration simplicity (browser default) while enabling advanced workflows
- Dashboard creators can set appropriate context (UTC for incidents, regional time for geographic monitoring, local for personal dashboards)
- Individual users maintain autonomy to override when their workflow demands it
- Mirrors Grafana's proven hierarchical model (industry leader with established user acceptance)
- Clear visual indicators prevent timestamp misinterpretation during critical operations

**Distributed Team Collaboration Impact**:
- Shared incident response dashboards configured to UTC eliminate "what timezone?" questions during high-stress situations
- Regional monitoring dashboards (EU cluster, APAC models, US-West services) display region-local time for business context
- Personal dashboards respect individual preferences without forcing team consensus
- Verbal communication clarity: "Check the spike at 14:23" is unambiguous when everyone viewing same dashboard sees same time
- Follow-the-sun handoffs benefit from consistent shared dashboard timestamps with personal override capability

**Incident Response Implications**:
- Shared incident dashboards default to UTC (industry SRE best practice, supported by Google SRE research)
- Post-mortem timeline reconstruction has single source of truth (no timezone conversion errors)
- Individual responders can temporarily toggle to local time for correlation with local systems/logs without affecting team view
- Alert timestamps include both UTC (canonical) and user local time (convenience)
- Screenshots and exported dashboards retain timezone context via visual watermark

**Alternatives Considered**:

**Option A: User Local Timezone (Browser-Based)**
- **Pros**: Zero configuration, familiar to users, lowest implementation complexity
- **Cons**:
  - Distributed team nightmare ("incident started at 3am" - whose 3am?)
  - Multi-region analysis requires mental conversion ("what time is EU business hours in PST?")
  - Incident coordination suffers when different team members see different times
  - Shared screenshots/reports lack consistent context
- **When it works**: Single-timezone teams, personal non-critical dashboards
- **Why rejected**: Our research shows distributed teams and multi-region ML monitoring are core use cases

**Option B: Cluster/System Timezone (Consistent UTC)**
- **Pros**:
  - Eliminates ambiguity in team communication
  - CI/CD friendly (deterministic, matches backend storage)
  - Incident response clarity (everyone sees identical timestamps)
  - Aligns with SRE best practices
- **Cons**:
  - Requires constant mental conversion for daily monitoring
  - Poor UX for multi-region model performance analysis (was spike during EU business hours?)
  - Ignores user context (data scientist in Tokyo forced to think in UTC)
  - Not beginner-friendly for casual dashboard viewers
- **When it works**: Highly technical SRE teams, automated/programmatic access, compliance requirements
- **Why rejected as sole option**: Too rigid for diverse user base; better as configurable choice

**Option C: Configurable Per User (User Preference Only)**
- **Pros**:
  - User autonomy (SRE chooses UTC, data scientist chooses regional, manager chooses local)
  - Personalized experience reduces cognitive load
  - Accommodates different workflows
- **Cons**:
  - Team coordination challenge if everyone uses different settings ("look at the 10am spike" - which timezone?)
  - Shared screenshots/Slack discussions require constant clarification
  - Configuration burden (must discover setting and choose wisely)
- **When it works**: Individual contributor dashboards, mature teams with explicit timezone communication norms
- **Why enhanced with Option D**: Needs dashboard-level context for shared collaboration scenarios

**Option D: Configurable Per Dashboard (Dashboard Metadata Only)**
- **Pros**:
  - Dashboard creator sets appropriate context (Incident→UTC, EU Cluster→Europe/London, APAC Models→Asia/Singapore)
  - Shared understanding (everyone viewing dashboard sees same time)
  - Screenshots/exports maintain consistent context
  - CI/CD dashboards can specify UTC for reproducibility
- **Cons**:
  - Dashboard proliferation if people want different timezone views of same data
  - Creator burden (must choose appropriate timezone)
  - Less individual flexibility if user's workflow differs from dashboard intent
- **When it works**: Shared team dashboards, use-case-specific contexts, curated dashboard libraries
- **Why enhanced with Option C**: Needs user override for individual workflow flexibility

**Implementation Guidelines**:

**Display Format**:
- **Standard timestamps**: `MMM DD, YYYY HH:mm:ss ZZZ` (e.g., "Nov 14, 2025 14:23:42 UTC" or "Nov 14, 2025 06:23:42 PST")
- **Compact timestamps** (space-constrained UI): `YYYY-MM-DD HH:mm ZZZ` (e.g., "2025-11-14 14:23 UTC")
- **Tooltips/detailed view**: Include dual timezone display: "14:23:42 UTC (06:23:42 your local time)"
- **Timezone abbreviation**: Always include timezone suffix (UTC, PST, GMT, etc.) to prevent ambiguity

**Timezone Indicator**:
- **Location**: Persistent badge in dashboard header, always visible
- **Format**: `[🌍 UTC]` `[🌍 PST]` `[🌍 Browser Local]`
- **Override indicator** when user timezone differs from dashboard: `[🌍 Your timezone: PST | Dashboard: UTC]`
- **Color coding**:
  - Green: Browser local / No configuration
  - Blue: Dashboard-specified timezone
  - Orange: User override active (different from dashboard default)
  - Purple: UTC (special case, always obvious)
- **Tooltip on hover**: Full timezone name, UTC offset, current time (e.g., "Pacific Standard Time (UTC-8), Current time: 06:23:42")

**Switching Mechanism**:
- **User preference configuration**:
  - Location: User profile settings
  - UI: Searchable dropdown with categories (Browser Local, UTC, Common timezones, IANA database search)
  - Live preview showing current time in selected timezone
  - Applies to all dashboards unless overridden by dashboard or quick toggle

- **Dashboard configuration**:
  - Location: Dashboard settings (edit mode only, requires edit permissions)
  - UI: Same dropdown as user settings
  - Default: "Inherit from user preference" (null/undefined)
  - Common presets: UTC (incidents), Browser Local (personal), Specific region (monitoring)

- **Quick toggle** (in-dashboard switching):
  - Location: Click timezone indicator badge in dashboard header
  - Options in dropdown menu:
    - Switch to UTC
    - Switch to Browser Local
    - Switch to Dashboard Default (if currently overridden)
    - Open full timezone selector
  - Persistence: Temporary (session-only) or save to user preference
  - Discoverability: Tooltip on badge says "Click to change timezone"

**Edge Cases**:

**UTC Events (Scheduled Jobs, Cron, Kubernetes)**:
- Display original UTC schedule time in tooltip: "Scheduled: 14:00 UTC (06:00 PST in your timezone)"
- Visual marker/icon for UTC-scheduled events to distinguish from timezone-converted display
- Option toggle: "Show all schedules in original timezone (UTC)" for troubleshooting

**Scheduled Reports (Email, PDF)**:
- Report configuration explicitly specifies timezone (defaults to dashboard timezone, fallback to UTC)
- Generated report header includes timestamp and timezone: "Generated at 2025-11-14 06:00 PST"
- PDF/email includes note: "All timestamps in this report are displayed in [PST]"
- If recipient timezone known (user database), show dual time: "06:00 PST (14:00 UTC)"

**Alerts**:
- Alert notifications ALWAYS include UTC timestamp as canonical reference
- Also include user's local time if known: "Alert fired at 14:23 UTC (06:23 your local time)"
- Alert history dashboard respects user/dashboard timezone preference for browsing
- Alert rule configuration shows next evaluation time in user's selected timezone with UTC in parentheses

**Automated Dashboards (CI/CD Pipelines)**:
- Default to UTC (deterministic, no browser dependency)
- Dashboard metadata can override if specific timezone needed (e.g., deployment schedule aligned to business hours)
- API access: All timestamps returned in ISO 8601 format with explicit timezone (e.g., "2025-11-14T14:23:42Z" for UTC)
- Programmatic dashboard creation should explicitly set timezone in dashboard spec

**Daylight Saving Time Transitions**:
- Use IANA timezone database via date-fns-tz (handles DST automatically)
- Time range selector prevents selecting invalid/ambiguous times (e.g., spring-forward hour gap)
- Display warning banner during DST transition weekends: "Note: [Timezone] transitions to/from DST on [Date]"
- Historical data during fall-back hour: Show annotation marking ambiguous hour (e.g., "2:30am occurred twice")

**Cross-Timezone Correlation**:
- Backend stores all data in UTC (no changes needed)
- UI conversion happens at display layer only
- Advanced feature (Phase 3): "Multi-timezone overlay" showing multiple timezone rulers on single timeline:
  ```
  14:00 UTC
  06:00 PST (US-West cluster events)
  15:00 CET (EU cluster events)
  ```

**Shared Screenshots/Exports**:
- Include timezone in screenshot watermark (subtle footer: "Displayed in UTC")
- Export dialog warns user: "This dashboard will export in [UTC]. Ensure recipients are aware of timezone."
- CSV/JSON exports include timezone metadata in header row/field
- Shareable URLs include timezone parameter: `/dashboard/abc123?tz=UTC` (Phase 3)
- Copy-paste timestamp feature includes timezone: "Nov 14, 2025 14:23:42 UTC" (not just "14:23:42")

**Implementation Notes**:
- Perses already has TimeZoneProvider React context (/workspace/sessions/agentic-session-1763159109/workspace/perses/ui/components/src/context/TimeZoneProvider.tsx)
- Supports: undefined (browser local), "local"/"browser", "UTC"/"utc", IANA timezones (e.g., "America/New_York")
- Requires: User preferences database schema, dashboard metadata field, UI components for selection/indicator
- Phase 1 (MVP): Basic user/dashboard configuration, visual indicator, standard display
- Phase 2 (Enhanced UX): Quick toggle, improved selectors, color-coded indicators
- Phase 3 (Advanced): URL parameters, multi-timezone overlay, DST warnings, export enhancements
