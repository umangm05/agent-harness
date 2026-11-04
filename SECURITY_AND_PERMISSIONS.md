# Agent Harness — Security and Permissions

**Status:** Security baseline / Draft v1  
**Principle:** Authorization is Harness-owned; enforcement must occur at the execution boundary.

## 1. Security objectives

- Isolate tenants and users.
- Protect private memory, documents and credentials.
- Prevent agents from granting themselves additional access.
- Ensure consequential actions can require human approval.
- Limit the blast radius of terminal, CLI and MCP execution.
- Maintain an audit trail for privileged and sensitive actions.
- Be explicit when a permission cannot be reliably enforced.

## 2. Threat model

Assume:
- An agent may misunderstand instructions or behave unexpectedly.
- A model may be prompt-injected by a document, website, email or tool result.
- A CLI may have broader access than its apparent command.
- An MCP server may expose more tools or data than intended.
- Credentials may be accidentally leaked through logs, environment dumps or generated files.
- A compromised agent/session may attempt lateral access to another agent's resources.
- Admin configuration may drift from runtime configuration.
- A malicious or compromised container image may attempt host access.

The Harness must not rely on the model to follow a policy as the sole control.

## 3. Authorization model

A grant should be scoped across:

`Principal → Capability → Resource → Connection → Constraints`

Example:

```yaml
agent: stark
capability: github.repository.write
resource:
  repository: umang/portfolio
connection: umang-github
effect: allow
constraints:
  branches:
    - feature/*
  approval:
    production_branch: required
```

A Gmail example:

```yaml
agent: family-assistant
capability: gmail.send
resource:
  account: dad-gmail
connection: dad-google
effect: allow
constraints:
  external_recipients: approval_required
```

These are conceptual examples, not final policy syntax.

### Policy rules

- Deny by default.
- Explicit deny overrides allow.
- Scope grants to specific resources and connections.
- Use least privilege.
- Grants can expire and be revoked.
- Agents may request access but cannot grant it.
- Approval decisions are recorded with actor, scope and time.
- Context names and agent roles do not automatically grant access.

## 4. Enforcement layers

| Capability path | Primary enforcement |
|---|---|
| Harness memory/knowledge APIs | Harness authorization on every request |
| Task delegation/approval APIs | Harness authorization on every request |
| MCP tools | MCP gateway or server-side authorization; Hermes tool visibility is an additional control |
| CLI through Hermes terminal | Hermes execution environment, scoped credentials, filesystem/network sandbox |
| Hermes native tools | Hermes configuration plus runtime sandbox/host controls |
| Dashboard/API | Authentication, tenant-scoped authorization and server-side validation |

If the required boundary cannot enforce a policy, mark the capability `unsupported` or require a safer execution path. Do not label it protected merely because it is hidden from the model.

## 5. Tenant and user isolation

- Every tenant-owned record includes `tenant_id`.
- Derive tenant access from authenticated membership, not request-supplied tenant IDs.
- Check authorization on every API request and background job.
- Private memory defaults to owner-only.
- Specialist memory defaults to that specialist and explicitly authorized Harness services.
- Shared memory requires an explicit grant.
- A user's credentials are never automatically available to another user's agent.
- Cross-tenant sharing is disabled initially unless a deliberate product requirement is designed and audited.

Use PostgreSQL row-level security as defense in depth where practical, alongside application authorization.

## 6. Memory and knowledge protection

- Authorize before retrieval, not after content reaches the model.
- Apply tenant and namespace filters to vector search.
- Keep provenance and source references.
- Treat retrieved documents and tool output as untrusted data, not instructions.
- Prevent one agent from reading another agent's private memory unless explicitly granted.
- Separate permission to read, propose, write and administer.
- Record sharing changes and memory corrections.
- Define retention and deletion semantics for source data, derived chunks, embeddings and provider copies.

## 7. Credentials and connections

