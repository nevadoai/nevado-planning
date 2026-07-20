# 001. Use GitHub Issues as Project Tracker Instead of Jira

**Status:** Accepted
**Date:** 2026-07-15

## Context

Nevado needed a project tracking system for cross-repo initiative planning. The team is one person + agents, all work lives in GitHub, and the existing workflow already uses `gh` CLI heavily.

Options considered:
- Jira — full-featured but separate system, context-switching overhead, not accessible to agents via CLI
- Linear — modern but same separation problem
- GitHub Issues + Projects — native to where code lives, CLI-scriptable, agents can create/query/update issues programmatically

## Decision

Use GitHub Issues with:
- A dedicated `nevado-planning` repo for epics (cross-repo initiatives)
- Implementation issues in their respective repos
- Sub-issues linking implementation work to epics
- Shared labels (`priority:p0/p1/p2`, `size:S/M/L`, initiative tags)
- Per-initiative milestones (not per-quarter)
- An org-level GitHub Project board for kanban/status tracking
- Custom issue type "Epic" for initiative-level issues

## Consequences

- **Easier:** Agents can create, query, and manage issues via `gh` CLI without authentication to a separate system. Everything stays in one ecosystem.
- **Easier:** No context switching between code and tracker — issues, PRs, and code are cross-linked natively.
- **Easier:** Milestones per-initiative give clear progress bars without needing sprints.
- **Harder:** GitHub's filtering across repos is weaker than Jira's JQL. Org-wide queries require search syntax (`org:nevadoai label:X`).
- **Harder:** No native dependency graph or blocking visualization. Cross-links are manual text references.
- **Harder:** Labels must be created in each repo separately (no org-wide label management).
