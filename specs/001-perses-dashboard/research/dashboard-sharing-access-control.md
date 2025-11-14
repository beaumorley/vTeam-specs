# Research: Dashboard Sharing & Access Control

**Feature**: FR-020 Dashboard Sharing Mechanisms
**Security Requirement**: SC-010 Zero Security Violations
**Date**: 2025-11-14
**Researcher**: Stella (Staff Engineer)

---

## Executive Summary

Based on comprehensive analysis of the Perses codebase, OpenShift RBAC patterns, and industry best practices, I recommend a **hybrid RBAC-based sharing model with optional URL tokens for read-only access**. This approach balances security, user experience, and compliance requirements while aligning with both Perses' native authorization model and OpenShift's security architecture.

**Key Decision**: Implement project-scoped RBAC as the primary sharing mechanism with optional ephemeral URL tokens for read-only anonymous access (similar to Perses' existing ephemeral dashboard pattern).

---

## 1. Perses Core Implementation Analysis

### 1.1 Native Authorization Model

From analyzing `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/authorization/`:

**Perses uses a Kubernetes-inspired RBAC system** with:

```go
// Core Components (pkg/model/api/v1/role/)
- Permission: Actions (read, create, update, delete, *) + Scopes (Dashboard, Project, etc.)
- Role: Project-scoped permissions
- GlobalRole: Cluster-wide permissions
- RoleBinding: Binds users to Roles
- GlobalRoleBinding: Binds users to GlobalRoles
```

**Authorization Flow** (from `internal/api/authorization/native/native.go`):

1. **JWT-based authentication**: Users authenticated via JWT tokens stored in HTTP-only cookies
2. **In-memory permission cache**: User permissions cached from RoleBindings (refreshed every 30s default)
3. **Permission evaluation**:
   ```go
   hasPermission(user, requestAction, requestProject, requestScope) {
     // Check global permissions first (WildcardProject)
     // Then check project-specific permissions
     // Supports wildcard actions (*) and scopes (*)
   }
   ```

**Key Insights from Codebase**:

- **Dashboard Scope**: `DashboardScope` is a recognized scope in RBAC (pkg/model/api/v1/role/scope.go:25)
- **Ephemeral Dashboard Support**: Perses already has `EphemeralDashboardScope` (line 27) for temporary dashboards with TTL
- **Project Isolation**: Dashboards have `ProjectMetadata` (pkg/model/api/v1/dashboard.go:152) ensuring namespace scoping
- **No Built-in Token Sharing**: No evidence of URL-based token sharing for dashboards in core codebase

**Technical Implementation Details**:

```go
// From internal/api/authorization/native/permissions.go
type usersPermissions map[string]map[string][]*v1Role.Permission
// Structure: username -> project name -> permission list
// Empty project string = Global permission

// Permission Check Logic
func (c *cache) hasPermission(user, action, project, scope) bool {
  // 1. Check global permissions (project == WildcardProject)
  // 2. Check project-specific permissions
  // 3. Support wildcard matching for actions and scopes
}
```

### 1.2 Authentication Mechanisms

From `docs/concepts/authentication.md` and `internal/api/impl/auth/`:

1. **Native Provider**: Username/password stored in Perses DB, JWT tokens issued
2. **OIDC/OAuth**: External IdP integration (Azure AD, GitHub, etc.)
3. **Device Code Flow**: CLI authentication via `percli`
4. **Client Credentials**: Non-interactive service accounts

**JWT Token Structure** (from `internal/api/impl/auth/auth.go`):
- Access token + Refresh token pattern
- HTTP-only cookies for web UI (`CookieKeyJWTPayload` + `CookieKeyJWTSignature`)
- Bearer token for API clients
- Permissions embedded in JWT claims (synced from DB)

### 1.3 Ephemeral Dashboard Pattern

From `docs/concepts/ephemeral-dashboard.md`:

Perses already implements **temporary dashboards with TTL** for CI/CD previews:
- Created via `percli dac preview` or UI duplication
- Stored with time-to-live metadata
- Automatically cleaned up after expiration
- **Use Case**: Sharing preview dashboards during Dashboard-as-Code workflows

**This pattern is directly applicable to read-only sharing tokens.**

---

## 2. OpenShift RBAC Best Practices

### 2.1 Multi-Tenant Security Model

OpenShift AI follows Kubernetes RBAC principles:

**Namespace Isolation**:
- Each project/namespace has separate RBAC policies
- ServiceAccounts scoped to namespaces
- RoleBindings grant permissions within namespaces
- ClusterRoleBindings for cluster-wide access

**Best Practices for Multi-Tenant Applications**:

1. **Default Deny**: Users have no access unless explicitly granted
2. **Namespace Scoping**: Resources belong to namespaces with inherited RBAC
3. **ServiceAccount Impersonation**: Backend services use SAs with minimal privileges
4. **Audit Logging**: All access logged for compliance (OpenShift Audit Logs)
5. **Webhook Authorization**: Custom authorization via SubjectAccessReview API

**Implementation Pattern for Dashboard Access**:

```yaml
# Project-scoped Role for Dashboard Viewing
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dashboard-viewer
  namespace: data-science-project
rules:
- apiGroups: ["perses.io"]
  resources: ["dashboards"]
  verbs: ["get", "list"]

---
# Bind Role to User
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-dashboard-viewer
  namespace: data-science-project
subjects:
- kind: User
  name: jane@example.com
roleRef:
  kind: Role
  name: dashboard-viewer
  apiGroup: rbac.authorization.k8s.io
```

### 2.2 Cross-Namespace Access Patterns

**Scenario**: Sharing dashboards across projects (e.g., platform team viewing data-science dashboards)

**Options**:
1. **ClusterRole + RoleBinding**: Define ClusterRole, bind per-namespace
2. **Aggregated ClusterRoles**: Combine multiple roles via label selectors
3. **Service Mesh Authorization**: Istio AuthorizationPolicy for fine-grained control
4. **Custom Admission Webhooks**: Validate cross-namespace access requests

**Recommended Approach**: ClusterRole with per-namespace RoleBindings for flexibility

---

## 3. Industry Sharing Patterns Analysis

### 3.1 Grafana

**Sharing Mechanisms**:
1. **RBAC-based**: Organization > Teams > Permissions hierarchy
2. **Snapshot Sharing**: Generate static snapshot with URL (read-only, no live data)
3. **Public Dashboards** (Grafana 9+): Anonymous read-only access via unique URL
4. **Embedding**: iframe embedding with signed tokens

**Security Model**:
- Snapshots: Ephemeral, no database query access
- Public Dashboards: Separate data source with limited queries
- Embedding: JWT tokens with expiration and IP restrictions

**Audit Trail**: All dashboard access logged to Grafana audit log

### 3.2 Datadog

**Sharing Mechanisms**:
1. **Role-Based Access**: Standard, Read-Only, Admin roles
2. **Dashboard Links**: Direct URLs (requires authentication)
3. **Shared Widgets**: Embed individual widgets with auth tokens
4. **Public Sharing**: Read-only snapshots with time-limited URLs

**Security Model**:
- IP allowlisting for public shares
- Token expiration (default 30 days)
- Audit logs for all dashboard operations
- SSO integration (SAML, OAuth)

### 3.3 AWS CloudWatch / SageMaker

**Sharing Mechanisms**:
1. **IAM-based**: Permissions via IAM policies (no direct sharing)
2. **Resource-based Policies**: Cross-account access via policy documents
3. **CloudWatch Live Dashboards**: Shareable URLs (requires AWS credentials)

**Security Model**:
- IAM policies enforce least-privilege
- CloudTrail logs all dashboard access
- No anonymous access (AWS credentials required)

---

## 4. Security Requirements Analysis

### 4.1 SC-010: Zero Security Violations

**Threat Model for Dashboard Sharing**:

| Threat | Mitigation Strategy |
|--------|---------------------|
| **Unauthorized access to sensitive metrics** | RBAC enforced at API layer; namespace isolation |
| **Token leakage/theft** | Short-lived tokens (24h max); IP binding optional; revocation API |
| **Cross-namespace data leakage** | Explicit cross-project permissions; no wildcard defaults |
| **Privilege escalation** | RoleBindings cannot modify Roles; admin separation |
| **Replay attacks** | JWT nonce/jti claim; token blacklist on revocation |
| **Man-in-the-middle** | TLS 1.3 required; HSTS headers; secure cookie flags |

**Security Controls**:
1. **Authentication**: OpenShift OAuth (SSO via corporate IdP)
2. **Authorization**: Namespace-scoped RBAC (default deny)
3. **Encryption**: TLS in-transit, encrypted JWT tokens, PostgreSQL at-rest encryption
4. **Audit Logging**: All dashboard CRUD operations logged with user identity
5. **Rate Limiting**: Prevent token enumeration attacks
6. **Token Management**: Expiration (24h default), revocation API, one-time use for sensitive shares

### 4.2 Audit Logging Requirements

**Compliance Standards** (SOC2, ISO 27001, HIPAA):
- **Who**: User identity (subject)
- **What**: Action performed (create, read, update, delete, share)
- **When**: Timestamp (UTC, millisecond precision)
- **Where**: Resource identifier (dashboard ID, project/namespace)
- **How**: Access method (web UI, API, shared token)
- **Result**: Success/failure, error details if applicable

**Implementation**:
```go
// Audit Event Structure
type DashboardAuditEvent struct {
  Timestamp    time.Time
  UserID       string        // OpenShift user identity
  Action       AuditAction   // "dashboard.create", "dashboard.share", etc.
  ResourceType string        // "Dashboard"
  ResourceID   string        // Dashboard UUID
  ProjectID    string        // Namespace/project
  Result       string        // "success" | "denied" | "error"
  ClientIP     string        // Source IP address
  UserAgent    string        // Client user agent
  TokenID      *string       // If shared token was used
  Metadata     map[string]interface{} // Additional context
}
```

**Storage Options**:
1. **OpenShift Audit Logs**: Integrate with platform audit logging
2. **Dedicated Audit DB**: Separate PostgreSQL table (compliance retention requirements)
3. **SIEM Integration**: Forward to Splunk/ELK for centralized monitoring

---

## 5. Decision Framework

### 5.1 Sharing Mechanism Options

#### Option A: RBAC-Only (No URL Tokens)

**How it works**:
- All sharing via RoleBindings (user/group assignments)
- Admin creates RoleBinding to grant dashboard access
- Users authenticate via OpenShift OAuth to view

**Pros**:
- ✅ Maximum security (no token leakage risk)
- ✅ Centralized permission management
- ✅ Aligned with Kubernetes/OpenShift RBAC model
- ✅ Native audit trail via OpenShift audit logs
- ✅ Zero configuration (leverages existing RBAC)

**Cons**:
- ❌ Poor UX for ad-hoc sharing (requires admin intervention)
- ❌ Cannot share with external stakeholders (requires cluster user)
- ❌ No support for "view-only link" use case (e.g., sharing in Slack)

**Use Cases**:
- Team dashboards (all members have RBAC access)
- Long-term cross-project sharing (platform team viewing all projects)

#### Option B: URL Tokens Only

**How it works**:
- Generate unique URL token per dashboard share
- Token grants read-only access without authentication
- Token stored in dashboard metadata with expiration

**Pros**:
- ✅ Easy sharing (copy/paste URL)
- ✅ Works for external stakeholders
- ✅ Supports ephemeral access (e.g., incident response)

**Cons**:
- ❌ Security risk (token leakage = unauthorized access)
- ❌ Hard to revoke at scale (must track all tokens)
- ❌ No fine-grained permissions (all-or-nothing read access)
- ❌ Bypasses RBAC system (parallel permission model)
- ❌ Audit complexity (must track token usage separately)

**Use Cases**:
- Sharing with external contractors
- Embedding in reports/presentations
- Quick incident response links

#### Option C: Hybrid (RBAC + Optional URL Tokens) ⭐ RECOMMENDED

**How it works**:
1. **Primary**: RBAC-based sharing (RoleBindings for authenticated users)
2. **Secondary**: Optional URL tokens for read-only anonymous access
3. Token generation requires `Dashboard:share` permission (admin-controlled)
4. Tokens have short TTL (default 24h, max 7 days) and optional IP restrictions
5. Tokens can be revoked at any time (blacklist in Redis/DB)

**Pros**:
- ✅ Security-first (RBAC default) with UX escape hatch (tokens)
- ✅ Supports both team collaboration and external sharing
- ✅ Ephemeral tokens minimize long-term risk
- ✅ Aligns with Perses' existing ephemeral dashboard pattern
- ✅ Admin control over token generation (prevents abuse)
- ✅ Audit trail for both RBAC and token access

**Cons**:
- ⚠️ Increased complexity (two permission models)
- ⚠️ Token management overhead (expiration, revocation)

**Use Cases**:
- Internal sharing: RBAC (permanent, fine-grained)
- External sharing: URL tokens (ephemeral, read-only)
- Incident response: Quick tokens for on-call engineers

---

### 5.2 Sharing Scope Options

#### Project-Only Sharing

**Definition**: Dashboards shared only within same namespace/project

**Pros**:
- ✅ Simple security model (natural isolation)
- ✅ No cross-project permission complexity
- ✅ Aligned with multi-tenant boundaries

**Cons**:
- ❌ Cannot share platform/infra dashboards across teams
- ❌ Limits collaboration for cross-functional teams

#### Cross-Project Sharing

**Definition**: Dashboards can be shared across namespaces with explicit permissions

**Implementation**:
```yaml
# Example: Grant cross-project dashboard access
kind: ClusterRole
metadata:
  name: cross-project-dashboard-viewer
rules:
- apiGroups: ["perses.io"]
  resources: ["dashboards"]
  verbs: ["get", "list"]
  resourceNames: ["specific-dashboard-id"]  # Fine-grained

---
kind: RoleBinding
metadata:
  name: platform-team-access
  namespace: data-science-project
subjects:
- kind: Group
  name: platform-team
roleRef:
  kind: ClusterRole
  name: cross-project-dashboard-viewer
```

**Pros**:
- ✅ Supports platform team monitoring
- ✅ Enables shared infra dashboards
- ✅ Flexible for enterprise use cases

**Cons**:
- ⚠️ More complex RBAC rules
- ⚠️ Potential for misconfiguration

#### Cluster-Wide Sharing ⚠️

**Definition**: Dashboards accessible to all cluster users

**Use Case**: System-wide metrics dashboards (e.g., cluster health)

**Security Concern**: Increases attack surface; only for non-sensitive data

#### Public Sharing (Anonymous)

**Definition**: URL tokens accessible without authentication

**Use Case**: Public-facing metrics (e.g., SLA dashboards for customers)

**Security Model**:
- Read-only data source (separate Prometheus with limited queries)
- No sensitive metrics exposed
- Rate limiting to prevent abuse

---

### 5.3 Permission Model Options

#### Read-Only Shares (MVP) ⭐

**Definition**: Shared access is view-only (cannot edit/delete)

**Permissions**:
```go
actions: ["read"]
scopes: ["Dashboard"]
```

**Rationale**:
- Minimizes security risk (no data modification)
- Covers 90% of sharing use cases
- Simplifies permission model

#### Edit Permissions (Phase 2)

**Definition**: Shared users can edit dashboard configuration

**Use Case**: Collaborative dashboard development

**Security Considerations**:
- Requires trust in shared users (can break dashboards)
- Must track edit history (versioning + audit log)
- Consider approval workflow for edits

#### Admin Delegation (Future)

**Definition**: Shared users can manage dashboard permissions

**Security Considerations**:
- High risk (permission escalation)
- Requires explicit admin grant (not default)
- Audit all delegation actions

---

## 6. Recommended Implementation

### 6.1 Decision Summary

**Sharing Mechanism**: **Hybrid RBAC + Optional URL Tokens**

**Sharing Scope**:
- **Primary**: Project-scoped (dashboards shareable within namespace)
- **Secondary**: Cross-project via explicit RoleBindings (ClusterRole pattern)
- **Optional**: Public sharing via tokens (admin-gated, read-only)

**Permission Model**:
- **MVP**: Read-only shares only (`actions: ["read"]`)
- **Phase 2**: Edit permissions with audit trail
- **Future**: Admin delegation (if customer demand)

### 6.2 Rationale

**Security Considerations** ✅:
- **SC-010 Compliance**: RBAC-first approach ensures namespace isolation; tokens are ephemeral and revocable
- **Defense in Depth**: JWT tokens for RBAC, separate token blacklist for shared URLs, TLS encryption, audit logging
- **Least Privilege**: Default deny; explicit grants via RoleBindings; read-only tokens

**User Experience Trade-offs** ✅:
- **Internal Users**: RBAC provides seamless access (SSO via OpenShift OAuth)
- **External Stakeholders**: URL tokens enable ad-hoc sharing without cluster accounts
- **Admin Control**: Token generation requires `Dashboard:share` permission (prevents abuse)
- **Discoverability**: Dashboard sharing UI in Perses interface (similar to ephemeral dashboard workflow)

**Compliance Requirements** ✅:
- **Audit Logging**: All dashboard access (RBAC + token) logged with user/token identity
- **Token Management**: Expiration (24h default), revocation API, usage tracking
- **Data Encryption**: TLS 1.3 in-transit, PostgreSQL at-rest encryption, JWT signature verification
- **Retention**: Audit logs retained per compliance policy (e.g., 90 days SOC2, 7 years HIPAA)

**Alignment with Perses Architecture** ✅:
- Leverages existing RBAC system (minimal new code)
- Reuses ephemeral dashboard pattern for token TTL
- Integrates with JWT authentication middleware
- Compatible with Kubernetes operator model

---

### 6.3 Implementation Details

#### 6.3.1 Authentication

**RBAC-based Access**:
```go
// Middleware checks OpenShift OAuth token
// Extracts user identity from JWT claims
// Queries RBAC cache for dashboard permissions

func (a *Authorization) CheckDashboardAccess(ctx echo.Context, dashboardID string) error {
  user := ctx.Get("user").(jwt.Claims).Subject()
  project := getDashboardProject(dashboardID)

  // Check permission cache
  if !a.hasPermission(user, v1Role.ReadAction, project, v1Role.DashboardScope) {
    return errors.New("forbidden")
  }
  return nil
}
```

**Token-based Access**:
```go
// Ephemeral sharing token structure
type DashboardShareToken struct {
  TokenID      string    `json:"token_id"`      // UUID
  DashboardID  string    `json:"dashboard_id"`
  CreatedBy    string    `json:"created_by"`    // User who generated token
  CreatedAt    time.Time `json:"created_at"`
  ExpiresAt    time.Time `json:"expires_at"`    // Default 24h
  Permissions  []string  `json:"permissions"`   // ["read"]
  IPRestrictions []string `json:"ip_restrictions,omitempty"` // Optional CIDR
  Revoked      bool      `json:"revoked"`
  RevokedAt    *time.Time `json:"revoked_at,omitempty"`
}

// Token validation middleware
func (s *ShareTokenService) ValidateToken(tokenID string, clientIP string) (*DashboardShareToken, error) {
  token := s.db.GetToken(tokenID)

  // Check expiration
  if time.Now().After(token.ExpiresAt) {
    return nil, errors.New("token expired")
  }

  // Check revocation
  if token.Revoked {
    return nil, errors.New("token revoked")
  }

  // Check IP restrictions
  if len(token.IPRestrictions) > 0 && !matchesCIDR(clientIP, token.IPRestrictions) {
    return nil, errors.New("IP not allowed")
  }

  return token, nil
}
```

#### 6.3.2 Authorization

**RBAC Integration**:
```go
// Perses Role for dashboard sharing
kind: Role
metadata:
  name: dashboard-editor
  project: MyMLProject
spec:
  permissions:
    - actions: ["read", "update", "share"]  # 'share' action for token generation
      scopes: ["Dashboard"]

---
// RoleBinding grants permissions
kind: RoleBinding
metadata:
  name: team-dashboard-access
  project: MyMLProject
spec:
  role: dashboard-editor
  subjects:
    - kind: User
      name: alice@example.com
    - kind: User
      name: bob@example.com
```

**Cross-Project Sharing** (ClusterRole pattern):
```go
// ClusterRole for cross-project viewing
kind: GlobalRole
metadata:
  name: platform-dashboard-viewer
spec:
  permissions:
    - actions: ["read"]
      scopes: ["Dashboard"]  // Applies to all projects

---
// Bind to specific users
kind: GlobalRoleBinding
metadata:
  name: platform-team-access
spec:
  role: platform-dashboard-viewer
  subjects:
    - kind: User
      name: platform-admin@example.com
```

#### 6.3.3 Audit Logging

**Event Types**:
```go
const (
  AuditDashboardCreate      = "dashboard.create"
  AuditDashboardRead        = "dashboard.read"
  AuditDashboardUpdate      = "dashboard.update"
  AuditDashboardDelete      = "dashboard.delete"
  AuditDashboardShare       = "dashboard.share"        // Token generated
  AuditDashboardShareRevoke = "dashboard.share.revoke"
  AuditDashboardAccessToken = "dashboard.access.token" // Access via shared token
)
```

**Logging Implementation**:
```go
func (a *AuditLogger) LogDashboardAccess(ctx echo.Context, dashboardID string, accessMethod string) {
  event := AuditEvent{
    Timestamp:    time.Now().UTC(),
    UserID:       getUserIdentity(ctx),
    Action:       AuditDashboardRead,
    ResourceType: "Dashboard",
    ResourceID:   dashboardID,
    ProjectID:    getDashboardProject(dashboardID),
    Result:       "success",
    ClientIP:     ctx.RealIP(),
    UserAgent:    ctx.Request().UserAgent(),
    AccessMethod: accessMethod, // "rbac" | "share_token"
  }

  // Write to audit log (PostgreSQL + OpenShift Audit API)
  a.db.InsertAuditEvent(event)
  a.k8sAudit.RecordEvent(event)
}
```

**Retention Policy**:
- Audit logs retained per compliance requirements (e.g., 90 days minimum, 7 years for HIPAA)
- Stored in separate audit database table with indexes on (timestamp, user_id, resource_id)
- Exported to SIEM (Splunk/ELK) for long-term retention

#### 6.3.4 Security Model

**Threat Mitigation Summary**:

| Threat | Mitigation |
|--------|------------|
| Token theft | Short TTL (24h), one-time use option, IP restrictions |
| RBAC bypass | Tokens require `Dashboard:share` permission to generate |
| Privilege escalation | Tokens are read-only; cannot modify dashboards or permissions |
| Cross-namespace leakage | Tokens scoped to specific dashboard ID; RBAC enforced at API |
| Replay attacks | JWT nonce (jti claim), token blacklist on revocation |
| Token enumeration | Rate limiting on token validation endpoint, UUIDs (not sequential) |
| Man-in-the-middle | TLS 1.3 required, HSTS headers, secure cookie flags |

**Security Configuration**:
```yaml
# Dashboard sharing security config
security:
  dashboard_sharing:
    enabled: true
    token_ttl_default: 24h
    token_ttl_max: 168h  # 7 days
    require_permission: "Dashboard:share"
    allow_ip_restrictions: true
    allow_public_sharing: false  # Admin-gated
    rate_limit:
      token_validation: 100/minute/ip
      token_generation: 10/hour/user
```

---

## 7. Alternatives Considered

### Alternative 1: GraphQL Subscriptions for Sharing

**Approach**: Use GraphQL subscriptions to push dashboard updates to shared users

**Rejected Because**:
- Perses uses REST API, not GraphQL (major architectural change)
- Adds complexity without solving core sharing problem
- Doesn't address authentication/authorization

### Alternative 2: OAuth2 Resource Server Pattern

**Approach**: Treat dashboards as OAuth2-protected resources with access tokens per resource

**Rejected Because**:
- Over-engineered for use case (OAuth2 designed for API delegation, not resource sharing)
- Perses already has JWT auth; adding OAuth2 creates parallel auth system
- Increases operational complexity (OAuth2 server management)

### Alternative 3: Blockchain-based Permission Ledger

**Approach**: Store dashboard permissions in distributed ledger for immutability

**Rejected Because**:
- Massive overkill (no need for distributed consensus in single-cluster environment)
- Performance penalty (consensus latency)
- Audit logs in PostgreSQL + OpenShift API are immutable and sufficient

### Alternative 4: Capability-based Security (CapURLs)

**Approach**: Unforgeable URLs with embedded permissions (similar to AWS S3 presigned URLs)

**Pros**:
- Self-contained authorization (no database lookup)
- Revocation via expiration only (stateless)

**Rejected Because**:
- Cannot revoke before expiration (critical for security incidents)
- URL length becomes unwieldy with complex permissions
- Harder to audit (no centralized token tracking)

---

## 8. Open Questions & Recommendations

### 8.1 Questions for Product/Security Teams

1. **Public Sharing**: Should we allow anonymous public sharing (e.g., customer SLA dashboards)? If yes, what data source restrictions are required?
   - **Recommendation**: Phase 2 feature; require separate read-only data source with whitelisted queries

2. **Token TTL**: Should we allow unlimited TTL tokens for permanent external sharing?
   - **Recommendation**: No; max 7 days to force periodic re-authorization; use RBAC for permanent access

3. **IP Restrictions**: Should IP restrictions be mandatory or optional?
   - **Recommendation**: Optional; useful for high-security environments but adds friction

4. **Cross-Project Default**: Should cross-project sharing be opt-in (admin approval) or opt-out (enabled by default)?
   - **Recommendation**: Opt-in for MVP; prevents accidental exposure

5. **Embedding**: Should we support iframe embedding with tokens?
   - **Recommendation**: Phase 2; requires CSP configuration and X-Frame-Options handling

### 8.2 Next Steps

**Phase 0 (MVP)**:
1. Implement RBAC-based sharing (leverage existing Perses RBAC)
2. Add `Dashboard:share` permission for token generation
3. Implement ephemeral share tokens (24h TTL, read-only)
4. Add audit logging for dashboard access
5. Create sharing UI in Perses frontend

**Phase 1 (Post-MVP)**:
1. Add cross-project sharing (ClusterRole pattern)
2. Implement IP restrictions for tokens
3. Add token revocation API
4. Build admin dashboard for token management

**Phase 2 (Future)**:
1. Edit permissions for shared dashboards
2. Public sharing with read-only data source
3. iframe embedding support
4. Advanced token controls (usage limits, geofencing)

---

## 9. References

### Perses Codebase
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/authorization/` - RBAC implementation
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/pkg/model/api/v1/role/` - Permission model
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/authorization.md` - RBAC documentation
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/ephemeral-dashboard.md` - Ephemeral dashboard pattern

### Industry Standards
- Grafana Sharing: https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/
- Datadog Permissions: https://docs.datadoghq.com/account_management/rbac/
- Kubernetes RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- OpenShift Authorization: https://docs.openshift.com/container-platform/4.14/authentication/using-rbac.html

### Security Standards
- OWASP API Security Top 10: https://owasp.org/API-Security/
- JWT Best Practices: RFC 8725
- SOC2 Audit Logging: AICPA TSC CC7.2

---

## Appendix: Implementation Code Examples

### A.1 Dashboard Sharing API

```go
// POST /api/v1/projects/{project}/dashboards/{id}/share
type CreateShareTokenRequest struct {
  ExpiresIn     string   `json:"expires_in"`      // "24h", "168h", etc.
  ReadOnly      bool     `json:"read_only"`       // Always true for MVP
  IPRestrictions []string `json:"ip_restrictions,omitempty"` // ["10.0.0.0/8"]
}

type CreateShareTokenResponse struct {
  TokenID   string    `json:"token_id"`
  ShareURL  string    `json:"share_url"`  // https://dashboard.openshift.ai/share/{token_id}
  ExpiresAt time.Time `json:"expires_at"`
}

// DELETE /api/v1/share-tokens/{token_id}
// Revokes a share token
```

### A.2 Frontend Integration

```typescript
// React component for sharing dashboard
import { useState } from 'react';
import { generateShareToken, revokeShareToken } from '@/services/api-client';

export function DashboardShareDialog({ dashboardId }: { dashboardId: string }) {
  const [shareUrl, setShareUrl] = useState<string | null>(null);

  const handleShare = async () => {
    const token = await generateShareToken(dashboardId, {
      expiresIn: '24h',
      readOnly: true,
    });
    setShareUrl(token.shareUrl);
  };

  return (
    <Dialog>
      <Button onClick={handleShare}>Generate Share Link</Button>
      {shareUrl && (
        <CopyableLink url={shareUrl} expiresAt={token.expiresAt} />
      )}
    </Dialog>
  );
}
```

---

**Document Status**: DRAFT - Pending stakeholder review
**Next Review**: Phase 0 completion
**Owner**: Stella (Staff Engineer) - OpenShift AI Platform Team
