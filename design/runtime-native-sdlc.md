# Runtime-Native SDLC Sessions

## Implemented — and six decisions below were superseded

**Read this section before the rest of the document.** The migration was built and merged as of 2026-09-26 (35 PRs across `nevadoai/command-center` and `nevadoai/nevado-sherpa-tui`). The sections below are the original proposal and six of their decisions did not survive implementation. Where this section and the text below disagree, **this section is what shipped.**

The authoritative detail is in `sdlc-runtime-execution-architecture.md` (decisions) and `sdlc-runtime-impl-spec-part1-foundation.md` / `…-part2-cutover.md` (code facts), all in this directory.

| Proposed below | What shipped |
|---|---|
| **An SDLC WebSocket channel** at `/v1/workspace/ws/sdlc`, with sequence numbers and reconnect replay | **SQS FIFO.** The runtime publishes through a `setObserver` hook on the existing `Broadcaster`: `messageGroupId = sessionId`, a per-envelope UUID dedup id, a `v: 1` envelope, and an opaque `correlation` (`applicationId`/`cycleId`/`stage`) the runtime never interprets. A Go consumer drains it. There is no SDLC WebSocket, and nothing long-lived survives in the machine path. |
| A route under `/v1/workspace` | **`/v1/sdlc/*`**, its own encapsulated plugin with its own authorization hook, so the machine surface is separately routable at the ALB and the hook cannot leak onto human routes. |
| `sdlc` config plus `systemPrompt` override on `POST /sessions` | A **`profile`** on `POST /v1/sdlc/sessions` carrying `tools`, `maxTurns` and `instructions`. |
| **`reportCompletion`** as the terminal tool | **Three** terminal tools — `submitPlan`, `submitReview`, `reportCompletion` — with **exactly one granted per session**, derived from `profile.tools`. Omitting the list grants none, which is fail-closed and was itself a shipped defect (`command-center#810`). |
| **SDK changes** (`pauseReason`, `completionResult`, protocol types, engine exit paths) as sub-issue 1, with a dependency chain `sherpa-sdk → runtime → command-center` | **The dependency chain does not exist**, and the SDK change that mattered was declined. All of it was built in `apps/runtime/src/sdlc/` against unchanged **4.0.1**, which Command Center still pins. `sherpa-sdk#227` did ship as **v4.1.2**, but only to widen `SessionApplication` with `'sdlc'` so the ACL comparison is type-checked rather than cast — nobody waited on it. `sherpa-sdk#226`, unforcing `createdBy` and `application`, was **closed unmerged**: `createdBy` backs a live ownership check in `renameSession`. Emitting `'completed'` from the engine loop was also declined — it is a human complete/archive flag and would break `nevado-sherpa-ide`'s `/resume`, and no session status distinguishes a session that submitted a result from one that did not. The pause reason travels on the published `sessionEnd` event instead. |
| "**The agent does not commit or push**"; the runtime commits after `reportCompletion` | **The agent commits and pushes.** `reportCompletion` then *verifies* it from git: it reads `pushedSha` from `refs/remotes/origin/<branch>` and refuses the call unless it equals `headSha`, naming both. It derives `branch`, `commitSha` and `filesModified` from git rather than accepting them as arguments. |

**Also settled, and previously listed as open items:** the working branch is derived as `cycle/<cycleNumber>` from the cycle sort key at approve-dispatch — the `cycle/**` prefix is a wire constraint, because the CI workflow the provisioner writes into every customer repo triggers on nothing else. `maxTurns` is carried per session on the profile. Authentication is a bearer token the runtime verifies itself, with mTLS retained as an inactive `sdlc_ingress_mode`; mTLS was declined on **cost**, not feasibility.

**Verification state.** Code-level verification is complete across all 35 PRs, with mutation testing on every Go change. One local end-to-end run proved dispatch → session → commit → push → git-verified completion. **No cycle has run on AWS.** The runtime is deployed; the ingress is not durable until `command-center#858` is fixed, because `sdlc_ingress_mode` is persisted nowhere and every merge to `develop` reverts it. Only **human-approved** cycles reach the Go path — PM auto-approval still routes through the JavaScript gate and its stub dispatcher.

## Context

SDLC (automated coding cycles) in Command Center originally used a bespoke execution path:

```
CC Orchestrator Lambda → Engineering Agent Lambda → SSM SendCommand → EC2
```

The Engineering Agent runs a 40-turn agentic loop (file ops, shell commands, tool calling) via SSM. The Sherpa runtime, which evolved later, provides these same capabilities natively — concurrent sessions, workspace management, real-time streaming, and a shared tool surface — all on the same EC2 instance, already exposed via CloudFront → ALB.

