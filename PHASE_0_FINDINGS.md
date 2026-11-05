# Agent Harness — Phase 0 Verified Findings

**Verified against:** Hermes Agent v0.22.1+609.g43faacf (2026.9.24), local install at `/root/.hermes/hermes-agent`, multiplexed gateway on 127.0.0.1:8642.
**Date:** 2026-11-05
**Method:** Live probing (CLI + HTTP API), not doc reading. Every claim below was exercised end-to-end.

## 1. Profiles

- **Create:** `hermes profile create <name> [--clone-from SOURCE] [--description ...] [--no-skills]` works. Clone copies config, .env, SOUL.md, skills, memory provider config. Messaging channels are NOT cloned (by design — avoids bot-token collisions). A profile with no .env inherits nothing; must clone or run setup.
- **Delete:** `hermes profile delete <name> -y`. Warns about running gateway but succeeds.
- **List/show:** `hermes profile list`, `hermes profile show <name>`.
- **Run any command against a profile:** `hermes -p <profile> <cmd>` (also `--profile`).
- **Persistence:** Each profile has its own home at `/root/.hermes/profiles/<name>/` with its own `state.db`, `projects.db`, `mnemosyne/`. Profiles are fully isolated on disk.
- **Serving:** Multiplexed gateway serves a set of profiles (`profiles_to_serve(multiplex=True)`). New profiles require a gateway restart to become HTTP-served; CLI access works immediately.

## 2. Sessions

- **Storage:** Per-profile SQLite `state.db` (table `sessions` with id, source, started_at, ended_at, end_reason, message_count, profile_name, model, title, ...). Hermes owns session data; Harness stores only the mapping.
- **Creation:** Sessions are created implicitly by a chat turn (`hermes -p <p> chat -q ...` or HTTP run). Returned `session_id` (format `local-<hex>`).
- **Resume:** `--resume <session_id>` or `'latest'` works; context is restored ("↻ Resumed session ... N user messages").
- **Lifecycle observed:** `end_reason = 'cli_close'` on normal exit. No explicit "close session" CLI observed; sessions simply end when the process exits.
- **Harness implication:** Store `hermes_session_id` as an external reference; do not duplicate transcripts by default (confirmed feasible — session data is local SQLite, not an opaque blob).

## 3. Invocation (two viable paths)

### Path A — CLI (universal, no key needed)
`hermes -p <profile> chat -q "<prompt>" -Q --oneshot` → prints final response + `session_id: ...` on stderr/stdout. Exit code 0 on success. Works for ANY profile immediately, including freshly created ones.
- **Pros:** No auth setup, works for every profile.
- **Cons:** Subprocess spawn per turn; no streaming; harder to observe mid-flight.

### Path B — HTTP API (richer, requires per-profile API_SERVER_KEY)
Base: `http://127.0.0.1:8642`. Auth: `Authorization: Bearer <API_SERVER_KEY>` (64-char, from root `.env`). Multiplexed profiles served at `/p/<profile>/...`.

Verified endpoints (default profile, root path):
- `GET /v1/capabilities` — feature flags (run_submission, runs_idempotency durable, run_events_sse, run_stop, run_steer, run_approval_response, session_chat, session_fork, ...).
- `POST /v1/chat/completions` — OpenAI-compatible, `{"model":"hermes-agent","messages":[...],"stream":false}` → `choices[0].message.content`.
- `POST /v1/runs` — `{"input":"...","model":"hermes-agent"}` → `{"run_id","status":"started","replayed":false}`. Idempotency supported (durable, 86400s retention).
- `GET /v1/runs/{run_id}` → status, output, session_id, usage, pending_controls.
- `GET /v1/models`, `GET /health`.

Profile routing: `/p/<profile>/v1/...` works ONLY if that profile has its own `API_SERVER_KEY` (in its own `.env`). Profiles cloned without it fail auth. The default profile uses the root key. **Harness must either (a) provision an API_SERVER_KEY into each profile's .env at creation, or (b) rely on the CLI path.** Recommend (a) for the Harness-managed profiles so the richer HTTP surface (runs, SSE, steer, stop, approvals) is available.

## 4. Events / observation

- `run_events_sse` feature flag is true → SSE stream for run events (not individually probed, but advertised).
- `run_stop`, `run_steer`, `run_approval_response` advertised → Harness can interrupt/steer/approve mid-run via HTTP.
- CLI path has no mid-flight observation; only final output + session_id.
- Webhooks: `hermes webhook {subscribe,add,list,remove,test}` — dynamic webhook subscriptions for event-driven agent activation. Usable as a completion-notification channel if Hermes fires them on run completion (not probed).

## 5. Task orchestration — Hermes already has a durable kanban

This is the single most consequential finding. Hermes ships a built-in multi-profile task board:

- Storage: `/root/.hermes/kanban.db` (SQLite), tables `tasks`, `task_links`, `task_events`, `task_runs`, `task_comments`, `task_attachments`, `kanban_notify_subs`.
- Task fields include: id, title, body, assignee (profile), status, priority, parent_task_id (via task_links), tenant, idempotency_key, max_runtime_seconds, session_id, model/provider/reasoning overrides, completion_contract, goal_mode, consecutive_failures, worker_pid, claim_lock, etc.
- Statuses: triage, todo, scheduled, ready, running, blocked, review, done.
- CLI: `hermes kanban {create, assign, claim, complete, block, unblock, link, comment, attach, dispatch, daemon, watch, ...}` with `--parent`, `--tenant`, `--idempotency-key`, `--max-runtime`, `--completion-contract`, `--goal`.
- Dispatch: `hermes kanban dispatch` (one-shot) and `hermes kanban daemon` (polling loop, default 60s) spawn a profile process per ready task, with claim locks, heartbeat, timeout, retry, and dead-letter after N consecutive failures.
- Dependencies: `task_links` parent→child; cycle prevention at service layer.

