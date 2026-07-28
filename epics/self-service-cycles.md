# Self-Service Cycles

**Epic:** [nevado-planning#12](https://github.com/nevadoai/nevado-planning/issues/12)
**Milestone:** Self-Service Cycles
**Initiative:** A2 (CC Integration)

## Goal

Cycle initiation, interaction, and observability from any Sherpa surface (IDE, TUI, Web). Remove Tyler as the bottleneck — no context-switch to the CC web UI required.

## Personas

| Persona | Description |
|---------|-------------|
| **Developer** (Tyler) | Builds across surfaces, configures agents, runs cycles |
| **Sherpa agent** | The AI agent acting autonomously within a cycle |

## User Stories

### Initiation

| # | Story | Issue |
|---|-------|-------|
| 1 | Start a cycle from any Sherpa surface | sherpa-sdk#94 |
| 2 | Select an app and describe the scope | sherpa-sdk#95 |
| 3 | Set constraints (budget, model, autonomy) | command-center#499 |

### Mid-Cycle Interaction

| # | Story | Issue |
|---|-------|-------|
| 4 | Approve or reject a cycle's proposed plan from any surface | sherpa-sdk#96 |
| 5 | Give iteration feedback targeting a specific cycle | sherpa-sdk#97 |
| 6 | Cancel a running cycle | sherpa-sdk#98 |

### Observability

| # | Story | Issue |
|---|-------|-------|
| 7 | See status of all active cycles from any surface | sherpa-sdk#99 |
| 8 | Be notified when a cycle completes or needs input | command-center#500 |
| 9 | See token spend during and after a cycle | command-center#501 |

### Output

| # | Story | Issue |
|---|-------|-------|
| 10 | Review and merge the cycle's PR from the same surface | sherpa-sdk#100 |

### Error / Recovery

| # | Story | Issue |
|---|-------|-------|
| 11 | See why a cycle failed and retry from last checkpoint | command-center#502 |

### Configuration

| # | Story | Issue |
|---|-------|-------|
| 12 | Define which operations require approval vs. auto-proceed | command-center#503 |
| 13 | Review past cycles and their outcomes | sherpa-sdk#101 |

## Current State

The CC already has a full cycle API:
- 15+ REST endpoints (JWT + IAM auth)
- MCP server exposing: `start_cycle`, `list_cycles`, `get_cycle_status`, `approve_plan`, `iterate_cycle`, `cancel_cycle`, `create_pr`, `approve_pr`, `merge_pr`
- M2M Cognito credentials for programmatic access

The IDE already has:
- CC API client (`SherpaClient`) with auth (SigV4 + Cognito)
- Used for chat/conversations today, not cycles

The gap is surface-level integration — exposing these capabilities from where the developer already works.

## Dependencies (not owned by this epic)

- **Auth across surfaces** — TUI/Web need real identity before this works beyond IDE
- **Knowledge Loop** — agent memory read/write is a separate initiative
- **Notification mechanism** — ADR needed before story #8 can be fully specified

## Technical Enablers

- Typed protocol messages for cycle operations
- Shared CC client in SDK (MCP vs HTTP — pending decision)

## Spikes

| Spike | Issue | Informs |
|-------|-------|---------|
| What scope/constraint params does `start_cycle` accept? | command-center#504 | #95, #499 |
| Notification delivery mechanism for cycle events | command-center#505 | #500 |
| Does CC aggregate TokenUsage per-cycle? | command-center#506 | #501 |
| Does CC support partial cycle retry? | command-center#507 | #502 |
| What operations are gateable for autonomy config? | command-center#508 | #503 |

## Open Decisions

1. **MCP vs typed SDK client:** Should surfaces connect to CC's MCP server (agent-friendly, already exists) or build a new HTTP client in the SDK (UI-friendly, typed)? Likely both — but which for MVP?
2. **Auth identity:** Who is "Tyler" across IDE (AWS SSO), CC (Cognito), and runtime (hardcoded `dev-user`)?
3. **Notification channel:** WebSocket push, VS Code notifications, Slack, email, or poll?

## Scope Boundaries

**In scope:**
- Cycle lifecycle from Sherpa surfaces (start, interact, observe, output, recover, configure)
- IDE-first is acceptable as first milestone; TUI/Web follow

**Out of scope:**
- Multi-user access control (defer to auth decision)
- Cycle scheduling / cron-triggered cycles
- Knowledge loop (agent memory read/write — separate epic)
- Billing/metering pipeline
- Custom cycle templates
