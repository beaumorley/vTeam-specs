# Dashboard Export - Quick Reference Card

## Endpoint

```
GET /api/v1/projects/{project}/dashboards/{id}/export
```

## Basic Usage

```bash
# Export active dashboard
curl -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/api/v1/projects/my-project/dashboards/dashboard-123/export" \
  -o dashboard.json

# Export draft dashboard
curl -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/api/v1/projects/my-project/dashboards/dashboard-123/export?include_draft=true" \
  -o dashboard.json
```

## Parameters

| Type | Name | Required | Description |
|------|------|----------|-------------|
| Path | project | Yes | Project namespace |
| Path | id | Yes | Dashboard ID |
| Query | include_draft | No | Allow draft export (default: false) |

## Response

### Success (200 OK)

```json
{
  "kind": "Dashboard",
  "apiVersion": "v1",
  "metadata": { "name": "...", "project": "...", ... },
  "spec": { /* Perses dashboard definition */ },
  "annotations": { "openshift.io/dashboard-id": "...", ... }
}
```

**Headers**:
- `Content-Type: application/json`
- `Content-Disposition: attachment; filename="dashboard-{name}.json"`

### Errors

| Code | Reason | Solution |
|------|--------|----------|
| 400 | Missing parameters | Check project and ID |
| 400 | Draft without flag | Add `?include_draft=true` |
| 401 | Not authenticated | Add Authorization header |
| 403 | Insufficient permissions | Check RBAC for namespace |
| 404 | Not found | Verify dashboard exists |

## Export Format

```json
{
  "kind": "Dashboard",
  "apiVersion": "v1",
  "metadata": {
    "name": "dashboard-name",
    "project": "namespace",
    "createdAt": "2025-01-15T12:00:00Z",
    "updatedAt": "2025-01-15T14:30:00Z",
    "version": 3
  },
  "spec": {
    // Perses dashboard spec (panels, layouts, datasources)
  },
  "annotations": {
    "openshift.io/dashboard-id": "dashboard-abc123",
    "openshift.io/created-by": "user@example.com",
    "openshift.io/updated-by": "user@example.com",
    "openshift.io/state": "active",
    "openshift.io/refresh-interval": 15,
    "openshift.io/time-range-default": "last_1h"
  }
}
```

## RBAC Requirements

- **Authentication**: Bearer token required
- **Namespace Access**: User must have access to project
- **Permission**: `Dashboard:read` in namespace

## Common Use Cases

### Backup
```bash
# Backup all dashboards in a project
for ID in $(curl -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/api/v1/projects/my-project/dashboards" | \
  jq -r '.dashboards[].id'); do
  curl -H "Authorization: Bearer $TOKEN" \
    "$API_BASE/api/v1/projects/my-project/dashboards/$ID/export" \
    -o "backup-$ID.json"
done
```

### Migration
```bash
# Export from dev
curl -H "Authorization: Bearer $DEV_TOKEN" \
  "$DEV_API/api/v1/projects/dev/dashboards/dashboard-123/export" \
  -o dashboard.json

# Import to prod (future feature)
curl -X POST -H "Authorization: Bearer $PROD_TOKEN" \
  -H "Content-Type: application/json" \
  --data @dashboard.json \
  "$PROD_API/api/v1/projects/prod/dashboards/import"
```

### Version Control
```bash
# Export and commit to Git
curl -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/api/v1/projects/ml-workspace/dashboards/dashboard-123/export" \
  -o dashboards/gpu-metrics.json

git add dashboards/gpu-metrics.json
git commit -m "Update GPU metrics dashboard"
git push
```

## Implementation Files

- **Handler**: `backend/pkg/api/handlers/dashboard_handlers.go` (lines 494-817)
- **Router**: `backend/pkg/api/router.go` (line 102)

## Key Functions

1. **ExportDashboard()** - Main handler
2. **toPersesExportFormat()** - Format converter
3. **sanitizeFilename()** - Filename sanitizer

## Security Notes

- Same RBAC as GetDashboard
- Filename sanitization prevents path traversal
- Draft dashboards require explicit opt-in
- All exports are audit logged

## Performance

- Typical dashboard: 10-50ms
- Large dashboard (1-5MB): 100-500ms
- Single database query
- In-memory processing (no temp files)

## Documentation

- Detailed Guide: `EXPORT_ENDPOINT_USAGE.md`
- Implementation: `T043_EXPORT_DASHBOARD_IMPLEMENTATION.md`
- Example Format: `export-format-example.json`

---

**Quick Tips**:
- Use `-o` with curl to save file
- Add `?include_draft=true` for draft dashboards
- Check 403 errors for RBAC issues
- Filename is auto-generated from dashboard name
- Export format is Perses-native JSON
