# Dashboard Sharing Architecture Diagram

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OpenShift AI Platform                            │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Perses Dashboard UI                         │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │   │
│  │  │  Dashboard  │  │  Dashboard   │  │   Share Dialog        │  │   │
│  │  │  List View  │  │  Viewer      │  │  [Generate Token]     │  │   │
│  │  └─────────────┘  └──────────────┘  └───────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                 ▲                                         │
│                                 │ HTTPS (TLS 1.3)                        │
│                                 ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Perses Backend API                            │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │           Authentication Middleware                       │  │   │
│  │  │  ┌──────────────────┐     ┌──────────────────────────┐  │  │   │
│  │  │  │ OpenShift OAuth  │  OR │  Share Token Validator   │  │  │   │
│  │  │  │  (JWT Cookies)   │     │  (UUID Token in Query)   │  │  │   │
│  │  │  └──────────────────┘     └──────────────────────────┘  │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │           Authorization Middleware                        │  │   │
│  │  │  ┌──────────────────┐     ┌──────────────────────────┐  │  │   │
│  │  │  │  RBAC Cache      │     │  Token Blacklist         │  │  │   │
│  │  │  │  (In-Memory)     │     │  (PostgreSQL)            │  │  │   │
│  │  │  └──────────────────┘     └──────────────────────────┘  │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │              API Endpoints                                │  │   │
│  │  │  • GET    /api/v1/dashboards/{id}                        │  │   │
│  │  │  • POST   /api/v1/projects/{p}/dashboards/{id}/share    │  │   │
│  │  │  • DELETE /api/v1/share-tokens/{token_id}               │  │   │
│  │  │  • GET    /api/v1/share-tokens (list active tokens)     │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                 │                                         │
│                                 ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      PostgreSQL Database                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │   │
│  │  │ Dashboards   │  │ Share Tokens │  │  Audit Events        │  │   │
│  │  │ (Configs)    │  │ (Active/Rev) │  │  (Compliance Logs)   │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    OpenShift RBAC System                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │   │
│  │  │ Roles        │  │ RoleBindings │  │  Users/ServiceAccts  │  │   │
│  │  │ (Permissions)│  │ (User→Role)  │  │  (Identities)        │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Access Flow Comparison

### Flow 1: RBAC-Based Access (Internal Users)

```
┌──────────┐                                                ┌──────────────┐
│  User    │──(1) Login via OpenShift OAuth───────────────▶│  OAuth IdP   │
│  Alice   │                                                │  (Azure AD)  │
└──────────┘                                                └──────────────┘
     │                                                             │
     │◀─────────(2) JWT Access Token (HTTP-only cookies)──────────┘
     │
     │         (3) GET /api/v1/dashboards/{id}
     │             Authorization: Bearer {jwt_token}
     ▼
┌──────────────────────────────────────────────────────┐
│            Perses Backend API                        │
│  ┌────────────────────────────────────────────────┐ │
│  │  Authentication Middleware                     │ │
│  │  • Verify JWT signature (HS512)                │ │
│  │  • Extract user identity: alice@example.com    │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  ┌────────────────────────────────────────────────┐ │
│  │  Authorization Middleware                      │ │
│  │  • Lookup user permissions in RBAC cache       │ │
│  │  • Check: hasPermission(alice, "read",         │ │
│  │           "MyMLProject", "Dashboard")          │ │
│  │  • Result: ✅ ALLOWED (via RoleBinding)        │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  ┌────────────────────────────────────────────────┐ │
│  │  Audit Logger                                  │ │
│  │  Event: dashboard.read                         │ │
│  │  User: alice@example.com                       │ │
│  │  Method: rbac                                  │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│         Return dashboard JSON                        │
└──────────────────────────────────────────────────────┘
     │
     │◀─────(4) Dashboard data
     ▼
┌──────────┐
│  User    │  Dashboard rendered in browser
│  Alice   │
└──────────┘
```

### Flow 2: Token-Based Access (External Users)

