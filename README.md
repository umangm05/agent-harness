# Agent Harness — Architecture Documents

This folder contains the initial product and technical architecture baseline for a Hermes-first Agent Harness.

## Documents

1. `PRODUCT_REQUIREMENTS.md` — vision, goals, users, scope and functional requirements.
2. `SYSTEM_ARCHITECTURE.md` — components, boundaries, processes, data flow and deployment.
3. `DATA_MODEL.md` — conceptual PostgreSQL entities, relationships and isolation.
4. `HERMES_INTEGRATION.md` — Harness/Hermes ownership boundary and integration contract.
5. `SECURITY_AND_PERMISSIONS.md` — threat model, authorization, enforcement and security acceptance tests.
6. `IMPLEMENTATION_ROADMAP.md` — phased delivery plan and exit criteria.

## Important status note

These are architecture baseline documents, not a claim that every Hermes integration detail has already been verified. Phase 0 of the roadmap is specifically intended to validate the selected Hermes version's supported profile, session, invocation, event, MCP, CLI and sandbox interfaces before implementation depends on them.

## Agreed product direction

- Hermes is the only supported runtime for the foreseeable future.
- Harness complements Hermes; it does not recreate its agent loop or native tools.
- Harness owns identity, long-term memory policy, knowledge, orchestration, permissions, connections, evaluations and administration.
- Hermes owns profiles, sessions, reasoning, model interaction, native tools, terminal/CLI execution and MCP client behavior.
- The initial architecture is a modular monolith deployed with Docker Compose, with separate worker processes where useful.
- Admin dashboard first. Personal mobile experience and customizable workspaces are later backlog.
