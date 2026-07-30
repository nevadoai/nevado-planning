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
| Task | Issue in implementation repo | `sherpa-sdk#55` |

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
| `compliance` | Compliance program, control evidence, audit readiness |

## When to Use What

| Concept | Mechanism | Why |
|---------|-----------|-----|
| **Urgency** | Project board → Priority field (P0–P3) | Single source of truth; filterable in board views |
| **Time-boxing** | Project board → Target Quarter field | Decoupled from milestones so initiatives can span quarters |
| **Initiative grouping** | Milestone (per-repo, named after the initiative) | Built-in GitHub field; links issues to an initiative with a progress bar |
| **Workflow state** | Project board → Status (Todo / In Progress / Done) | Built-in; drives the Kanban columns |
| **Issue classification** | Issue Type (Epic / Task / Bug / Feature) | Built-in org-level field; no labels needed for this |
| **Effort estimate** | Label `size:S` / `size:M` / `size:L` | No built-in equivalent; labels are visible in lists without opening the board |
| **Initiative tag** | Label matching initiative name (e.g. `session-handoff`) | Enables cross-repo filtering by initiative in search/notifications |
| **Owner** | Assignees field | Built-in; assign the person doing the work |

### Principles

1. **Prefer built-in fields over custom.** GitHub's built-in fields (Status, Milestone, Assignees, Labels, Repository, Sub-issues progress) auto-sync and work in search/filters. Only use custom project fields (Priority, Target Quarter) when no built-in covers the need.
2. **No duplication.** Don't use labels for something a project board field already handles. Priority lives on the board — not as `priority:p0` labels.
3. **Labels are for cross-repo filtering and categorization** that doesn't fit a project field. Keep the set small and consistent across repos.
4. **Milestones are per-initiative, not per-quarter.** A milestone tracks all work for one initiative in one repo. Use Target Quarter on the board for time-boxing.

## Labels

Shared across all repos. Keep this set minimal — add a label only when no project field serves the purpose.

| Label | Purpose |
|-------|---------|
| `size:S` / `size:M` / `size:L` | Effort estimate (T-shirt sizing) |
| `<initiative-name>` (e.g. `session-handoff`) | Cross-repo initiative filter |
| `blocked` | Issue cannot progress (explain in a comment) |

Do **not** create labels for: priority (use board field), status (use board field), bug/enhancement/feature (use Issue Type field), or quarter (use board field).

## Project Board Fields