This design migrates SDLC execution to the runtime, enabling:

1. **CC SDLC cycles execute on the runtime** — eliminating the bespoke SSM agentic loop in favor of the shared AgentEngine
2. **Any surface can initiate a managed cycle** (future) — the architecture doesn't preclude IDE, TUI, or Web from starting sessions that feed into CC's pipeline, but the mechanism for how a surface-initiated session connects back to CC (authentication, cycle record creation, pipeline triggering) is not yet designed

## Goals

- Single execution layer — SDLC uses the same AgentEngine as interactive sessions
- Clear completion semantics — structured success/failure with metadata
- Clean boundary — runtime emits events, consumers decide what to do with them
- Minimal runtime additions — extend the existing session model, don't build SDLC-specific logic

## Non-Goals

- Multi-tenant isolation (same trust boundary as today)

## Why Migrate

| SSM Path (current) | Runtime Path (proposed) |
|---|---|
| Polls SSM every 2s for command results | Direct process execution, no polling |
| 5 bespoke tools reimplemented in Lambda | Shared AgentEngine tool surface (shell, files, MCP, sub-agents) |
| Manual git clone/branch/push scripting | Built-in workspace management |
| No observability during execution | Real-time event streaming |
| One session at a time per invocation | Concurrent sessions with queue |
| Ephemeral — workspace deleted after each run | Session persistence (S3 sync, resume) |

## Architecture

```mermaid
flowchart TB
    subgraph "CC Backend"
        ORCH[Orchestrator Lambda<br/>SQS-triggered, up to 15 min]
    end

    subgraph "EC2 Instance"
        RT[Sherpa Runtime<br/>Fastify :3000]
        AE[AgentEngine]
    end

    subgraph "CC Frontend"
        UI[Cycle Progress UI<br/>polls DynamoDB every 10s]
    end

    DDB[(DynamoDB: cycles)]

    ORCH -->|"1. POST /sessions (SDLC mode)"| RT
    ORCH -->|"2. Connect to SDLC WS channel"| RT
    RT --> AE
    AE -->|"tool events + completion"| ORCH
    ORCH -->|"write activities + result"| DDB
    UI -->|"poll"| DDB
```

### Sequence

```mermaid
sequenceDiagram
    participant UI as CC Frontend
    participant SQS as SQS
    participant Orch as Orchestrator Lambda
    participant RT as Runtime
    participant DDB as DynamoDB

    UI->>SQS: User approves plan
    SQS->>Orch: Trigger
    Orch->>RT: POST /v1/workspace/sessions (SDLC mode)
    RT-->>Orch: { sessionId }
    Orch->>RT: Connect /v1/workspace/ws/sdlc, subscribe to sessionId

    loop Agent executing
        RT-->>Orch: toolActivity {seq, tool, summary}
        Orch->>DDB: Write activity entry (batched as needed)
    end

    Note over Orch: Approaching Lambda timeout?
    Orch->>SQS: Send reconnect message {sessionId, cycleId, lastSeq}
    SQS->>Orch: New invocation
    Orch->>RT: Reconnect WS, replay from lastSeq

    RT-->>Orch: completion {seq, success, summary, filesModified}
    Orch->>RT: Commit (message) + push
    RT-->>Orch: committed {seq, commitSha, branchName}
    Orch->>DDB: Update cycle: engineeringResult, status
    Orch->>Orch: routeAfterEngineering() (PR, QA, deploy)

    UI->>DDB: Poll (10s interval)
    DDB-->>UI: Updated activities + status
```

## Runtime Changes

### SDLC Session Mode

`POST /v1/workspace/sessions` accepts new fields for SDLC:

```typescript
interface CreateSessionRequest {
  // ... existing fields ...
  systemPrompt?: string;  // overrides S3-loaded prompt when provided
  sdlc?: {
    autoApprove: boolean;            // all tool calls auto-approved
    includeCompletionTool: boolean;  // add reportCompletion to tool set
  };
}
```

When `sdlc` is present:
- Approval callback always returns `true`
- `reportCompletion` tool is registered
- Interactive tools are excluded (`askQuestion`, `switchMode`, `updateTaskTracker`) — there's no human to answer or observe these
- Session is tagged for the SDLC WS channel

**Note:** The exact tool allowlist/blocklist for SDLC sessions needs to be defined during implementation. The principle is: include execution tools (file ops, shell, search, sub-agents) + `reportCompletion`, exclude anything that expects human interaction or is only meaningful in an interactive context.

