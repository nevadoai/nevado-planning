# Handoff: Data Sharing Warning for Model Picker (Web)

## Context

PR #40 in sherpa-sdk added `requiresDataSharing?: boolean` to `ModelEntry` in `packages/core/src/models.ts`. Claude Fable 5 was changed from `unavailable: 'Revoked...'` to `requiresDataSharing: true` — meaning it now appears in model pickers as a selectable option.

The TUI (agent-workflow PR #38) implements the data sharing warning UX. The web package (`@nevadoai/sherpa-web`) needs the same treatment.

## What "requiresDataSharing" means

Models with this flag require the AWS Bedrock account to have `provider_data_share` retention enabled. This means:

- Prompts & outputs are shared with the model provider (e.g. Anthropic) for trust/safety review
- Retention period: up to 30 days
- Purpose: abuse detection, not training
- The user must explicitly opt in at the AWS account level before the model is invocable

## TUI implementation (reference)

**File:** `agent-workflow/apps/tui/src/components/ModelPicker.tsx`

### Model list row
- Models with `requiresDataSharing` render their label in **yellow** with `underline` when focused
- Suffix badge: `⚠ provider safety review` in yellow
- Other models keep default styling (cyan on focus)

### Confirmation dialog
When user selects a data-sharing model, a confirmation replaces the list content (same container, no unmount):

```
⚠ This model shares prompts & outputs with the
  model provider for safety review (30 day retention).
  Not used for training.
Proceed? [y] yes  [n] back  [Esc] cancel
```

- `y` — confirms selection, calls `onSelect(modelId)`
- `n` — returns to model list (preserves cursor position)
- `Esc` — dismisses the entire picker

### Status bar
When the active model has `requiresDataSharing`, the model name in the status bar renders in **yellow** with a `⚠` prefix:

```
mode: do · ⚠ Claude Fable 5 · workspace: ~/project
```

Instead of the normal dimmed style.

### Helper function

```typescript
export function modelRequiresDataSharing(modelId: string): boolean {
  return MODELS.find(m => m.id === modelId)?.requiresDataSharing === true;
}
```

Used by the status bar component to derive the warning state from `modelId` (no extra prop threading needed).

## What the web package needs

**File:** `packages/web/src/components/ModelPicker.tsx` (or equivalent)

1. **Row styling** — Yellow text + warning badge for `requiresDataSharing` models
2. **Confirmation step** — Before switching, show a dialog/modal explaining data sharing. Same copy as TUI.
3. **Status indicator** — Wherever the active model name is displayed, show it in yellow/warning style with ⚠ prefix when data sharing is active

**File:** `packages/web/src/SherpaTUI.tsx`

The `handleModelSelect` callback should gate on `requiresDataSharing` before calling through — or the ModelPicker component itself should handle confirmation internally (as the TUI does).

## Design decisions made

- **Provider-agnostic wording** — "model provider" not "Anthropic", in case non-Anthropic models get the flag
- **Not a blocker** — user can still select the model after confirmation, it's informed consent
- **Escape dismisses entirely** — don't trap the user in a two-step escape
- **Derive from modelId** — don't thread a `dataSharing` boolean prop; derive it where needed from `modelRequiresDataSharing(modelId)`
