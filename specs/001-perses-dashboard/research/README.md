# Research Deliverables: Dashboard Sharing & Access Control

**Feature Requirement**: FR-020
**Security Requirement**: SC-010 (Zero Security Violations)
**Research Completed**: 2025-11-14
**Lead Researcher**: Stella (Staff Engineer)

---

## Quick Reference

**Decision**: Hybrid RBAC + Optional Ephemeral Tokens

**Implementation Timeline**:
- Phase 0 (MVP): RBAC + 24h read-only tokens - 2 weeks
- Phase 1: Cross-project sharing + IP restrictions - 1 week
- Phase 2: Edit permissions + public sharing - 2 weeks

**Key Security Controls**:
- OpenShift OAuth authentication (primary)
- JWT-based RBAC with namespace isolation
- Ephemeral tokens (24h TTL, read-only, revocable)
- Comprehensive audit logging (SOC2/HIPAA compliant)

---

## Research Documents

### 1. [FR-020 Decision Summary](./fr-020-decision-summary.md) 📋 **START HERE**

**Purpose**: Executive summary for stakeholders

**Contents**:
- Final decision and rationale
- Security considerations (SC-010 compliance)
- User experience trade-offs
- Compliance requirements (audit logging)
- Implementation phases
- API design
- Success metrics

**Audience**: Product managers, security team, engineering leads

**Length**: ~14 KB (20-minute read)

---

### 2. [Dashboard Sharing & Access Control - Full Research](./dashboard-sharing-access-control.md) 📚

**Purpose**: Comprehensive research analysis

**Contents**:
- Perses core implementation deep dive
- OpenShift RBAC best practices
- Industry analysis (Grafana, Datadog, AWS)
- Security threat model & mitigation strategies
- Decision framework with alternatives considered
- Implementation code examples
- Open questions & recommendations

**Audience**: Staff+ engineers, security architects, technical reviewers

**Length**: ~29 KB (45-minute read)

**Key Sections**:
- Section 1: Perses RBAC analysis (roles, permissions, JWT auth)
- Section 2: OpenShift multi-tenant security
- Section 3: Competitor sharing patterns
- Section 4: Security requirements (SC-010 compliance)
- Section 5: Decision framework (3 options analyzed)
- Section 6: Recommended implementation
- Section 7: Alternatives rejected

---

### 3. [Sharing Architecture Diagrams](./sharing-architecture-diagram.md) 🎨

**Purpose**: Visual system design and data flows

**Contents**:
- System overview (components, layers, integrations)
- RBAC-based access flow (internal users)
- Token-based access flow (external users)
- Perses RBAC hierarchy diagram
- Permission model examples
- Token lifecycle state machine
- Security layers (defense in depth)

**Audience**: All stakeholders (diagrams for visual learners)

**Length**: ~30 KB (15-minute skim, diagrams speak for themselves)

**Visual Aids**:
- System architecture diagram
- Sequence diagrams (RBAC flow vs token flow)
- RBAC hierarchy diagram
- Token state machine
- Security layer stack

---

### 4. [Scalability Architecture](./scalability-architecture.md) 🚀

**Purpose**: Performance and scale considerations

**Contents**:
- Horizontal scaling strategy
- Caching architecture (Redis for tokens, in-memory RBAC)
- Database optimization (connection pooling, read replicas)
- Rate limiting design
- Load testing recommendations

**Audience**: SRE team, performance engineers

**Length**: ~35 KB (30-minute read)

**Key Topics**:
- Scale targets: 500+ users, 500+ dashboards, 50+ concurrent
- Performance SLOs: <3s p95 dashboard load, <30s metric latency
- Caching strategy: In-memory RBAC + Redis token cache
- Database scaling: PostgreSQL connection pooling, read replicas

---

## Implementation Checklist

### Phase 0: MVP (2 weeks)