### SDLC WebSocket Channel

A dedicated WS endpoint (or subscription mode) that emits only tool-level events:

```typescript
// Tool activity event
{
  seq: number;           // monotonically increasing, for reconnect replay
  type: 'toolActivity';
  sessionId: string;
  tool: string;          // 'writeFile', 'editFile', 'runCommand', etc.
  summary: string;       // 'src/billing/index.ts', 'npm test', etc.
  status: 'started' | 'completed' | 'error';
  detail?: string;       // exit code, error message, etc.
  timestamp: string;
}

// Completion event
{
  seq: number;
  type: 'completion';
  sessionId: string;
  success: boolean;
  summary: string;
  filesModified: string[];
  metadata?: Record<string, unknown>;
  timestamp: string;
}

// Committed event (after orchestrator triggers commit)
{
  seq: number;
  type: 'committed';
  sessionId: string;
  commitSha: string;
  branchName: string;
  timestamp: string;
}

// Failure event (agent didn't call reportCompletion)
{
  seq: number;
  type: 'failed';
  sessionId: string;
  reason: SessionPauseReason;
  error?: string;
  timestamp: string;
}
```

The runtime emits these events. It doesn't know or care what the consumer does with them.

### `reportCompletion` Tool

```typescript
{
  name: 'reportCompletion',
  description: 'Signal that the implementation task is complete. Call when all code changes are ready.',
  inputSchema: {
    type: 'object',
    properties: {
      success: { type: 'boolean', description: 'Whether the task completed successfully' },
      summary: { type: 'string', description: 'What was implemented and verified' },
      metadata: { type: 'object', description: 'Optional structured data (test output, etc.)' }
    },
    required: ['success', 'summary']
  }
}
```

When called in SDLC mode:
1. Engine loop exits
2. Runtime stages `session.filesModified`, holds workspace open
3. `session.pauseReason` is set to `'completion_reported'`
4. Runtime emits `completion` event on SDLC WS channel (includes `filesModified`, `summary`)
5. Orchestrator receives completion, constructs commit message (using cycle context + agent summary)
6. Orchestrator tells runtime to commit (with the message) and push
7. Runtime commits, pushes, sets `commitSha`/`branchName`, cleans up workspace
8. Runtime emits `committed` event with final `commitSha` and `branchName`

