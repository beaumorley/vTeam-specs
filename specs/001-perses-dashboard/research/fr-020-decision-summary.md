# Dashboard Sharing & Access Control - Decision Summary

**Feature Requirement**: FR-020
**Security Requirement**: SC-010 (Zero Security Violations)
**Date**: 2025-11-14
**Decision Owner**: Stella (Staff Engineer)

---

## Decision: Hybrid RBAC + Optional Ephemeral Tokens

**Primary Mechanism**: Kubernetes-style RBAC via Perses Role/RoleBinding system
**Secondary Mechanism**: Optional URL-based ephemeral tokens for read-only access (24h TTL default)

**Sharing Scopes**:
- **Project-scoped** (MVP): Dashboards shareable within same namespace
- **Cross-project** (Phase 2): Via ClusterRole + RoleBinding pattern
- **Public** (Future): Admin-gated anonymous access with isolated data source

**Permission Model**:
- **MVP**: Read-only shares only (`actions: ["read"]`, `scopes: ["Dashboard"]`)
- **Phase 2**: Edit permissions with audit trail
- **Future**: Admin delegation (if customer demand)

---

## Rationale

### Security Considerations ✅

**SC-010 Compliance - Zero Violations**:
- **RBAC-first approach**: Leverages existing Perses/Kubernetes RBAC model with namespace isolation
- **Ephemeral tokens**: Short-lived (24h default, 7d max), revocable, read-only
- **Defense in depth**: JWT authentication + token blacklist + TLS 1.3 + audit logging
- **Least privilege**: Default deny; explicit grants via RoleBindings; tokens require `Dashboard:share` permission

**Threat Mitigation**:
| Threat | Mitigation |
|--------|------------|
| Token theft | Short TTL (24h), optional IP restrictions, revocation API |
| RBAC bypass | Token generation requires `Dashboard:share` permission |
| Privilege escalation | Tokens read-only; cannot modify dashboards/permissions |
| Cross-namespace leakage | Tokens scoped to specific dashboard; RBAC at API layer |
| Replay attacks | JWT nonce (jti), token blacklist, expiration enforcement |

**Encryption**:
- TLS 1.3 in-transit (enforced)
- PostgreSQL at-rest encryption for dashboard configs
- JWT signature verification (HS512)
- HTTP-only secure cookies (CSRF protection)

### User Experience Trade-offs ✅

**Internal Users** (80% use case):
- **RBAC**: Seamless SSO via OpenShift OAuth
- **No friction**: Automatic access based on namespace membership
- **Discoverability**: Dashboard list filtered by user permissions

**External Stakeholders** (20% use case):
- **URL tokens**: Ad-hoc sharing without cluster accounts
- **Use cases**: Incident response, contractor access, report embedding
- **UX flow**: Click "Share" → Generate token → Copy URL → Send to recipient
- **Similar to**: Perses ephemeral dashboard workflow (already implemented)

**Admin Control**:
- Token generation gated by `Dashboard:share` permission (prevents abuse)
- Token management dashboard (view active tokens, revoke, audit trail)
- Configurable TTL limits (default 24h, max 7d)

### Compliance Requirements Met ✅

**Audit Logging** (SOC2, ISO 27001, HIPAA):
- **Events logged**: dashboard.create, dashboard.read, dashboard.update, dashboard.delete, dashboard.share, dashboard.share.revoke, dashboard.access.token
- **Data captured**: User ID, action, resource ID, project/namespace, timestamp (UTC), client IP, user agent, access method (RBAC vs token)
- **Storage**: PostgreSQL audit table + OpenShift Audit API integration
- **Retention**: Configurable (90d default, 7y for HIPAA)
- **SIEM integration**: Export to Splunk/ELK for centralized monitoring

**Data Governance**:
- Dashboard configs stored in PostgreSQL (backed up per DR policy)
- Token revocation persisted (blacklist never pruned for audit trail)
- Metric data retention per Prometheus/Thanos policy (7-30d typical)

---

## Alternatives Considered

### Alternative 1: RBAC-Only (No URL Tokens)
**Rejected**: Poor UX for ad-hoc sharing; cannot share with external stakeholders; requires admin intervention for every share

### Alternative 2: URL Tokens Only
**Rejected**: Security risk (token leakage); hard to revoke at scale; bypasses RBAC; audit complexity

### Alternative 3: OAuth2 Resource Server
**Rejected**: Over-engineered; Perses has JWT auth; parallel auth system; operational overhead

### Alternative 4: Capability URLs (CapURLs)
**Rejected**: Cannot revoke before expiration; unwieldy URLs; harder to audit

**Why Hybrid Wins**:
- Balances security (RBAC default) with UX (tokens for edge cases)
- Aligns with existing Perses architecture (ephemeral dashboard pattern)
- Minimal new code (reuses RBAC + JWT middleware)
- Admin control prevents abuse (gated token generation)

