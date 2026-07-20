# Session Handoff

**Epic:** [nevado-planning#1](https://github.com/nevadoai/nevado-planning/issues/1)
**Milestone:** Session Handoff
**Initiative:** A3

## Goal

Sessions started on any Sherpa surface (IDE, TUI, Web) can be resumed on any other surface. Users see all active sessions in Command Center and can pick them up from wherever they left off.

## User Stories

| # | Story |
|---|-------|
| 11 | As a **Tyler/operator**, I want to see all active Sherpa sessions (IDE, TUI, Web) in one place in CC so that I know what's running |
| 12 | As a **user**, I want to resume a session I started in the IDE from the Web terminal so that I can continue work from a different device |
| 13 | As a **user**, I want to resume a session I started in the Web from the TUI so that I can pick up work locally |
| 14 | As a **user**, I want sessions to survive runtime restarts so that I don't lose work when the server reboots |

## Current State

| Surface | Session Store | Persists? | Resumable? |
|---------|--------------|-----------|------------|
| **IDE** | SQLite (`.sherpa/sessions.db`) | Yes | Yes (local only) |
| **TUI (local mode)** | `FileSessionStore` (`~/.config/nevado/`) | Yes | Yes (local only) |
| **Runtime server** (TUI remote / Web) | `FileSessionStore` (fixed from InMemory) | Yes | Yes (local only) |

## What Already Works

- IDE broadcasts session metadata to S3 (`sessions/{app}/{id}.json`) — CC renders these as read-only cards
- `AgentEngine.run()` handles resume natively (detects prior messages, fixes dangling tool uses)
- `FileSessionStore` in SDK persists everything needed (sessions.json + messages/{id}.json)
- Protocol types (`AgentSession`, `SessionMessage`) are shared across all packages
- TUI local mode uses `FileSessionStore` — proves the pattern works
- Shared S3 session store implemented (`SessionSyncService` + `SyncedSessionStore`)

## Gaps

- Messages are not broadcast from all surfaces — S3 gets metadata only in some cases
- No shared "adopt/import foreign session" flow on all surfaces
- Workspace state: resuming needs the same files, but no commit SHA or diff is recorded
- Profile config resolution from CC not yet wired up

## Architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│   IDE    │    │   TUI    │    │   Web    │
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     │  broadcast     │  broadcast     │  broadcast
     ▼               ▼               ▼
┌─────────────────────────────────────────────┐
│           S3 Session Bucket                  │
│  sessions/{application}/{session-id}.json    │
│  transcripts/{application}/{session-id}.json │
└─────────────────────────────────────────────┘
     │
     │  reads
     ▼
┌─────────────────┐
│ Command Center  │
│ (session cards) │
└─────────────────┘
```

## Key Design Decisions

- **Push vs pull for messages:** Hybrid — metadata broadcast continuously, full transcript synced to S3, pulled on resume
- **Workspace continuity:** Requires committed state (branch + SHA). Uncommitted changes are the user's responsibility.
- **Conflict resolution:** TBD — what happens if two surfaces try to resume the same session simultaneously?

## Work Items

| # | Work Item | Repo | Issue |
|---|-----------|------|-------|
| A3.1 | Fix runtime to use FileSessionStore | nevado-sherpa-tui | [#41](https://github.com/nevadoai/nevado-sherpa-tui/issues/41) ✅ |
| A3.2 | Define session handoff data requirements | sherpa-sdk | [#55](https://github.com/nevadoai/sherpa-sdk/issues/55) ✅ |
| A3.3 | Canonical message persistence format | sherpa-sdk | [#56](https://github.com/nevadoai/sherpa-sdk/issues/56) ✅ |
| A3.4 | Broadcast messages to S3 | nevado-sherpa-ide | [#88](https://github.com/nevadoai/nevado-sherpa-ide/issues/88) |
| A3.5 | Shared S3-backed session store | sherpa-sdk | [#57](https://github.com/nevadoai/sherpa-sdk/issues/57) ✅ |
| A3.6 | Resume foreign session flow on runtime | nevado-sherpa-tui | [#42](https://github.com/nevadoai/nevado-sherpa-tui/issues/42) |
| A3.7 | CC UI: Resume in Terminal action | command-center | [#445](https://github.com/nevadoai/command-center/issues/445) |
| A3.8 | IDE: resume session from another surface | nevado-sherpa-ide | [#89](https://github.com/nevadoai/nevado-sherpa-ide/issues/89) |
| A3.9 | Workspace state handoff strategy | sherpa-sdk | [#58](https://github.com/nevadoai/sherpa-sdk/issues/58) |

## Open Questions

1. **Conflict resolution:** What happens if two surfaces resume the same session simultaneously? Lock? Last-write-wins? Fork?
2. **Transcript size:** Full transcripts can be large. Do we cap what's stored, or paginate on pull?
3. **Session ownership:** Can any user resume any session, or is there an ACL?