The [Nevado project board](https://github.com/orgs/nevadoai/projects/1) uses these fields:

| Field | Type | Values | When to set |
|-------|------|--------|-------------|
| Status | Built-in | Todo → In Progress → Done | Always; move when work state changes |
| Assignees | Built-in | GitHub users | When someone picks up the work |
| Milestone | Built-in | Initiative name | Always; set at issue creation |
| Labels | Built-in | See above | At creation; update if blocked |
| Repository | Built-in | (auto) | Automatic |
| Sub-issues progress | Built-in | (auto) | Automatic for epics with sub-issues |
| Priority | Custom (single-select) | P0 - Urgent, P1 - High, P2 - Medium, P3 - Low | Always; set at creation, reassess in triage |
| Target Quarter | Custom (single-select) | Q3 2026, Q4 2026, Q1 2027, Q2 2027 | When the work is scheduled |

### Priority Definitions

| Level | UI Label | Meaning | Response |
|-------|----------|---------|----------|
| **P0** | Urgent | System down / data loss / security | Drop everything, fix now |
| **P1** | High | Major functionality broken or blocking others | This week |
| **P2** | Medium | Important but not urgent | This quarter |
| **P3** | Low | Nice to have / tech debt | When capacity allows |

## Working on Items

When you pick up an issue:

1. **Assign yourself** — so others know it's taken.
2. **Move to In Progress** — on the project board, shift the item from Todo to In Progress.
3. **Work on a branch** — branch from the default branch in the implementation repo. Branch name should reference the issue (e.g. `42-resume-foreign-session`).
4. **Open a PR** — use a [closing keyword](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue) in the PR body so the issue closes automatically on merge (`Closes #42` for same-repo, `Closes nevadoai/repo#42` for cross-repo).
5. **Move to Done** — the board should auto-transition when the issue closes. Verify it did.

If you get blocked, add the `blocked` label and leave a comment explaining what's blocking.

### For Agents

```bash
# Assign yourself (use your GitHub username or the bot's)
gh issue edit ISSUE_NUM --repo nevadoai/REPO_NAME --add-assignee "@me"

# Move to In Progress on the board
ITEM_ID=$(gh project item-list 1 --owner nevadoai --format json \
  | jq -r '.items[] | select(.content.number == ISSUE_NUM and .content.repository == "nevadoai/REPO_NAME") | .id')
gh project item-edit --project-id PVT_kwDODJ8T8s4Bd6rR --id "$ITEM_ID" \
  --field-id PVTSSF_lADODJ8T8s4Bd6rRzhYYrnM --single-select-option-id 47fc9ee4

# When done — move to Done
gh project item-edit --project-id PVT_kwDODJ8T8s4Bd6rR --id "$ITEM_ID" \
  --field-id PVTSSF_lADODJ8T8s4Bd6rRzhYYrnM --single-select-option-id 98236657
```

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

   # Set issue type (Task, Bug, or Feature)
   ISSUE_ID=$(gh api graphql -f query='{ repository(owner: "nevadoai", name: "REPO_NAME") { issue(number: ISSUE_NUM) { id } } }' --jq '.data.repository.issue.id')
   gh api graphql -f query="mutation { updateIssue(input: { id: \"$ISSUE_ID\", issueTypeId: \"IT_kwDODJ8T8s4BmeAa\" }) { issue { issueType { name } } } }"

   # Get epic node ID and link sub-issue
   EPIC_ID=$(gh api graphql -f query='{ repository(owner: "nevadoai", name: "nevado-planning") { issue(number: EPIC_NUM) { id } } }' --jq '.data.repository.issue.id')
   gh api graphql -f query="mutation { addSubIssue(input: { issueId: \"$EPIC_ID\", subIssueId: \"$ISSUE_ID\" }) { issue { id } } }"
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

### Setting Project Board Fields

After adding an issue to the board, set its Priority and Target Quarter:

```bash
# Get the project item ID
ITEM_ID=$(gh project item-list 1 --owner nevadoai --format json \
  | jq -r '.items[] | select(.title == "Issue Title") | .id')

# Set Priority (field: PVTSSF_lADODJ8T8s4Bd6rRzhYYs9Q)
# Options: P0/Urgent=b6ff0f5f, P1/High=a1c6a391, P2/Medium=062ebd5f, P3/Low=1221ae12
gh project item-edit --project-id PVT_kwDODJ8T8s4Bd6rR --id "$ITEM_ID" \
  --field-id PVTSSF_lADODJ8T8s4Bd6rRzhYYs9Q --single-select-option-id a1c6a391

# Set Target Quarter (field: PVTSSF_lADODJ8T8s4Bd6rRzhYYtAo)
# Options: Q3_2026=6c0d819b, Q4_2026=00935f23, Q1_2027=49d40e4d, Q2_2027=759e2fe1
gh project item-edit --project-id PVT_kwDODJ8T8s4Bd6rR --id "$ITEM_ID" \
  --field-id PVTSSF_lADODJ8T8s4Bd6rRzhYYtAo --single-select-option-id 6c0d819b
```

### Setting Status

```bash
# Status field: PVTSSF_lADODJ8T8s4Bd6rRzhYYrnM
# Options: Todo=f75ad846, In_Progress=47fc9ee4, Done=98236657
gh project item-edit --project-id PVT_kwDODJ8T8s4Bd6rR --id "$ITEM_ID" \
  --field-id PVTSSF_lADODJ8T8s4Bd6rRzhYYrnM --single-select-option-id 47fc9ee4
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

### Closing an Initiative

When all sub-issues are done:

1. **Close the epic** — close the issue in `nevado-planning`. The board auto-moves it to Done.
2. **Close milestones** — close the matching milestone in each repo that had one.
3. **Leave Target Quarter as-is** — it records when the work was scheduled, not when it finished.

```bash
# Close the epic
gh issue close EPIC_NUM --repo nevadoai/nevado-planning

# Close milestones in each repo
gh api -X PATCH repos/nevadoai/REPO_NAME/milestones/MILESTONE_NUM \
  -f state=closed
```