---

## Implementation Details

### Authentication

**RBAC-based Access** (Primary):
```go
// OpenShift OAuth → JWT → Perses RBAC cache
func (a *Authorization) CheckDashboardAccess(ctx echo.Context, dashboardID string) error {
  user := extractUserFromJWT(ctx)
  project := getDashboardProject(dashboardID)

  // Check in-memory RBAC cache
  if !a.hasPermission(user, v1Role.ReadAction, project, v1Role.DashboardScope) {
    return ErrForbidden
  }
  return nil
}
```

**Token-based Access** (Secondary):
```go
// Ephemeral share token
type DashboardShareToken struct {
  TokenID        string    // UUID (not sequential)
  DashboardID    string
  CreatedBy      string    // User who generated token
  CreatedAt      time.Time
  ExpiresAt      time.Time // Default 24h
  Permissions    []string  // ["read"]
  IPRestrictions []string  // Optional CIDR whitelist
  Revoked        bool
}

// Validation middleware
func ValidateShareToken(tokenID, clientIP string) (*Token, error) {
  token := db.GetToken(tokenID)

  if time.Now().After(token.ExpiresAt) { return nil, ErrExpired }
  if token.Revoked { return nil, ErrRevoked }
  if !matchesCIDR(clientIP, token.IPRestrictions) { return nil, ErrIPDenied }

  return token, nil
}
```

### Authorization

**Perses RBAC Integration**:
```yaml
# Example: Project-scoped dashboard editor
kind: Role
metadata:
  name: dashboard-editor
  project: MyMLProject
spec:
  permissions:
    - actions: ["read", "update", "share"]  # 'share' for token generation
      scopes: ["Dashboard"]

---
kind: RoleBinding
metadata:
  name: team-dashboard-access
  project: MyMLProject
spec:
  role: dashboard-editor
  subjects:
    - kind: User
      name: alice@example.com
```

**Cross-Project Sharing** (Phase 2):
```yaml
# ClusterRole for platform team
kind: GlobalRole
metadata:
  name: platform-dashboard-viewer
spec:
  permissions:
    - actions: ["read"]
      scopes: ["Dashboard"]  # All projects

---
kind: GlobalRoleBinding
metadata:
  name: platform-team-access
spec:
  role: platform-dashboard-viewer
  subjects:
    - kind: User
      name: platform-admin@example.com
```

### Audit Logging

**Event Structure**:
```go
type DashboardAuditEvent struct {
  Timestamp    time.Time              // UTC, millisecond precision
  UserID       string                 // OpenShift user identity
  Action       string                 // "dashboard.share", "dashboard.access.token", etc.
  ResourceType string                 // "Dashboard"
  ResourceID   string                 // Dashboard UUID
  ProjectID    string                 // Namespace
  Result       string                 // "success" | "denied" | "error"
  ClientIP     string                 // Source IP
  UserAgent    string                 // Client UA
  TokenID      *string                // If shared token used
  Metadata     map[string]interface{} // Additional context
}
```

**What Gets Logged**:
- RBAC access: User views dashboard (authenticated via OpenShift OAuth)
- Token generation: User creates share token (logs token ID, expiration, permissions)
- Token access: Anonymous user views dashboard via token (logs token ID, IP, UA)
- Token revocation: User/admin revokes token (logs revoked token ID, reason)

### Security Model

**Configuration**:
```yaml
security:
  dashboard_sharing:
    enabled: true
    token_ttl_default: 24h
    token_ttl_max: 168h  # 7 days
    require_permission: "Dashboard:share"
    allow_ip_restrictions: true
    allow_public_sharing: false  # Admin-gated (future)
    rate_limit:
      token_validation: 100/minute/ip   # Prevent enumeration
      token_generation: 10/hour/user    # Prevent abuse
```

**Security Controls**:
- Rate limiting on token validation endpoint (prevent brute-force)
- UUID tokens (not sequential IDs)
- Token blacklist persisted (revocation immediate)
- Optional IP restrictions (CIDR whitelist)
- TLS 1.3 required (HSTS headers, secure cookie flags)

---

## API Design

### Share Token Endpoints

```http
POST /api/v1/projects/{project}/dashboards/{id}/share
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "expires_in": "24h",          # "24h" | "168h" (max 7d)
  "read_only": true,            # Always true for MVP
  "ip_restrictions": [          # Optional
    "10.0.0.0/8",
    "203.0.113.0/24"
  ]
}

Response 201:
{
  "token_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "share_url": "https://dashboard.openshift.ai/share/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2025-11-15T22:00:00Z"
}
```

```http
DELETE /api/v1/share-tokens/{token_id}
Authorization: Bearer <jwt_token>

Response 204: No Content
```