- [ ] Add `Dashboard:share` permission to Perses RBAC model
- [ ] Implement share token generation API
  - [ ] POST `/api/v1/projects/{project}/dashboards/{id}/share`
  - [ ] Token model: UUID, expiration, permissions, creator
- [ ] Build token validation middleware
  - [ ] Expiration check
  - [ ] Revocation check
  - [ ] Read-only enforcement
- [ ] Add audit logging
  - [ ] PostgreSQL audit table
  - [ ] OpenShift Audit API integration
  - [ ] Events: dashboard.read, dashboard.share, dashboard.access.token
- [ ] Create share dialog in Perses UI
  - [ ] Generate token button
  - [ ] Copy share URL
  - [ ] Display expiration time
- [ ] Write tests
  - [ ] Unit tests: token generation, validation
  - [ ] Integration tests: RBAC + token flows
  - [ ] E2E tests: share workflow
  - [ ] Target: 70% backend, 60% frontend coverage

### Phase 1: Cross-Project Sharing (1 week)

- [ ] Implement ClusterRole pattern for cross-project access
- [ ] Add IP restriction support to tokens
  - [ ] CIDR validation
  - [ ] IP matching middleware
- [ ] Build token revocation API
  - [ ] DELETE `/api/v1/share-tokens/{token_id}`
  - [ ] Blacklist persistence
- [ ] Create admin dashboard for token management
  - [ ] List active tokens
  - [ ] Revoke tokens
  - [ ] View audit trail

### Phase 2: Advanced Features (2 weeks)

- [ ] Edit permissions for shared dashboards
  - [ ] Add `Dashboard:update` to share tokens
  - [ ] Implement versioning/edit history
  - [ ] Audit all edits
- [ ] Public sharing (admin-gated)
  - [ ] Isolated data source with query whitelist
  - [ ] Public dashboard flag
  - [ ] Rate limiting (stricter for public)
- [ ] iframe embedding support
  - [ ] CSP configuration
  - [ ] X-Frame-Options handling
  - [ ] Token in URL fragment (security)
- [ ] Advanced token controls
  - [ ] Usage limits (max 10 accesses)
  - [ ] Geofencing (country-based restrictions)
  - [ ] One-time use tokens

---

## Key Findings Summary

### From Perses Codebase Analysis

1. **Perses has robust RBAC**: Kubernetes-inspired Role/RoleBinding model with project scoping
2. **JWT authentication**: HTTP-only cookies, HS512 signature, in-memory permission cache
3. **Ephemeral dashboard pattern exists**: Can be reused for ephemeral share tokens
4. **Dashboard scope recognized**: `DashboardScope` already in RBAC model
5. **No existing sharing mechanism**: No URL-based token sharing (opportunity to add cleanly)

### From OpenShift Best Practices

1. **Namespace isolation critical**: Multi-tenant security requires project-scoped RBAC
2. **SubjectAccessReview API**: OpenShift provides authorization hooks for custom logic
3. **Audit logging mandatory**: All access must be logged for compliance
4. **ClusterRole pattern**: Standard for cross-namespace access (use for Phase 1)
5. **ServiceAccount model**: Backend uses SAs with minimal privileges

### From Industry Analysis

1. **Grafana**: Snapshots (static) + public dashboards (limited queries) + embedding (JWT)
2. **Datadog**: Role-based + time-limited tokens + IP restrictions + SSO
3. **AWS CloudWatch**: IAM-only (no anonymous sharing)
4. **Common pattern**: Hybrid RBAC + optional tokens for external sharing
5. **Security focus**: Short TTL (24h), revocation, read-only, audit logging

---

## Open Questions (Pending Product/Security Review)

1. **Public Sharing**: Should we allow anonymous public sharing for customer SLA dashboards?
   - **Impact**: Requires separate read-only data source with query whitelist
   - **Recommendation**: Phase 2 feature (not MVP)

2. **Token TTL Limits**: Should we allow unlimited TTL tokens?
   - **Impact**: Security vs convenience trade-off
   - **Recommendation**: Max 7 days; use RBAC for permanent access

