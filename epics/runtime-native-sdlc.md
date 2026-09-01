# Runtime-Native SDLC Sessions

**Epic:** [nevado-planning#TBD](https://github.com/nevadoai/nevado-planning/issues/TBD)
**Milestone:** Runtime-Native SDLC
**Design Doc:** [design/sdlc/runtime-native-sdlc.md](../design/sdlc/runtime-native-sdlc.md)

## Goal

Migrate SDLC engineering execution from the bespoke SSM agentic loop to the Sherpa runtime's session API. The orchestrator Lambda starts a runtime session, listens for tool events and completion on a dedicated WebSocket channel, and proceeds through the existing post-engineering pipeline unchanged.

## Scope

- Engineering execution only — post-engineering pipeline (PR creation, build verification, QA, deploy) remains as-is
- Feature-flagged per application (`buildMode: 'runtime'`) — SSM path coexists until deprecated
- CC frontend unchanged — Lambda writes activities to DynamoDB, frontend polls as before

## Sub-Issues

| # | Title | Repo | Scope |
|---|-------|------|-------|
| 1 | SDLC session protocol + engine support | `sherpa-sdk` | Protocol types (`pauseReason`, `completionResult`, `sdlc` config, SDLC WS event types), `reportCompletion` tool spec + handler, set `pauseReason` at all engine exit paths, SDLC tool filtering, configurable `maxTurns` |
| 2 | SDLC session mode + WS channel | `agent-workflow` | Accept `sdlc` config + `systemPrompt` override in `POST /sessions`, SDLC WS endpoint (`/v1/workspace/ws/sdlc`) with seq numbers and reconnect replay, tool event emission, commit/push handling |
| 3 | Orchestrator runtime integration | `command-center` | Feature flag (`buildMode: 'runtime'`), session creation via HTTP, SDLC WS client, activity writing to DynamoDB, completion/failure handling, Lambda timeout reconnect via SQS |

## Dependencies

```
1 (sherpa-sdk) → 2 (agent-workflow) → 3 (command-center)
```

SDK must publish before runtime can consume. Runtime must be deployed before orchestrator can target it.

## Open Items

- Exact SDLC tool allowlist/blocklist (determined during sub-issue 1)
- Commit message format convention (orchestrator decides)
- Whether `maxTurns` default of 200 is sufficient or needs adjustment for SDLC
- Concurrency handling if contention arises at scale
