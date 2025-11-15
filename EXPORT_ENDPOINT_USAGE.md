# Dashboard Export Endpoint - Usage Guide

## Quick Reference

### Endpoint Details

- **Method**: GET
- **Path**: `/api/v1/projects/{project}/dashboards/{id}/export`
- **Authentication**: Required (Bearer token)
- **Authorization**: Dashboard:read permission in namespace
- **Response**: JSON file download (application/json)

## Basic Usage

### 1. Export Active Dashboard

Export a dashboard that is in the 'active' state:

```bash
curl -X GET \
  -H "Authorization: Bearer ${OPENSHIFT_TOKEN}" \
  "https://dashboard-api.apps.cluster.example.com/api/v1/projects/ml-workspace/dashboards/dashboard-abc123/export" \
  -o my-dashboard.json
```

**Response Headers**:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Disposition: attachment; filename="dashboard-ml-model-training.json"
```

### 2. Export Draft Dashboard

Export a dashboard in 'draft' state (requires explicit flag):

```bash
curl -X GET \
  -H "Authorization: Bearer ${OPENSHIFT_TOKEN}" \
  "https://dashboard-api.apps.cluster.example.com/api/v1/projects/ml-workspace/dashboards/dashboard-xyz789/export?include_draft=true" \
  -o draft-dashboard.json
```

## Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project` | string | Yes | Namespace/project identifier |
| `id` | string | Yes | Dashboard ID (format: dashboard-{uuid}) |

## Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `include_draft` | boolean | No | false | Allow export of draft dashboards |

## Response Format

### Success Response (200 OK)

```json
{
  "kind": "Dashboard",
  "apiVersion": "v1",
  "metadata": {
    "name": "GPU Utilization Metrics",
    "project": "ml-workspace",
    "createdAt": "2025-01-10T08:30:00Z",
    "updatedAt": "2025-01-15T14:45:00Z",
    "version": 5
  },
  "spec": {
    // Perses dashboard definition
    "display": { ... },
    "panels": { ... },
    "layouts": [ ... ],
    "datasources": { ... },
    "variables": [ ... ]
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

### Error Responses

#### 400 Bad Request - Missing Parameters

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "project and dashboard ID are required"
  }
}
```

#### 400 Bad Request - Draft Export Without Flag

```json
{
  "error": {
    "code": "INVALID_STATE",
    "message": "Cannot export draft dashboard unless include_draft=true is specified",
    "details": {
      "dashboard_id": "dashboard-xyz789",
      "state": "draft",
      "hint": "Add query parameter ?include_draft=true to export draft dashboards"
    }
  }
}
```

#### 401 Unauthorized - Missing Authentication

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "authentication required"
  }
}
```

#### 403 Forbidden - Insufficient Permissions

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "insufficient permissions to export dashboard"
  }
}
```

#### 404 Not Found - Dashboard Doesn't Exist

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Dashboard not found: dashboard-abc123",
    "details": {
      "dashboard_id": "dashboard-abc123"
    }
  }
}
```

#### 404 Not Found - Dashboard Not in Project

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Dashboard not found in project 'ml-workspace'",
    "details": {
      "dashboard_id": "dashboard-abc123",
      "project": "ml-workspace"
    }
  }
}
```

#### 500 Internal Server Error

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "failed to retrieve dashboard"
  }
}
```

## Examples

### Example 1: Export with wget

```bash
wget --header="Authorization: Bearer ${TOKEN}" \
  "https://api.example.com/api/v1/projects/my-project/dashboards/dashboard-123/export" \
  -O exported-dashboard.json
```

### Example 2: Export with Python

```python
import requests
import os

def export_dashboard(project, dashboard_id, output_file, include_draft=False):
    token = os.getenv('OPENSHIFT_TOKEN')
    base_url = "https://dashboard-api.apps.cluster.example.com"

    url = f"{base_url}/api/v1/projects/{project}/dashboards/{dashboard_id}/export"

    params = {}
    if include_draft:
        params['include_draft'] = 'true'

    headers = {
        'Authorization': f'Bearer {token}'
    }

    response = requests.get(url, headers=headers, params=params)

    if response.status_code == 200:
        with open(output_file, 'w') as f:
            f.write(response.text)
        print(f"Dashboard exported successfully to {output_file}")
    else:
        print(f"Export failed: {response.status_code}")
        print(response.json())

# Usage
export_dashboard('ml-workspace', 'dashboard-abc123', 'my-dashboard.json')
```

### Example 3: Export with JavaScript/Node.js

```javascript
const axios = require('axios');
const fs = require('fs');

async function exportDashboard(project, dashboardId, outputFile, includeDraft = false) {
  const token = process.env.OPENSHIFT_TOKEN;
  const baseUrl = 'https://dashboard-api.apps.cluster.example.com';

  const url = `${baseUrl}/api/v1/projects/${project}/dashboards/${dashboardId}/export`;

  const params = {};
  if (includeDraft) {
    params.include_draft = 'true';
  }

  try {
    const response = await axios.get(url, {
      headers: {
        'Authorization': `Bearer ${token}`
      },
      params: params
    });

    fs.writeFileSync(outputFile, JSON.stringify(response.data, null, 2));
    console.log(`Dashboard exported successfully to ${outputFile}`);
  } catch (error) {
    console.error('Export failed:', error.response?.status);
    console.error(error.response?.data);
  }
}

// Usage
exportDashboard('ml-workspace', 'dashboard-abc123', 'my-dashboard.json');
```

### Example 4: Export Multiple Dashboards (Bash Script)

```bash
#!/bin/bash

