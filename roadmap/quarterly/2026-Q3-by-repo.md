# Nevado Q3 2026 — Roadmap Work by Repo/Project

**Generated:** 2026-07-01 | **Quarter ends:** 2026-07-31 (1 month remaining)

---

## Status Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Complete |
| 🟡 | In progress / partially complete |
| ⬜ | Not started |

---

## 1. `sherpa-sdk` — SDK Stabilization

**Roadmap section:** Sherpa IDE + SDK — "The Hands"
**Vision:** Lock the protocol contract between IDE, TUI, and Command Center as the foundation for everything else.

| # | Work Item | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Lock protocol contract — version `@nevadoai/sherpa-protocol` as stable API surface | ✅ | PR #49: model registry moved to protocol (breaking v2). Types cover WebSocket messages, REST API, settings, sessions, models. All consumers upgraded. |
| 2 | CC integration hooks — `cycle.start`, `memory.read`, `knowledge.submit` message types | 🟡 ~50% | Settings API + CommandCenterProfile types exist (PR #41). Domain-specific cycle/memory/knowledge messages not yet defined. |
| 3 | Protocol contract documentation — one-page consumer doc | ⬜ | — |

**Key PRs (June–July 2026):**
- #49 `feat!: move model registry to @nevadoai/sherpa-protocol`
- #41 `feat: add settings API support and export isAutoApprovedCommand`
- #40 `feat(core): add requiresDataSharing flag; re-enable Fable 5`
- #33 `feat: add @nevadoai/sherpa-web package`
- #31 `feat(protocol): add typed sub-agent and engine event messages`

---

## 2. `command-center` — Self-Service Cycles + Agent Admin

**Roadmap section:** Command Center — "The Brain"
**Vision:** Remove Tyler as bottleneck. Make cycle initiation, agent configuration, and context ingestion self-service from the UI.

| # | Work Item | Status | Evidence |
|---|-----------|--------|----------|
| 4 | Self-service cycle initiation API — backend endpoint to start a cycle from UI | ⬜ | `continueCycle.js` exists for in-progress cycles; no UI-triggered start endpoint |
| 5 | Cycle initiation UI — select app, define scope, trigger, show progress | ⬜ | — |
| 6 | Agent admin UI — CRUD for personas, skills, memory | ⬜ | Only `pages/admin/QAAgent.jsx` exists; no general persona/skills/memory management |
| 7 | Upload Portal → CC integration — docs auto-feed into project context | ⬜ | `doc-parser/` and `fileManager/` exist but aren't wired to `agentDrivenOrchestrator` |
| 8 | Spreadsheet forensics automation — Excel → extract rules → draft spec | ⬜ | `doc-parser/` has Python parsers; no spreadsheet-specific extraction |

**Key PRs (June–July 2026):**
- #372 `feat: integrate @nevadoai/sherpa-web popout for terminal sessions`
- #367 `Make provisioner app-templates fetch layout-agnostic`
- #365 `Reconcile CloudFront log delivery drift`
- #364 `Reconcile Lambda state drift before apply in customer deploy`

**Note:** Recent CC work focused on infrastructure stability and Sherpa Web integration — not yet on self-service features.

---

## 3. `nevado-sherpa-ide` — Command Center Integration

**Roadmap section:** Sherpa IDE + SDK — "The Hands"
**Vision:** Close the loop — Sherpa can trigger cycles, read CC memory, and submit learnings back.

| # | Work Item | Status | Evidence |
|---|-----------|--------|----------|
| 9 | Trigger cycles from IDE — call CC API to start cycles from Sherpa | ⬜ | — |
| 10 | Read CC memory from IDE — surface agent memory/knowledge in context | ⬜ | — |
| 11 | Submit to knowledge base — push learned patterns back to CC | ⬜ | — |

**Key PRs (June–July 2026):**
- #64 `v1.10.13: bump @nevadoai packages to v2; import models from protocol`
- #62 `v1.10.12: deeper throttle backoff + clearer rate-limit message`
- #60 `v1.10.10: gate Fable 5 on account provider_data_share retention`
- #57 `v1.10.7: bound command/search output buffering to prevent ext-host OOM`
- #52 `v1.10.2: session concurrency + running indicators + CDP test engine`
- #51 `v1.9.0: Browser MCP Scripting Engine (/test)`

**Note:** IDE work in June/July focused on stability (OOM, stream fixes), model support (Fable 5), and protocol v2 adoption. CC integration is unblocked now that protocol is stable.

---

## 4. `nevado-requirements` — Formalize PAS + Claims

**Roadmap section:** Requirements IP Library — "The Knowledge"
**Vision:** The deepest codified library of insurance product specs. Makes "days not months" possible because you never start from zero.

| # | Work Item | Status | Evidence |
|---|-----------|--------|----------|
| 12 | PAS shared core extraction — common domains into `core/` vs `variants/` | ⬜ | 28 domain folders exist under `policy-admin-system/domains/` but no core/variant separation |
| 13 | Per-variant delta specs — document what each PAS variant overrides | ⬜ | — |
| 14 | Claims product spec — expand beyond current 6 domains (fnol, investigation, litigation, payments, reserving) | ⬜ | — |
| 15 | Claims adapter pattern — integration spec for PAS and external TPAs | ⬜ | — |

**Current state:**
- PAS: 28 domains defined (accounting, billing, rating-engine, renewals, underwriting, etc.)
- Claims: 6 domains defined (fnol, investigation, litigation, payments, reserving + UI screens)
- No commits since before June 2026

---

## 5. `agent-workflow` — Multi-Tenant Runtime + Claims Agent

**Roadmap section:** Embedded Business Agents — "The Wedge"
**Vision:** Domain-aware AI agents deployed into customer's tools, owned by the customer, operated by Nevado. Land at $5-15K/mo, expand to $350K+/yr.

| # | Work Item | Status | Evidence |
|---|-----------|--------|----------|
| 16 | Multi-tenant persona/skills/memory separation | 🟡 ~40% | SettingsStore for per-session config (PR #38), AgentEngine with typed events (PR #29), GitHub App tokens per workspace (PR #16). Persona/skills/memory isolation not yet formalized. |
| 17 | Customer-account deployment pattern — runtime in customer's AWS | ⬜ | Infrastructure TF exists but targets Nevado account only |
| 18 | Productized Claims Agent — repeatable claims-intake from EMA pattern | ⬜ | — |
| 19 | Claude Tag / Slack-native delivery | ⬜ | — |

**Key PRs (June–July 2026):**
- #39 `feat: bump @nevadoai packages to v2; import models from protocol`
- #38 `feat: runtime-authoritative auto-approve using SettingsStore`
- #37 `fix(runtime): fall back to default branch when explicit branch clone fails`
- #36 `fix: upgrade to Ink v7 with incremental rendering`
- #29 `feat: migrate to AgentEngine with typed protocol events`
- #26 `feat: migrate to @nevadoai/sherpa-sdk packages`

**Note:** Strong foundation work (protocol migration, SettingsStore, Ink v7). Ready for multi-tenant formalization.

---

## 6. `nevado-customer-provisioner` + `aws-nevado-management` — SOC2 Readiness

**Roadmap section:** Compliance & Security — "The Trust Layer"
**Vision:** Provable, auditable, continuously-enforced compliance. SOC2 Type II report target: August 2026.

| # | Work Item | Status | Evidence |
|---|-----------|--------|----------|
| 20 | SOC2 control gap analysis — map infra controls to Type II requirements | ⬜ | Existing TF covers Organizations, SCPs, CloudTrail, Config, KMS, audit logs — but no gap mapping |
| 21 | Evidence collection automation — exportable compliance artifacts | ⬜ | `github-audit-logs.tf` and `cloudwatch-observability.tf` exist; no evidence export pipeline |
| 22 | Per-app compliance report generation | ⬜ | `devopsAuditAnalyzer/` exists in CC backend but doesn't produce SOC2 evidence format |

**Current infrastructure (aws-nevado-management):**
- AWS Organizations + SCPs/RCPs
- CloudTrail (org-wide)
- AWS Config
- KMS encryption
- GitHub audit log ingestion
- CloudWatch observability
- Bedrock quota monitoring
- Agent toolbox + Sherpa support StackSets
- DevOps read-only StackSets

**No commits since before June 2026 in either repo.**

---

## Dependencies

```
#1 SDK protocol (✅) ──────→ #9-11 Sherpa↔CC integration
#1 SDK protocol (✅) ──────→ #4-5 cycle API/UI (can use protocol types)
#4-5 cycle API/UI ─────────→ #9 trigger cycles from IDE
#12-13 PAS core ───────────→ #14-15 claims spec (references PAS)
#16 multi-tenant runtime ──→ #18-19 claims agent needs the runtime
#20 SOC2 gap analysis ─────→ #21-22 know what evidence to collect
```

---

## Suggested Sequencing — Remaining 4 Weeks

### Week 1 (Jul 1–7)
**Focus: Unblock the next wave + SOC2 urgency**

| Item | Repo | Effort |
|------|------|--------|
| #3 Protocol contract docs | sherpa-sdk | S |
| #20 SOC2 gap analysis | aws-nevado-management + customer-provisioner | M |
| #12 PAS shared core extraction | nevado-requirements | L |

### Week 2 (Jul 8–14)
**Focus: Self-service CC + requirements structure**

| Item | Repo | Effort |
|------|------|--------|
| #4 Cycle initiation API | command-center | M |
| #5 Cycle initiation UI | command-center | M |
| #6 Agent admin UI | command-center | L |
| #13 Per-variant delta specs | nevado-requirements | M |

### Week 3 (Jul 15–21)
**Focus: Connect the pipes**

| Item | Repo | Effort |
|------|------|--------|
| #2 CC integration hooks (finish) | sherpa-sdk | M |
| #7 Upload Portal → CC | command-center | M |
| #9-11 Sherpa↔CC integration | nevado-sherpa-ide | L |
| #16 Multi-tenant formalization (finish) | agent-workflow | M |

### Week 4 (Jul 22–31)
**Focus: Revenue wedge + compliance deadline**

| Item | Repo | Effort |
|------|------|--------|
| #8 Spreadsheet forensics | command-center | M |
| #14-15 Claims spec + adapter | nevado-requirements | L |
| #17-18 Customer deployment + Claims Agent | agent-workflow | L |
| #19 Claude Tag / Slack | agent-workflow | M |
| #21-22 SOC2 evidence automation | aws-nevado-management + CC | L |

---

## Summary

| Category | Total Items | ✅ Done | 🟡 Partial | ⬜ Not Started |
|----------|-------------|---------|------------|----------------|
| sherpa-sdk | 3 | 1 | 1 | 1 |
| command-center | 5 | 0 | 0 | 5 |
| nevado-sherpa-ide | 3 | 0 | 0 | 3 |
| nevado-requirements | 4 | 0 | 0 | 4 |
| agent-workflow | 4 | 0 | 1 | 3 |
| compliance/infra | 3 | 0 | 0 | 3 |
| **Total** | **22** | **1** | **2** | **19** |

**Bottom line:** The SDK protocol stabilization (the critical dependency) shipped today. 19 of 22 Q3 items haven't started yet with one month remaining. The SOC2 Type II deadline (Aug 2026) is the hardest constraint — gap analysis should start this week.
