# Decisions (ADRs)

Architecture Decision Records — numbered, append-only records of significant technical choices.

## Format

```
NNN-short-title.md
```

## Template

```markdown
# NNN. Short Title

**Status:** Accepted | Superseded by [NNN](./NNN-short-title.md)
**Date:** YYYY-MM-DD

## Context

What is the issue that we're seeing that is motivating this decision?

## Decision

What is the change that we're proposing and/or doing?

## Consequences

What becomes easier or more difficult to do because of this change?
```

## Rules

- Once accepted, ADRs are immutable. If a decision is reversed, write a new ADR that supersedes it.
- Number sequentially. Don't reuse numbers.
- Keep it short — one page max.