# Configuration
PROJECT="ml-workspace"
TOKEN="${OPENSHIFT_TOKEN}"
API_BASE="https://dashboard-api.apps.cluster.example.com/api/v1"
OUTPUT_DIR="./exported-dashboards"

# Create output directory
mkdir -p "${OUTPUT_DIR}"

# List all dashboards in project
DASHBOARDS=$(curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${API_BASE}/projects/${PROJECT}/dashboards" | \
  jq -r '.dashboards[].id')

# Export each dashboard
for DASHBOARD_ID in ${DASHBOARDS}; do
  echo "Exporting ${DASHBOARD_ID}..."

  curl -s -H "Authorization: Bearer ${TOKEN}" \
    "${API_BASE}/projects/${PROJECT}/dashboards/${DASHBOARD_ID}/export" \
    -o "${OUTPUT_DIR}/${DASHBOARD_ID}.json"

  if [ $? -eq 0 ]; then
    echo "✓ Exported ${DASHBOARD_ID}"
  else
    echo "✗ Failed to export ${DASHBOARD_ID}"
  fi
done

echo "Export complete. Files saved to ${OUTPUT_DIR}"
```

## Use Cases

### 1. Backup and Restore

Export dashboards for backup purposes:

```bash
# Backup all dashboards
./backup-dashboards.sh ml-workspace ./backups/2025-01-15/

# Restore dashboard (using future import endpoint)
curl -X POST \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  --data @backups/2025-01-15/dashboard-abc123.json \
  "https://api.example.com/api/v1/projects/ml-workspace/dashboards/import"
```

### 2. Dashboard Migration

Export from development, import to production:

```bash
# Export from dev
curl -H "Authorization: Bearer ${DEV_TOKEN}" \
  "${DEV_API}/api/v1/projects/dev-workspace/dashboards/dashboard-123/export" \
  -o dashboard.json

# Import to prod (future feature)
curl -X POST \
  -H "Authorization: Bearer ${PROD_TOKEN}" \
  -H "Content-Type: application/json" \
  --data @dashboard.json \
  "${PROD_API}/api/v1/projects/prod-workspace/dashboards/import"
```

### 3. Version Control

Track dashboard changes in Git:

```bash
# Export dashboard
curl -H "Authorization: Bearer ${TOKEN}" \
  "${API_BASE}/api/v1/projects/ml-workspace/dashboards/dashboard-abc123/export" \
  -o dashboards/gpu-metrics.json

# Commit to version control
git add dashboards/gpu-metrics.json
git commit -m "Update GPU metrics dashboard - add temperature panel"
git push
```

### 4. Dashboard Templating

Export a dashboard as a template for other teams:

```bash
# Export reference dashboard
curl -H "Authorization: Bearer ${TOKEN}" \
  "${API_BASE}/api/v1/projects/reference/dashboards/dashboard-template/export" \
  -o template-ml-monitoring.json

# Share with other teams
# They can modify and import to their projects
```

## Best Practices

### 1. Authentication

- Store tokens securely (environment variables, secrets management)
- Use short-lived tokens when possible
- Rotate tokens regularly

### 2. Error Handling

- Always check HTTP status codes
- Parse error responses for detailed information
- Implement retry logic for transient failures

### 3. File Management

- Use descriptive filenames
- Organize exports by project/date
- Validate JSON after export
- Store backups in version control or secure storage

### 4. Automation

- Schedule regular backups
- Automate dashboard migration between environments
- Integrate with CI/CD pipelines for dashboard-as-code

### 5. Security

- Respect RBAC permissions
- Don't export dashboards from unauthorized projects
- Sanitize sensitive data in annotations if sharing externally
- Audit export operations for compliance

## Troubleshooting

### Issue: "authentication required"

**Solution**: Ensure Authorization header is set correctly:
```bash
# Get token from OpenShift
oc whoami -t

# Use token in request
curl -H "Authorization: Bearer $(oc whoami -t)" ...
```

### Issue: "Dashboard not found in this project"

**Solution**: Verify dashboard ID belongs to the specified project:
```bash
# List dashboards in project
curl -H "Authorization: Bearer ${TOKEN}" \
  "${API_BASE}/api/v1/projects/ml-workspace/dashboards"
```

### Issue: "Cannot export draft dashboard"

**Solution**: Add `include_draft=true` query parameter:
```bash
curl -H "Authorization: Bearer ${TOKEN}" \
  "${API_BASE}/api/v1/projects/ml-workspace/dashboards/${ID}/export?include_draft=true"
```

### Issue: "insufficient permissions to export dashboard"

**Solution**: Verify you have read permissions in the namespace:
```bash
# Check permissions
oc auth can-i get dashboards -n ml-workspace
```

## Related Endpoints

- **List Dashboards**: `GET /api/v1/projects/{project}/dashboards`
- **Get Dashboard**: `GET /api/v1/projects/{project}/dashboards/{id}`
- **Create Dashboard**: `POST /api/v1/projects/{project}/dashboards`
- **Update Dashboard**: `PUT /api/v1/projects/{project}/dashboards/{id}`
- **Delete Dashboard**: `DELETE /api/v1/projects/{project}/dashboards/{id}`

## See Also

- [Perses Dashboard Schema Documentation](https://perses.dev/docs/schemas/dashboard)
- [OpenShift AI Dashboard API Reference](../api-reference.md)
- [RBAC Configuration Guide](../rbac-guide.md)
- [Export Format Example](./export-format-example.json)
