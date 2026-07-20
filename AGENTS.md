# Nevado Planning

This is the planning hub for Nevado AI. It contains no code — only epics, roadmaps, decisions, and design docs.

See `README.md` for the full structure and agent workflow instructions.

## Rules

- One-line commit messages, no body, no attribution footers
- All work on feature branches, never commit to main directly
- Epic issues in this repo use type "Epic" (`IT_kwDODJ8T8s4CGZpg`)
- Implementation issues go in their respective repos, not here
- ADRs are append-only — never edit an accepted decision, write a new one that supersedes it

## Key Commands

```bash
# List epics
gh issue list --repo nevadoai/nevado-planning

# List all open issues across the org
gh issue list --repo nevadoai/nevado-planning --state open

# Add an issue to the project board
gh project item-add 1 --owner nevadoai --url "https://github.com/nevadoai/REPO/issues/NUMBER"

# Link a sub-issue to an epic
gh api graphql -f query='mutation { addSubIssue(input: { issueId: "EPIC_NODE_ID", subIssueId: "ISSUE_NODE_ID" }) { issue { id } } }'
```
