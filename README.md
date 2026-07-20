# Nevado Planning

Central planning hub for Nevado AI. Epics, roadmaps, architecture decisions, and design docs live here. Implementation issues live in their respective repos; this repo provides the cross-cutting view.

## Quick Links

- [Project Board](https://github.com/orgs/nevadoai/projects/1) — Kanban view of all active work
- [Epics](https://github.com/nevadoai/nevado-planning/issues) — Cross-repo initiatives tracked as issues (type: Epic)
- [Org-wide issues](https://github.com/issues?q=org%3Anevadoai+is%3Aopen) — All open issues across Nevado repos

## Repository Structure

```
nevado-planning/
├── roadmap/              # Strategic direction
│   ├── 2026-2027.md      # Master roadmap
│   └── quarterly/        # Quarter-level breakdowns
├── epics/                # Context docs for each epic (goals, constraints, open questions)
├── decisions/            # Architecture Decision Records (ADRs)
└── design/               # Technical design documents
```

## How It Works

### Hierarchy

| Layer | Where | Example |
|-------|-------|---------|
| Roadmap | `roadmap/` docs | Strategic themes and timelines |
| Epic | Issue in this repo (type: Epic) | "Session Handoff" |
| Milestone | Per-repo milestone matching the epic | `sherpa-sdk` milestone "Session Handoff" |
| Story/Task | Issue in implementation repo | `sherpa-sdk#55` |

### Epics

An epic is a GitHub issue in this repo with:
- **Type:** Epic
- **Milestone:** Named after the initiative (e.g. "Session Handoff")
- **Sub-issues:** Linked implementation issues in other repos
- **Context doc:** A matching file in `epics/` with goals, constraints, and open questions

### Milestones

Milestones are per-initiative, not per-quarter. Each implementation repo gets a milestone matching the epic name. Use the Project board's "Target Quarter" field for time-boxing.

### Decisions (ADRs)

Numbered, append-only records of architectural choices. Format:

```
decisions/NNN-short-title.md
```

Each ADR has: Context, Decision, Consequences. Once written, they don't change — if a decision is reversed, write a new ADR that supersedes it.

### Design Docs

Longer-form technical documents for significant features or system-level architecture. These are living documents — update them as the system evolves.

## Implementation Repos

| Repo | Domain |
|------|--------|
| `sherpa-sdk` | Shared SDK packages (protocol, core, commands) |
| `nevado-sherpa-tui` | TUI client and Fastify runtime server |
| `nevado-sherpa-ide` | VS Code extension |
| `command-center` | Web app, backend, infrastructure |
| `nevado-customer-provisioner` | Customer AWS account provisioning |

## Labels

Shared label taxonomy across repos:

| Label | Purpose |
|-------|---------|
| `priority:p0` / `p1` / `p2` | Urgency |
| `size:S` / `M` / `L` | Effort estimate |
| `session-handoff` | Initiative tag (one per epic) |

## Project Board Fields

| Field | Type | Purpose |
|-------|------|---------|
| Status | Built-in | Todo → In Progress → Done |
| Priority | Custom (single-select) | P0, P1, P2, P3 |
| Target Quarter | Custom (single-select) | Q3 2026, Q4 2026, Q1 2027, Q2 2027 |
| Milestone | Built-in | Initiative grouping |
| Repository | Built-in | Which repo the issue belongs to |
| Sub-issues progress | Built-in | Completion percentage for epics |

---

## For Agents

This section describes how to work with this planning system programmatically.

### Workflow: Roadmap → Planning → Issues

1. **Roadmap → Epics:** Break a roadmap initiative into an epic.
   ```bash
   # Create the epic in this repo
   gh issue create --repo nevadoai/nevado-planning \
     --title "Epic Title" \
     --milestone "Initiative Name" \
     --body "## Goal\n\n...\n\n## Surfaces\n\n...\n\n## Sub-issues\n\nTracked as sub-issues linked from implementation repos."
   ```

   Then set the issue type to Epic:
   ```bash
   # Get the issue node ID
   ISSUE_ID=$(gh api graphql -f query='{ repository(owner: "nevadoai", name: "nevado-planning") { issue(number: NUMBER) { id } } }' --jq '.data.repository.issue.id')

   # Set type to Epic
   gh api graphql -f query="mutation { updateIssue(input: { id: \"$ISSUE_ID\", issueTypeId: \"IT_kwDODJ8T8s4CGZpg\" }) { issue { issueType { name } } } }"
   ```

2. **Epic → Implementation issues:** Create issues in the relevant repos, then link as sub-issues.
   ```bash
   # Create issue in implementation repo
   gh issue create --repo nevadoai/REPO_NAME \
     --title "Task title" \
     --milestone "Initiative Name" \
     --label "initiative-label" \
     --body "## Context\n\n...\n\n## Acceptance Criteria\n\n- [ ] ..."

   # Get both node IDs
   EPIC_ID=$(gh api graphql -f query='{ repository(owner: "nevadoai", name: "nevado-planning") { issue(number: EPIC_NUM) { id } } }' --jq '.data.repository.issue.id')
   SUB_ID=$(gh api graphql -f query='{ repository(owner: "nevadoai", name: "REPO_NAME") { issue(number: ISSUE_NUM) { id } } }' --jq '.data.repository.issue.id')

   # Link sub-issue to epic
   gh api graphql -f query="mutation { addSubIssue(input: { issueId: \"$EPIC_ID\", subIssueId: \"$SUB_ID\" }) { issue { id } } }"
   ```

3. **Add to Project Board:**
   ```bash
   gh project item-add 1 --owner nevadoai --url "https://github.com/nevadoai/REPO_NAME/issues/NUMBER"
   ```

### Creating a Milestone

Milestones are per-initiative per-repo. Create in both planning and implementation repos:
```bash
# In the planning repo
gh api repos/nevadoai/nevado-planning/milestones \
  -f title="Initiative Name" \
  -f description="One-line description" \
  -f state=open

# In each implementation repo that has work for this initiative
gh api repos/nevadoai/REPO_NAME/milestones \
  -f title="Initiative Name" \
  -f description="One-line description" \
  -f state=open
```

### Issue Types (Org-wide)

| Type | ID | Use for |
|------|----|---------|
| Epic | `IT_kwDODJ8T8s4CGZpg` | Initiatives in nevado-planning |
| Task | `IT_kwDODJ8T8s4BmeAa` | Implementation work |
| Bug | `IT_kwDODJ8T8s4BmeAb` | Defects |
| Feature | `IT_kwDODJ8T8s4BmeAc` | New capabilities |

### Setting Priority on the Project Board

```bash
# Get the project item ID after adding
ITEM_ID=$(gh project item-list 1 --owner nevadoai --format json | jq -r '.items[] | select(.title == "Issue Title") | .id')

# Get the Priority field ID and option ID
# Priority field and options can be queried from the project schema
gh project field-list 1 --owner nevadoai --format json
```

### Checklist: Creating a New Initiative

- [ ] Create `epics/initiative-name.md` with context doc
- [ ] Create epic issue in `nevado-planning` with type Epic
- [ ] Create milestone "Initiative Name" in planning + implementation repos
- [ ] Create initiative label (e.g. `initiative-name`) in implementation repos
- [ ] Break down into implementation issues in relevant repos
- [ ] Link implementation issues as sub-issues of the epic
- [ ] Add all issues to the Project board
- [ ] Set Priority and Target Quarter on board items
