# Agent Harness — Hermes Integration Contract

**Status:** Design contract / validation required  
**Scope:** Hermes is the only supported agent runtime in the initial product.

## 1. Purpose

Define the boundary between Harness-owned product capabilities and Hermes-owned execution capabilities.

The Harness should integrate through documented and supported Hermes interfaces. This document intentionally distinguishes architectural intent from details that must be verified against the exact Hermes version selected for implementation.

## 2. Responsibilities

### Harness responsibilities

- Own user, tenant and agent identity.
- Own agent-to-person relationships and primary-agent assignment.
- Own long-term memory policy, namespace boundaries and shared-memory authorization.
- Own task orchestration, dependencies, approvals and audit.
- Own integration connection metadata and secret ownership.
- Manage approved MCP server lifecycle and agent-specific access policy.
- Provision/configure approved CLI binaries in execution environments.
- Maintain desired configuration and observed Hermes mappings.
- Provide authorized context and Harness capabilities to agents through supported interfaces.

### Hermes responsibilities

- Agent loop and reasoning execution.
- Model interaction and runtime-level model configuration.
- Profile and session functionality.
- Terminal/process, filesystem, browser and other built-in tools.
- CLI invocation through terminal.
- MCP client, tool discovery and tool invocation.
- Skills and runtime-specific execution behavior.

## 3. Integration surfaces to verify

Before implementation, inspect the exact deployed Hermes version and document the supported contract for each item:

| Surface | Required evidence |
|---|---|
| Profiles | Supported create/read/update/delete/list operations; config locations and validation |
| Sessions | Supported create/list/resume/close behavior; session identity and lifecycle |
| Messages/invocation | Supported way to submit work and receive responses |
| Events | Webhooks, event stream, logs or polling options |
| Configuration | Supported profile/tool/MCP/model configuration mechanism |
| MCP | Server configuration, per-server filtering, reload behavior and credential handling |
| Skills | Supported installation, scoping and update mechanism |
| Runtime health | Health endpoint/CLI/status contract |
| Versioning | Compatibility guarantees and migration notes |
| Sandbox | Available isolation controls and enforcement boundaries |

Do not infer a stable API from an internal database schema or an undocumented implementation detail.

## 4. Agent/profile mapping

A Harness agent maps to a Hermes profile. The Harness stores the mapping and desired configuration version.

Recommended invariants:
- A Harness agent has one designated active Hermes profile binding in the initial implementation.
- The profile identifier is treated as an external reference.
- Profile names are not primary keys in the Harness.
- Reconciliation is idempotent.
- If the Hermes profile disappears, Harness marks the binding degraded and offers a controlled repair/recreate flow.
- Harness identity remains intact if a profile is recreated.

Whether multiple Harness agents may share one Hermes profile should be disallowed initially unless Hermes's isolation semantics and Harness policy enforcement prove that it is safe.

## 5. Session strategy

Use Hermes sessions; do not build a parallel session engine.

Harness stores only the association and product metadata needed for:
- Which Harness agent owns/uses a session.
- Which task or workflow caused the session.
- Session purpose (conversation, task, scheduled, evaluation).
- Correlation IDs and timestamps.
- Observed state and last activity.

Do not assume session transcript ownership, retention or deletion semantics. Validate them and document whether the Harness needs a separate indexed copy for search/evaluation. Default to not duplicating transcripts.

A session should not be used as the durable task database. Tasks, approvals and dependencies remain Harness-owned.

## 6. Invocation and delegation

The Harness needs a supported way to:
- Deliver a task to the intended agent/profile.
- Supply a bounded task envelope and authorized context references.
- Associate execution with a Harness task ID.
- Receive completion/failure/status.
- Interrupt or cancel work where supported.
- Recover from runtime restarts.

Validate whether Hermes offers a native API, gateway, CLI, event mechanism or supported integration for each. The Harness should use the narrowest supported surface and avoid shelling out to undocumented internals.

If Hermes does not expose reliable completion callbacks, use a documented polling/observation strategy with idempotent task state transitions.

## 7. Context and Harness-native capabilities

Harness-owned capabilities (memory recall/store, knowledge search, task delegation, approvals and sharing) need a path into Hermes agents.

Preferred order:
1. Supported Hermes extension/tool interface.
2. MCP server exposing Harness APIs, with authenticated agent-scoped identity.
3. Supported skill/instruction integration for low-risk workflows, while keeping authorization server-side.

A prompt instruction alone is not an authorization boundary. Any read/write operation must validate the agent, tenant, scope and resource on the server.

## 8. MCP management

Harness manages:
- Approved MCP server catalog and configuration.
- Server container lifecycle and health.
- Connection-to-server credential binding.
- Agent-specific server/tool allowlists.
- Audit and revocation.