3. **IP Restrictions**: Mandatory or optional?
   - **Impact**: Security vs friction trade-off
   - **Recommendation**: Optional (Phase 1); useful for high-security environments

4. **Cross-Project Default**: Opt-in (admin approval) or opt-out (enabled by default)?
   - **Impact**: Security vs collaboration trade-off
   - **Recommendation**: Opt-in for MVP; prevents accidental exposure

5. **Timezone Handling**: User local time vs UTC vs configurable?
   - **Impact**: Global teams need consistent time display
   - **Recommendation**: UTC default with user preference option (separate feature)

---

## Success Metrics

### Security (SC-010: Zero Violations)

- **Target**: 100% unauthorized access prevention
- **Measurement**: Zero security incidents in production
- **Monitoring**: Daily audit log review, SIEM alerts for suspicious patterns

### Performance

- **Dashboard load time**: <3s p95 for 8-chart dashboards
- **Token validation latency**: <50ms p95
- **RBAC cache hit rate**: >95%
- **Audit log write latency**: <100ms p95

### Adoption

- **Internal sharing**: 80% via RBAC (seamless SSO)
- **External sharing**: 20% via tokens (ad-hoc use cases)
- **Token revocation rate**: <5% (indicates proper usage)
- **User satisfaction**: 85% report sharing meets needs (post-release survey)

### Compliance

- **Audit coverage**: 100% of dashboard access logged
- **Retention**: 90 days minimum (SOC2), 7 years for HIPAA environments
- **SIEM integration**: All audit events forwarded to Splunk/ELK
- **Audit query performance**: <2s for 90-day range queries

---

## Next Steps

1. **Stakeholder Review** (This week)
   - [ ] Present FR-020 Decision Summary to product team
   - [ ] Security team review of threat model
   - [ ] Compliance team review of audit logging

2. **Phase 0 Design** (Next week)
   - [ ] Detailed API specification (OpenAPI)
   - [ ] Database schema design (share_tokens table)
   - [ ] UI mockups for share dialog
   - [ ] Security review of token generation algorithm

3. **Phase 0 Implementation** (Weeks 3-4)
   - [ ] Backend: Token generation + validation
   - [ ] Frontend: Share dialog + URL handling
   - [ ] Testing: Unit + integration + E2E
   - [ ] Documentation: API docs + user guide

4. **Phase 0 Deployment** (Week 5)
   - [ ] Staging environment testing
   - [ ] Security penetration testing
   - [ ] Load testing (50+ concurrent users)
   - [ ] Production rollout (canary deployment)

---

## References

### Perses Codebase

- `/workspace/sessions/agentic-session-1763159109/workspace/perses/internal/api/authorization/` - RBAC implementation
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/pkg/model/api/v1/role/` - Permission model
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/authorization.md` - RBAC documentation
- `/workspace/sessions/agentic-session-1763159109/workspace/perses/docs/concepts/ephemeral-dashboard.md` - Ephemeral pattern

### External Resources

- **Grafana**: https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/
- **Datadog**: https://docs.datadoghq.com/account_management/rbac/
- **Kubernetes RBAC**: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- **OpenShift**: https://docs.openshift.com/container-platform/4.14/authentication/using-rbac.html
- **JWT Best Practices**: RFC 8725
- **OWASP API Security**: https://owasp.org/API-Security/

---

## Contact

**Research Lead**: Stella (Staff Engineer)
**Team**: OpenShift AI Platform
**Slack**: #openshift-ai-observability
**Email**: stella@example.com (placeholder)

**For Questions**:
- Security concerns: Contact security team (#security-reviews)
- Product priorities: Contact PM (#product-management)
- Implementation details: Stella (this research)

---

**Status**: READY FOR STAKEHOLDER REVIEW
**Last Updated**: 2025-11-14
**Next Review**: Phase 0 design completion
