# Git Worktree Isolation

**Epic:** TBD (create in `nevado-planning`)
**Milestone:** Git Worktree Isolation
**Initiative:** TBD (follows A3 — see Dependencies)

> **A3** = [Session Handoff](session-handoff.md), the initiative this epic follows.

## Goal

Give Sherpa surfaces native git worktree support so concurrent sessions against the same repo never share git state — and so a session's uncommitted workspace state becomes a capturable, restorable artifact instead of being lost outside of committed history.

## Dependencies

**Starts after Session Handoff (Initiative A3) is complete.** A3 is still open (6/9 `sherpa-sdk` sub-issues closed as of 2026-08-12; several open across `nevado-sherpa-tui`, `nevado-sherpa-ide`, `command-center`). This epic is not scheduled concurrently with A3.

A3.9 ("Workspace state handoff strategy," `sherpa-sdk#58`) will close using A3's existing committed-state approach (branch + SHA; uncommitted changes are the user's responsibility). This epic revisits that limitation for same-machine resume only — it does not block A3's closure. Cross-machine resume of uncommitted work is a permanent out-of-scope boundary for both epics (see Scope). Add a forward-reference note on A3.9 pointing here once this epic exists as an issue.

## Problem

Today, surfaces that operate directly on a developer's on-disk checkout (IDE, TUI local mode) share a single working directory per repo. Two concurrent sessions against the same repo — e.g., two VS Code windows, or an IDE session and a local TUI session — have no git-level separation: checkout, staged changes, and uncommitted edits collide in the same directory.

Separately, the Runtime server (backing TUI remote and Command Center's embedded SherpaTUI) already isolates concurrent sessions, but via a full `git clone` per session (`design/runtime-native-sdlc.md`), not a worktree. This is heavier than necessary and doesn't share the local object store, which matters as concurrent session count grows past the current cap of 5.

Neither mechanism produces something A3.9 can use for handoff: today, resuming a session on another surface only carries committed state (branch + SHA). Uncommitted work has no representation and is silently dropped.

## Decisions

### Trigger model

- **Resume → always automatic, no user choice.** Resuming a session always uses its worktree. If the original worktree was pruned (see Open Questions — Lifecycle, still unresolved), resume recreates it via `git worktree add` against the session's known branch name — branches survive worktree removal, so this is a plain re-add, not a special case.
- **New work → on-demand, never forced, never auto-detected.** No collision detection — a user's/agent's request is the only trigger. New work has no existing branch to anchor to, so starting it requires a name (often a ticket reference — Jira, Linear, etc.) that becomes the worktree's branch identifier. Two ways to trigger it:
  - **Human-invoked:** `/worktree <name>` (see Command shape below).
  - **Agent-invoked:** a tool definition the agent can call on its own judgment, with system prompt guidance on when isolation is appropriate (genuinely separate/parallel work, risky/experimental changes).
- **Agent-initiated creation defaults to asking the user first**, consistent with the existing tool permission-mode pattern (ask / always-allow) — configurable per user for those who want the agent to just act without confirming.
- **Moving existing work into a worktree is not a plain `git worktree add`.** That command only checks out committed history — it does not carry uncommitted changes. Isolating mid-session requires: capture the uncommitted diff (stash), create the worktree, reapply the diff inside it. Committed-but-unpushed commits on the branch being left behind are already safe and stay untouched — but the tool must surface that explicitly ("branch X has 3 unpushed commits, left as-is") so the user isn't left wondering where that work went.
- **New worktrees are always derived from the repo's actual default branch, detected dynamically** (via `origin/HEAD` or the GitHub API's `defaultBranchRef`) — never hardcoded as `main`. The base is a freshly-fetched `origin/<default>`, not the local copy, to avoid stale local state or stray unpushed commits sitting on a local default branch.
- **Sub-agent isolation within a single session is out of scope.** That's the orchestrating model's responsibility, not this epic's.

### Command shape