```
┌──────────┐
│  Alice   │──(1) Click "Share" button in UI
│ (Author) │
└──────────┘
     │
     │         (2) POST /api/v1/projects/MyMLProject/dashboards/123/share
     │             Authorization: Bearer {alice_jwt_token}
     │             Body: { "expires_in": "24h", "read_only": true }
     ▼
┌──────────────────────────────────────────────────────┐
│            Perses Backend API                        │
│  ┌────────────────────────────────────────────────┐ │
│  │  Authorization Check                           │ │
│  │  • Verify Alice has "Dashboard:share" perm     │ │
│  │  • Result: ✅ ALLOWED                          │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  ┌────────────────────────────────────────────────┐ │
│  │  Token Generator                               │ │
│  │  • Generate UUID: a1b2c3d4-e5f6-...            │ │
│  │  • Set expiration: now + 24h                   │ │
│  │  • Store in PostgreSQL                         │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  ┌────────────────────────────────────────────────┐ │
│  │  Audit Logger                                  │ │
│  │  Event: dashboard.share                        │ │
│  │  User: alice@example.com                       │ │
│  │  TokenID: a1b2c3d4-...                         │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  Return: { "share_url": "https://.../share/a1b2..." }
└──────────────────────────────────────────────────────┘
     │
     │◀─────(3) Share URL
     ▼
┌──────────┐
│  Alice   │──(4) Send URL to Bob (email/Slack)
└──────────┘       share_url = https://dashboard.openshift.ai/share/a1b2c3d4...
                        │
                        │
                        ▼
                   ┌──────────┐
                   │   Bob    │──(5) Click share URL (no auth required)
                   │(External)│
                   └──────────┘
                        │
                        │  GET /api/v1/dashboards/123?share_token=a1b2c3d4...
                        ▼
┌──────────────────────────────────────────────────────┐
│            Perses Backend API                        │
│  ┌────────────────────────────────────────────────┐ │
│  │  Token Validation Middleware                   │ │
│  │  • Lookup token in PostgreSQL                  │ │
│  │  • Check expiration: ✅ Valid (< 24h)          │ │
│  │  • Check revoked: ✅ Not revoked               │ │
│  │  • Check IP: ✅ Matches CIDR (if configured)   │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  ┌────────────────────────────────────────────────┐ │
│  │  Audit Logger                                  │ │
│  │  Event: dashboard.access.token                 │ │
│  │  TokenID: a1b2c3d4-...                         │ │
│  │  ClientIP: 203.0.113.42                        │ │
│  │  User: anonymous (token-based)                 │ │
│  └────────────────────────────────────────────────┘ │
│                     ▼                                │
│  Return dashboard JSON (read-only)                   │
└──────────────────────────────────────────────────────┘
     │
     │◀─────(6) Dashboard data
     ▼
┌──────────┐
│   Bob    │  Dashboard rendered (read-only mode)
│(External)│
└──────────┘
```

## RBAC Permission Model

```
┌─────────────────────────────────────────────────────────────────┐
│                      Perses RBAC Hierarchy                       │
│                                                                   │
│  ┌─────────────────┐           ┌─────────────────────────────┐ │
│  │   GlobalRole    │           │  GlobalRoleBinding          │ │
│  ├─────────────────┤           ├─────────────────────────────┤ │
│  │ • Name          │           │ • Role: GlobalRole name     │ │
│  │ • Permissions[] │───────────│ • Subjects: [User|Group]    │ │
│  │   - Actions     │           └─────────────────────────────┘ │
│  │   - Scopes      │                                            │
│  └─────────────────┘           Example:                         │
│        │                       - Role: platform-admin           │
│        │                       - Subjects: [alice@example.com]  │
│        │  Applies to ALL projects                               │
│        ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Project: MyMLProject                                    │   │
│  │  ┌─────────────┐         ┌───────────────────────────┐  │   │
│  │  │    Role     │         │     RoleBinding           │  │   │
│  │  ├─────────────┤         ├───────────────────────────┤  │   │
│  │  │ • Project   │         │ • Project: MyMLProject    │  │   │
│  │  │ • Name      │─────────│ • Role: dashboard-editor  │  │   │
│  │  │ • Perms[]   │         │ • Subjects: [bob@...]     │  │   │
│  │  └─────────────┘         └───────────────────────────┘  │   │
│  │                                                           │   │
│  │  Scoped to MyMLProject only                              │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

Permission Evaluation:
hasPermission(user, action, project, scope) {
  1. Check GlobalRoleBindings (applies to all projects)
  2. Check RoleBindings (project-specific)
  3. Return true if ANY binding grants permission
}
```

## Permission Examples

### Example 1: Dashboard Viewer (Read-Only)

```yaml
kind: Role
metadata:
  name: dashboard-viewer
  project: MyMLProject
spec:
  permissions:
    - actions: ["read"]
      scopes: ["Dashboard"]

---
kind: RoleBinding
metadata:
  name: team-viewer
  project: MyMLProject
spec:
  role: dashboard-viewer
  subjects:
    - kind: User
      name: viewer@example.com
```

**Grants**: Read dashboards in MyMLProject
**Denies**: Edit, delete, share dashboards

### Example 2: Dashboard Editor (Create + Share)

