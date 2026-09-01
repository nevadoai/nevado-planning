# SDLC Step Contract — Cycle Record, Events, and Gates

**Status:** Sketch, not decided. Fleshes out the field-level contract that [`sdlc-execution-architecture.md`](sdlc-execution-architecture.md) flags as undrafted (see "Design principle: contracts at boundaries" and the Layer 2 step-list note). Data/API contract only — no UI is specified here. [`sdlc-ux.html`](sdlc-ux.html) renders one possible view of this data as an illustrative sketch; nothing here requires that specific view.

## Scope

What Layer 2 writes into Command Center's existing DynamoDB cycle record, so any consumer — today's frontend, a redesigned one, `sdlc-ux.html`'s mockup — can show role/step/gate state without hardcoding a pipeline shape or reaching into session internals. Follows the same contracts-at-boundaries split as the rest of the design: the runtime emits a generic vocabulary with no knowledge of CC's table shape; Layer 2 is the only thing that translates and enriches.

## Today, for comparison

- `cycle.currentStage` — a numeric index into a hardcoded `PIPELINE_STAGES` array (`command-center/frontend/src/components/CycleProgress.jsx:17-58`). Engineering-only, fixed shape, can't represent five roles or a per-tenant step list.
- `cycle.activityLog` / `cycle.activities` — flat `{timestamp, message}` (`CycleProgress.jsx:75-76`). No role attribution, no tool-level detail.
- Two separate, bespoke gate-resolution endpoints: `POST /v1/applications/{id}/cycles/{cycleId}/approve-plan` (`ApplicationDetail.jsx:355-364`) and `PUT /v1/applications/{id}/cycles/{cycleId}` with `approvalStatus` (`ApplicationDetail.jsx:286-310`).

## 1. Step list

Per-tenant config (owned by Layer 2, the thing #37 needs to make editable):

```
StepConfig {
  stepId: string          // stable id, e.g. "engineering", "qa", "docs"
  role: Role               // "engineering" | "qa" | "documentation" | "devops" | "compliance"
  sessionConfig: object     // role-specific system prompt + sdlc config; opaque to Layer 2 beyond pass-through
  gate?: GateConfig         // optional — pause before or after this step
  onFail?: "retry" | "escalate" | "abort"
}
```

Per-cycle runtime instance, written into the cycle record as `cycle.steps[]` — this replaces `currentStage`:

```
StepState {
  stepId: string
  role: Role
  status: "queued" | "active" | "blocked" | "complete" | "failed"
  sessionId?: string        // set once the session for this step starts
  startedAt?: string
  completedAt?: string
}
```

`cycle.currentStepId` can be stored explicitly or derived client-side as the first non-terminal entry in `steps[]`.

## 2. Session events → activity record

Layer 1's generic event vocabulary is unchanged from what's already specified in `sdlc-execution-architecture.md`: `sessionId`, `type`, `tool`, `summary`, `status`, `seq`. Layer 2 enriches each event with role/step context it already has — it started the session, so it knows what role and step it belongs to — before writing it into `cycle.activities[]`:

```
ActivityEntry {
  seq: number
  timestamp: string
  role: Role
  stepId: string
  sessionId: string
  type: "toolActivity" | "completion" | "committed" | "failed"
  tool?: string
  summary: string
}
```

The runtime never needs to know about roles, steps, or CC's table shape — it only ever emits the generic event. Layer 2 is the sole place `role` and `stepId` get attached.

## 3. Gates

Generic read, embedded on the cycle record rather than a separate polled endpoint — the frontend already polls the cycle record every 10s (`ApplicationDetail.jsx:209-211`), so a second poll loop isn't needed:

```
GateState {
  gateId: string
  stepId: string
  question: string
  channel: "cc-ui" | "slack" | "linear"
  openedAt: string
  timeoutAt?: string
  status: "open" | "resolved" | "timed_out"
}
```

One resume call, replacing both of today's bespoke endpoints:

```
POST /v1/gates/{gateId}/resume
{ answer: "approve" | "reject" | string, actor: string }
```

Each channel adapter (CC-UI, Slack, Linear) calls this the same way after its own channel-specific auth check. The primitive itself doesn't know or care which channel called it — matching the adapter model already described in the Gates section of `sdlc-execution-architecture.md`.

## What this doesn't specify

No UI — `sdlc-ux.html` is one exercise of this contract, not a requirement derived from it. Also out of scope here: the exact DynamoDB item/partition-key layout (existing cycle-table conventions apply, this only adds fields) and the internal shape of `sessionConfig` (owned by session bootstrap, opaque at this layer).