The agent does not need to commit or push. The orchestrator controls the commit message (it has cycle context the runtime doesn't). The runtime executes the commit deterministically using the tracked `filesModified` list.

**Implementation detail:** Whether the orchestrator sends the commit command via the SDLC WS channel (bidirectional) or via a REST endpoint (e.g., `POST /sessions/:id/commit`) is left to implementation. The existing interactive WS is read-only (all input via REST), so either pattern has precedent.

### `pauseReason` Field

Every exit path in `AgentEngine.run()` sets a `pauseReason` on the session:

| Exit condition | `pauseReason` |
|---|---|
| Agent calls `reportCompletion` | `'completion_reported'` |
| `signal.aborted` (cancel) | `'user_cancel'` |
| `stopReason === 'end_turn'` | `'natural_stop'` |
| Context window full | `'context_exhausted'` |
| LLM inactivity timeout | `'llm_timeout'` |
| LLM error | `'llm_error'` |
| Tool rejection | `'tool_rejected'` |
| Max turns reached | `'max_turns'` |

If the agent finishes without calling `reportCompletion` and a session has `sdlc` config, the runtime emits a `failed` event with the `pauseReason`.

## Orchestrator Changes (CC)

### Starting a Session

```javascript
const response = await fetch('http://localhost:3000/v1/workspace/sessions', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    prompt: buildEngineeringPrompt(task, plan),
    mode: 'do',
    repoUrl: application.repoUrl,
    branch: `cycle/${cycleNumber}`,
    sdlc: {
      autoApprove: true,
      includeCompletionTool: true,
    },
  }),
});
const { sessionId } = await response.json();
```

### Listening for Events

The orchestrator connects to the SDLC WS channel and:
- Receives `toolActivity` events → decides how to batch/summarize → writes `activities[]` to DynamoDB
- Receives `completion` event → maps to `engineeringResult` → proceeds to next stage
- Receives `failed` event → sets cycle status to FAILED

The orchestrator decides the chunking strategy (every event, every N events, every 3 turns — whatever makes sense for the UI). The runtime doesn't care.

### Completion Mapping

```javascript
// After receiving 'completion' + 'committed' events:
const engineeringResult = {
  success: completionEvent.success,
  branchName: committedEvent.branchName,
  commitSha: committedEvent.commitSha,
  changes: { files: completionEvent.filesModified.map(f => ({ path: f })) },
  message: completionEvent.summary,
};
// Then: routeAfterEngineering() — existing logic unchanged
```

### Feature Flag

```javascript
// application.agentConfig in DynamoDB
{ "buildMode": "runtime" }  // 'ssm' (default) | 'runtime' (new)
```

Both paths coexist. Per-application switch. SSM path remains until fully deprecated.

## Interactive Flow (Any Surface → Managed Cycle)

A user on IDE/TUI/Web can also initiate a managed cycle by starting a session with `sdlc` config and a callback mechanism. The surface itself acts as the WebSocket consumer (showing progress), and on completion, notifies CC that a cycle completed.

This is a future extension — the core architecture supports it because the runtime doesn't distinguish who connected to the SDLC WS channel.

## Decisions

1. **SDLC WS endpoint** — Separate path (`/v1/workspace/ws/sdlc`). The interactive WS streams everything (token-by-token text, all events). SDLC consumers need a fundamentally different event set. Separate path keeps the contract clean.

2. **SDLC WS events include a sequence number** — Each event has a monotonically increasing `seq: number`. If the Lambda needs to reconnect (approaching timeout), it sends an SQS message to itself with `{ sessionId, cycleId, lastSeq }`. The new invocation reconnects and replays from that sequence. Gap between invocations is seconds — well within the buffer.

3. **Concurrency** — Runtime caps concurrent sessions at 5 (env `MAX_CONCURRENT_SESSIONS`). When hit, new sessions queue (FIFO wait, no rejection). Not a limitation at current scale. If contention arises: bump cap, priority queue, or reserve slots.

4. **Runaway prevention** — `AgentEngine` has a 200 LLM roundtrip cap per `engine.run()` invocation. For SDLC (one prompt, one run), this is the effective session limit. Hasn't been a problem (SSM runner only used 40), but should be configurable via session creation so SDLC can raise or remove it if needed.

5. **Token tracking** — Not needed per-session. Current SSM runner logs usage to console (Lambda → CloudWatch) but doesn't persist it on the cycle record. The runtime doesn't ship logs to CloudWatch today — token usage would only be visible via Bedrock billing. Acceptable for now; if per-cycle cost attribution is needed later, the runtime can log or persist usage.

6. **Post-engineering pipeline** — Remains as-is. The orchestrator creates the draft PR (GitHub API) after receiving the engineering result, then build verification (EventBridge poller), QA (Lambda + Bedrock), and deploy monitoring (EventBridge poller) proceed. All just need the same `{ success, branchName, commitSha, files }` result shape. Only the engineering execution stage changes.

7. **Real-time CC frontend** — Not needed. The Lambda writes activities to DynamoDB, the frontend polls every 10s.

8. **System prompt** — Lambda passes it via `systemPrompt` field on session creation. Overrides the runtime's S3-loaded prompt.

9. **Git commit/push** — Handled by the runtime after `reportCompletion`, triggered by the orchestrator (which provides the commit message). The agent does not commit or push.

## Repos Affected

**Superseded — see the table at the top of this document.** None of the `sherpa-sdk` rows happened, and the runtime row describes a WebSocket channel that was replaced by SQS. Retained to show what was proposed; `nevadoai/nevado-sherpa-tui` was written here as `agent-workflow`, a local clone directory name, which is the error that made a later revision of the spec conclude the repository did not exist.

| Repo | Proposed changes (not what shipped) |
|---|---|
| `nevadoai/sherpa-sdk` | Protocol types: `pauseReason`, `completionResult`, `sdlc` config on session. `reportCompletion` tool spec. SDLC WS event types. — **Not done.** Built in the runtime against unchanged 4.0.1. |
| `nevadoai/sherpa-sdk` (core) | `AgentEngine`: set `pauseReason` at each exit. Handle `reportCompletion` tool (set result, exit loop). — **Not done.** The pause reason already existed per turn; it travels on the published `sessionEnd` event. |
| `nevadoai/nevado-sherpa-tui` | Runtime: SDLC WS channel. Accept `sdlc` config in `POST /sessions`. Auto-approve for SDLC sessions. Emit tool events + completion on SDLC channel. — **Shipped differently:** `/v1/sdlc/*` as its own plugin, a `profile` rather than an `sdlc` config, and SQS instead of a WS channel. |
| `nevadoai/command-center` | Orchestrator: `POST /sessions` to start. WS client for SDLC channel. Write activities to DynamoDB. Handle completion/failure. Feature flag. — **Shipped with an SQS consumer** in place of the WS client; the rest holds. |
