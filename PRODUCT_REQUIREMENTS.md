# Agent Harness — Product Requirements

**Status:** Architecture baseline / Draft v1  
**Product direction:** Hermes-first agent harness  
**Audience:** Product, architecture, engineering and future contributors

## 1. Product vision

Build an installable management and intelligence layer around Hermes that turns individual Hermes agents into a coordinated, persistent, multi-user agent system.

The Harness complements Hermes. It does not replace its agent loop, model interaction, terminal, browser, filesystem, skills, tool invocation or session machinery.

**Core principle**

> The Harness owns who an agent is, what it knows, what it can access, what it should do and how it improves. Hermes owns how the agent thinks, maintains execution sessions and acts.

The first deployment is the product's own proving ground, but the product must not encode one household, one business, one set of agents or one operator as permanent assumptions.

## 2. Product goals

1. Provide a single admin control plane for users, agents, memory, knowledge, integrations, permissions, evaluations and operations.
2. Support multiple users in one installation, with clear private and shared data boundaries.
3. Give each person one primary agent as their main point of interaction.
4. Support specialist agents that perform work behind the scenes and report through the primary agent or shared task workflows.
5. Enable autonomous task execution with explicit ownership, state transitions, approvals and audit history.
6. Preserve agent identity and long-term knowledge outside Hermes-specific profile/session storage.
7. Reuse Hermes capabilities instead of rebuilding them.
8. Make installation, backup, restore, upgrades and recovery practical for a home machine or cloud VM.

## 3. Non-goals for the initial release

- Supporting runtimes other than Hermes.
- Building a replacement agent loop, model router, terminal, browser, filesystem or MCP client.
- Building a consumer-grade personal chat/mobile application. This is a later product surface.
- Building user-customizable workspaces, tabs and widgets.
- Kubernetes, microservice sprawl or independently deployed domain services.
- Arbitrary untrusted container-image execution.
- A general-purpose workflow automation platform competing with dedicated products.
- Silently changing agent identity, access policy, credentials or sharing permissions through autonomous learning.

## 4. Target users and tenancy

### 4.1 Installation and tenant

An installation is one deployed Harness environment. It can contain one or more tenants. A tenant represents a logical family, business, team or personal environment and owns its users, agents, resources, policies and integrations.

For an initial single-owner installation, the setup wizard may create one default tenant. The data model should not assume that an installation can only ever contain one.

### 4.2 Person and agents

- **Person/User:** A human account in a tenant.
- **Primary Agent:** The person's main agent and chief-of-staff interface.
- **Specialist Agent:** An agent assigned a domain or operational role and usually invoked through orchestration.
- **Agent:** A Harness-owned identity and policy record mapped to a Hermes profile.
- **Hermes Profile:** The Hermes-side configuration/profile used to execute an agent.
- **Hermes Session:** A Hermes-managed execution/conversation session associated with an agent/profile.

A person has one primary agent in the initial product model. Specialist agents can serve multiple workflows, subject to authorization. Users do not need to switch agents for ordinary tasks.

### 4.3 Contexts

Personal, engineering, career, family and business are activity/context scopes, not separate agents or security boundaries. The primary agent should resolve context from the request and available information. A context does not grant access by itself.

## 5. Core product capabilities

### 5.1 Admin dashboard (first product surface)

The first UI is an admin/control dashboard. It must let an authorized administrator:

- View installation health and Hermes runtime status.
- Create and manage tenants, users and agents.
- Assign primary agents and specialist roles.
- Inspect Hermes profile/session mappings and lifecycle status.
- Manage private, specialist and shared memory.
- Upload, organize, index and permission knowledge documents.
- Configure MCP servers and connections.
- Inspect detected CLI tools in approved execution environments.
- Configure agent capability policies and approvals.
- Manage model/provider settings supported by Hermes.
- Define evaluations and inspect results/regressions.
- View tasks, dependencies, approvals and agent activity.
- Inspect logs, failures, costs and resource usage.
- Configure backups, updates and system settings.

The dashboard is an API client. It must not read or mutate application databases directly.

### 5.2 Future personal agent experience

A future web/mobile experience will present a person's primary agent as the main interface. This is explicitly not the first implementation milestone.

### 5.3 Future customizable workspace

Users may eventually ask their agent to create or configure views, tabs, menus and widgets. This is backlog, not foundational scope.

## 6. Memory and learning requirements

Memory is Harness-owned in policy and data model, even if an external memory provider such as Mnemosyne is used initially.

Required memory scopes:

- **Private:** Visible to the relevant person/primary agent and explicitly authorized services.
- **Specialist:** Domain-specific operational memory for a specialist agent.
- **Shared:** Explicitly shared with selected agents, a family or an organization.
- **Tenant knowledge:** Governed shared knowledge, distinct from personal memory.

Each memory record should support owner, tenant, namespace/scope, visibility, provenance, confidence, timestamps, retention/expiry, source reference and correction status.

Agents may propose that a learning be shared. The Harness decides whether it is allowed and records the resulting scope. Agents must not silently broaden visibility.