Hermes remains the MCP client and invokes tools.

For sensitive integrations, consider a controlled MCP gateway/proxy that:
- Exposes only the allowed tools.
- Resolves credentials server-side.
- Checks authorization on every invocation.
- Records audit metadata.
- Supports revocation without relying only on refreshing the model's tool list.

Whether a shared MCP server instance can safely serve multiple accounts must be assessed per server. If it cannot isolate credentials/data scopes, use separate instances or a credential-aware proxy.

## 9. CLI provisioning

The Harness may inventory and provision approved CLI binaries in the relevant Hermes execution environment.

- Discovery must happen inside the environment where Hermes terminal will execute the command.
- Record binary, version, path and non-secret authentication status.
- Do not copy CLI credentials into the Harness database or agent prompt.
- Prefer scoped credentials and environment injection supported by the execution environment.
- Do not assume a CLI installed on the host is available in a Hermes container.
- CLI inventory is not a permission grant.
- Direct shell access is governed by Hermes tool configuration and sandbox restrictions; command hiding is not security.

## 10. Configuration reconciliation

Maintain desired and observed configuration versions.

Example flow:
1. Admin changes agent policy in Harness.
2. Harness validates whether the policy can be enforced.
3. Harness computes the Hermes-facing configuration delta.
4. Integration worker applies it through supported Hermes interfaces.
5. Worker reads back observed state.
6. Harness marks synchronized or degraded and records diagnostics.

Do not silently overwrite manual runtime changes. Define whether a field is Harness-managed, runtime-managed or shared. Surface drift with a repair action.

## 11. Failure handling

- Hermes unavailable: preserve queued tasks and mark execution blocked.
- Profile missing: mark binding degraded; do not silently bind to a different profile.
- Session missing: use documented recovery behavior; preserve Harness task state.
- MCP server unhealthy: mark integration degraded and block affected capabilities.
- Configuration apply fails: retain previous known-good configuration and surface the failure.
- Version mismatch: block unsupported upgrades or enter compatibility warning mode.
- Duplicate events: deduplicate by external event ID or idempotency key.
- Worker restart: resume durable jobs from database/outbox state.

## 12. Version and compatibility policy

Pin a tested Hermes version for each Harness release. Maintain an integration test suite covering:
- Profile mapping and configuration.
- Session lifecycle.
- Task invocation and completion observation.
- MCP configuration and filtering.
- Tool availability and sandbox assumptions.
- Restart/recovery behavior.
- Upgrade compatibility.

Do not promise compatibility with every Hermes version until the suite validates it.

## 13. Kanban integration contract

Hermes owns the task engine; the Harness integrates with it. See `PHASE_0_FINDINGS.md` §5.

Rules:
- Task state, dependencies, events, runs, comments, attachments live in Hermes's shared `kanban.db`.
- The Harness stores only a thin wrapper keyed on the kanban task id, plus Harness-owned metadata (requester, acceptance criteria, context refs, approval requirements, tenant enforcement).
- All mutations go through `hermes kanban` CLI or a supported API. Direct writes to `kanban.db` are forbidden.
- Harness agents map to kanban assignees (Hermes profile names).
- Tenant isolation is enforced by the Harness on top of kanban's `tenant` column; kanban's own visibility is not a security boundary.
- Approval gates, owner decisions and audit are Harness-layer concerns layered on kanban state transitions.
- The Harness must not use a Hermes session as the durable task database.

## 14. Open validation checklist
- [ ] Identify exact supported Hermes APIs and configuration interfaces.
- [ ] Verify profile lifecycle operations.
- [ ] Verify session creation/resume/close semantics.
- [ ] Verify agent invocation and response retrieval.
- [ ] Verify event/webhook/polling options.
- [ ] Verify MCP per-server and per-tool filtering.
- [ ] Verify configuration reload/restart requirements.
- [ ] Verify CLI environment and credential provisioning.
- [ ] Verify sandbox capabilities and limitations.
- [ ] Verify supported runtime health and version reporting.
- [ ] Document all unsupported enforcement cases.

## Phase 0 status (updated 2026-11-05)

Phase 0 probing is complete. See `PHASE_0_FINDINGS.md` for the verified contract. Key outcomes:
- Profile/session/invocation lifecycle verified end-to-end (CLI + HTTP runs API).
- Hermes ships a built-in multi-profile kanban (tasks, links, events, runs, comments, attachments, notify subs, dispatch daemon) — the Harness should INTEGRATE with it, not rebuild orchestration.
- Sandbox is `local` backend by default → CLI capabilities must be marked `sandbox-limited` unless docker backend is enabled. Do not ship a security claim the runtime can't back.
- Multi-profile HTTP invocation requires a per-profile `API_SERVER_KEY`; provision one at profile creation or fall back to the CLI path.
