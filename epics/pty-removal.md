# PTY Removal

**Epic:** [nevado-planning#64](https://github.com/nevadoai/nevado-planning/issues/64)
**Milestone:** —
**Initiative:** Standalone (no dependencies)

## Goal

Remove `/v1/workspace/ws/pty` and the `node-pty` dependency from the Sherpa runtime, and retire the one feature built on it — command-center's web terminal.

Success is: no PTY endpoint on the runtime, no native module in the runtime image or its CI, and no protocol surface describing PTY session state.

## Problem

The endpoint was documented as deprecated on 2026-09-16 (`nevado-sherpa-tui@6bfd5ad`), with the note that it "should not acquire new callers and may be removed."

command-center's web terminal shipped against it on 2026-06-05 ([command-center#302](https://github.com/nevadoai/command-center/pull/302)) — three months *before* the deprecation — with CloudFront multi-origin routing built specifically to reach it. No replacement transport was ever published, and command-center was never given a migration path.

So the deprecation was declared over a live consumer and then left unresolved. The endpoint is simultaneously deprecated and load-bearing: we pay to keep something alive that we've labelled dead, and a shipped feature depends on something we've labelled removable.

## Why it is worth doing now

The cost is concentrated in native-module handling, carried entirely for this one consumer:

- `Dockerfile:11` installs a `python3 make g++` toolchain into the builder stage solely for `node-pty`; `:20-21` rebuilds its bindings for linux.
- `.github/workflows/publish-sherpa.yml:60` pins install to a specific runner because `node-pty` is native; `:90-91` verifies `pty.node` landed.
- [nevado-sherpa-tui#72](https://github.com/nevadoai/nevado-sherpa-tui/issues/72) (Fargate Phase 0) carries "compiled `node-pty`" as a Dockerfile requirement, so the re-platform track is actively building around it.
- Five open HIGH-severity screen-flicker findings against `pty-manager.ts` sit untriaged, because the endpoint's status has been ambiguous.

Resolving the disposition removes the native-module burden from the Fargate re-platform and closes the flicker findings as moot rather than fixing them.

## Constraints

- **Consumer first.** command-center retires the terminal and deploys before the runtime deletes the endpoint, so production never routes to a deleted path.
- **The protocol change can lag.** `activePtySessions?` on `HealthResponse` is already optional, so the runtime can stop emitting it with no coordinated release.
- **Verify before deleting shared build steps.** The `python3 make g++` toolchain in the Dockerfile is believed to exist only for `node-pty`; confirm no other native dependency needs it.

## Architecture

Current, and what goes away:

```
command-center frontend
  TerminalView.jsx → usePtyTerminal.js → ptyWebSocketService.js
        │ WebSocket /v1/workspace/ws/pty  (via CloudFront multi-origin route)
        ▼
Sherpa runtime (nevado-sherpa-tui)
  ws/pty-handler.ts → ws/pty-manager.ts → node-pty → shell running the TUI
        │
        └── /health { activePtySessions } ──→ HealthResponse (sherpa-sdk protocol)
```

Every box below the frontend is deleted. Nothing replaces it.

## Sequence

| Order | Repo | Issue |
|-------|------|-------|
| 1 | command-center | [#779](https://github.com/nevadoai/command-center/issues/779) — retire the web terminal + CloudFront route |
| 2 | nevado-sherpa-tui | [#90](https://github.com/nevadoai/nevado-sherpa-tui/issues/90) — delete the endpoint, `node-pty`, build/CI handling, docs |
| 3 | sherpa-sdk | [#211](https://github.com/nevadoai/sherpa-sdk/issues/211) — drop `activePtySessions` from `HealthResponse` |

## Open questions

- **Does the web terminal have active users?** The decision to remove rather than migrate assumes it does not. If command-center finds real usage, that reopens this epic's core decision — raise it on #64 rather than working around it in #779.
- **Is the 3600s ALB idle timeout sized for terminals?** `docs/AWS_DEPLOYMENT.md:405` justifies it by "agent sessions and web terminal PTY." Agent sessions need a long timeout regardless, so this is expected to be a doc edit only — but it should be confirmed rather than assumed before touching infrastructure.

## Scope boundary

Replacing the web terminal with a different transport is **out of scope**. If browser-based terminal access is wanted again later, that is a new initiative with its own design — not a migration inside this one.