Two flat commands, not a subcommand family — deliberately rejected a `/cycle`-style `start`/`status`/`list` shape (that pattern doesn't exist anywhere yet either; it would be new infrastructure, not reuse, per `roadmap/quarterly/2026-Q3.md:155`, which is itself only planned, unbuilt work):

- **`/worktree <name>`** — create. The name is required and becomes the branch identifier.
- **`/worktree-list`** — an interactive, navigable list (not a static printout) showing every worktree for the repo, current session's row included. Status (clean / in-progress / needs-attention) shown per row; an inline action removes a flagged row (confirm-if-dirty, same shape as the Command Center "Clean up" flow). This single command absorbs what would otherwise have been separate `status` and `remove` commands — there's no independent lookup or cleanup verb.
- **Why not `/worktree list`:** collapsing verb and argument into one slot means `list` becomes a reserved word in the name position — a worktree genuinely named `list` would silently misfire. A separate flat command has no such collision, for any name.
- **Why not subcommand parsing generally:** with only one real create-verb and one view-and-act surface, there's no repeated shape worth building shared parsing/hinting infrastructure for. Two flat commands cost nothing new to build (same pattern as `/audit`, `/discover`).
- **Open implementation question:** whether Sherpa TUI already has an interactive-selectable-list-with-inline-actions component to reuse for `/worktree-list`, or whether this is new component work, is unconfirmed — check before scoping this as cheap.

### Lifecycle

- **Dirty check is easy and reliable:** `git status` on a worktree's path tells us definitively whether it has uncommitted/untracked changes — unlike "did the session end," this isn't ambiguous.
- **Explicit quit is the reliable trigger.** When a user runs an explicit exit action, the surface controls that moment directly and can check synchronously before exiting: clean → auto-remove silently; dirty → prompt keep-or-remove. `/exit` (TUI's existing quit command) is the concrete home for this on TUI; IDE needs an equivalent explicit "end session" action, not just the window closing.
- **Everything else falls to a lazy sweep.** Window close, crash, `kill -9` — none give a reliable "session ended" signal to intercept. Rather than chase that, a lazy sweep (run whenever something else relevant happens, e.g. a new worktree is requested for the same repo) prunes worktrees that are clean and orphaned. Dirty-and-orphaned worktrees are left alone and flagged for attention. Refining cleanup for the crash/kill case specifically is explicitly not a priority right now.
- **`/new` is a no-op for worktrees.** Starting a fresh session doesn't quit anything, so there's nothing to reconcile — the previous worktree just isn't the *current* one anymore, still intact, still checkable later via `/exit` or the lazy sweep. Treating it like quit would mean removing a worktree (a real git op) and then needing to relocate the session's CWD somewhere — the obvious fallback, the repo root, is exactly the shared/unisolated state this epic exists to avoid, which would mean un-isolating a session at the moment it starts fresh work. The two commands simply compose instead: `/new` to start fresh, then `/worktree new-feature-work` if/when that work should be isolated.

### Runtime workspace sync

- **One shared repo copy per app, worktrees carved from it** — replaces today's full-clone-per-session. Created lazily on first session.
- **Fetch as late as practically possible** — right before a worktree's base ref is chosen, not at session start, to minimize (not eliminate) staleness.
- **Accepted limitation, not solved:** if an agent explores for a while before deciding to isolate, the shared copy can still be stale relative to the remote by the time a worktree is actually created. This is inherent to any fetch-then-use-later approach — no fetch timing eliminates it. Not chasing a fix for this now.

### Branch collisions

Git refuses to check out a branch in a worktree if that branch is already checked out somewhere else — including the repo's own root/main checkout, which counts as a worktree in git's bookkeeping. This isn't inherently a problem for us (new work already gets a unique name per the naming convention), but it can still happen and must not surface as a raw git error. Two cases, two messages:

- **Branch already checked out in another worktree** — that worktree is effectively an active/orphaned session for that same branch. Surface its location and suggest resuming/reusing it instead of creating a duplicate.
- **Branch already checked out at the repo's root** (the user's plain, non-worktree checkout) — surface that the root is currently on that branch, and suggest either picking a different name for the new worktree or switching the root checkout first.

## User Stories

| # | Story |
|---|-------|
| 1 | As a developer, I want to run two independent Sherpa sessions (e.g. two VS Code windows) against the same repo without their git state colliding, so I can work on two features in parallel |
| 2 | As a developer, I want to run `/worktree <name>` to isolate a session into its own worktree, naming it (e.g. with a ticket reference), so I control when isolation starts and what it's called |
| 3 | As a Sherpa agent, I want a worktree-creation tool available so I can isolate work on my own judgment when the situation calls for it, not only when the user remembers to ask |
| 4 | As a developer, I want to control whether the agent asks before creating a worktree or just does it, so I can tune the friction to my preference |
| 5 | As a developer, when I isolate an in-progress session, I want my uncommitted changes to move with me into the new worktree, so I don't strand in-progress work in the old checkout |
| 6 | As a developer, I want new worktrees to branch off my repo's actual default branch, not an assumed `main`, so I never end up isolated from the wrong base |
| 7 | As a developer, when I resume a session, I want its worktree to come back automatically — recreated if it was cleaned up — so resuming never requires me to think about git |
| 8 | As an operator, I want concurrent Runtime-backed sessions (SherpaTUI in Command Center) to use lightweight worktrees off a shared repo instead of full clones, so more sessions can run concurrently without linear disk/clone-time cost |
| 9 | As a developer, I want to see which sessions own which worktrees for a repo via `/worktree-list` — not a dedicated Command Center or IDE view — so I can understand what's isolated where and clean up when done |
| 10 | As a Sherpa session, I want my uncommitted workspace changes captured as part of session state (not just branch + SHA), so a same-machine resume has something to restore beyond committed history (cross-machine resume is out of scope — see Scope) |

### Acceptance Criteria (stories 2–7, 9)

**#2 — `/worktree <name>`**
- Requires a name argument; the resulting branch/worktree is named from it.
- Always creates a new worktree when invoked — no collision check, no "already isolated" ambiguity (branch-collision handling per Decisions still applies if the name collides with an existing branch).

**#3 — Agent worktree tool**
- A tool definition for worktree creation is available to the agent, separate from the human-facing command.
- System prompt guidance describes when to consider it: genuinely separate/parallel work, risky/experimental changes.

**#4 — Ask vs. always-allow setting**
- Default: agent-initiated worktree creation prompts for user confirmation before acting.
- A setting exists to switch this to always-allow.
- The human-invoked command (#2) is never gated by this setting — it's already an explicit request.

**#5 — Carry uncommitted changes on isolate**
- Uncommitted changes present at the moment of isolation are captured (stash/diff) before the worktree is created.
- That diff is reapplied inside the new worktree after creation.
- Committed-but-unpushed commits on the branch being left are not touched, and the tool explicitly reports that they were left in place.

**#6 — Default branch detection**
- Default branch is resolved dynamically per repo (`origin/HEAD` or GitHub API), never hardcoded as `main`.
- New worktree is created from a freshly-fetched `origin/<default>`, not a local copy.

**#7 — Resume recreates worktree**
- If a session's worktree still exists, resume reuses it directly.
- If it was pruned, resume recreates it via `git worktree add` against the session's known branch name, with no user action required.

**#9 — `/worktree-list` (TUI)**
- Interactive, navigable list of every worktree for the repo — not a static printout.
- Each row shows status (clean / in-progress / needs-attention); the current session's own worktree is included and identifiable.
- A flagged (dirty + orphaned) row has an inline remove action, with a confirm step before discarding uncommitted changes.
- No separate `status` or `remove` commands exist — this list is the only surface for both.

## Scope

**In scope:**
- Worktree-per-session isolation on local surfaces (IDE, TUI local mode)
- Worktree-per-session on the Runtime server, replacing full-clone-per-session
- Automatic worktree use on session resume (recreated if pruned)
- On-demand worktree creation for new work — human-invoked command and agent-invoked tool, both requiring a name/identifier for the new branch
- Ask-vs-always-allow permission setting for agent-initiated worktree creation
- Uncommitted-diff stash/reapply flow so isolating mid-session carries in-progress work into the new worktree
- Dynamic default-branch detection (never hardcoded `main`), basing new worktrees on a freshly-fetched `origin/<default>`
- Worktree state (branch + uncommitted diff) exposed in a form the `sherpa-sdk` protocol can carry, for same-machine consumers only

**Deliberately not building:**
- Dedicated worktree UI in Command Center or IDE. A live session's worktree status isn't decision-relevant (whoever's in the session already knows what they've changed) — the only case with real decision value is an orphaned, dirty worktree needing keep-or-discard, and that's already covered by `/worktree-list` (see Command shape) wherever a Sherpa command surface runs — including Command Center's own embedded terminal (`@nevadoai/sherpa-web`), which reaches Runtime-hosted worktrees that would otherwise be unreachable from any local surface. Worktrees stay tooling, not a native CC/IDE view. Revisit only if a concrete glanceable-without-a-terminal need shows up later.

**Permanently out of scope (not deferred, not planned under A3.9 either):**
- Shipping a worktree's uncommitted diff across machines (e.g., IDE session on a laptop → resumed in Web/CC on a different machine). Uncommitted files are out of scope for cross-machine resume, full stop — this epic makes uncommitted state capturable on the machine that holds it, but no epic currently plans to transmit that diff to a different machine's filesystem.

> **Local ↔ Runtime resume requires committing WIP — permanently.** Local surfaces (IDE, TUI local) hold their worktree on the developer's machine; Runtime-backed surfaces (TUI remote, Web/CC's SherpaTUI) hold theirs on the Runtime server's disk. This epic doesn't move a worktree's uncommitted diff between those two filesystems, and no follow-on is planned to do so. So resuming a **local** session on a **Runtime** surface, or vice versa, always loses uncommitted work unless it's committed first — this is a permanent commit-first boundary, not a gap to be closed later. This epic only removes uncommitted-state loss when resume stays within the same machine (e.g., one local surface picking up another's worktree).

**Cross-boundary warning — warn, don't block.** When a resume crosses the local↔Runtime boundary and the source worktree has uncommitted changes, the resuming surface warns (e.g. "3 files changed on \[surface] won't be included here") but doesn't prevent the resume — nothing is actually destroyed, the original changes stay intact on the source machine, this is purely informational. Session Handoff (A3) may already have relevant mechanisms in place (e.g., its existing continuous metadata broadcast) that this can fold into — verify against A3's actual implementation when this epic starts, rather than assuming new infrastructure is needed.

## Open Questions

None remaining — trigger model, lifecycle (including `/new`), Runtime workspace sync, the cross-boundary warning, and UI depth are all resolved. See Decisions and Scope above.

## Work Items (draft — not yet filed)

| Work Item | Repo | Notes |
|-----------|------|-------|
| Worktree management primitives (create/list/remove) | `sherpa-sdk` | Core logic + protocol types for session-to-worktree mapping |
| Default-branch detection + fetch-based base | `sherpa-sdk` | Resolve `origin/HEAD`/GitHub `defaultBranchRef` dynamically; base new worktrees on a fresh `origin/<default>` |
| Uncommitted-diff stash/reapply on isolate | `sherpa-sdk` | Mid-session move into a worktree; surfaces unpushed-commit state explicitly |
| Resume-recreates-worktree logic | `sherpa-sdk` | Re-add via known branch name if the worktree was pruned |
| Branch-collision detection + messaging | `sherpa-sdk` | Replace git's raw error with a clear message + suggested fix (resume existing worktree, or rename, or switch root) |
| `/worktree <name>` command | `sherpa-sdk` (`sherpa-commands`) | Flat command, required name argument; no new parsing infra needed |
| `/worktree-list` interactive command | `nevado-sherpa-tui` (+ `sherpa-commands` for registration) | Navigable list + inline remove action; folds in status/remove; TUI component reuse unconfirmed — verify before scoping as cheap |
| Agent worktree tool + system prompt guidance | `sherpa-sdk` | Agent-invoked path; ties into ask/always-allow permission setting |
| Ask-vs-always-allow permission setting | `sherpa-sdk` + surfaces | Protocol/setting in SDK; each surface implements the confirmation prompt |
| Local TUI worktree integration | `nevado-sherpa-tui` | Local mode uses worktree management primitives |
| IDE worktree integration (multi-window isolation) | `nevado-sherpa-ide` | New-session and resume paths use worktree management primitives |
| Runtime server: bare-repo + worktree-per-session | `nevado-sherpa-tui` | Replaces `POST /v1/workspace/sessions` full-clone path |
