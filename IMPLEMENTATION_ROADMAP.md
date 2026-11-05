# Agent Harness — Implementation Roadmap

**Status:** Proposed delivery sequence / Draft v1  
**Approach:** Validate Hermes integration first; then build a vertical slice.

## 1. Delivery principles

- Prove the Hermes integration contract before building broad product surfaces.
- Build a modular monolith, not microservices.
- Deliver end-to-end vertical slices.
- Keep Harness-owned state durable and independently backed up.
- Add security tests alongside each domain, not at the end.
- Prefer a small working system over a large dashboard with mocked controls.
- Do not build future mobile/custom workspace features in the initial release.

## 2. Phase 0 — Technical discovery and risk reduction

**Goal:** Remove uncertainty around the Hermes boundary.

Tasks:
1. Select and pin the initial Hermes version.
2. Inspect official documentation and supported integration surfaces.
3. Verify profile lifecycle and configuration operations.
4. Verify session lifecycle and how work is submitted/observed.
5. Verify supported event, webhook, polling or logging options.
6. Verify MCP server configuration and per-agent/tool filtering.
7. Verify CLI provisioning and credential behavior in the actual execution environment.
8. Verify sandbox controls and identify gaps.
9. Build a small proof of concept: Harness creates/maps two profiles, starts work in a session, observes completion and survives a Hermes restart.
10. Record integration contract, version assumptions and unsupported controls.

**Exit criteria:** No critical assumption about profile/session/invocation lifecycle remains undocumented. Security gaps have an explicit mitigation or are marked unsupported.

## 3. Phase 1 — Foundation and installation

**Goal:** A repeatable install with durable Harness state.

Deliver:
- Monorepo and pnpm workspace.
- NestJS API skeleton and domain module boundaries.
- React/Vite dashboard shell.
- PostgreSQL migrations and health checks.
- Docker Compose for local and VPS deployment.
- Environment/configuration validation.
- Secret encryption baseline.
- Admin bootstrap and first-run setup wizard.
- Structured logs, request IDs and audit foundation.
- Backup and restore prototype.

**Exit criteria:** Fresh install works; admin can log in; data survives restart; backup can be restored into a clean environment.

## 4. Phase 2 — Identity and Hermes integration

**Goal:** Harness agents map reliably to Hermes profiles and sessions.

Deliver:
- Tenant, user, membership and role model.
- Agent CRUD and primary-agent assignment.
- Specialist-agent model.
- Hermes instance registration and health.
- Hermes profile bindings and reconciliation.
- Hermes session bindings and observation.
- Agent configuration mapping.
- Runtime version compatibility checks.
- Restart/reconnect recovery.
- Admin views for profiles, sessions and health.

**Exit criteria:** Admin can create two Harness agents, map them to separate Hermes profiles, inspect their sessions and recover mappings after restart.

## 5. Phase 3 — Memory and knowledge

**Goal:** Give agents durable, permissioned knowledge.

Deliver:
- Memory provider interface.
- Mnemosyne adapter (if validated as the initial provider).
- Private, specialist and shared namespaces.
- Memory grants and authorization checks.
- Memory proposal/correction workflow.
- Document upload and object storage.
- Ingestion and chunking pipeline.
- Knowledge retrieval with permission filtering.
- Provenance, expiry and retention metadata.
- Agent-facing Harness memory/knowledge capability via supported Hermes extension or MCP.

**Exit criteria:** Two users/agents can store and retrieve private memories without cross-access; an explicitly shared item is available only to authorized agents.

## 6. Phase 4 — Orchestration and approvals

**Goal:** Agents can delegate durable work without losing tasks, by integrating with Hermes kanban (not rebuilding it).

Deliver:
- Harness task wrapper model keyed on kanban task id; tenant scoping and authorization.
- Agent-to-kanban-assignee mapping (Harness agent → Hermes profile).
- `hermes kanban create/assign/claim/complete/block/link/comment/attach` invocation from Harness/agent flows.
- Approval gates and owner-decision queue layered on kanban state transitions.
- kanban dispatch daemon integration and health observation.
- Primary-agent result consolidation mechanism (read kanban task results/events).
- Admin task board view reading kanban state + Harness audit.
- Security tests for tenant isolation on task data.