- Store credentials in an encrypted secret store or encrypted-at-rest secret subsystem.
- Ordinary database rows contain secret references, never plaintext values.
- Never place secrets in agent identity prompts, shared knowledge or task descriptions.
- Prefer short-lived/scoped tokens where providers support them.
- Bind a connection to approved agents and resources.
- Do not inject all installation credentials into every Hermes container.
- Redact tokens, authorization headers and sensitive environment variables from logs.
- Support rotation and revocation.
- On revocation, block future use immediately at the connector/gateway boundary and terminate or refresh active sessions where supported.

## 8. MCP security

- Use a curated catalog by default.
- Validate image, command, transport, network and configuration before launch.
- Do not run arbitrary user-provided images without an explicit advanced/admin flow.
- Run MCP servers with least privilege, non-root users where possible, read-only filesystems where feasible, resource limits and restricted networks.
- Expose only approved tools to each agent.
- Check authorization on every sensitive tool invocation, not just discovery.
- Keep account credentials server-side when possible.
- Separate instances when a server cannot safely isolate multiple account credentials.
- Audit server start/stop/configuration and sensitive invocations.

## 9. CLI and terminal security

Hermes terminal executes CLI commands. The Harness provisions/inventories tools and controls policy/environment; it does not build a second shell.

Controls:
- Install only approved CLI binaries and pin/track versions.
- Discover binaries inside the actual Hermes execution environment.
- Use scoped credentials, not a shared admin token.
- Restrict writable mounts to assigned project/workspace directories.
- Do not mount host root, home directories broadly, Docker socket or unrelated secrets.
- Apply CPU, memory, process, execution-time and disk limits.
- Restrict network egress where practical.
- Avoid broad environment variables and credential files.
- Treat arbitrary shell access as high risk.
- Where command-level policy is required, enforce it with a reliable execution boundary; a prompt or tool allowlist alone is insufficient.
- Production deploys, destructive commands and sensitive operations can require approval.

## 10. Approval policy

Approval is an explicit state machine, not a conversational promise.

Potential approval classes:
- No approval: low-risk, reversible, scoped read operations.
- Agent/owner policy approval: routine writes within assigned resources.
- Human approval: external communications, production deployment, destructive changes, financial or account-sensitive actions.
- Denied: prohibited actions.

An approval should bind to the exact action payload, target resource, connection and expiry. If the action changes materially, request approval again.

## 11. Audit requirements

Record:
- Authentication and authorization failures.
- Policy changes and grant/revocation.
- Credential lifecycle events (not values).
- MCP server lifecycle and configuration changes.
- Sensitive tool invocation metadata and outcomes where observable.
- Approval requests and decisions.
- Agent profile/configuration changes.
- Memory/document sharing changes.
- Administrative operations, upgrades, backup and restore.

Audit records should be append-only to application roles. Define retention and export controls.

## 12. Container and host security

- Do not expose unrestricted Docker socket to the API.
- Use a constrained container-management service/proxy or narrowly scoped orchestration process.
- Curate images and pin digests where possible.
- Use isolated networks for MCP and runtime containers.
- Apply CPU/memory/PID limits.
- Avoid privileged containers and host networking.
- Persist only required volumes.
- Separate secrets from images and source control.
- Keep host patching and Docker daemon security in installation guidance.

## 13. Revocation and incident response

When a grant is revoked:
1. Update Harness policy transactionally.
2. Block future requests at Harness/MCP enforcement points immediately.
3. Apply Hermes configuration changes to reduce exposed tools.
4. Revoke/rotate credentials if exposure is suspected.
5. Stop or isolate affected runtime/container if the active execution cannot be safely interrupted.
6. Record the incident and affected resources.

The Harness must show whether revocation is fully enforced, pending runtime reconciliation or limited by an unsupported execution path.

## 14. Security acceptance tests

- Agent A cannot read Agent B's private memory by guessing an ID.
- A specialist cannot access a user's Gmail connection without an explicit grant.
- An MCP tool call is denied after its grant is revoked, even if the model still has a stale tool listing.
- A CLI cannot read host files outside approved mounts.
- An agent cannot grant itself a capability.
- Unauthorized document chunks never enter model context.
- A task approval cannot be reused for a materially different action.
- Secrets do not appear in logs, task payloads or agent-visible prompts.
- Tenant IDs supplied by a client cannot bypass membership checks.
- Container management cannot launch arbitrary privileged images through the API.
