# Nevado Planning

This is the planning hub for Nevado AI. It contains no code — only epics, roadmaps, decisions, and design docs.

See `README.md` for the full structure and agent workflow instructions.

## Rules

- One-line commit messages, no body, no attribution footers
- All work on feature branches, never commit to main directly
- Epic issues in this repo use type "Epic" (`IT_kwDODJ8T8s4CGZpg`)
- Implementation issues go in their respective repos, not here
- ADRs are append-only — never edit an accepted decision, write a new one that supersedes it

## Planning System

See `README.md` for the full guide, including:
- **"When to Use What"** — decision table for fields vs labels vs milestones
- **"For Agents"** — complete CLI workflows with field IDs for the project board

Key rules:
- Priority lives on the project board field (P0–P3), not as labels
- Milestones are per-initiative (not per-quarter)
- Target Quarter is a project board field
- Issue Type (Epic/Task/Bug/Feature) is the built-in org field, not a label
- Every issue added to the board must have: Status, Priority, Milestone set
- Labels are only for: size (`size:S/M/L`), initiative tags, and `blocked`

## Commands & Workflows

All `gh` CLI workflows (creating epics, linking sub-issues, setting board fields, working on items) are in `README.md` under **"For Agents"** and **"Working on Items"**.