**Exit criteria:** A primary agent delegates a task to a specialist via kanban; task state survives restarts; dependencies, approval and completion are visible and auditable. No custom dispatch engine shipped.

## 7. Phase 5 — Integrations and permissions

**Goal:** Safely expose real work tools through Hermes.

Deliver:
- Capability definitions and agent grants.
- Connection records and secret references.
- MCP catalog and approved server lifecycle manager.
- MCP server health and configuration UI.
- Per-agent MCP access configuration.
- MCP gateway/proxy for integrations requiring server-side credential and authorization enforcement.
- CLI inventory per Hermes execution environment.
- Approved CLI installation/version tracking.
- Credential provisioning strategy for CLI tools.
- Permission/revocation reconciliation.
- Sandbox validation and security acceptance tests.

**Exit criteria:** A selected agent can use an approved MCP integration and CLI in its assigned environment; another agent cannot access the connection or resources without a grant; revocation is verified at the enforcement boundary.

## 8. Phase 6 — Evaluations and learning

**Goal:** Improve agents through evidence-based, controlled learning.

Deliver:
- Evaluation profile definitions.
- Evaluation runner and result storage.
- Baseline evaluation sets per agent.
- Capture corrections and task outcomes as candidate signals.
- Learning proposals with provenance.
- Approval/promotion workflow for identity or memory changes.
- Regression checks before promotion.
- Rollback and change history.
- Dashboard for evaluation trends and learning proposals.

**Exit criteria:** A proposed learning can be evaluated, promoted or rejected; changes do not silently broaden access or cause known evaluation regressions.

## 9. Phase 7 — Operational hardening and release

**Goal:** Prepare for repeatable self-hosted use.

Deliver:
- Installation documentation for local and VPS.
- Update and migration workflow.
- Backup/restore and retention documentation.
- Health and diagnostics page.
- Resource limits and capacity guidance.
- Secure defaults and hardening checklist.
- Version compatibility matrix for Hermes.
- End-to-end and multi-tenant isolation tests.
- Failure injection for Hermes/MCP/database restarts.
- Release packaging and upgrade/recovery tests.

**Exit criteria:** A new operator can install, configure, use, update and restore the product using documentation without developer intervention.

## 10. Suggested MVP boundary

A useful first MVP is not every domain at once. It should include:

- Installation and admin authentication.
- One tenant with multiple users.
- Primary and specialist agents.
- Hermes profile/session mapping.
- Private and explicitly shared memory.
- Durable task assignment/delegation.
- One approved MCP integration.
- CLI inventory for the Hermes environment.
- Basic permission enforcement and audit.
- Runtime/task health dashboard.

Advanced learning, broad integration catalog, family-specific UX, mobile and customizable workspaces can follow.

## 11. Testing strategy

### Unit tests
- Domain rules, authorization decisions, task transitions, memory scope and policy validation.

### Integration tests
- PostgreSQL transactions, outbox/worker, memory provider adapter, Hermes integration, MCP lifecycle and secret handling.

### End-to-end tests
- Setup wizard, user/agent creation, task delegation, approval, memory retrieval and recovery.

### Security tests
- Tenant isolation, cross-agent memory access, credential isolation, MCP revocation, CLI sandbox boundaries and admin privilege checks.

### Reliability tests
- Kill/restart API, worker, Hermes and MCP containers during active tasks.
- Verify no task loss, duplicate dispatch or false completion.
- Restore backup into a fresh installation.
- Test upgrade and rollback paths.

## 12. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| Hermes integration surface changes | Pin versions, contract tests, compatibility matrix |
| Undocumented Hermes internals become dependencies | Use supported interfaces only; isolate any unavoidable workaround |
| Tool visibility mistaken for security | Enforce at gateway/sandbox/credential boundary |
| Memory provider lock-in | Harness-owned provider interface and namespaces |
| Cross-user data leakage | Tenant-aware queries, grants, retrieval tests |
| Autonomous task loss | Durable DB state, outbox, idempotency and recovery |
| Docker API compromise | Restricted container manager, curated images and network isolation |
| Scope expansion into a generic platform | Keep Hermes-only non-goal explicit |