Learning loop:

1. Observe interactions, corrections, task outcomes and evaluation feedback.
2. Propose candidate memories or operational improvements.
3. Validate against privacy, provenance and retention rules.
4. Evaluate the change against agent-specific evaluations.
5. Promote, reject or roll back the change.
6. Record why the change was made and its observed effect.

Separate:
- Personal learning: communication preferences, goals and user-specific facts.
- Operational learning: how an agent performs its assigned work.
- Platform learning: Harness-wide reliability and product improvements.

Learning cannot independently change identity, permissions, credential access or sensitive sharing.

## 7. Orchestration requirements

The Harness does **not** build a parallel task engine. Hermes ships a durable, multi-profile task board (kanban) with tasks, dependency links, append-only events, runs, comments, attachments, notify subscriptions, a dispatch daemon (claim locks, heartbeat, timeout, retry, dead-letter) and idempotency keys. The Harness **integrates with and wraps** Hermes kanban rather than reimplementing it.

The Harness's orchestration responsibilities are therefore:
- Tenant-scoping and authorization on top of kanban's `tenant` column.
- Mapping Harness agents/users to kanban assignees (Hermes profiles).
- Approval gates and owner-decision queues layered on task state transitions.
- Audit and observability beyond kanban's own event log.
- Primary-agent consolidation of specialist results.

Kanban's own lifecycle is reused: `triage → todo → scheduled → ready → running → blocked → review → done`. A Harness task wraps a kanban task ID and adds Harness-owned metadata (requester, acceptance criteria, context refs, approval requirements). Tasks, approvals and dependencies remain durable in kanban's store; the Harness never uses a Hermes session as the durable task database.

Task requirements (Harness layer):
- Stable ID and tenant (kanban task id + Harness wrapper).
- Requester and owner agent.
- Optional parent task and dependencies (kanban `task_links`).
- Description, acceptance criteria and relevant context references.
- Priority, status, timestamps and due date if applicable.
- Approval requirements.
- Result/artifact references.
- Append-only event history (kanban `task_events` + Harness audit).
- Idempotent, bounded retries (kanban dispatch daemon).
- Agent-to-agent messages do not replace durable task state.
- The primary agent consolidates specialist results for the user.

Harness agents invoke kanban via the Hermes CLI (`hermes kanban create/assign/claim/complete/block/link/comment/attach/dispatch/daemon`) or, where a supported API exists, through it. Direct writes to `kanban.db` are not permitted; all mutations go through Hermes interfaces.

## 8. Integrations and tools

The Harness complements Hermes rather than replacing its tool system.

- Hermes executes terminal, browser, filesystem and other native runtime tools.
- Hermes invokes CLI tools through its terminal.
- Hermes is the MCP client and invokes MCP tools.
- The Harness manages approved MCP server installation/configuration/lifecycle, account connections, credential ownership and agent-specific access policy.
- The Harness detects CLI availability inside the relevant execution environment and reports inventory/status. It does not build a second shell or command executor.
- Custom integrations should generally be exposed to Hermes through MCP or a supported Hermes extension/skill, rather than a parallel universal tool framework.

A tool being hidden from the model is not sufficient security. Enforce policy at the strongest available boundary: MCP gateway/connector, scoped credentials, Hermes configuration and execution sandbox.

## 9. Deployment and operations

Initial deployment target:
- Docker Compose.
- Home/personal machine and cloud VM/VPS.
- Local persistent storage by default; external object storage as an option.
- PostgreSQL for structured state; pgvector may be used initially if it meets retrieval needs.
- API, worker and learning worker can be separate processes from one modular monolith/codebase.
- Hermes runtime and approved MCP servers run in managed containers/environments.
- No direct unrestricted Docker socket access from the application API.

Must-have operational capabilities:
- Health checks and startup dependency handling.
- Persistent data separated from disposable containers/images.
- Backup and restore tested before general availability.
- Versioned database migrations.
- Upgrade compatibility checks and rollback/recovery strategy.
- Secret encryption and rotation strategy.
- Audit trail for privileged changes and sensitive actions.

## 10. Success criteria

Initial release is successful when an administrator can:

1. Install the stack and complete first-run setup.
2. Connect/configure Hermes and create at least one Harness agent mapped to a Hermes profile.
3. Create users and assign primary agents.
4. Create specialist agents and delegate a durable task between agents.
5. Store private and shared memory with enforced scope boundaries.
6. Add an approved MCP server and configure agent-specific access.
7. Detect approved CLI tools in an agent's execution environment.
8. Inspect task lifecycle, approvals, activity and failures.
9. Restart services without losing Harness-owned state.
10. Back up and restore an installation successfully.

## 11. Product principles

- Hermes-first, not runtime-agnostic.
- Reuse before rebuild.
- Harness-owned identity and policy; Hermes-owned execution.
- Explicit boundaries for private and shared information.
- Durable state over chat-only coordination.
- Autonomy with observable lifecycle and human approval at consequential boundaries.
- Secure defaults; unsupported enforcement must be reported honestly.
- Modular monolith before microservices.