```http
GET /api/v1/share-tokens
Authorization: Bearer <jwt_token>

Response 200:
{
  "tokens": [
    {
      "token_id": "...",
      "dashboard_id": "...",
      "created_by": "alice@example.com",
      "created_at": "...",
      "expires_at": "...",
      "revoked": false
    }
  ]
}
```

### Dashboard Access with Token

```http
GET /api/v1/dashboards/{id}?share_token={token_id}

# No Authorization header required
# Token validated via middleware
# Returns read-only dashboard data
```

---

## Frontend Integration

```typescript
// Dashboard share dialog component
import { Button, Dialog, TextField, IconButton } from '@mui/material';
import { Share, ContentCopy } from '@mui/icons-material';
import { generateShareToken } from '@/services/api-client';

export function DashboardShareDialog({ dashboardId }: { dashboardId: string }) {
  const [shareUrl, setShareUrl] = useState<string | null>(null);

  const handleGenerateToken = async () => {
    const token = await generateShareToken(dashboardId, {
      expiresIn: '24h',
      readOnly: true,
    });
    setShareUrl(token.shareUrl);
  };

  return (
    <Dialog>
      <DialogTitle>Share Dashboard</DialogTitle>
      <DialogContent>
        <Typography variant="body2">
          Generate a read-only share link that expires in 24 hours.
        </Typography>
        <Button onClick={handleGenerateToken} startIcon={<Share />}>
          Generate Share Link
        </Button>
        {shareUrl && (
          <TextField
            fullWidth
            value={shareUrl}
            InputProps={{
              readOnly: true,
              endAdornment: (
                <IconButton onClick={() => navigator.clipboard.writeText(shareUrl)}>
                  <ContentCopy />
                </IconButton>
              ),
            }}
          />
        )}
      </DialogContent>
    </Dialog>
  );
}
```

---

## Open Questions & Next Steps

### Questions for Product/Security Review

1. **Public Sharing**: Allow anonymous public sharing for customer SLA dashboards?
   - **Recommendation**: Phase 2; requires separate read-only data source with query whitelist

2. **Token TTL**: Allow unlimited TTL for permanent external access?
   - **Recommendation**: No; max 7 days forces periodic re-authorization; use RBAC for permanent

3. **IP Restrictions**: Mandatory or optional?
   - **Recommendation**: Optional; useful for high-security but adds friction

4. **Cross-Project Default**: Opt-in (admin approval) or opt-out (enabled by default)?
   - **Recommendation**: Opt-in for MVP; prevents accidental exposure

### Implementation Phases

**Phase 0 (MVP - 2 weeks)**:
1. Add `Dashboard:share` permission to Perses RBAC
2. Implement share token generation API
3. Build token validation middleware
4. Add audit logging for dashboard access
5. Create share dialog in Perses UI
6. Unit + integration tests (70% coverage)

**Phase 1 (Post-MVP - 1 week)**:
1. Cross-project sharing (ClusterRole pattern)
2. IP restrictions for tokens
3. Token revocation API
4. Admin dashboard for token management
5. E2E tests for sharing workflows

**Phase 2 (Future - 2 weeks)**:
1. Edit permissions for shared dashboards (with versioning)
2. Public sharing with isolated data source
3. iframe embedding support (CSP configuration)
4. Advanced token controls (usage limits, geofencing)

---

## Success Metrics

**Security** (SC-010):
- Zero unauthorized access incidents (target: 100% enforcement)
- All dashboard access logged (target: 100% audit coverage)
- Token leakage detection (monitor for suspicious access patterns)

**User Experience**:
- Time to share dashboard: <30 seconds (generate token → copy URL)
- RBAC access latency: <100ms p95 (in-memory cache)
- Token validation latency: <50ms p95 (PostgreSQL query)

**Adoption**:
- 80% of sharing via RBAC (internal teams)
- 20% of sharing via tokens (external stakeholders)
- <5% token revocations due to security incidents (indicates proper usage)

---

## References

**Perses Codebase**:
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/authorization/` - RBAC implementation
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/pkg/model/api/v1/role/` - Permission model
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/ephemeral-dashboard.md` - Ephemeral pattern (reusable for tokens)

**Industry Best Practices**:
- Grafana: https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/
- Datadog: https://docs.datadoghq.com/account_management/rbac/
- Kubernetes RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- JWT Best Practices: RFC 8725

**Detailed Research**: See `/workspace/sessions/agentic-session-1763159109/workspace/artifacts/specs/001-perses-dashboard/research/dashboard-sharing-access-control.md`

---

**Status**: READY FOR REVIEW
**Next Action**: Stakeholder approval → Phase 0 implementation
**Owner**: Stella (Staff Engineer) - OpenShift AI Platform Team