```yaml
kind: Role
metadata:
  name: dashboard-editor
  project: MyMLProject
spec:
  permissions:
    - actions: ["read", "create", "update", "share"]
      scopes: ["Dashboard"]

---
kind: RoleBinding
metadata:
  name: team-editor
  project: MyMLProject
spec:
  role: dashboard-editor
  subjects:
    - kind: User
      name: alice@example.com
```

**Grants**: Read, create, edit dashboards; generate share tokens
**Denies**: Delete dashboards (requires "delete" action)

### Example 3: Platform Admin (All Projects)

```yaml
kind: GlobalRole
metadata:
  name: platform-admin
spec:
  permissions:
    - actions: ["*"]  # All actions
      scopes: ["*"]   # All scopes (Dashboard, Project, User, etc.)

---
kind: GlobalRoleBinding
metadata:
  name: admin-access
spec:
  role: platform-admin
  subjects:
    - kind: User
      name: admin@example.com
```

**Grants**: Full access to all resources in all projects
**Use Case**: Platform administrators

## Token Lifecycle

```
┌────────────────────────────────────────────────────────────────┐
│                    Share Token Lifecycle                        │
│                                                                  │
│  ┌──────────┐   Generate Token   ┌──────────┐                  │
│  │  User    │──────────────────▶│  Active  │                  │
│  │  (Alice) │                    │  Token   │                  │
│  └──────────┘                    └──────────┘                  │
│       │                                │                        │
│       │                                │                        │
│       │                                │  Token used for access │
│       │                                ▼                        │
│       │                          ┌──────────┐                  │
│       │                          │  Valid   │                  │
│       │                          │ (< 24h)  │                  │
│       │                          └──────────┘                  │
│       │                                │                        │
│       │  Manual Revocation             │  Expiration           │
│       │  (security incident)           │  (after 24h)          │
│       ▼                                ▼                        │
│  ┌──────────┐                    ┌──────────┐                  │
│  │ Revoked  │                    │ Expired  │                  │
│  │  Token   │                    │  Token   │                  │
│  └──────────┘                    └──────────┘                  │
│       │                                │                        │
│       │                                │                        │
│       └────────────┬───────────────────┘                        │
│                    │                                             │
│                    ▼                                             │
│              ┌──────────┐                                       │
│              │ Blacklist│  (persisted forever for audit)       │
│              │  (DB)    │                                       │
│              └──────────┘                                       │
│                                                                  │
│  Token States:                                                  │
│  • Active: Can be used to access dashboard                     │
│  • Expired: Past expiration time (24h default)                 │
│  • Revoked: Manually revoked by user/admin                     │
│  • Blacklist: Persisted record of revoked/expired tokens       │
└────────────────────────────────────────────────────────────────┘
```

## Security Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Defense in Depth                              │
│                                                                   │
│  Layer 1: Transport Security                                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ • TLS 1.3 (enforced)                                        │ │
│  │ • HSTS headers (Strict-Transport-Security)                 │ │
│  │ • Secure cookie flags (HttpOnly, Secure, SameSite=Strict) │ │
│  └────────────────────────────────────────────────────────────┘ │
│                             ▼                                     │
│  Layer 2: Authentication                                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ RBAC Path:                  Token Path:                     │ │
│  │ • OpenShift OAuth           • UUID validation               │ │
│  │ • JWT signature (HS512)     • Expiration check              │ │
│  │ • Cookie verification       • Revocation check              │ │
│  └────────────────────────────────────────────────────────────┘ │
│                             ▼                                     │
│  Layer 3: Authorization                                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ RBAC Path:                  Token Path:                     │ │
│  │ • Permission cache lookup   • Token permissions check      │ │
│  │ • Project isolation         • Dashboard scope validation   │ │
│  │ • Action/scope matching     • Read-only enforcement        │ │
│  │                             • Optional IP restrictions      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                             ▼                                     │
│  Layer 4: Rate Limiting                                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ • Token validation: 100/min/IP (prevent enumeration)       │ │
│  │ • Token generation: 10/hour/user (prevent abuse)           │ │
│  │ • API requests: 1000/min/user (prevent DoS)                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                             ▼                                     │
│  Layer 5: Audit Logging                                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ • All access logged (who, what, when, where, how)          │ │
│  │ • PostgreSQL audit table + OpenShift Audit API             │ │
│  │ • SIEM integration (Splunk/ELK)                            │ │
│  │ • Compliance retention (90d default, 7y HIPAA)             │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

**Diagram Status**: DRAFT - For stakeholder review
**Last Updated**: 2025-11-14
**Owner**: Stella (Staff Engineer)