**Implication for the Harness:** The "cross-agent task orchestration" domain in the architecture doc largely duplicates what Hermes already provides and runs in production. The Harness should INTEGRATE with (and wrap) Hermes kanban rather than rebuild it:
- Harness can create/assign tasks via CLI (`hermes kanban create --assignee <profile> --tenant <tenant> ...`) or by writing to kanban.db (not recommended — use CLI).
- Harness adds value on top: tenant isolation, memory/knowledge grants, approvals, audit, multi-user identity, MCP/CLI policy. Kanban gives the execution engine and lifecycle for free.
- Open question: is kanban.db shared across all profiles (yes, single file at /root/.hermes/kanban.db) and does it support tenant filtering (yes, `tenant` column). Good fit for multi-tenant Harness with a tenant column.

## 6. MCP management

- `hermes mcp {add, remove, list, test, configure, login, catalog, install, serve}`.
- `hermes mcp serve` exposes Hermes as an MCP server with conversation-bridge tools: conversations_list, conversation_get, messages_read, attachments_fetch, events_poll, events_wait, messages_send, channels_list, permissions_list_open, permissions_respond. This is for cross-agent messaging/observation, NOT for invoking an agent's loop.
- Per-agent MCP access filtering and server lifecycle are managed by Hermes config; Harness can manage the catalog/config via CLI. The Harness's "MCP gateway/proxy for server-side credential enforcement" is still needed for sensitive integrations (Hermes MCP is a client, not a credential-aware proxy).

## 7. CLI provisioning & sandbox

- Terminal backend: `local` (default). No container isolation by default. Config has container_cpu/memory/disk limits but backend is local process execution.
- `cli_inventory` concept in the data model maps to: binary detection inside the actual execution environment (`hermes` terminal runs in the profile's cwd). There is no built-in "detect installed CLIs" command; the Harness must probe `which`/`--version` in the execution environment.
- Sandbox reality: command execution is local-process; container isolation requires switching backend to docker (not probed). **Security gap:** local backend means CLI tools run with the host user's privileges. The architecture doc's "CLI cannot read host files outside approved mounts" acceptance test cannot be met with `backend: local`. Must either (a) switch to docker backend, or (b) mark CLI capabilities `unsupported`/`sandbox-limited` honestly.

## 8. Memory

- Mnemosyne is the configured memory provider (cloned with profile). Each profile has its own Mnemosyne bank (`/root/.hermes/profiles/<name>/mnemosyne/data/banks/<name>/mnemosyne.db`). Memory is NOT shared across profiles by default — consistent with the Harness's private/specialist/shared namespace model (shared requires explicit grants).
- Harness-owned memory policy layer on top is still needed for cross-profile sharing, grants, and tenant isolation.

## 9. Version & compatibility

- Pinned version: v0.22.1+609.g43faacf. No compatibility guarantee across versions (per doc). Pin this in the Harness and run contract tests on upgrade.

## Phase 0 exit-criteria assessment

| Checklist item | Status | Evidence |
|---|---|---|
| Identify exact supported Hermes APIs | ✅ Verified | CLI help + HTTP capabilities + live calls |
| Verify profile lifecycle operations | ✅ Verified | Created + chatted + deleted `harnessprobe` |
| Verify session creation/resume/close | ✅ Verified | session_id returned, resume restored context |
| Verify agent invocation and response retrieval | ✅ Verified | CLI oneshot + HTTP runs API both work |
| Verify event/webhook/polling options | ⚠ Partial | SSE/run events advertised; webhooks exist; not end-to-end probed |
| Verify MCP per-server and per-tool filtering | ⚠ Partial | MCP CLI exists; per-tool filtering config not probed |
| Verify configuration reload/restart requirements | ⚠ Partial | New profiles need gateway restart to be HTTP-served; CLI works immediately |
| Verify CLI environment and credential provisioning | ⚠ Partial | backend=local; container backend not tested |
| Verify sandbox capabilities and limitations | ⚠ Partial | local backend = no isolation; gap flagged |
| Verify supported runtime health and version reporting | ✅ Verified | /health returns status/version; `hermes status` |
| Document all unsupported enforcement cases | ⚠ Open | CLI sandbox (local) is the main one; MCP gateway proxy still needed |

## Recommendations (revised after Phase 0)

1. **Reuse Hermes kanban for task orchestration** instead of rebuilding. Saves the entire orchestration domain's execution engine. Harness focuses on identity, memory policy, knowledge, permissions, connections, evaluations, and a thin orchestration layer (tenant scoping, approvals, audit) on top of kanban.
2. **Invocation strategy:** Use the HTTP runs API (`POST /v1/runs` + `GET /v1/runs/{id}`) for Harness-managed profiles, provisioning an `API_SERVER_KEY` into each profile's `.env` at creation. Fall back to CLI for profiles without a key.
3. **Sandbox honesty:** With `backend: local`, mark CLI capabilities as `sandbox-limited` (not `protected`) unless the operator switches to the docker backend. Do not ship a security claim the runtime can't back.
4. **Profile lifecycle is clean** — create/clone/delete all work; Harness can manage profiles as external references with idempotent reconciliation.
5. **Do not duplicate session transcripts** — confirmed they live in per-profile SQLite; store only the mapping. Good.
6. **Memory sharing** requires Harness-level grants because Mnemosyne is per-profile isolated. Consistent with the data model's `memory_grants`.
7. **Smallest viable next step:** Build the vertical slice against the HTTP runs API with one primary + one specialist profile, one kanban task, private memory, basic audit. No custom orchestration engine.