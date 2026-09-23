# Project-local AGENTS.md

**Epic:** [nevado-planning#66](https://github.com/nevadoai/nevado-planning/issues/66)
**Milestone:** Project-local AGENTS.md
**Initiative:** Standalone (no dependencies)

## Goal

A project's own `AGENTS.md` governs the agent working in it, on every Sherpa surface.

Success is: a developer drops an `AGENTS.md` in a repo root, and the next message they send,
the agent is following it — on the TUI, the runtime, and the IDE alike, with no configuration,
no knowledge base, and no deploy.

## Problem

Sherpa's operating instructions come from S3, deployed from `command-center/bootstrap/` by
`scripts/deploy-bootstrap.sh`. That content is written for the client accounts our tools run
in: `agents/sherpa/bootstrap/AGENTS.md` describes a Command Center assistant that lists
applications and submits feedback. It is not the right prompt for someone building the tools.

Today the only lever is editing the bootstrap source in a different repo and running a deploy
script — which changes the prompt for every Sherpa surface in the account. There is no
per-project option.

The two surface families fail differently:

- **SDK-backed (TUI, runtime).** `loadSystemPrompt` (`sherpa-sdk/packages/core/src/prompt-loader.ts:79`)
  reads only S3. The sole `AGENTS.md` string in the package sources is the S3 key at `:100`.
  There is no local path at all.
- **IDE.** `nevado-sherpa-ide` *does* discover workspace-local instruction files
  (`src/config/promptLoader.ts:281`), but its candidate chain ranks `CLAUDE.md` third and
  `AGENTS.md` ninth, and `loadProjectInstructions` (`:326`) reads the match raw with no
  `@`-import resolution. In a repo following the `CLAUDE.md` → `@AGENTS.md` convention it
  loads an 11-byte pointer as the project instructions and never opens the real file.
  Demonstrable in `sherpa-sdk`: `CLAUDE.md` is 11 bytes containing `@AGENTS.md`; `AGENTS.md`
  is 7091 bytes and is never read. The failure is silent — the section looks populated.

## Why it is worth doing now

`AGENTS.md` is the cross-vendor convention, and Claude Code now discovers it natively
(the binary carries the string "Claude Code hardcodes CLAUDE.md / AGENTS.md discovery" as its
reason for rejecting Codex's `project_doc_fallback_filenames` setting). So the `CLAUDE.md`
pointer that breaks the IDE is becoming vestigial, while the file the IDE ranks ninth is
becoming the one that matters. The ordering gets more wrong over time, not less.

## Architecture

Three surfaces, two mechanisms.

- **`sherpa-sdk`** — `PromptLoaderConfig` gains an optional `workspaceRoot`. When
  `<workspaceRoot>/AGENTS.md` is present and non-blank it stands in for the whole S3 side of
  the prompt; the built-in tool-guidance block is appended as it is today. Read before the
  cache check, so local content is never cached and never shared between workspaces.
- **`nevado-sherpa-tui`** — both `apps/tui` and `apps/runtime` wrap `loadSystemPrompt` and
  currently discard the `ctx` that carries the workspace root. Each forwards
  `ctx.workspaceRoot`, but only for sessions actually bound to a checkout.
- **`nevado-sherpa-ide`** — independent implementation, needing two unrelated changes:
  reordering the candidate chain to prefer `AGENTS.md` over the vendor-specific files above it,
  and switching project instructions from appended to replacing, per the decision below. The
  first is a bug; the second is behavioral alignment. Neither depends on the other.

## Sequence

1. SDK capability — self-contained, ships first.
2. TUI + runtime adoption — blocked on (1) and an SDK version bump.
3. IDE candidate ordering — independent of both; can run in parallel.
4. IDE append → replace — independent of (1)–(3), and the last piece needed before the same
   `AGENTS.md` behaves the same way on every surface.

## Decisions

**Project instructions replace knowledge-base instructions, on every surface.**

Appending does not solve the problem this initiative exists for. The knowledge-base content is
written for the client accounts our tools run in; layering a project file beneath it still
loads the Command Center persona, which is the thing a developer building the tooling is
trying to get away from. A project that states its own instructions is stating them *instead
of* the org's, not in addition to them.

Precisely: a project-local instructions file replaces everything sourced from the knowledge
base — instructions, personality, tool descriptions, memory, skills. Content that is already
local is unaffected: the IDE's workspace intelligence and session plans are not knowledge-base
content and continue to compose as they do today.

Consequence: the IDE currently appends (`promptLoader.ts:625-634`) and has to change. That is
a separate change from the candidate-ordering bug and is tracked on its own; the ordering fix
stands alone and does not depend on it.

**The duplicated S3 transport is out of scope for this initiative.**

The IDE's `promptLoader.ts` (806 lines) imports nothing from `@nevadoai/sherpa-core` and
reimplements `readS3File`, `loadSkills`, and the bootstrap keys. Per-surface *assembly* is
intentional — `agent-runner.ts:105-112` names the IDE's pipeline as the reason `loadPrompt` is
a port — but the transport beneath it is duplicated and has drifted.

Consolidating it is the right direction and is tracked separately. It stays out of this
initiative deliberately: it is a refactor with no user-visible change, and gating a small
behavioral fix behind a large structural one would delay the thing developers actually need.

## Scope boundary

Not in scope: local overrides for `SOUL.md`, `TOOLS.md`, memory or skills; parent-directory
traversal or a `.sherpa/` fallback; any size cap on the local file; and making the built-in
tool-guidance block itself overridable (it names specific tools that may not match a surface
under development — a real problem, tracked separately).
