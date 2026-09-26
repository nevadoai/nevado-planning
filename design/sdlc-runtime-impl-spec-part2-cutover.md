# SDLC Runtime Execution — Implementation Spec, Part 2: Cutover

**Phases 4, 4b, 5, 6, 7.** Derived from `sdlc-runtime-execution-architecture.md` (the architecture doc, cited below as **A§n**).

**Status:** implementation specification. The architecture is settled; this document does not re-open it. Where the architecture is wrong about the code, §1 says so and gives the corrected instruction — follow §1 over A§n.

**Verified against the repo on 2026-09-23 at `develop` = `938be378`**, working tree carrying four uncommitted frontend files (see §1.9). Every file path, line number and behavioural claim below was read from the actual file.

**Revision 2 — after principal-architect review.** Revised against `sdlc-runtime-impl-spec-architecture-review.md` (cited below as **R§n**). Five things changed materially and anyone who read revision 1 must re-read these:

| What changed | Where | Why |
|---|---|---|
| **The runtime repo exists and every runtime claim is now verified.** Revision 1 said it did not and marked runtime items “⚠ VERIFY”. It is **`nevadoai/nevado-sherpa-tui`** — revision 2 recorded it under a local directory name, `agent-workflow`, which is what made revision 1 unable to find it; the SDK is a third repo, **`nevadoai/sherpa-sdk`** | §1.10, rewritten | R§4(m) |
| **OD-4 is closed, and the answer is worse than the question.** A `commitSha` checkout does **not** detach HEAD, `branch` **is** returned, and `checkoutCommit` **never throws** — a bad sha silently serves the branch tip | §1.11 (new), Task 6.3 | R§1.2, R§4(a) |
| **Release-before-create is reversed to create-then-release.** Its premise — git refusing a branch in two worktrees — is false; the runtime provisions independent clones | Task 4.7, 6.3, 6.6 | R§1.2 |
| **`submitPlan` and `submitReview` collapse into one generic `submitResult`.** CC's plan and QA schemas leave the runtime's protocol package and become CC-side payload schemas — **superseded by revision 3 below: the schema ownership holds, the name collapse is withdrawn** | Phases 5 and 6, rewritten | R§2.1 |

**Revision 3 — after an independent cold adjudication of the terminal-tool design** (`context/specs/terminal-tool-design-decision.md`, cited below as **D§n**). Revision 2's direction was upheld; its mechanism was not. Two changes, and both are improvements on what this document proposed:

| What changed | Where | Why |
|---|---|---|
| **`submitResult` is withdrawn. Three named tools return: `submitPlan`, `submitReview`, `reportCompletion`** — with `submitPlan` and `submitReview` carrying an **opaque `payload: unknown`**. Collapsing the *names* was never needed to fix the real problem, and it cost the `kind` discriminator | §1.7, Phases 5 and 6 | D§ verdict, D§5(b) |
| **`kind` is restored**, as the tool name, so Part 1's `kind` ↔ `correlation.stage` cross-check is meaningful again. Revision 2 dropped it and made the stage its own discriminator, which left a planning/QA stage mislabel undetectable | Tasks 5.1, 6.5 | D§5(b) |
| **New optional create-time field `resultSchema`** — a JSON Schema authored by CC, stored beside `correlation`, which the runtime runs `payload` through **generically** before publishing. A failure is a `toolError`, so the agent corrects **in-turn**. Phases 5 and 6 author the schemas that get sent | `DEP-P1-13`, Tasks 5.1, 6.1 | D§1d, D§4 |
| **The plan schema's required set was derived from dead code.** Revision 2 justified requiring `approach` partly by "the plan-review dialog consumes it". That dialog is unreachable at `HEAD`. Re-derived from the live consumers | §1.12, Tasks 5.1, 5.3 | D§5(a) — **and see §12.4, where this document partly disagrees** |

**The decisive fact none of the three prior documents had:** `inputSchema` is **never read at runtime**. Verified in the deployed 4.0.1: `bedrock-client.ts:159-165` assembles it into the Bedrock `toolConfig` and nothing consults it again, and `grep -rIlE 'ajv|\bzod\b|json-schema'` across both `packages/*/package.json` trees returns **zero hits**. So a typed tool schema buys generation-time biasing and nothing else — it is not a gate. The runtime's own two most recently added tools prove the consequence: `saveLearnedPattern` declares `{title, category, trigger, content}` while its handler requires `input.pattern`, and `updateTechDebt` declares `{priority, category, location}` while its handler reads `severity`/`file`/`line`. Both drifts are shipped, identical at HEAD, and no conformance test notices. **Enforcement is a property of code running inside the session, not of where the schema lives** — which is why `resultSchema` gets the enforcement back without putting CC's taxonomy in the SDK.
| **The CI-build-failure loop moves into Phase 4.** Deferring it made Phase 4's own exit criterion unachievable | Task 4.7b (new), Task 4.9, Phase 6b (new) | R§1.5, R§7.4 |

**Revision 8 — brought current on 2026-09-26 against shipped code.** The transport, the machine ingress, the `/v1/sdlc` namespace, the three terminal tools and the branch derivation all landed between revision 7 and this date, so six things this document recorded as open, pending, blocking or Part 1's-to-build are now settled *by code* and are stated here as facts instead. **Everything not in this table is unchanged and is still a specification of work not yet done** — this is not a changelog and the body has not been turned into one.

| What changed | Where | Evidence |
|---|---|---|
| **The machine ingress is a bearer token the runtime verifies itself — not mTLS, and not a CA ceremony.** `sdlc_ingress_mode = "token"` puts one rule on the **existing :443 listener at priority 6**, matching `/v1/sdlc/*` **AND** an `Authorization: Bearer *` header, above an unconditional **priority-8 deny**; Terraform generates and writes the secret in the same apply, with **no human step at all**. `mtls` and `header` stay implemented and inactive | Task 4.10's precondition block (new), §9.2, §3.4 `DEP-P1-18` (new) | `infrastructure/sherpa-sdlc-ingress.tf` — `random_password`/`aws_secretsmanager_secret{,_version}.sdlc_auth_token` at `:488-546`, `aws_lb_listener_rule.sherpa_sdlc_token_forward` at `:600-640`, `.sherpa_sdlc_deny_443` at `:394-398`, `variable "sdlc_ingress_mode"` at `:133-141` |
| **The operator cutover sequence is three ordered preconditions, and the first of them is broken today.** Apply with `sdlc_ingress_mode = "token"` → deploy the runtime artifact and set `SDLC_AUTH_TOKEN_ARN` **in one restart window** → only then set `buildMode` on one application. Command Center **#858** destroys step one on every merge to `develop` | Task 4.10, §9.2, §10.1 R16 (new) | command-center #858 |
| **There is no `sherpa-sdk → runtime → command-center` blocking chain, and §0.2's "patch 4.0.1 and publish 4.0.2" is STRUCK.** 4.0.2 was never cut. One SDK release did ship — `sherpa-sdk` **#227**, merged 2026-09-25, released as **4.1.2** (#228) — but it is a **type widening only** (`SessionApplication` gains `'sdlc'` so the runtime ACL type-checks; `session-manager.ts` untouched), and **nobody waited on it**: Command Center still pins `@nevadoai/sherpa-protocol` **4.0.1**. The SDK change that *was* authorised and then **declined on merit** is `sherpa-sdk` **#226**, *"default `createdBy` and `application` on session create instead of forcing them"* — **CLOSED, not merged**, because `createdBy` backs a live ownership check in `renameSession`. The forcing is correct; **strike any suggestion that it be relaxed** | §0.2, §1.10, §1.14 (new), §3.4 | `sherpaSdkDriftGuard.test.js:40` (`PINNED_SDK_VERSION = '4.0.1'`), `frontend/package.json:27`; sherpa-sdk #226 / #227 / #228 |
| **A session's status is never its outcome, and no future status could be.** `'completed'` is a human complete/archive flag: all ten `finish` sites in the engine pass `'paused'`, emitting `'completed'` from the loop would stop `nevado-sherpa-ide`'s `/resume` (`listByStatus('paused')`) seeing sessions, and **delivery is not a lifecycle state** — a session that finished having submitted a result and one that finished having submitted nothing are in the same state | §0.2, §1.14 (new), Task 4.4 | source of record `backend/go/internal/orchestrator/runtimeevents/runtimeevents.go:9-45`; executable pin nevado-sherpa-tui PR **#101** (exactly ten sites, status set exactly `{paused}`) |
| **The working branch is DERIVED, not agent-chosen**, and `cycle/**` is a **wire constraint** rather than a naming preference. #851 (nothing wrote `cycle.branch`) is **FIXED and CLOSED** by PR **#857**; the defect that replaces it is **#854** — the reported branch overwrites the requested one unchecked — paired with its silent half, nevado-sherpa-tui **#103** | §1.13 (new), Tasks 4.3, 4.7b, 4.10, §10.1 R17 (new) | `cyclerecord.go:144-150`, `approvedispatch.go:101-147`, `runtimesession.go:465-491`, `engcomplete.go:345`, `applicationProvisioner/githubClient.js:1670-1687` |
| **Only HUMAN-APPROVED runtime cycles are unblocked.** PM auto-approval still routes through the JavaScript `approvePlan` and its **stub** dispatcher; the Go `planning.autoApprove` has no `cmd/` caller. A pre-existing cutover gap, **not** a regression | §1.13 (new), §9.2, §13 | PR #857's own body; `agentDrivenOrchestrator/index.js:1331-1352`, `plangeneration.go:254-263` |
| **Exactly ONE of `submitPlan` / `submitReview` / `reportCompletion` is granted per session**, refused at create if `profile.tools` names zero or two. That is **stricter** than A§4.3's allowlist, because it is code rather than configuration — `listTools()` advertises one and `executeTool` refuses every other name | §0.2, §3.4 `DEP-P1-12`, Task 6.7 | `nevado-sherpa-tui apps/runtime/src/routes/sdlc.ts:337-353`, `sdlc/terminal-tools.ts:47-59` |
| **`reportCompletion` VERIFIES; it does not stage, commit and push.** A§2.6's three-retry push was not built and must not be: `useMcpTool`'s 30 s budget is non-overridable (metadata is keyed on the dispatcher name) and `withTimeout` fires **without cancelling the work**, so a slow push would land while the agent was told it failed. The agent pushes with `runCommand` instead, and a failure reaches it as stderr — which is what A§2.6 wanted. **Task 7.2's `gitPushWithRetry` harvest is withdrawn** | §0.2, Task 7.2 item 1, §13.3 | `nevado-sherpa-tui apps/runtime/src/sdlc/git-state.ts:6-30`, `sdlc/terminal-tools.ts:24-35` |
| **`BQ-GO-2` and `DEP-P1-16` are answered by code.** Both halves are Go — the consumer is `backend/go/cmd/sdlc-event-consumer`, the runtime-facing HTTP client is `backend/go/internal/runtimeclient` — so `mapSessionEvent` is Go too (`internal/orchestrator/runtimeevents`) and Tasks 4.4, 5.1 and 6.5 extend *that*. The Go shape is nearest **G2**: new Go Lambdas over `internal/orchestrator/*` packages, with `agent-orchestrator` still `nodejs20.x`. **`DEP-P1-8` is confirmed superseded** — the shared home is `internal/orchestrator/cycleeffects`, not `backend/common/cycleEffects.js` | §13.4, §3.4 | `infrastructure/sdlc-events.tf:113`, `:204-206`; `lambdas.tf:1554-1560`, `:3040-3122` |
| **Three Part 1 dependencies are GRANTED and shipped, two of them more strongly than asked.** `DEP-P1-10` (a failed `commitSha` pin is a create-time `400`, via `provisionWorkspace` throwing rather than `checkoutCommit` returning `false`), `DEP-P1-15` (`tokenManager` is non-optional — `registerSdlcRoutes` throws at registration without it), and `DEP-P1-12`'s one-terminal-tool rule. **Read the runtime's `apps/runtime/src/sdlc/` over §3.4 where they disagree** — §3.4 records what Part 1 was asked for | §1.10 closing, §3.4 | `routes/sdlc.ts:299-307`, `:407-408`, `:619-658` |
| **Repo naming corrected throughout.** `agent-workflow` is `nevadoai/nevado-sherpa-tui`. Absolute paths on one person's laptop are replaced by repository names, which mean something to a second reader | throughout, §1.10 | — |

**Nothing in this document has been verified running a cycle on AWS.** One **local** end-to-end run proved dispatch through completion, and that is the whole of the evidence. Every "verified by" below that names an `aws` command is still a check **to perform**, not a check performed — Task 4.10 most of all.

**Authority.** Part 1 is authoritative on the shared contracts and on all runtime-side facts. This document is authoritative on Command-Center-side cutover facts. Where this document disagrees with the review, §12 says so with evidence rather than complying silently.

---

## 0. Scope, audience, and the boundary with Part 1

### 0.1 What this document owns

| Phase | Subject | Tasks | Architecture source |
|---|---|---|---|
| **4b** | Step-specific failure statuses (`ENGINEERING_FAILED`) | 4b.1–4b.5, **one commit** | A§6 Phase 4b, A§1.4 |
| **4** | Engineering step cutover, including the CI-build-failure loop | 4.0–4.10 (11) | A§3.3, A§3.4, A§3.8, A§3.9, A§6 Phase 4 |
| **5** | Planning as a session | 5.1–5.6 (7) | A§3.1, A§3.2, A§6 Phase 5, D§4 |
| **6** | QA as a session | 6.1–6.7 (7) | A§3.5, A§3.6, A§6 Phase 6, D§4 |
| **6b** | The deploy-failure loop and the request-changes path | 6b.1 | A§3.7, R§7.4 |
| **7** | Deletes | 7.1–7.7 | A§5.1, A§5.2, A§5.3, A§6 Phase 7 |

Note the phase numbers are the architecture's labels, not the execution order. **4b ships before 4** (§1.1, §3.1). Phase 6b is new in revision 2 (R§7.4) and is a hard gate on Phase 7.

**Reading order changed in revision 5: start at §0.15, then §13.** The Go mandate makes most of Phases 4–6b Go work rather than JS edits, and §13 gives the disposition for all 39 tasks. A task body below still tells you *what* to build and *why* — the findings, line references and acceptance criteria are language-independent and were expensive to establish — but §13 tells you *where* it lands and in what language. **Do not pick up a task without reading its §13 row.**

Then: **§1 before anything else** — it is the list of places the architecture doc is wrong about the code, and following A§n over §1 will cost a day. **§1.12 in particular** if you are touching the plan UI, because the surface both prior revisions called "the plan-review dialog" is not the one that renders. **§2** is the convention set. **§3** is the ordering. **§9.3** before writing any code that touches shared modules, because it is the honest rollback story and revision 1's was not. **§10.4** for the decisions already taken, so they are not relitigated in a PR. **§12** for the four places this document argues with its reviewers — §12.4 is the one still live.

**Three contract facts that changed under you if you read revision 2.** There are three terminal tools again, not one (`submitPlan`, `submitReview`, `reportCompletion`); `kind` is back on the completion payload and is the tool name; and CC now ships a `resultSchema` at session create that the runtime enforces in-session, which is why a malformed plan or review costs one model turn instead of a whole session. §0.2 has the contract, §1.7 has the reasoning.

### 0.15 The Go mandate — read this before anything else in this document

**All Command Center Lambdas must be written in Go, and any Lambda this project touches must be ported to Go and written test-first in Go. In-place JS edits are not permitted — not even one-liners. This is a CTO mandate, not a preference, and it is not weighed against cost.** It overturns the architecture doc's settled input *"TypeScript in the existing JS orchestrator rather than a new Go executor"* (A§ decisions, A§5.5), which was the premise this entire document was written on.

**The consequence for this document is close to total.** Nearly every task in Phases 4, 5, 6 and 6b edits `agentDrivenOrchestrator` internals in JS. Under the mandate those are not edits at all: touching that Lambda means porting it. There is no "small change to an existing JS handler" category any more.

**Phase 4b is stopped mid-implementation** on `feat/sdlc-phase0-cleanups` — it was editing `sdlcEngine.js` and `sdlcStepGraph.js` in JS. §13.1 respecs it as Go.

**What that means for this document, stated plainly: most of its tasks describe edits to JS code that is being replaced.** §13 gives a per-task disposition for all 39. Read it before picking up any task.

#### Verified ground state

| Fact | Evidence |
|---|---|
| Migration is advanced, not greenfield | **31** handlers in `backend/go/cmd`, **54** in `backend/lambda_handlers`, many names in both. `GO_LAMBDA_NAMES`/`GO_LAMBDA_CMDS` at `scripts/deployment/package-lambdas.sh:420-421` list all 31 |
| The cycle orchestrator is still JS | `infrastructure/lambdas.tf:1519-1528` wires `aws_lambda_function.agent_orchestrator` from `agentDrivenOrchestrator.zip` |
| **The existing Go `sdlc-manager` is not a cycle orchestrator and cannot be reused for these phases** | `backend/go/internal/sdlc/handler.go` (17 KB) + `cmd/sdlc-manager` + tests = **828 lines**. Its types are `SDLCTemplate`, `Stage`, `Config`, `PlanningConfig`, `QAConfig`, `CreateSDLCRequest` (`:74-142`) — **template and preset CRUD**. There is no cycle record, no status transition, no step routing. And SDLC templates are explicitly out of scope for this project (A§ scope). So "fold the cycle path into the existing Go handler" means adding a second, unrelated responsibility to a preset CRUD service |
| Go Lambda conventions | `handler = "bootstrap"`, `runtime = "provided.al2023"` (`lambdas.tf:2161-2164`, `sdlc_manager` as the exemplar); source layout `cmd/<name>/main.go` + `internal/<domain>/handler.go`; reusable packages `internal/{config,middleware,dynamo,storage,ghclient,agent,awsprov,sqs,cloudwatch,response,uuid}` |
| **The migration plan's own sequencing contradicts this project's needs** | `context/plans/go-migration-plan-for-node-js-lambda-handlers.md` puts the orchestrator **last**, in Wave 5, on the stated principle *"defer the orchestrator (most complex, most critical) until patterns are proven"* (`:15`, `:193-215`). `stuckCycleDetector` is Wave 3 (`:151`); `qaAgent` and `engineeringAgent` are Wave 4 (`:174`, `:178`). This project needs the cycle path **now**. That tension is real and is **not this document's to resolve** — it is `DEP-P1-16` |
| The plan's orchestrator sizing is stale | It says *"3,970 lines, 42 functions"* (`:195`); `agentDrivenOrchestrator/index.js` is **4,984 lines** with ~55 top-level functions. The Wave 5 estimate is against a file 25% smaller than the one that exists |

#### The three shapes, and why the disposition depends on which Part 1 picks

Part 1 owns this choice (`DEP-P1-16`). This document classifies each task against all three so the answer is mechanical once the choice lands:

- **G1 — extend the existing Go `sdlc-manager`.** Bolts the cycle path onto a 828-line preset CRUD handler.
- **G2 — a new Go handler for the cycle path** (e.g. `cmd/sdlc-cycle-orchestrator` + `internal/orchestrator/`), leaving `agentDrivenOrchestrator` to the legacy paths until they retire. This is the migration plan's Wave 5 decomposition (`:204-215`) applied to the cycle path only.
- **G3 — migrate `agentDrivenOrchestrator` wholesale**, i.e. Wave 5 in full, as a prerequisite to Phase 4.

**Under G2 and G3, most of Phases 4, 5, 6 and 6b stop being edits and become Go implementation.** Under G3 several tasks dissolve entirely, because the JS code they modify ceases to exist. §13 says which.

#### What the mandate does to the shared JS modules

**The mandate is about Lambdas. `backend/common/cycleStatuses.js` and `cycleActivities.js` are not Lambdas — they are shared modules.** That distinction does not exempt them; it **relocates** them, and the direction is the opposite of what an earlier draft of this section said.

Their consumers today are (a) JS Lambdas and (b) the frontend, via `frontend/vite.config.js:19-20`, which aliases `common/cycleStatuses` and `common/cycleActivities` straight at the backend files. Port every JS Lambda consumer to Go and **the only remaining consumer of the JS artifact is the browser.** So the authoritative definition moves to Go and the JS module degrades to a frontend-facing copy. Three ways to hold that together, and the choice is `BQ-GO-1`:

- **generate** the JS artifact from the Go source at build time (the only option with no hand-maintained duplicate);
- **hand-maintain** the JS copy behind a conformance test that compares the two at CI time (needs a real comparison, not a source grep — see below);
- **serve** the vocabulary from an API and drop the compile-time alias (changes the frontend's startup path).

An earlier draft of this section argued that the Vite alias meant `cycleStatuses.js` "must stay JS" and that Phase 4b.1 therefore stood as written. **That was wrong** — it confused "a browser consumer needs a JS artifact" with "the definition lives in JS". The artifact stays JS; the definition does not.

**The status/activity contract is therefore spread across three languages today and the mandate makes it four.** This is the most consequential structural consequence for these phases and it is not in any prior document:

   | Contract | JS | Go | Python | Frontend |
   |---|---|---|---|---|
   | Cycle statuses | `common/cycleStatuses.js` (authoritative) | hand-maintained `AgentID` correspondence — `sdlcStepGraph.js:51` documents that its values were chosen to match `internal/sdlc/handler.go`'s shape (`:384-394`) | `cc_cycles.py` reads raw status strings | via the Vite alias |
   | Activity feed selector | `common/cycleActivities.js` (`selectActivities`, added `51ed3fe0`) | — (needed if the cycle path moves) | **a hand port** in `cc_cycles.py` (`e29f8264`) | via the Vite alias |

   `backend/__tests__/cycleProgressContract.test.js` is the only thing binding the JS and Python copies, and it is a **source-text guard** — which cannot catch a logic divergence, only a missing reference. Adding a Go copy adds a third implementation of a selector whose whole reason for existing is that the obvious one-line version is wrong in a way invisible on inspection (R0).

   **This is `BQ-GO-1` (§13.4), and it is now blocking rather than merely cheap-to-answer-early.** Phase 4b's entire job is to add a status. Under the mandate it must add it in Go, to a Go transition table, while the frontend still needs it in JS and `cc_cycles.py` still reads it in Python. **4b cannot be respec'd coherently until BQ-GO-1 has an answer**, because "where does the constant live" is the first line of the task.

#### TDD in Go is a hard rule, and most of this document's source guards do not satisfy it

Tests first, failing for the right reason, then implementation. **Regex-over-source assertions do not count as the test.** Phase 0 proved the point at this project's own expense: the selector this document prescribed (`cycle?.progressLog || cycle?.activities`) failed **7 of 11** behavioural tests that the field-name grep it also prescribed would have passed without complaint — because `[]` is truthy and a grep cannot see truthiness. That is the single best argument in the whole project for behavioural tests, and it was produced by this document being wrong.

So the guards in this document split into two classes and must not be conflated:

| Class | Examples | Status under the mandate |
|---|---|---|
| **Illegitimate substitutes** — assert a *behaviour* by looking at source text | Task 4.8's `signalPmItem` wiring grep; Task 5.3's field-name guard; Task 4b.5's `FAILED`-producer greps | **Replace with behavioural Go tests.** These are exactly the class Phase 0 disproved. Where the code under test is a Go handler, a table-driven test on the real function replaces the grep outright |
| **Legitimately structural** — assert a fact about *deployment or wiring* that has no in-process behaviour | Task 4.0's `-target=` membership; Task 4.9's dispatch-site count; Task 7.x's dead-code assertions | **Keep, and stop calling them tests-first.** You cannot behaviourally test "this resource appears in a Terraform target list" without deploying. Label them **deployment guards**, run them in CI alongside the Go tests, and do not let their presence imply a behaviour is covered |

§2.1's three JS test styles are superseded for anything that becomes Go: the pure-function suites port cleanly to table-driven `_test.go`; the deployment guards port fine (reading files as text is as easy in Go); and the mocked-AWS-SDK harness in `applyRuntimeEffect.test.js` — which §2.1 names as the exemplar to copy wholesale — **has no Go equivalent** and is rebuilt around interface injection, which is what the existing Go handlers already do. §13.3 has the mapping.

### 0.2 What Part 1 owns — reference it, never redefine it

Part 1 (`sdlc-runtime-impl-spec-part1-foundation.md`) owns the shared contracts and Phases 0–3:

- **The ACL** (A§2.2, Part 1 §A.3). **v1 scopes on `application === 'sdlc'`**, enforced on every `/v1/sdlc` read / cancel / delete — that is machine/human separation, which is the whole of v1's real security value. **Part 1 owns the mechanism and is authoritative on it.** Do not re-derive it here; read Part 1 §A.3.
  - **Revision 8 — `createdBy` cannot carry a machine identity, and that is now settled rather than deferred.** Revision 2 asserted the ACL required a two-line SDK change unforcing `createdBy` and `application` in `SessionManager.create`. That change was written (`sherpa-sdk` #226) and **closed unmerged, on merit**: `createdBy` backs a live ownership check — `renameSession` throws `RemoteSessionNotOwnedError` when `record.createdBy !== this.userId` — so defaulting it instead of forcing it would let a machine-supplied label defeat a human surface's authorization gate. The SDLC create route therefore *passes* a label (`m2m:bearer`) which `SessionManager.create` **discards**, and every SDLC session persists the runtime's own `userId`. `routes/sdlc.ts:178-220` states this at length; the assertion is against the **persisted record**, not the create argument (nevado-sherpa-tui `d8da6a1`).
  - **So per-client partitioning is not deferred — it is unavailable, and the reason is the credential, not the field.** A shared bearer token carries no per-client identity to partition *on*. `application === 'sdlc'` is the only discriminator v1 has, and it is the one that matters: it is what keeps a machine call away from a human's session. A per-client predicate over `createdBy` is ruled out until there is both a second M2M client **and** a credential that distinguishes them.
- **The HTTP surface** (A§2.3, Part 1 §A.2). Four calls, and **they live in `backend/common/runtimeClient.js`** — not `runtimeDispatch.js`. Part 1's packaging reasoning is correct and decides it: `package_lambda_with_common` copies only the handler directory's own top-level `.js` files (`scripts/deployment/package-lambdas.sh:141`) while serving `backend/common/` from the shared layer, so `sdlcEventConsumer` and `stuckCycleDetector` — both separate Lambdas — cannot require anything from `agentDrivenOrchestrator/`. This document calls them `createSession(...)`, `getSession(id)`, `cancelSession(id)`, `releaseSession(id)`.
  - **`POST /sessions` returns `201`**, not `202`; and `201` with `status: 'queued'` for a create that exceeds the concurrency cap.
  - **`DELETE /sessions/{id}` returns `200 {ok: true}`**, not `204`. Verified at `nevado-sherpa-tui apps/runtime/src/routes/sessions.ts:438`. CC branches on status class, so this is a documentation fix, not a behaviour change.
  - **Revision 8 — the namespace is `/v1/sdlc/*` and it is its own encapsulated Fastify plugin with its own authorization hook**, not a route under `/v1/workspace`. `registerSdlcRoutes` is registered under the `/v1/sdlc` prefix and installs a `preHandler` hook of its own (`routes/sdlc.ts:401-525`), pinned by `routes/sdlc-prefix.integration.test.ts` — *"registers the SDLC routes under `/v1/sdlc`, not `/v1/workspace`"*. Encapsulation is the point: the machine hook cannot leak onto the human routes and the human auth cannot be the thing guarding a machine call. Where a task below writes a `/v1/workspace/sessions` path, read `/v1/sdlc/sessions`.
  - **Revision 8 — the CC-side client is Go: `backend/go/internal/runtimeclient`.** It defines four `/v1/sdlc` methods, sends `Authorization: Bearer <token>` read from Secrets Manager (`transport.go:69`, `:78-83`, `:111-138`), and **cancel has no production caller** — `runtimesession.go:39-44` argues *"RELEASE, NEVER CANCEL"*, for the reason §1.10's last row gives. A delivery note describing this package as *"mTLS, Secrets Manager identity"* is wrong about the operated mode: mutual TLS is a code path it retains for an `mtls` deployment, and the mode CC runs is bearer token. Note also that `transport.go:14-19` and `secrets.go:55-58` still carry the **old, false** framing of why mTLS is not the operated mode — do not cite those two docblocks; the accurate account is in `infrastructure/sherpa-sdlc-ingress.tf:25-52`.
  - **`GET` returns a `200` that does not guarantee `.session`** — one existing response shape is `{status: 'unshared', tombstone}` (`sessions.ts:157`). The record uses **`command`, not `mode`**, and `createdAt`/`updatedAt` are **epoch numbers**, not ISO strings.
- **The event transport** (A§2.4, A§2.5) — the FIFO queue, the envelope, the two-tier publish cadence, the consumer, and the conditional status write that makes replay safe. **`seq` is no longer durable state:** the FIFO `MessageDeduplicationId` is a **per-envelope UUID v4** minted at envelope construction and reused across retries of that one `SendMessage`; ordering remains `messageGroupId = sessionId` (R§2.4, R§7.2). Risk R5c is deleted as a consequence.
  - **Revision 8 — this shipped, and these are the shapes to build against. There is no SDLC WebSocket; do not design one.** The publisher attaches to the runtime's **existing `Broadcaster`** via `setObserver` (`apps/runtime/src/index.ts:97`, `ws/broadcaster.ts:29`) — so SDLC observation is a tap on the message path the human surfaces already use, not a second transport. `messageGroupId = sessionId` and a `randomUUID` `MessageDeduplicationId` per envelope, both mandatory on this queue (`sdlc/sqs-transport.ts:12-33`); envelope version `v: 1` (`sdlc/publisher.ts:55`). **`correlation` is opaque to the runtime** — `applicationId` / `cycleId` / `stage` are carried and never interpreted — which is what keeps CC's cycle vocabulary out of the runtime.
  - **The consumer is Go, and the naming in this document is stale.** Where a task says `sdlcEventConsumer`, the thing that exists is `backend/go/cmd/sdlc-event-consumer`, deployed as `${project_name}-sdlc-event-consumer-${environment}` with an event-source mapping off `aws_sqs_queue.sdlc_events` (`infrastructure/sdlc-events.tf:113`, `:204-206`). This is the **repo's first FIFO queue**: both the queue and its DLQ must be FIFO and end in `.fifo`, and a standard DLQ on a FIFO queue is rejected at apply time (`sdlc-events.tf:8-19`). See §13.4's `BQ-GO-2`, which this answers.
- **The terminal tools** (A§2.6, Part 1 §A.5). **Three, named, with two of them carrying an opaque payload** (D§ verdict) — **and exactly ONE granted per session**:
  - **Revision 8 — one per session, refused at create otherwise, and this is the only place the per-role separation happens.** `profile.tools` must name **exactly one** of `submitPlan` / `submitReview` / `reportCompletion`; zero or two is a `400` (`routes/sdlc.ts:337-353`, `sdlc/terminal-tools.ts:47-59`). The reason it cannot be done anywhere else is mechanical: all three reach the engine through `useMcpTool`, so `PLAN_BLOCKED_TOOLS` cannot see them and **session-mode gating cannot separate a planning session from an engineering one**. Zero means a session with no way to submit a result, which burns its whole turn budget before anyone finds out; two means the published `kind` is ambiguous and the runtime has no basis for choosing. Anywhere below that reads as though a session holds more than one terminal tool is wrong — the grant is singular, derived from `profile.tools`, and the tool name survives the `useMcpTool` proxy as the `completion` event's `kind`.
  - **`reportCompletion` stays fully typed.** Branch, commit sha, files changed are git and session vocabulary, and it needs the working tree. Keeping it typed also removes a round trip: `commitSha`/`branch` are absent from the event stream today, so the completion effect would otherwise have to fetch the `AgentSession` for them (`runtimeEvents.js:20-23`, `:88-90`).
    - **Revision 8 — it VERIFIES rather than stages/commits/pushes, and A§2.6's three-retry push was not built.** `useMcpTool` carries a non-overridable 30 s budget (metadata is keyed on the *dispatcher* name, so an injected tool cannot be exempted) and `withTimeout` resolves a `toolError` at the deadline **without cancelling the work** — so a slow push would land while the agent was told it failed, and the retry would duplicate the commit or record failure over landed work. Instead **the agent pushes with `runCommand`** (`timeoutExempt: true`, auto-approved on an unattended session) and `reportCompletion` confirms the tree and the sha read-only (`apps/runtime/src/sdlc/git-state.ts`). A push failure therefore reaches the agent as `runCommand` stderr, which is what A§2.6 actually asked for, and the runtime never reports a sha it did not confirm was pushed. Task 7.2's `gitPushWithRetry` harvest is withdrawn on this basis.
  - **`submitPlan` and `submitReview` carry `payload: unknown`.** No CC business field — `severity`, `acceptanceCriteria`, `requirementsMet`, `overallQuality`, `category` — appears in the runtime's protocol package. The tool *names* are fine: the runtime already knows "plan" as a first-class generic concept (`SessionCommand = 'do' | 'plan'`, `writePlan`, `PLAN_BLOCKED_TOOLS`, `planCreated`, `AgentSession.plan`). The boundary objection was always to the **contents**, and it is much narrower than "a tool named `submitPlan` breaches §1.1" (D§3).
  - **`kind` on the `completion` payload is the tool name**, so Part 1's `kind` ↔ `correlation.stage` cross-check discriminates all three stages. A mismatch is a **protocol violation** — CC granted a tool for a step it is not running — and goes to the DLQ, per §A.4.6.
- **`resultSchema`, a new optional create-time field** (`DEP-P1-13`). CC authors a JSON Schema, the runtime stores it beside `correlation` and runs `payload` through it **generically** before publishing; a failure is a `toolError` naming the failing paths, and the agent corrects inside the same turn. Bounded keyword subset — `type`, `properties`, `required`, `items`, `enum`, `maxLength`, `maxItems`, `minItems`, `additionalProperties: false`; **no `$ref`, no `allOf`/`anyOf`/`oneOf`, no `pattern`** (a caller-supplied regex on an unattended long-lived process is a ReDoS vector for no benefit). It is **optional**: absence degrades to unvalidated forwarding, so CC can disable the validator by config with **no runtime deploy** — which is what makes a generic validator safe to put behind an outage-gated release when a plan-specific one would not be.
- **Instruction injection** (A§2.7) and the **non-interactive session profile** (A§4). The profile that carries information is `{tools: string[], maxTurns: number}`, with `application: 'sdlc'` as the discriminator; `workspaceIsolation: 'per-session'` is decoration, because the runtime clones per session unconditionally (`nevado-sherpa-tui apps/runtime/src/agent/workspace.ts:33-47`).
- ~~**SDK version: patch 4.0.1 and publish 4.0.2.**~~ **STRUCK in revision 8 — 4.0.2 was never cut and nothing needed it.** The deployed runtime declares `"@nevadoai/sherpa-core": "^4.0.1"` (`nevado-sherpa-tui apps/runtime/package.json:22`) and Command Center still pins `@nevadoai/sherpa-protocol` **4.0.1** (`sherpaSdkDriftGuard.test.js:40`, `frontend/package.json:27`). Every line reference to SDK source in this document remains a reference to **4.0.1**, read out of the runtime's own installed tree at `node_modules/.pnpm/@nevadoai+sherpa-core@4.0.1/node_modules/@nevadoai/sherpa-core/src/`. **The three terminal tools, `resultSchema` and its validator were built in the runtime, not in the SDK** — `apps/runtime/src/sdlc/terminal-tools.ts` and `sdlc/result-schema.ts`, injected into the deployed 4.0.1 rather than shipped inside it. See §1.14 for what that buys and what it does not.
- **No phase in this document requires an SDK release. Confirmed in revision 8 against what shipped, not merely reasoned.** All three tools, `resultSchema`, its validator and the "already submitted" guard live in the runtime, so their delivery vehicle is a **runtime deploy**, not an npm release — which is the more expensive of the two and is the constraint §1.10's last paragraph and `OQ-1` are about. The generic `payload` plus `resultSchema` makes every subsequent plan-schema and QA-schema change a Command Center edit on CC's normal pipeline. D§1d puts the economics precisely: **`resultSchema`'s marginal cost over an opaque-payload-only design is zero deploy windows now, and one deploy window if deferred** — so the seam goes in on the release already being taken. This is the operational payoff of the opaque-`payload` decision (R§2.1; revision 3 withdrew the *name* collapse, not the schema ownership) and it is worth stating as a property rather than leaving implicit: **after Part 1 Phase 3, nothing here waits on a runtime deploy window.** If a task below appears to need one, that is a design error in the task rather than a scheduling problem — raise it.
- **Phase 0** free cleanups, **Phase 1** auth proof, **Phase 2** session create/read/cancel/release, **Phase 3** SQS transport + the extraction of `applyRuntimeEffect` / `routeAfterEngineering` / `writeStatus` / the `addProgressLog` wrapper into **`backend/common/cycleEffects.js`**.

When a phase below needs a contract detail Part 1 might have missed, it is stated as a **dependency on Part 1**, flagged `DEP-P1-n`, and collected in §3.4. Do not invent a competing definition.

### 0.3 How to read a task

Each task is one commit. Each states:

- **Files** — full paths, marked `NEW` or `MOD`.
- **Tests first** — test file path, test names as sentences, fixtures, assertions. Where a unit of work is not unit-testable, the task says **not TDD — verified by** and gives the concrete command and the expected output. No filler tests.
- **Implementation** — what to change, with the exemplar file to copy from.
- **Acceptance** — a binary check.
- **Commit** — the one-line conventional-commit message. `fix` for a bug that exists today; `feat` only for a new capability.

---

## 1. Verification log — what the architecture doc gets wrong

The architecture doc claims verification on 2026-09-23 and its line references are mostly exact. The corrections below are the ones that change what you should build. Everything not listed here was checked and is correct.

### 1.1 Blocking: `applyTransition` throws on an unknown status — Phase 4b cannot be deferred

A§6 Phase 3 says of `ENGINEERING_FAILED`: *"Either pull 4b forward to here, or stub the constant now… Do not ship a consumer that writes a status `cycleStatusConfig.js` has never heard of."* That understates the failure by an order of magnitude.

`sdlcEngine.applyTransition` (`backend/lambda_handlers/agentDrivenOrchestrator/sdlcEngine.js:507`) opens with:

```js
if (!(toStatus in TRANSITIONS)) {
  throw new Error(`Unknown target status "${toStatus}"`);
}
```

`TRANSITIONS` is a literal keyed on `cycleStatuses.*` (`sdlcEngine.js:48-230`). A status that is not a key **throws**, unconditionally, before any of the report-and-proceed logic at `:520-542`. So a consumer that maps a planning-stage `error` event to `ENGINEERING_FAILED` before the constant is in `TRANSITIONS` does not log a violation — it throws, SQS retries, the message group freezes for ~9 minutes, and the message lands in the DLQ. Every failing runtime cycle would freeze.

**Instruction: Phase 4b lands before any code writes `ENGINEERING_FAILED`, i.e. before Part 1's Phase 3 consumer ships its per-stage `error` mapping.** Phase 4b has no dependency on Phases 0–3 and can be built in parallel with them from day one.

A§5.3 lists `sdlcEngine.js` and `sdlcStepGraph.js` as **untouched**. For Phase 4b that is wrong: both must change. See §5 tasks 4b.1 and 4b.2.

### 1.2 Wrong: `routeAfterEngineering` is *not* untouched by this change

A§5.3 and A§7 open question 4 both state `routeAfterEngineering` is untouched. True for Phase 4. **False for Phase 6.**

`routeAfterEngineering` (`index.js:175-309`) has four branches. The `ai-qa` default branch (`index.js:192-305`) does not merely set `QA_TESTING` — it calls `invokeQAAgent(qaPayload)` **synchronously at `index.js:237`** and then processes the returned `qaResult` inline across ~65 lines (`:247-305`), writing `cycle.iterations[n].qaResult`, `cycle.testBranch`, `cycle.testsGenerated`, `cycle.testMetadata`, `cycle.qaFailures`, `cycle.qaFeedback`, and choosing between `QA_WAITING_FOR_TESTS`, `PENDING_APPROVAL`, `QA_FAILED` and `FAILED`.

Moving QA onto a session means that branch must **start a session and return**, with the result arriving later through the consumer. That is a structural change to `routeAfterEngineering` and it is the largest piece of work in Phase 6. See §7 task 6.4.

### 1.3 Wrong: there are **five** engineering dispatch sites, not one

A§5.2 describes the cutover as flipping the branch at `index.js:1628` and says *"The SQS engineering dispatch (`:1652+`) survives during the transition and deletes when `buildMode: 'runtime'` becomes unconditional."* That accounts for the entry into engineering and nothing else.

`invokeEngineeringAgent` (declared `index.js:2488`) has **five** call sites:

| Line | Enclosing function | Trigger |
|---|---|---|
| `index.js:1779` | `processSQSCycleExecution` (`:1688`) | First engineering run, via the SQS queue |
| `index.js:2132` | `approveCycle` (`:1973`), `request_changes` branch (`:2101`) | Operator requested changes at the PR gate |
| `index.js:3056` | `continueBuildIteration` (`:2951`) | CI build failed |
| `index.js:3223` | `continueDeployIteration` (`:3133`) | Post-merge deploy failed (A§3.7) |
| `index.js:3613` | `continueIteration` (`:3467`) | QA failed / operator iterate (A§3.5 loop-back) |

Four of the five are **synchronous in-Lambda invokes**, not the SQS dispatch. `buildMode` is read and forwarded at `:1788`, `:2143`, `:3032`, `:3217`, `:3608` — so all five already pass `buildMode` down to `engineeringAgent/orchestratorHandler.handle`, whose SSM branch is at `:378-397`. None of the four has a `buildMode === 'runtime'` branch, so on a runtime application they all fall through to the JSON-blob Bedrock path.

**Instruction:** Phase 4 must not leave four silent fall-throughs. Task 4.9 makes them fail loudly; Phase 6 task 6.6 converts the QA loop-back (`:3613`); the deploy loop (`:3223`) and the build loop (`:3056`) are explicitly out of scope for Phases 4–7 and are called out as open decision **OD-3** (§10.4).

### 1.4 Wrong: the cycle record has no per-step session id

A§3.2 and A§3.9 both key off `cycle.planSessionId` / `cycle.engineeringSessionId` / `cycle.qaSessionId`. A§1.4 lists the real fields as `runtimeSessionId` / `runtimeSessionStubbed` and cites `index.js:1636-1638`.

Verified: `index.js:1629-1641` writes exactly two fields:

```js
await updateItem(pk, sk, {
  runtimeSessionId: sessionId,
  runtimeSessionStubbed: stubbed,
  updatedAt: new Date().toISOString(),
});
```

`planSessionId`, `engineeringSessionId` and `qaSessionId` **do not exist anywhere in the repo**. One field cannot hold three concurrent sessions — and Phase 6 has two live at once (an engineering session's workspace is released only when QA starts, and QA runs while the cycle still records which engineering commit it is reviewing). Task 4.1 introduces the three fields.

### 1.5 Wrong: `stuckCycleDetector` is not in the deploy pipeline, and it does not use the status write path

Two separate problems, both blocking for Phase 4, both unmentioned in the architecture.

**(a) It is never deployed.** `.github/workflows/deploy-dev.yml` has **three** targeted apply steps — Core (`:419`, apply at `:426`, targets `:427-586`), DevOps Phase 1 (`:645`, targets `:653-791`), DevOps Phase 2-4 (`:822`, targets `:830-956`) — 426 `-target=` entries in total. Grep the whole file for `stuck_cycle_detector`: **zero hits.** The Lambda (`infrastructure/lambdas.tf:2269`), its log group (`:2299`), its schedule (`:2311`, `rate(5 minutes)` at `:2314`), its target (`:2324`) and its permission (`:2331`) are all in Terraform and in **no** target list. The zip *is* built (`scripts/deployment/package-lambdas.sh:463`), but because code deploys happen through `filename` + `source_code_hash` on the `aws_lambda_function` resource, and that resource is never targeted, **a change to `stuckCycleDetector/index.js` on `develop` never reaches AWS.**

A§3.9 makes session-aware stall handling a hard Phase 4 prerequisite. It is unshippable until the target list is fixed. Task 4.0.

**(b) It writes `status` raw.** A§1.2 claims *"`status` is never assigned raw — all 54 write sites go through `applyTransition` or `writeStatus`."* `stuckCycleDetector/index.js:174-197` is a counter-example: a bare `UpdateCommand` setting `':failed': 'failed'` (`:186`) as a hardcoded string, with no `ConditionExpression`, no `cycleStatuses` import anywhere in the file, and **no `REMOVE GSI4PK, GSI4SK`**. Since `qa_waiting_for_tests` is both in its scan list (`:110-117`) and in `POLLED_STATUSES` (`cycleStatuses.js:90-97`), a cycle timed out from that status keeps a stale GSI4 entry and the poller keeps returning it. Pre-existing bug; Task 4.0 fixes it as a precondition for touching the file.

Also note the status list is duplicated **three times** in that file — `TIMEOUT_THRESHOLDS` (`:23-29`), `IN_PROGRESS_STATUSES` (`:32-38`), and the scan `ExpressionAttributeValues` (`:110-117`). Adding a status means editing all three plus the `:s1..:s5` placeholder list at `:106`.

### 1.6 Wrong: `generatedBy` is stamped by the engineering agent, not by the orchestrator

A§1.4 says `expandedRequirements` gets *"`generatedBy` stamped by the orchestrator (`index.js:1305-1315`)"*. It does not. `index.js:1315` **reads** it for checkpoint evidence:

```js
generatedBy: expandedRequirements?.generatedBy ?? null,
```

The only writer is `engineeringAgent/orchestratorHandler.js:346`: `plan.generatedBy = 'nevado';` (with `plan.generatedAt` on `:345`).

This matters for Phase 5. `PlanCompletion` as defined in A§2.6 carries **neither `generatedAt` nor `generatedBy`**. If the consumer persists the payload verbatim, `recordCheckpoint`'s evidence gets `generatedBy: null`, and `backend/__tests__/stepAuditEvidence.test.js:187` asserts on `generatedBy: 'nevado'`. Task 5.1 stamps both consumer-side, and `DEP-P1-3` asks Part 1 whether the tool should carry them instead.

### 1.7 Wrong / misleading: `ReviewCompletion` does not fit CC's existing QA consumers — and the fix is to move it out of the protocol

A§2.6 says `ReviewCompletion.findings` is *"shaped to survive CC's existing validation rather than to be convenient for the agent"* and cites `qaAgent/orchestratorHandler.js:288-291` and `:337`. The GitHub API call at `:337` does take `{path, line, side, body}` — but the CC code that builds those comments reads a **different** finding shape, and two other CC consumers read fields `ReviewCompletion` does not have at all.

Verified at `qaAgent/orchestratorHandler.js:306-323`, the loop reads `f.severity` (`'error' | 'warning' | anything-else`, mapped to an emoji at `:307`), `f.issue`, `f.context`, `f.suggestion`, and `f.file` / `f.line`. Diff-line validation is at `:291-303` (building `validDiffLines`) and `:318-323` (the `validLines?.has(f.line)` test, with a fallback into the review body). The review verdict is `result.passed === true && result.overallQuality !== 'needs-work'` (`:286`).

Then `routeAfterEngineering`'s ai-qa branch (`index.js:247-305`) reads `qaResult.success`, `.requirementsMet` (via `updateCycleWithQAResult`, `qaAgent/orchestratorHandler.js:480-490`), `.testsGenerated`, `.testBranch`, `.testFiles`, `.testPlan`, `.githubActionsRequired`, `.allTestsPassed`, `.findings[].{file,line,issue,suggestion}`, `.recommendations`, `.summary`, `.message`, `.error`.

So the mapping `ReviewCompletion → qaResult` is a real adapter with a real risk of silently dropping fields, not a rename. Task 6.1 specifies it as a pure function with tests.

**Revision 3 — resolved by a design change, and the reason to state is not the boundary.** The fields CC reads — `overallQuality`, `requirementsMet`, `githubActionsRequired`, `testsGenerated`, `testBranch`, and per-finding `file` / `issue` / `context` / `suggestion` with a three-level severity — stop being a gap between a protocol type and a consumer, and become **a CC-side payload schema that CC owns end to end**: stated in the instructions CC composes (A§2.7), shipped to the runtime as `resultSchema` for in-session enforcement, and validated by CC on receipt. Task 6.1 is the schema owner, the validator and the adapter; there is no version skew between what a protocol package says a review is and what CC reads, because there is only one object.

**But put the real reason in the headline.** Revision 2 led with A§1.1 — *"the one genuine breach of the boundary"* — and D§3 is right that this framing is weaker than it looks. The runtime already knows "plan" as a generic concept; `instructions` crosses the same boundary carrying vastly more SDLC content and nobody calls that a breach, because the runtime does not *read* or *branch on* it. And this document conceded the point itself: §10.5 OQ-1 makes the decision explicitly contingent on deploy economics, which means the boundary argument was never the load-bearing one.

**So: CC's plan and QA schemas do not live in the runtime because the runtime has no deploy pipeline, no staging instance, no session drain, no version CC can observe, and a demonstrated history of schema/handler drift with no test that catches it.** That framing is falsifiable and it correctly predicts what changes if the deploy story improves. Keep §10.5 OQ-1 as the revisit trigger, and add a second condition to it: a spec↔handler conformance test in the SDK.

**What the adjudication adds that revision 2 gave away.** Revision 2 traded in-session enforcement for schema ownership, and treated the two as inseparable. They are not (D§1d): enforcement is a property of code running inside the session. `resultSchema` restores it — the runtime validates generically against a schema CC ships per session, a failure is a `toolError`, and the agent corrects inside the same turn at the cost of one extra model call. The most valuable property is not the retry: it is that **one JSON object in Command Center is simultaneously the prompt contract, the in-session gate and the consumer's validator**, so the three cannot drift.

**The cost, stated plainly, because it is real.** Without a tool `inputSchema`, the model is no longer constrained by the tool-use API's JSON-schema enforcement, so a malformed payload is possible. CC never messages a running session (A§3.2), so a malformed payload means a failed session and a retry from scratch rather than a correction. Task 6.1's validator must therefore fail **loudly and specifically** — naming the offending field — so the failure is diagnosable from the cycle record.

**Which disposition that failure gets is settled by Part 1 §A.4.6, and the test is one question: would a redrive after a code deploy succeed?** For a malformed payload, no — the same bytes fail the same way — so it is a **terminal cycle status with the message consumed**: `QA_FAILED` here, `PLANNING_FAILED` in Phase 5, both `ATTENTION_STATUSES` with working retries. The DLQ is reserved for the failures a redrive *would* fix: a drifted effect kind, an unparseable envelope, a stage no phase supports yet. Carry the question, not a memorised case list — §12.2 records why the distinction exists at all.

Also noted, unfixed, pre-existing: `:341` sends `event: approved ? 'APPROVE' : 'COMMENT'` while the log line at `:345` says `REQUEST_CHANGES`. The review is posted as a comment, so QA has never actually requested changes on GitHub. And the whole block is wrapped in a `try/catch` that only `console.warn`s (`:346-348`), so a failed review post never fails the cycle.

### 1.8 Wrong: `expandBusinessGoals` has a third site, and the plan-revision replacement is an async conversion

A§5.1 says *"`expandBusinessGoals` has two callers"* — `index.js:1222` (the planning fallback) and `index.js:4904` (plan revision). Correct as far as function calls go. But there is a third place that must change in Phase 7: `backend/scripts/verify-converse.js:1093-1141` reproduces `expandBusinessGoals`' entire prompt inline as verification case 11. It is not a caller, so it will not break — it will silently keep verifying a prompt for a deleted function.

More importantly, the architecture treats the plan-revision replacement as merely *"work"*. It is a **synchronous-to-asynchronous conversion**, and the doc does not say so. `handleRequestRevision` (`index.js:4871-4952`) today: reads open comments → calls `expandBusinessGoals` inline → `planComments.storePlanVersion` → `planComments.markCommentsAddressed` → writes `PLANNING_REVIEW` + `expandedRequirements` → returns `{plan: revisedPlan}` in the HTTP body. A plan *session* cannot do any of that in the request.

Good news, verified: the only caller is `frontend/src/components/PlanReview.jsx:43`, which discards the response body and calls `onRefreshCycles()`. So the conversion is frontend-compatible **provided the cycle leaves `PLANNING_REVIEW`** while the revision runs, or the UI will show the stale plan as reviewable. Task 5.5.

### 1.9 Drift and minor corrections

Systematic: line references inside `index.js`'s cycle-record region (A§1.4) run about one line low. `agentRouter.js` references run about six lines low. Files the architecture places under `packages/core/src/engine/tools/` or `packages/core/src/platform/` are actually at `packages/core/src/` top level (verified against 4.0.1).

**Revision 2 — the engineering-completion sequence's boundaries, settled.** R§4(i) flagged that Part 1 (`index.js:1839-1945`, PR guard `:1858`) and revision 1 of this document (`:1841-1941`, guard `:1857`) disagree by 1–4 lines at both ends of a ~100-line move, and instructed "find by content". Both are wrong, and the reason matters more than the offsets. Re-read and pinned:

| Anchor | Line | Verified text |
|---|---|---|
| First statement of the sequence | **`:1839`** | `cycle.updatedAt = new Date().toISOString();` |
| Success branch opens | `:1841` | `if (engineeringResult.success) {` |
| Validation | `:1842-1847` | the `Missing:` progress line and `applyTransition(cycle, cycleStatuses.FAILED)` |
| **The outer PR guard** | **`:1858`** | `if (!cycle.pullRequest?.number) {` — Part 1 is right. Note there are **three** occurrences of that exact expression in the file (`:1858`, `:1903` inside the 422 fallback, `:2001` in a different function), so it cannot be located by grep alone |
| Success branch closes / `else` for a failed result | `:1930-1933` | `} else { applyTransition(cycle, cycleStatuses.FAILED); cycle.error = engineeringResult.error; }` |
| Persist | `:1935-1943` | `// Save final cycle state (guard against concurrent cancellation)` through the `PutCommand`'s closing `}));` |
| **`catch (condErr)` with `continue`** | `:1944-1951` | `continue;` at **`:1948`** |

**The load-bearing fact neither document states: `:1948`'s `continue` is loop control.** It belongs to `for (const record of event.Records)` at `:1692`. An extraction that moves the `try/catch` into a function makes `continue` a syntax error, and an implementer who notices that mid-task will improvise. The indentation confirms the natural seam: `:1839` and `:1841` sit at two-space indent (someone de-indented the block in an earlier edit) while `:1935` returns to the loop body's six spaces.

**Instruction, which Task 4.3 implements:** extract `:1839-1943` — decision logic **and** the conditional `PutCommand`, because that `ConditionExpression` is hard-won and must not be duplicated — and **let `ConditionalCheckFailedException` propagate to the caller, which keeps its own `catch` / `results.push` / `continue` exactly as today.** Assert the first and last moved statements by name in the test, per R§4(i).

Revision 2 first paired `:1943` with a `'persisted' | 'cancelled'` return value, which is incoherent: a function whose last statement is the `PutCommand` does not catch, so it cannot report `'cancelled'`. Both coherent options were available, and the choice belongs to this document:

| | Shape | Verdict |
|---|---|---|
| **(i)** | Extract `:1839-1943`. The exception propagates. The caller keeps `catch (condErr)` → `results.push` → `continue` | **Taken.** The exception path is unchanged, so "behaviour must be identical" is literally true and the diff is a pure move |
| (ii) | Extract `:1839-1951`, catch internally, return `'persisted' \| 'cancelled'`, caller branches on the string | Nicer shape, and where this should end up eventually. But it **converts an exception path into a return value**, which is a control-flow change, and it must not ride a commit whose stated contract is behaviour preservation. If it is wanted, it is a separate follow-up commit after Legacy Gate 2 has passed |

So: the extracted function returns nothing meaningful, and **both** callers own their own disposition — the SQS loop with `continue`, the consumer with a clean return. That is two small duplications of a three-line `catch`, which is the price of a provably behaviour-preserving move.

| A§ claim | Corrected |
|---|---|
| `writeStatus` at `index.js:104-129` | Declared `:104`, body ends `:127` |
| `applyRuntimeEffect` at `index.js:140-166` | Declared `:140`, body ends `:168` |
| `routeAfterEngineering` at `index.js:175-309` | Correct. The ai-qa `PutCommand` is `:198-204` ✓ |
| `runtimeEvents.js` hardcodes stage at `:56,68,75` | `:56`, **`:67`**, `:75` |
| cycle record at `index.js:1067-1104` | `:1068-1109`. `startedAt` `:1090`, `currentIteration` `:1091`, `maxIterations` `:1092`, `progressLog: []` `:1095` |
| `cycle.currentIteration += 1` at `:2118-2119` | `:2120` |
| max-iterations `FAILED` at `:2108` | `:2107` |
| `handleRequestRevision` at `:4870` | `:4871`; `expandBusinessGoals` call `:4903-4906` |
| `addProgressLog` `list_append` at `progressLogger.js:44` | `:43`. `:44` is the `ConditionExpression: 'attribute_exists(PK)'` |
| `engineeringAgent/orchestratorHandler.js` SSM branch `:380-398` | `:378-397` |
| `agenticBuildLoop` at `ssmBuildRunner.js:732-739` | Declared `:555`; the only call is `:934` |
| `ssmBuildRunner` 300s timeout at `:24` | `COMMAND_TIMEOUT_SECONDS = 300` at `:23` |
| `engineeringAgent/index.js` placeholder markdown `:218-222` | `placeholderContent` is `:236`, used `:283`. `:222-226` is a TODO comment |
| unused bedrock-agent import `:4-10` | Only `:4` is dead (plus unused `bedrockClient` `:12`, `BEDROCK_AGENT_ID` `:16`, `BEDROCK_AGENT_ALIAS_ID` `:17`). `:5-10` are live requires. **And `@aws-sdk/client-bedrock-agent-runtime` is not in `engineeringAgent/package.json`** |
| `agentRouter.js` maps `:14-26`, `getDevAgent:77`, `getQAAgent:105`, `'none'` `:108-115`, `generatePlan:138-152`, `supportsPlanning:143` | `AGENT_ARNS` `:14-27`; `getDevAgent` `:84` (`devAgent \|\| 'nevado'` `:85`); `getQAAgent` `:111` (`:112`); `'none'` `:115-122`; `generatePlan` `:147-158`; `supportsPlanning` guard `:151` |
| `testResultPoller` — *"any `findCyclesByStatus(FAILED)` query"* in the 4b touch list | **Vacuous.** `findCyclesByStatus` (`testResultPoller/index.js:93`) queries GSI4 and has six call sites (`:348, 427, 656, 774, 1302, 1343`), none of them `FAILED`. `updateCycleStatus` removes `GSI4PK` on a non-polled transition (`:1463`), so a `FAILED` cycle is unqueryable there by construction. Drop this item; a new terminal status needs no poller change |
| *"the card renders an unstyled unknown status"* (4b) | `getStatusConfig` (`cycleStatusConfig.js:171-173`) falls back to `{label: status.replace(/_/g,' '), color: 'default'}`. An unmapped status renders a **grey chip with no button** — an inescapable dead end of exactly the class `cycleStatusInvariants.test.js` guards, not a visual glitch |
| `retryStalled` *"acts unilaterally on a time heuristic"* | Half right. It does not consult the session (correct, and Phase 4 fixes it), but it is **not** unguarded against concurrency: `index.js:3952-3961` refuses with 409 unless `cycleStatuses.isStalled(cycle)`, and the write at `:3989+` carries `ConditionExpression: '#status = :expected AND updatedAt = :lastSeen'` |

### 1.10 Runtime-side: the repo is `nevadoai/nevado-sherpa-tui`, and the claims are verified

**Revision 1 of this document was wrong about this and the error mattered.** It said *"`nevado-sherpa-tui` does not exist on this machine"* and marked every runtime-side item "⚠ VERIFY BEFORE IMPLEMENTING". R§4(m) is right: **there are three repos, not two.**

**Revision 8 — named by repository, not by path.** Revision 2 fixed the first error and introduced a smaller one: it recorded the runtime as `agent-workflow` and cited a directory under one person's home. `agent-workflow` is a local clone directory name, not a repository — the repository is **`nevadoai/nevado-sherpa-tui`**, which is exactly the name revision 1 went looking for and concluded did not exist. Every reference below is by repository name. A second reader clones wherever they like.

| Repo | Role |
|---|---|
| **`nevadoai/command-center`** | Cycles, steps, roles, tenants. This document's subject |
| **`nevadoai/nevado-sherpa-tui`** | **The runtime server.** `apps/runtime` is the HTTP service; `apps/tui` and `apps/web` are the human surfaces. Also the home of the `/v1/sdlc` plugin, the terminal tools, the `resultSchema` validator and the SQS publisher — `apps/runtime/src/sdlc/` |
| **`nevadoai/sherpa-sdk`** | `@nevadoai/sherpa-core` and `@nevadoai/sherpa-protocol`. Its `develop` runs well ahead of the deployed `v4.0.1` |

**Read 4.0.1, not HEAD**, and read it out of the runtime's installed tree: `nevado-sherpa-tui apps/runtime/package.json:22` declares `"@nevadoai/sherpa-core": "^4.0.1"`, and the deployed source sits at `node_modules/.pnpm/@nevadoai+sherpa-core@4.0.1/node_modules/@nevadoai/sherpa-core/src/`. Every SDK line reference below is to that tree. **The "patch 4.0.1 and publish 4.0.2" plan is struck** (§0.2, §1.14): the SDLC work was built in `apps/runtime/src/sdlc/` against the unchanged 4.0.1 instead.

Verified in `nevado-sherpa-tui`, and these are the facts Phases 4–7 rest on:

| Claim | Verified | Consequence for this document |
|---|---|---|
| **Workspace provisioning is clone-only.** `git clone --depth 1 [-b <branch>] -- <repoUrl> <baseDir>/<sessionId>` | `apps/runtime/src/agent/workspace.ts:33-47`. `destroyWorkspace` is `fs.rm(dir, {recursive, force})` at `:83-86`. **Zero `worktree` hits in all of `apps/runtime/src/`** | Release-before-create loses its entire justification. Two independent clones can both hold `cycle/12`. **Task 4.7 reverses to create-then-release** |
| **`DELETE /sessions/:id` already exists and returns `200 {ok: true}`** | `apps/runtime/src/routes/sessions.ts:417-439`; cancels an active session `:424-426`, destroys an ephemeral workspace `:428-430`, releases the GitHub token `:432-435`, deletes the record `:437`, `reply.send({ok: true})` `:438` | `DEP-P1-1`'s `204` is wrong. Corrected in §3.4 |
| **Create returns `201`** | `sessions.ts:347`, `:370` | `DEP-P1-4` corrected |
| ~~**`commitSha` and `baseBranch` are unreachable from the create path**~~ **— SUPERSEDED in revision 8: they shipped, on the SDLC route only** | Then: `cloneWorkspace` took `(sessionId, repoUrl, branch?)` only (`workspace.ts:33`). Now: the SDLC route calls **`provisionWorkspace`**, which clones, fetches `baseBranch` into `origin/<base>` so a diff target exists, and pins `commitSha` — **throwing**, after destroying the tree, if the pin fails (`routes/sdlc.ts:619-658`). `branch` and `commitSha` are mutually exclusive; `baseBranch` is required with `commitSha` | **Phase 6 can depend on existing behaviour now.** `DEP-P1-10` is granted: a bad sha is a `400` at create, not a `201` that reviews the branch tip (§1.11b). Task 6.3's own verification stays — it catches a correct pin of the *wrong* sha, which no create-time check can see |
| **The concurrency cap defaults to 5, and the cap is exactly `max`** | `apps/runtime/src/config.ts:49` is `parseInt(process.env.MAX_CONCURRENT_SESSIONS \|\| '5', 10)`. The gate is `if (this.activeSessions.size > this.config.maxConcurrentSessions)` — `>`, not `>=`, **and that is what makes the cap exact, not off by one**: `:127` reserves into `activeSessions` *before* the gate at `:134`, so the starter is already counted, and the comment at `:130-133` says so — *"hence `>` (not `>=`) to keep the original admission of exactly maxConcurrentSessions running at once"* | Risk R4 in §10.1. Deployed value is 10 via cloud-init, so the live cap is **10**: the tenth starter sees `10 > 10` false and runs; the eleventh parks |
| **A released session's concurrency slot frees asynchronously, after `DELETE` has returned** | `worker.ts:228-232` — the slot is deleted and `this.queue.shift()?.()` fires in `startSession`'s `finally` | Release is not instantaneous. Another reason create-then-release is the safer order: waiting for a slot to free before creating would deadlock |
| **Release is not just disk reclamation — it revokes a live push credential** | `apps/runtime/src/github-token-manager.ts`. `acquire(repo)` refcounts and, on the first holder, mints a GitHub App installation token scoped to **every** repo in `activeRepos` (`:62-71`), writes `~/.config/gh/hosts.yml`, and arms a refresh timer (`:73-82`) that re-mints with `[0, 30s, 60s, 120s]` retries (`:84-90`). `release(repo)` decrements, and **only when `activeRepos.size` reaches 0** does it `stopRefresh()` + `deleteHostsFile()` (`:36-49`) | **Raises Task 4.7's release-failure severity from a warning to a cycle-visible error.** A skipped or failed release leaks a **live, self-refreshing push credential**, not a directory |
| **A leaked acquisition is observed, not theoretical** | On the live testing instance, `hosts.yml` mtime was `2026-09-24 13:44 UTC` while `/health` reported `{"activeSessions":0,"queuedSessions":0}`, unit active since `2026-09-22 17:04:58`. `doRefresh()` returns early when `activeRepos` is empty (`:63-64`), so a rewrite with zero sessions can only mean a leaked acquisition — one that has been re-minting a credential every ~50 minutes for two days | The failure mode Task 4.7 must prevent already happens on the box |
| **There is no workspace TTL sweep** | 86 of 89 `/workspaces` directories contain a `.git`, oldest mtime `2026-06-08`. Nothing prunes them; `pruneWorktrees` is a worktree primitive and the runtime has no worktrees | **A§3.8's "TTL sweep is the backstop" does not exist.** Revision 2 leaned on it to justify downgrading release failure; that justification is void |
| **One `hosts.yml` and one `activeRepos` map serve every concurrent session** | The token covers `Array.from(this.activeRepos.keys())` (`:63`, `:66`) and lands in a single file | **Concurrent sessions on different repositories each hold a credential for the other's repo.** Matters from Phase 4 onward, where 10 slots are shared across applications and with humans — see R14 |
| **Cancelling a *queued* session does not stop it — it later runs** | `cancelSession` (`worker.ts:235-248`) aborts the controller and sweeps pending approvals. It does **not** remove the parked resolver from `queue` (`:41`). When a slot frees, `:231`'s `queue.shift()?.()` resolves it and the aborted turn proceeds into `startSession`'s body | **Affects Task 4.7's cycle-termination release.** Cancelling a cycle whose session is queued would otherwise start an agent on a cancelled cycle's branch. Task 4.7 must call `releaseSession` (`DELETE`), not `cancelSession`, on termination — `DELETE` deletes the record, which is the only thing that makes the resumed turn harmless |

What remains genuinely unverifiable here: nothing load-bearing for Phases 4–7. The items this document previously marked "⚠ VERIFY" are now either verified above or are **Part 1's to build** (the `/v1/sdlc` namespace, the ACL, the per-session profile, the SQS publisher, the `commitSha`/`baseBranch` create support, and the tool allowlist). Those are dependencies, not unknowns — see §3.4.

**Revision 8 — most of that list is no longer a dependency, because it shipped.** Built and readable in `nevado-sherpa-tui apps/runtime/src/`: the `/v1/sdlc` namespace as its own encapsulated plugin with its own authorization hook (`routes/sdlc.ts`, pinned by `routes/sdlc-prefix.integration.test.ts`), the bearer verifier (`sdlc/auth-token.ts`), the ACL scoping on `application === 'sdlc'`, the per-session profile and its exactly-one-terminal-tool rule (`routes/sdlc.ts:337-353`), the three terminal tools (`sdlc/terminal-tools.ts`), the `resultSchema` validator (`sdlc/result-schema.ts`), the opaque correlation (`sdlc/correlation.ts`), the SQS FIFO publisher and transport (`sdlc/publisher.ts`, `sdlc/sqs-transport.ts`), and the read-only git verification `reportCompletion` uses (`sdlc/git-state.ts`). **Read those files rather than §3.4's rows where the two disagree** — §3.4 records what Part 1 was *asked* for, and the answers moved.

Three of those answers are better than the rows that asked for them, and a task author should know before designing around the old shape:

- **`commitSha` / `baseBranch` on the create path exist, and a failed pin is a `400`** — `DEP-P1-10` granted. The route calls `provisionWorkspace`, not `cloneWorkspace`: it clones, fetches `baseBranch` into `origin/<base>` so a diff target exists, and **throws** if the `commitSha` pin fails, after destroying the tree. That throw is the point, and it is precisely the §1.11b hazard closed at the cheapest moment: the SDK's `checkoutCommit` never throws, it returns `false` and the session proceeds against the branch tip — *a review of the wrong code with no error anywhere*. `branch` and `commitSha` are mutually exclusive, and `baseBranch` is **required** when `commitSha` is supplied.
- **The branch-candidate fallback is suppressed under `commitSha`.** One attempt, on the branch asked for, because falling back to `develop` and *then* pinning would silently review a different branch's history. On a plain branch create the fallback still runs, and **that is the unlogged substitution in nevado-sherpa-tui #103** (§1.13) — `session.branch` records which candidate won, but nothing says it differed from the request.
- **`tokenManager` is wired and non-optional** — `DEP-P1-15` granted, and more strongly than asked: `registerSdlcRoutes` **throws at registration** without one (*"an SDLC session exists to commit and push"*), so the failure cannot reach a running session. The post-acquire failure arm releases the credential, deletes the row and then refuses, in that order, and it covers the `commitSha` pin path too — the arm an implementer forgets, because the human route has no equivalent to copy.

### 1.11 OD-4, closed: the `commitSha` checkout is not detached, and a bad sha is silently ignored

Revision 1 declared this **BLOCKING and unverifiable** and recommended a ten-minute empirical test. It is now answered from source, and the answer is worse than the question — which is why it gets its own subsection rather than a table row.

> **Revision 8 — the SDK behaviour below is unchanged and still worth reading, but the SDLC path no longer reaches it.** `DEP-P1-10` shipped: the SDLC create route calls `provisionWorkspace`, which **throws** on a pin failure after destroying the tree, and the route returns a `400` naming it. The runtime's own comment gives the same reasoning this subsection arrived at independently — *"the SDK's `checkoutCommit` never throws: it returns false and the session proceeds against the branch tip, which on a review step is a review of the wrong code with no error anywhere."* So §1.11b's hazard is closed **at create** for `/v1/sdlc`, and the branch-candidate fallback is additionally suppressed whenever `commitSha` is supplied — one attempt, on the branch asked for, because falling back to `develop` and *then* pinning would silently review a different branch's history.
>
> **Read the rest of this subsection anyway, for two reasons.** It still describes the **human** route, which does reach `checkoutCommit`. And it is the argument for why Task 6.3's independent verification stays: a create-time check catches a sha that cannot be fetched, and nothing at create time can catch a **successful pin of the wrong sha**.

Read `sherpa-core@4.0.1/src/git-checkout.ts` in full; it is 66 lines and three of its comments are the answer.

**(a) HEAD is not detached, and that was a deliberate rejection.** `checkoutCommitArgv` (`:42-47`) is:

```
git fetch --depth 1 origin -- <commitSha>
git reset --hard FETCH_HEAD
```

and `:29-32` says why, verbatim: *"Uses fetch + `reset --hard`, NOT `checkout FETCH_HEAD`: a shallow `-b <branch>` clone is already on the branch, and `reset --hard` moves that branch ref to the commit while keeping us on it. Checking out FETCH_HEAD would leave a DETACHED HEAD, so the resumed agent's commits/branch-detection would misbehave."*

So `branch` **is** a real ref on a `commitSha` session and **is** returned by create. **Task 6.3's revision-1 acceptance check — "`branch` being `null`/absent" — would have failed on correct behaviour**, and its implementation note calling QA's clone "a detached clone (harmless)" was wrong. Both are fixed. Part 1 is right; this is R§4(a).

**(b) `checkoutCommit` never throws. A bad sha silently serves the branch tip.** `:56-66`:

```ts
export async function checkoutCommit(cwd, commitSha, run = defaultGitRunner): Promise<boolean> {
  if (!isValidCommitSha(commitSha)) return false;
  try { for (const args of checkoutCommitArgv(commitSha)) { await run(args, cwd); } return true; }
  catch { return false; }
}
```

with the docblock at `:49-54` stating the consequence outright: *"Returns true if HEAD landed on that commit, false if we couldn't (unpushed on origin, server rejects want-by-SHA, or the sha is invalid) — the caller then resumes against whatever the clone already checked out (the branch tip). Never throws."*

**This is the worst failure mode in this specification and it is not the one revision 1 identified.** Revision 1 worried that QA could not compute a diff and would review nothing. The real hazard is that QA reviews **the wrong code with no error anywhere**: a sha that is valid-looking but unfetchable (a force-push, a `--depth 1` server that rejects want-by-SHA, a sha from a different repo) yields `false`, the session proceeds on the branch tip, and QA returns a confident verdict on code the cycle never asked it to review. A check that silently checks the wrong thing is worse than a check that fails.

**Mitigations, and they are layered because no single one is sufficient:**

1. **Part 1 Task 2.3 must turn a failed `checkoutCommit` into a create-time `400`**, not a `201`. That is Part 1's, it is already scoped there, and it is the only mitigation that prevents the session existing at all. Recorded as `DEP-P1-10`.
2. **CC verifies independently, because a `400` depends on Part 1 having done (1).** Task 6.3 adds a post-create assertion: the created session's reported `commitSha` must equal the requested one, and if the runtime does not report it, `GET /sessions/{id}` must. A mismatch fails the cycle to `QA_FAILED` before any review is posted.
3. **The review itself carries the commit.** Task 6.5 already posts through `pulls.createReview`; GitHub records the review against a `commit_id`. Phase 6 exit criterion 2 compares it to `cycle.pullRequest.headSha`, which catches (2) escaping.

**(c) A three-dot diff is unsound on this clone, and two-dot is the honest form.** This is derivable, not a coin-flip. `git-checkout.ts:33-35` rejects ancestry reasoning for exactly this reason: *"NO `merge-base --is-ancestor`/`cat-file` — a grafted `--depth 1` commit has no known ancestry, so ancestor reasoning is unsound."* `git diff A...B` is defined as `git diff $(git merge-base A B) B`, so it needs the very ancestry the SDK's own comment says does not exist over two independent `--depth 1` grafts.

**Therefore: `git diff origin/<baseBranch> HEAD` (two-dot), not `origin/<baseBranch>...HEAD`.** R§4(g) is right that Part 1 leaving a hole in a normative section to be "written back during implementation" will not survive an unsupervised implementer, and Task 6.3 makes the command a **named deliverable**: a fixture file the QA instruction composer imports, not prose. If Part 1 Task 2.3's empirical test contradicts this reasoning, Part 1 wins and the fixture changes in one place.

### 1.12 The plan-review surfaces: which are live, and what they actually read

D§5(a) reports that both specs derive the plan schema's required set from a dead dialog, and that `approach` therefore has no live consumer. **Verified, and it is right about committed `HEAD` — but the uncommitted working tree changes the answer, and the difference decides two tasks.** See §12.4 for the disagreement stated as such; this section is the evidence.

#### At committed `HEAD`: the dialog is dead

`git show HEAD:frontend/src/pages/ApplicationDetail.jsx` — `planningReviewData` is `useState(null)` at `:75`, the dialog is guarded by `{planningReviewData && (` at `:1670`, and the **only** four references are `setPlanningReviewData(null)` at `:1670`, `:1675`, `:1740`, `:1746`. Nothing ever sets it to a cycle. The dialog is unreachable, exactly as D§5(a) says, and with it the `technicalApproach` panel.

#### In the working tree: the uncommitted change resurrects it

A grep for `setPlanningReviewData(` misses the site that matters, because it is passed **by reference**, without a paren:

- `ApplicationDetail.jsx:618` — `onReviewPlan={setPlanningReviewData}` (uncommitted; appears as an addition in `git diff`).
- `AttentionCard.jsx:25-26` — `case 'approve_plan': onReviewPlan?.(cycle); break;` and `case 'review_plan': onReviewPlan?.(cycle); break;` (uncommitted).
- `cycleStatusConfig.js` — `PLANNING_REVIEW.primaryAction` changed from `'approve-plan'` to `'review-plan'`, label "Review Plan" (uncommitted).

Those three changes are one coherent piece of work whose whole purpose is to make the dialog reachable from the attention card, feeding it a full cycle object. So on the tree that will actually ship there are **two** live plan-review surfaces, reached differently:

| Surface | Reached from | Reads |
|---|---|---|
| **`PlanReview.jsx`** (the primary) | `ActiveCycleHero.jsx:51-62`, when the cycle is the **active** cycle at `PLANNING_REVIEW` | `cycle.expandedRequirements \|\| cycle.plan` (`:16`); `cycle.planVersion` (`:17`); `plan.requirements \|\| plan.tasks` (`:19` — **a `tasks` alias neither document accounted for**); `plan.summary` (`:118`); per-requirement `id`, `title`, `description`, `complexity`, `acceptanceCriteria`, `dependencies` (`:125-178`); `plan.assumptions`, `plan.risks` (`:187-208`). **Never `approach`, never `technicalApproach`** |
| **The `ApplicationDetail.jsx` dialog** (`:1676-1800`) | The **attention card**, once the uncommitted work lands | `task`; `expandedRequirements.summary`; `.requirements[]`; **`.technicalApproach`** (`:1750`, `:1753`); `.risks[]`; `cycle.plan`; `cycle.status` |

#### What follows for the schema

1. **`approach` keeps its place in the required set, and the real reason is not a dialog at all.** Revision 2's reason — "the plan-review dialog consumes it" — was half dead code, and D§5(a) is right to strike it. But striking it does not leave `approach` unjustified; it leaves it justified by **four readers that have nothing to do with the plan UI**:

   | Reader | What breaks without it |
   |---|---|
   | `cursorAgent/devHandler.js:98` — `requirements: plan?.approach \|\| task` | **The strongest.** Under `application.devAgent: 'cursor'` the plan's `approach` **is the dev agent's entire requirements input**. A missing one silently degrades a full plan to the one-line business goal, and the agent builds the wrong thing with no error anywhere |
   | `cursorAgent/devHandler.js:150` — `` `**Approach:** ${plan.approach}` `` | Unguarded interpolation: an absent value renders the literal `undefined` into the dev prompt |
   | `integrationAgent/orchestratorHandler.js:112` — `plan?.approach \|\| ''` | Knowledge-pack detection loses a signal, degrading silently |
   | `recordCheckpoint`'s evidence — `index.js:1315` | `stepAuditEvidence.test.js:187` fails, which is the desirable direction |

   The attention-card dialog is a fifth, after Task 5.3. `generatePlan` has emitted `approach` since before this project (`orchestratorHandler.js:305`). **So `approach` is required regardless of what happens to the uncommitted frontend work** — which removes the contingency revision 3 flagged at the end of §12.4.
2. **Task 5.3 is not "fixing code no user reaches."** It is the change that makes the second surface work, and it is worth keeping for that reason — but its framing changes: it fixes a panel that is *about to become* reachable, not one that has been silently broken for months.
3. **The schema gains two fields the documents missed, and one apparent third that turns out not to be ours.** `requirements[].dependencies` is rendered by the live surface (`PlanReview.jsx:125-178`) and no revision before 3 required it. `cycle.planVersion` (`:17`) is stamped only on the revision path (`planComments.js:304-305`), so the primary surface shows no version badge for a first-pass plan.

   The third — `PlanReview.jsx:19`'s `plan?.requirements || plan?.tasks` — is **not** a synonym of our plan shape, and revision 3's first instinct (tolerate and normalise it) was wrong. `tasks` is the **cursor** backend's plan key: `cursorAgent/cursorClient.js:286-294` returns `{approach, tasks, filesToModify, filesToCreate, complexity, generatedAt, generatedBy: 'cursor'}` with no `requirements`, `summary`, `risks`, `assumptions` or `estimatedEffort`. The alias is frontend compatibility for a different agent backend. `PLAN_PAYLOAD_SCHEMA` stays single-keyed on `requirements`; Task 5.1 has the reasoning and the reversed test.
4. **The crash D§5(b) identifies is real and the schema is the fix.** `PlanReview.jsx:19-20` is `const tasks = plan?.requirements || plan?.tasks || []` then `tasks.reduce(...)`. A truthy-non-array `requirements` — a string is the likeliest malformation from a model — makes `.reduce` undefined and **the approval gate throws before render**. Everything else in that component is optional-chained; this line is not. `requirements` must therefore be `{type: 'array'}` in `resultSchema` so the session catches it, and an array in CC's validator so the consumer catches it if `resultSchema` was absent.

#### One more live consumer worth knowing before Phase 5 touches it

`cycle.approvedPlan` is **written and never read.** Verified: written at `index.js:1594`; the only other occurrences are a `null` initialiser (`:1087`) and a **local** variable of the same name in `processSQSCycleExecution` (`:1737`, `:1783`) sourced from `expandedRequirements`, not from the field. Task 5.6 previously said the engineering prompt is composed from *"`cycle.expandedRequirements` + `cycle.approvedPlan`"*; the second half would compose from a field nothing has ever read. Compose from `expandedRequirements` and treat `approvedPlan` as the audit record it is.

### 1.13 The working branch is DERIVED, `cycle/**` is a wire constraint, and the round trip is where the live defect is

**New in revision 8, and it supersedes every line in this document that reads as though an agent picks or reports the branch.** Issue **#851** — *nothing wrote `cycle.branch` before the first engineering dispatch, so a runtime cycle could not leave plan approval* — is **FIXED and CLOSED** by PR **#857**. Anywhere below that treats the branch as unset, hand-seeded or agent-chosen is stale; this section is the corrected account.

#### The name is a pure function of the sort key

`cyclerecord.WorkingBranch(sk)` is `"cycle/"` joined to `strings.TrimPrefix(sk, "CYCLE#")` (`backend/go/internal/orchestrator/cyclerecord/cyclerecord.go:144-150`). That is the whole derivation, and its docblock says why it must stay that shape: *"every writer, every replay and every concurrent invocation computes the identical string. There is no state for a race to corrupt … **Do not add a suffix of any kind** — an attempt counter, a timestamp, a slug — because determinism is the entire argument."*

Two consequences a task author needs:

- **An SK with no cycle number yields `""`, not the bare prefix.** `cycle/` is not a ref git will accept, so it would fail *inside* the agent's session after the create had been paid for; and if it ever did land, every cycle in that state would share one branch. The empty string falls through to `DispatchEngineering`'s refusal instead, which names the cycle and the attribute and leaves the record untouched.
- **It agrees with the legacy SSM path by construction**, which computes the same name agent-side at `engineeringAgent/orchestratorHandler.js:463-464` and `ssmBuildRunner.js:863-864`. Same input, same output, **no reconciliation and no JavaScript edit** — which matters because a cycle can begin on one path and be inspected from the other.

#### Two sites, and the second is the one to build against

1. **Derived at approve-dispatch.** `approvedispatch.ensureWorkingBranch` (`approvedispatch/approvedispatch.go:101-147`) sets it on the live `RuntimeSessionSpec.Cycle` map before the dispatch reads it. It is **`WHEN ABSENT, NEVER ALWAYS`** — it names the branch only if nothing has named one, because from the first engineering completion onward the recorded branch is the one carrying the commits, and overwriting it with a derived name would send a later iteration to an empty branch. Whitespace and non-strings count as absent.
2. **Recorded on the engineering CLAIM WRITE.** `runtimesession.claimWrite` adds it to the guarded `set` (`runtimesession/runtimesession.go:465-491`, field constant `BranchField = "branch"`), so it rides the one `ConditionExpression`-guarded write — the same reason the stub flag rides it. **A loser of the dispatch race has its branch refused wholesale with the rest of its claim.** It carries **no condition of its own and needs none**, because winner and losers compute the same string.

Why not cycle creation, which would be tidier: `cycle.BuildCycle` is not wired into any deployed Lambda — `agent-orchestrator` is still `nodejs20.x` running `agentDrivenOrchestrator.zip` (`infrastructure/lambdas.tf:1554-1560`) and cycle creation still runs in JavaScript. And not inside the approval gate itself, because that is a port with a recorded golden, and a derivation added there would be a port that quietly does something the JavaScript never did.

#### `cycle/**` is a wire constraint, not a naming preference

The CI workflow the provisioner commits into **every customer repository** has exactly one trigger — `on: push: branches: ['cycle/**']` (`backend/lambda_handlers/applicationProvisioner/githubClient.js:1670-1687`). No `pull_request`, no `workflow_dispatch`; the inline comment in that file claiming it runs on PRs is wrong. The AI-visual-test workflow lists the same glob alongside `main` and `develop` (`:1089-1096`); the two deploy workflows are `develop`/`main` only.

**So a working branch named outside `cycle/**` does not produce a failing build — it produces NO WORKFLOW RUN AT ALL**, and that is the expensive direction. `buildpoll` queries workflow runs for the cycle's branch and waits; with no run to find there is nothing to go red, nothing in the feed, and the cycle sits in `BUILDING` until the stall detector eventually reaps it. `cyclerecord.go:94-114` puts it exactly: *"a reader renaming this to something tidier would see every test pass and every cycle hang."* Treat the prefix as protocol. If a task below reads as though the prefix is cosmetic, it is wrong.

#### The round trip is unchecked, and that is command-center #854

From the first engineering completion onward the derived name is **replaced** by whatever the session reported: `engcomplete.go:345` writes `in.Cycle["branch"] = in.Result["branchName"]`, and nothing compares the reported value against the requested one. That is **#854**, and #851's fix is what made it reachable — and, for the first time, *checkable*, because the requested branch is now on the record before the session starts.

Its silent half lives in the runtime: a branch-candidate fallback substitutes a branch without logging that it differed from the request (nevado-sherpa-tui **#103**). Combine the two and an agent that landed on `develop` and honestly reported `develop` sets `cycle["branch"] = "develop"` — after which the draft PR is opened `develop → develop`, `buildpoll` polls `develop`'s CI, and the iteration loop appends subsequent engineering work **to `develop`**.

**Both are open. Where a task below specifies the branch round trip — Task 4.3 step 2, Task 4.7b, Task 6.6, Task 6b.1 — it must compare the reported branch against the recorded one and refuse or flag a mismatch rather than writing it through.** The comparison is local; no runtime fetch is needed. A silent misroute onto `develop` is worse than the hard failure #851 used to produce: a cycle that cannot start is obvious, and a cycle that commits to the wrong branch is not.

#### Scope: human approval only

PR #857 says so in terms — *"This unblocks **human-approved** runtime cycles only."* Human approval goes through Go: `cmd/cycle-approval/main.go` → `approvedispatch` → a real `POST /v1/sdlc/sessions`, and its placeholder **refuses rather than stubbing**, because *"returning a synthetic `stub-` id would record a session nothing will ever emit events for"*. **PM auto-approval does not.** `agentDrivenOrchestrator/index.js:1331-1352` calls `shouldAutoApprovePlan` and then the JavaScript `approvePlan`, which reaches `runtimeDispatch.js` — still the stub returning `` `stub-${crypto.randomUUID()}` `` with `stubbed: true` — and the Go equivalent `planning.autoApprove` (`plangeneration.go:254-263`) has **no `cmd/` caller at all.** So a PM-linked runtime cycle goes through the JS gate and its stub dispatcher regardless.

**This is a pre-existing cutover gap, not a regression**, and it is not Phase 4's to close incidentally — but no phase exit criterion may be read as covering PM-linked cycles until one of the two paths is wired. §9.2's rollout order is therefore human-approved applications first.

### 1.14 No SDK release gated any of this, and the one SDK change that mattered was declined on merit

**New in revision 8.** §0.2's *"patch 4.0.1 and publish 4.0.2"* is struck, and the dependency chain `sherpa-sdk → runtime → command-center` — which revisions 2 and 3 both leaned on for scheduling — never existed. Three facts, because the wrong one of them is easy to write:

1. **An SDK release did ship, and nobody waited on it.** `sherpa-sdk` **#227** (merged 2026-09-25) widened the `SessionApplication` union to include `'sdlc'` so the runtime's ACL is type-checked; released as **4.1.2** (#228). It is **types and tests only** — `session-manager.ts` untouched. Command Center shipped no version bump and still pins `@nevadoai/sherpa-protocol` **4.0.1** (`backend/__tests__/sherpaSdkDriftGuard.test.js:40`, `frontend/package.json:27`).
2. **The SDK change that would have mattered was authorised and then declined on merit.** `sherpa-sdk` **#226**, *"default `createdBy` and `application` on session create instead of forcing them"*, is **CLOSED, not merged.** `createdBy` backs a live ownership check — `renameSession` throws `RemoteSessionNotOwnedError` on `record.createdBy !== this.userId` — so defaulting it rather than forcing it would let a machine-supplied label defeat a human surface's authorization gate. **`SessionManager.create` forcing the field is correct. Do not propose relaxing it.** What that costs the ACL is nothing that matters: a shared bearer token has no per-client identity to partition on anyway (§0.2).
3. **`'completed'` was not adopted, and no future status could replace it.** The engine's ten `finish` call sites all pass `'paused'` — normal `end_turn` exits included — so `sessionStatus` reads `paused` for a session that delivered and one that died alike. Making the engine emit `'completed'` is **not** the alternative fix: `nevado-sherpa-ide`'s `/resume` enumerates resumable sessions with `listByStatus('paused')` and would stop seeing them. `'completed'` is a **human complete/archive flag** — `SessionManager.completeSession` still exists and the IDE still calls it — so *"nothing can emit it"* is too strong and *"the engine loop cannot"* is the true statement.

   **And the deeper reason, which is the one to carry into any future SDK conversation: delivery is not a lifecycle state.** A session that finished having submitted a result and a session that finished having submitted nothing are in the *same* lifecycle state, so no status — present or future, however the SDK spells it — can tell them apart. A later SDK growing a `'completed'` status is therefore not an invitation to restore the inference arm; it changes nothing about what a status can mean.

   Source of record: `backend/go/internal/orchestrator/runtimeevents/runtimeevents.go:9-45`, whose package docblock is where this argument is maintained. Pinned executably rather than by comment: nevado-sherpa-tui PR **#101** extracts the status argument of every `finish` call site out of the installed `@nevadoai/sherpa-core` bundle and asserts both that there are **exactly ten** and that the status set is exactly `{paused}` — the count is pinned so the probe cannot pass vacuously, since a rename or a re-bundle reads as zero sites rather than as a pass.

**What this means for scheduling.** The terminal tools, `resultSchema` and its validator were built in `nevado-sherpa-tui apps/runtime/src/sdlc/` against the unchanged 4.0.1, not inside the SDK. So the delivery vehicle for a contract change is a **runtime deploy**, not an npm release — which is the more expensive of the two, is an outage window that kills live human sessions, and is what `OQ-1` and §1.10's release rows are actually about. Do not write a task whose critical path runs through an SDK version.


## 2. Repo conventions you must follow

Read this section before writing a line. Every convention below was observed in the repo, with an exemplar named.

### 2.1 Tests

- **Runner:** Jest 30, config at `backend/jest.config.js`. `testEnvironment: 'node'`, `testMatch: ['**/__tests__/**/*.test.js', '**/?(*.)+(spec|test).js']`, `setupFilesAfterEach: ['<rootDir>/jest.setup.js']`, and a `moduleNameMapper` aliasing `^common/(.*)$ → <rootDir>/common/$1`.
- **Command:** `npm run test:backend` from the repo root, or `npm test` inside `backend/` (`NODE_OPTIONS='--experimental-vm-modules' jest`). Single file: `npx jest backend/__tests__/<name>.test.js` from `backend/`.
- **Layout:** one flat directory, `backend/__tests__/`, one file per unit, named after the unit (`runtimeEvents.test.js`, `sdlcEngine.test.js`, `cycleStatusInvariants.test.js`). Fixtures in `backend/__tests__/fixtures/`.
- **Naming:** `describe` is the unit plus a facet — `describe('mapSessionEvent — terminal events')`. `it` is a sentence about behaviour, not about mechanism: `it('reports a commit-less completion as failure, not silent success')`, `it('refuses to guess when updatedAt is missing or unparseable')`. Copy that voice.
- **Three distinct test styles exist and each has a purpose. Use the right one:**
  1. **Pure-function tests.** Exemplar: `backend/__tests__/runtimeEvents.test.js`. Direct `require`, no mocks, table-driven where it helps. Use for every mapper, adapter and predicate in this spec.
  2. **Orchestrator tests with mocked SDKs.** Exemplar: `backend/__tests__/applyRuntimeEffect.test.js`. `jest.mock('@aws-sdk/client-lambda')`, `jest.mock('@aws-sdk/client-bedrock-runtime')`, `jest.mock('../common/githubAppClient', …)`, then `jest.isolateModules(() => { … require('../lambda_handlers/agentDrivenOrchestrator/index.js') })` with `DynamoDBDocumentClient.from` stubbed to a capturing `send`. Note the `trackCommandArgs()` helper there: auto-mocked command instances carry no `.input`, so payloads are recovered from the constructor's `mock.calls` via a `WeakMap`. **Reuse that helper verbatim** — writing a new one is how these tests silently assert on `{}`.
  3. **Static source / wiring guards.** Exemplars: `backend/__tests__/runtimeDispatch.test.js:104-131` (reads `index.js` as text and asserts the branch exists), `backend/__tests__/stalledCycleDetection.test.js:150-177` (asserts a route is in Terraform **and** in the `-target=` list **and** handled), `backend/__tests__/routeCoverageGuard.test.js`, `backend/__tests__/frontendContractGuard.test.js`. Use for deploy-target membership, dead-code assertions, and "code exists but is not connected" guards. These are the only tool available for Phase 7 and for the `-target=` list.
- **There is no frontend test suite.** Zero `*.test.*` / `*.spec.*` under `frontend/`, no vitest or jest config, no `test` script in `frontend/package.json`, and CI (`deploy-dev.yml:46-58`) runs only `lint:frontend` (with `continue-on-error: true`), `test:backend`, and `build:frontend`. **Every frontend change in this spec is therefore "not TDD — verified by X"**, and where a frontend contract must be enforced, enforce it from a *backend* static guard that reads the `.jsx` as text (the `frontendContractGuard.test.js` pattern). Playwright E2E lives at repo root (`npm test`, specs in `tests/flows`, `tests/integration`) and is not wired into the deploy gate.

### 2.2 Handler and module structure

- Lambda handlers live at `backend/lambda_handlers/<camelCaseName>/index.js` with `exports.handler = async (event) => {…}` and a sibling `package.json` listing only the AWS SDK clients that handler needs. Exemplar: `backend/lambda_handlers/stuckCycleDetector/`.
- Shared code lives at `backend/common/*.js`, CommonJS, `module.exports = { … }` at the bottom. Exemplar: `backend/common/progressLogger.js` (62 lines, one exported function, a long docblock explaining *why*).
- **`process.env` is read only by handlers.** Shared modules in `backend/common/` take configuration as parameters. `progressLogger.addProgressLog(docClient, tableName, pk, sk, stage, message, detail)` is the canonical shape. `backend/common/dynamoHelpers.js` predates this and has a module-level `docClient`/`TABLE_NAME`; do not follow that half of it.
- **New `Queries.*` functions take `(client, tableName, …)`.** Exemplars: `dynamoHelpers.js:845` `createDiagnosis(client, tableName, …)`, `:1077` `acquireWriteLock(client, tableName, …)`. The older entries (`:520-560`) close over module state — do not extend that style.
- **Wrapper convention.** `agentDrivenOrchestrator/index.js:79-81` re-exports the shared `addProgressLog` under the same name with `docClient`/`TABLE_NAME` injected, and the docblock at `:66-78` explains the deliberate name collision. When you add a shared helper that the orchestrator calls often, follow that pattern rather than threading the client through 50 call sites.
- **Logging idiom:** `console.log('[ComponentName] message', structuredObject)` — one bracketed component tag, no logger library, no JSON-only convention in this codebase. `console.warn` for recoverable, `console.error` for failures. Exemplars throughout `index.js`.

### 2.3 The status write path — non-negotiable

Exactly two functions may write `cycle.status`:

- **`sdlcEngine.applyTransition(cycle, toStatus, {stage, validate, strict, now})`** (`sdlcEngine.js:507`) for the in-memory `cycle` object that is then `PutCommand`ed whole. It validates against `TRANSITIONS`, reports illegal transitions via `reportIllegalTransition`, assigns anyway by default, and maintains `GSI4PK`/`GSI4SK` (`:551-562`).
- **`writeStatus(pk, sk, toStatus, fields, options)`** (`index.js:104`) for a targeted `UpdateCommand`. It composes `statusUpdateFragments` from `common/pollingIndex.js` so the GSI4 index and the status can never be observed disagreeing (docblock `:90-103`), and it accepts an `options` spread — **this is where a `ConditionExpression` goes**.

Anything else is a bug. `stuckCycleDetector/index.js:174-197` is the one live violation (§1.5b); Task 4.0 removes it.

### 2.4 New Terraform must be added to the deploy target list — in the same PR

`deploy-dev.yml` runs three `terraform apply` invocations, each with an explicit `-target=` list. A resource absent from all three is written, reviewed, merged, and **silently never deployed**. This is a documented recurring failure (`stalledCycleDetection.test.js:150-153` cites #580).

Entry format, 12-space indent, one per line, backslash-continued:

```
            -target="aws_lambda_function.application_provisioner" \
            -target="aws_cloudwatch_log_group.application_provisioner_logs" \
            -target="aws_lambda_event_source_mapping.provisioner_sqs_trigger" \
```

The **Core** list (`:427-586`) is the one this spec adds to. A Lambda typically contributes 3–6 entries: function, log group, and its trigger (event source mapping, or event rule + target + permission), plus one per API route and integration.

**Every task in this spec that touches Terraform includes the `-target=` additions and a static guard test asserting them.**

### 2.5 New Lambda checklist

Four places, in this order:

1. `scripts/deployment/package-lambdas.sh` — `maybe_package "<zipName>" "backend/lambda_handlers/<dir>"` alongside the JS list at `:443-469`. `maybe_package` (`:386`) calls `package_lambda_with_common` (`:390`), which **rewrites `require('../../common/…')` → `require('common/…')` by sed (`:145-149`)** so the common layer resolves, and copies only `-maxdepth 1` `.js` files (`:141`) — **subdirectories are silently dropped**; use `package_devops_lambda_with_subdirs` (`:601`) if you need them. The zip lands at `infrastructure/<zipName>.zip`.
2. `infrastructure/lambdas.tf` — `aws_lambda_function` + `aws_cloudwatch_log_group` + trigger. Exemplar to copy verbatim: `aws_lambda_function.agent_orchestrator` (`lambdas.tf:1519-1558`) — `filename` and `source_code_hash = filebase64sha256(...)` on the same zip name, `role = aws_iam_role.lambda_execution_role.arn` (one shared role), `runtime = "nodejs20.x"`, `layers = [aws_lambda_layer_version.common_layer.arn]`, and the three-key `tags` block (`Name`/`Environment`/`Project`) used throughout `lambdas.tf`. Note `devops-*.tf` uses a different single-key `tags = { Component = "…" }` convention — follow the file you are in.
3. `.github/workflows/deploy-dev.yml` Core target list — one entry per resource (§2.4).
4. New IAM goes into the shared `aws_iam_role_policy.lambda_policy` (already targeted at `:428`). That role is near IAM's 10,240-byte inline-policy limit, which is why the workflow prunes orphaned policies at `:258` — keep additions minimal.

SQS-triggered Lambdas need **no** `aws_lambda_permission`; the event source mapping pulls.

### 2.6 Go vs JS — superseded by §0.15 and §13

**This section said the opposite of what is now true, and it misdirected real work. Read §0.15 and §13 instead.**

It originally read: *"The SDLC cycle state machine is JS only… Nothing in Phases 4–7 touches Go."* That was accurate against the architecture doc's settled input (A§5.5, "TypeScript in the existing JS orchestrator rather than a new Go executor") and it is now false three times over:

- **The Go mandate** (§0.15) makes every new or updated Lambda Go, so most of Phases 4–6b is Go work.
- **It has shipped.** The transition engine and step graph are Go. `backend/go/internal/sdlc` is no longer "a parallel, non-authoritative implementation" — for the transition table and step graph it is the implementation.
- **The one claim worth keeping** is the provenance note: `sdlcStepGraph.js:51` documents that its `agentId` values were chosen to match the Go side's shape. That cross-language correspondence was hand-maintained; it is now the thing `BQ-GO-1` (§13.4) exists to put on a proper footing.

`routeCoverageGuard.test.js`'s `GO_HANDLER_SOURCES` map still lists which Lambdas are Go-backed, and it is now **in scope** — it gains an entry as each cycle-path handler ports.

### 2.7 Git and PR conventions

Feature branch off `develop`, one commit per task, single-line conventional-commit subject with no body and no footers, PR targets `develop`. New `.tf` resources and their `-target=` entries go in the same PR as the code that needs them.

---

## 3. Ordering, dependencies, parallelism

### 3.1 Execution order

```
Part 1 Phase 0 (free cleanups)          ─┐
Phase 4b (failure statuses, ONE commit) ─┼─ independent, start both immediately
                                          │
Part 1 Phase 1 (auth proof, U1)         ─┘  ← gates everything runtime-side
        ↓
Part 1 Phase 2 (create/read/cancel/release, the 3 tools, resultSchema + validator)
        ↓
Part 1 Phase 3 (SQS transport, sdlcEventConsumer, cycleEffects extraction)
        ↓   ← LEGACY GATE 1: one non-runtime cycle end to end (§9.3)
Phase 4  (engineering cutover, incl. the CI-build-failure loop)
        ↓   ← LEGACY GATE 2: after Task 4.3, one non-runtime cycle end to end
Phase 5  (planning)   ──┐
Phase 6  (QA)         ──┴─ can overlap; see §3.3
        ↓
Phase 6b (deploy-failure loop + approveCycle request_changes)
        ↓
Phase 7  (deletes)   ← needs 5, 6 and 6b complete and buildMode unconditional
```

**Two changes from revision 1.** The CI-build-failure loop (`continueBuildIteration`) moves **into Phase 4** rather than a deferred Phase 6b (R§1.5, R§7.4; see Task 4.7b and the Phase 4 exit criteria). And the two **legacy gates** are new: they are the only thing standing between a shared-code refactor and every non-migrated application (§9.3).

### 3.2 Hard gates

| Gate | Blocks | Why |
|---|---|---|
| **`DEP-P1-16` answered — which Go shape** | **Everything in Phases 4b–7** | The Go mandate (§0.15) makes nearly every task a port rather than an edit, and which port depends on the shape. §13 resolves mechanically once the answer lands |
| **`BQ-GO-1` answered — the status/activity source of truth** | **Phase 4b** | 4b's whole job is adding a status, and under the mandate that status must exist in Go, in a JS artifact for the browser, and in Python. "Where does the constant live" is the first line of the task (§13.1) |
| **The affordance status gates are Go** (`cancellableStatuses`, `continueIteration`'s gate) | **anything writing `ENGINEERING_FAILED`** — Part 1 Phase 3's per-stage `error` mapping, and all of Phase 4 | **Narrowed in revision 7 from "Phase 4b merged, as one commit".** Two corrections. The *one-commit* half is withdrawn: 4b.1 now defines the status in Go, so it never enters `cycleStatusInvariants.test.js`'s `ATTENTION_STATUSES` and the affordance invariant never goes red — 4b ships as port-then-change (§13.0 rule 2). The *gate* half is tighter than "4b merged": those two gates are inline `cycleStatuses.*` array literals in `agentDrivenOrchestrator/index.js`, and adding the status to them needs both a forbidden edit to read-only `backend/common/cycleStatuses.js` and an in-place JS Lambda edit — **no order of operations satisfies both.** So part of the orchestrator port precedes the rest of 4b. Pinned executably in `backend/__tests__/fixtures/goOnlyStatuses.js`. Task 4b.4 fact (a) |
| Task 4.0 merged **and applied** | Phase 4 exit criterion 3 | `stuckCycleDetector` is in no `-target=` list, so a code change to it never reaches AWS (§1.5a) |
| Task 4.1 merged | Tasks 4.2, 4.7, 4.7b, 5.6, 6.3, 6.6 | Three concurrent sessions need three fields (§1.4) |
| Part 1 Task 3.3 (`cycleEffects.js`) merged **and soaked** | Tasks 4.3, 4.4, 6.2, 6.4 | The consumer cannot `require` from a handler directory. Part 1 calls this "the riskiest revert in Part 1"; let it sit through the rest of Phase 3 before building on it |
| **Legacy gate 1** — one non-runtime cycle end to end after Part 1 Task 3.3 | Phase 4 | Task 3.3 moves `routeAfterEngineering` on five live call paths with no flag (§9.3) |
| Part 1 Task 3.2's `completion` arm scoped to `engineering`, with `plan`/`review` reported as a batch item failure | Tasks 5.1, 6.5 | Otherwise a plan or review completion arriving between Phase 3 and Phase 5/6 is mapped to an effect nothing handles, `console.warn`ed at `index.js:164`, and the SQS message deleted — silent data loss (R§1.4). See `DEP-P1-11` |
| Part 1's `DELETE /sessions/{id}` client shipped | Tasks 4.7, 4.7b, and Phase 6 entirely | Release-on-supersede. **No longer a collision guard** — see §1.10 |
| **Legacy gate 2** — one non-runtime cycle end to end after Task 4.3 | Phases 5, 6 | Task 4.3 moves 100 lines out of the legacy SQS path (§9.3) |
| Phase 5 Tasks 5.5a and 5.5b merged | Phase 7 Task 7.5 | Deleting `expandBusinessGoals` breaks plan revision and silently drops the chat transcript otherwise |
| Phase 6b merged | Phase 7 Task 7.7 | Task 7.7 deletes `invokeEngineeringAgent`, and the deploy loop still calls it |
| `buildMode: 'runtime'` unconditional for a full release cycle | All of Phase 7 | Phase 7 is the point of no return (§9.3) |

### 3.3 What can run in parallel

- **Phase 4b** with all of Part 1 Phases 0–3. No shared files. One commit.
- Within Phase 4: **4.0** (deploy target list + `stuckCycleDetector`'s status write) is independent of **4.1–4.4**. **4.5 / 4.6** depend on 4.0 and 4.1. **4.7b** depends on 4.1 and 4.7.
- **Phase 5 and Phase 6** touch mostly disjoint code (5 touches `processPlanGeneration`, `startDevelopmentCycle`, `retryPlanning`, `retryStalled`, `handleRequestRevision`, `agentRouter`; 6 touches `routeAfterEngineering` in `cycleEffects.js`, `qaAgent/orchestratorHandler`, `continueIteration`). They collide on the per-step session-id helper from Task 4.1, on `dispatchEngineeringSession` from Task 4.7b, and on `runtimeEvents.mapSessionEvent`'s `completion` dispatch. **Serialise the `runtimeEvents.js` edits**; parallelise the rest.
- Within Phase 7: **7.1** (`codingAgentAdapter`) is independent of everything. **7.2–7.7** follow the phase gate.

### 3.4 Dependencies on Part 1

Revised against Part 1's answers and R§4. Rows marked **corrected** changed since revision 1.

| ID | What Part 1 must provide | Consumed by |
|---|---|---|
| `DEP-P1-1` | **corrected.** `releaseSession(sessionId)` in **`backend/common/runtimeClient.js`** (not `runtimeDispatch.js` — packaging, §0.2), idempotent, with a retry-once wrapper whose final failure **throws** rather than being swallowed. **The runtime returns `200 {ok: true}`, not `204`** (`sessions.ts:438`), so CC branches on status class. The concurrency slot frees asynchronously **after** the response (`worker.ts:228-232`) | 4.7, 4.7b, 6.3, 6.6 |
| `DEP-P1-2` | `getSession(sessionId)` returning the `.session` field and discarding `messages`. **A `200` does not guarantee `.session` is present** — one existing shape is `{status: 'unshared', tombstone}` (`sessions.ts:157`), which Task 4.5 must treat as "unknown", not "gone". The record uses **`command`, not `mode`**, and `createdAt`/`updatedAt` are **epoch numbers**. `?include=session` is a nice-to-have, not a blocker | 4.5, 4.6, 6.3 |
| `DEP-P1-3` | **answered.** `PlanCompletion` carries **no** `generatedAt` / `generatedBy`; CC stamps them (Task 5.1). Adopted | 5.1 |
| `DEP-P1-4` | **corrected.** `createSession(spec)` accepting `mode`, `repoUrl`, `branch`, `commitSha`, `baseBranch`, `name`, `instructions`, `model?`, `prompt`, `profile`, `correlation`. Returns **`201`**, not `202`, and **`branch` is a real ref even on a `commitSha` checkout** (§1.11a) — this document must not treat it as absent. `status: 'queued'` also comes back `201` | 4.1, 4.7b, 5.2, 6.3 |
| `DEP-P1-5` | **answered.** `correlation.stage` is exactly `'planning' \| 'engineering' \| 'qa'`, threaded through `mapSessionEvent`, removing the hardcoded `'engineering'` at `runtimeEvents.js:56`, **`:67`** and `:75`. Note R§2.2 argues the runtime should **not** validate the enum (a courier that rejects mail it cannot read is not a courier) and that the check belongs in the consumer's `validateEnvelope`. That is Part 1's call; this document only consumes the value | 4.4, 5.1, 6.5 |
| `DEP-P1-6` | **answered.** Per-stage `error` mapping uses `PLANNING_FAILED` / `ENGINEERING_FAILED` / `QA_FAILED`. **Requires Phase 4b merged first**; do not stub the constant | 4b gate |
| `DEP-P1-7` | **answered.** `writeStatus` gains an **optional** `expectedFrom`; supplied, it adds a `ConditionExpression` and absorbs `ConditionalCheckFailedException` as already-applied. Optional is load-bearing: 24 of 54 transition sites are error handlers reaching `FAILED`, and an error handler must never fail to record a failure | 4.4, 5.1, 6.5 |
| `DEP-P1-8` | **answered.** `backend/common/cycleEffects.js`. Phase 3 lands `applyRuntimeEffect`, `routeAfterEngineering`, `writeStatus` and the `addProgressLog` wrapper there; the engineering-completion sequence is Task 4.3's to add. New functions take `(client, tableName, …)` first, per Part 1 §0.2 | 4.3, 4.4, 4.7b, 6.2, 6.4, 6.5 |
| `DEP-P1-9` | **superseded.** There is no `ReviewCompletion` in the protocol to widen — `submitReview.payload` is `unknown`. The QA payload shape is a **CC-side schema** owned by Task 6.1, stated in the instructions CC composes, and shipped to the runtime as `resultSchema` | 6.1 |
| `DEP-P1-10` | **new in revision 1; GRANTED, and verified shipped in revision 8.** Part 1 Task 2.3 must make a failed `checkoutCommit` a create-time **`400`**, not a `201`. It does: the SDLC route calls `provisionWorkspace` rather than `cloneWorkspace`, which fetches `baseBranch` into `origin/<base>` and **throws on a pin failure** after destroying the tree, and the route turns that into a `400` naming the failure — *"a caller-fixable 400, not a retry: the sha does not exist on the remote or the server refuses want-by-SHA, and neither improves by trying again."* `branch` and `commitSha` are mutually exclusive and `baseBranch` is required alongside `commitSha`. **Task 6.3's independent verification is therefore belt-and-braces rather than the only guard — keep it anyway**, because it is what catches a *correct* pin of the *wrong* sha, which no create-time check can see | 6.3 |
| `DEP-P1-11` | **new.** Part 1 Task 3.2's `completion` arm must handle **`engineering` only**, mapping `plan` and `review` to an explicit unsupported-stage outcome that `sdlcEventConsumer` reports as a **batch item failure** (so it DLQs and pages). And `applyRuntimeEffect`'s `default` arm must **throw**, not `console.warn` (`index.js:164`). Both close R§1.4's silent-loss window between Phase 3 and Phases 5/6. Tasks 5.1 and 6.5 then replace the placeholder arms | 5.1, 6.5 |
| `DEP-P1-12` | **revised in revision 3; SHIPPED, with one addition, in revision 8.** Three tools, not one, **and exactly one granted per session** — `profile.tools` must name precisely one of them or the create is a `400` (`nevado-sherpa-tui apps/runtime/src/routes/sdlc.ts:337-353`). Every task below composing a `profile` must therefore name one and only one; naming all three is not a permissive default, it is a refused create. `submitPlan` and `submitReview` take `{outcome: 'success' \| 'blocked', summary: string, payload: unknown}`; `payload` reaches the `completion` event verbatim, and the event's `kind` is the tool name (`'plan'` / `'review'` / `'engineering'`) — the name survives the `useMcpTool` proxy, which is what makes it available as `kind` at all. **`payload` is object-only — not object-or-array** (Part 1's call, and correct: a top-level array makes consumer dispatch ambiguous, since there is nowhere to put `verdict` or `summary` and no key to discriminate on). The runtime must **not** cap or reshape `payload`, or a plan with 30 requirements is silently truncated | 5.1, 6.1, 6.5 |
| `DEP-P1-13` | **new in revision 3.** `CreateSessionRequest.resultSchema?: unknown` — optional, stored on the session beside `correlation`, run over `payload` by a generic bounded-subset validator before publishing; failure is a `toolError` listing failing JSON Pointer paths, so the agent retries in-turn. A **malformed schema is a `400` at create**, because it is a CC bug and must be loud immediately rather than at the end of a session. Bounds on the **schema itself** mirror the courier rule (≤ 32 KB, depth ≤ 8, ≤ 300 nodes). **Bounds on the payload travel inside the schema** as `maxLength` / `maxItems`, not as a fixed byte gate in the runtime — only CC knows how much room is left on the cycle item, and it varies per cycle (Part 1's call, and it fixes a real mismatch: the runtime's own size gate is sized against SQS's 256 KB message limit, while the binding constraint is the 400 KB DynamoDB item shared with `progressLog`, `iterations` and `activities`, so a band of payloads would otherwise be forwarded happily and then fail to persist **after the session is gone** — D§2 item 6). Phases 5 and 6 author the schemas; Part 1 builds the field and the validator | 5.1, 6.1 |
| `DEP-P1-17` | **new in revision 6.** Port granularity: does `agentDrivenOrchestrator` port as a whole handler, as the cycle path only, or module by module — and can a partial port leave a workable JS/Go boundary **inside one Lambda** (two runtimes in one zip is not a thing, so a partial port means either a second Lambda or a Go handler that shells nothing). This determines how many step A commits exist and where their seams fall. §13.0 | all of §13.2 |
| `DEP-P1-16` | **new in revision 5, and it gates everything.** Which Go shape the cycle path takes — G1 (extend the 828-line Go `sdlc-manager`, which today is preset CRUD), G2 (a new Go handler for the cycle path), or G3 (migrate `agentDrivenOrchestrator` wholesale, i.e. the migration plan's Wave 5). Part 1 owns it. **This supersedes `DEP-P1-8`:** `backend/common/cycleEffects.js` cannot be the shared home if the consumer is Go | all of §13 |
| `DEP-P1-15` | **new in revision 4; GRANTED, and verified shipped in revision 8** — more strongly than asked. `tokenManager` is **not optional** on `registerSdlcRoutes`: it throws at *registration* without one (*"an SDLC session exists to commit and push"*), so the gap cannot reach a running session, and the post-acquire failure arm releases the credential, deletes the row and then refuses, in that order. The original concern, retained because it is still true of the **human** route: it is an **optional** parameter of `registerSessionRoutes` (`nevado-sherpa-tui apps/runtime/src/routes/sessions.ts:177`), so omitting it there yields a session with **no GitHub credential at all** — and the failure surfaces only when the agent tries to push, i.e. after every expensive thing in the step has already happened. Every task in this document that asserts a successful push depends on this: Tasks 4.7b, 4.10, 6.6, 6b.1. Note also that the runtime's own `DELETE` wraps `release` in a `try/catch` that only `console.error`s (`sessions.ts:432-435`), so a release failure is invisible from the runtime side too — which is why Task 4.7 records it CC-side | 4.7, 4.7b, 4.10, 6.6, 6b.1 |
| `DEP-P1-18` | **new in revision 8, and it is a DEPENDENCY ON AN OPEN DEFECT rather than on Part 1.** Every task in this document that reaches the runtime over HTTP needs the token ingress *up*, and it does not stay up: `sdlc_ingress_mode` is a `workflow_dispatch` input persisted nowhere, so every merge to `develop` destroys the secret and the ALB rule with a green apply (**command-center #858**, observed live 2026-09-26). The mechanism itself is sound and needs nothing from Part 1 — Terraform generates and writes the bearer token with no human step, the rule is priority 6 on the existing :443 listener above a priority-8 deny — so this is purely a persistence defect in the deploy workflow. **Task 4.10 cannot be completed until it is fixed**, because a ten-cycle run spans merges. Task 4.10's precondition block has the full account | 4.10, 9.2, and every runtime-reaching task in 4–6b |
| `DEP-P1-14` | **new in revision 3.** The terminal tools are not terminal — the engine loops until `end_turn` (`4.0.1 agent-engine.ts:608-610`; `tool_use` falls through to `:613+` and the loop continues). So a model can call `submitPlan` twice with two **different** payloads, and the second envelope gets a fresh dedup id, so SQS delivers it. The handler must record that a terminal tool has fired and `toolError` a second call (*"result already submitted"*). Without it, first-write-wins is an accident of Task 3.4's `ConditionExpression` rather than a decision — and it matters more now, because `toolError`-and-retry makes several terminal-tool calls per session **normal** (D§5(e)) | 5.1, 6.5 |

---

## 4. Phase 4b — step-specific failure statuses

**Ships first. Ships as a port commit then a change commit — not as one commit.** See §1.1: the transition engine throws on a status absent from its table, so nothing may write `ENGINEERING_FAILED` until that table knows it.

**The one-commit rule is withdrawn, and the reason it existed is gone.** Revision 2 mandated a single commit because `cycleStatusInvariants.test.js:109-120` asserts every `ATTENTION_STATUSES` member appears in an affordance list — so adding the status in 4b.1 left `develop` red until 4b.4 supplied the affordances. **That coupling no longer exists:** under the Go mandate 4b.1 defines the status in Go, not in `backend/common/cycleStatuses.js`, so it never enters that JS suite's `ATTENTION_STATUSES` and the invariant never breaks.

It shipped as **two commits — port, then change** — which is §13.0 rule 2 applied correctly, and is the right shape: the Go transition engine and step graph are ported with parity tests, then `ENGINEERING_FAILED` is added to them with a test that fails first. The five sub-sections below remain a **work breakdown**; they map onto those two commits rather than onto five:

1. `refactor: port the SDLC transition engine and step graph to Go` — parity only, no new status.
2. `feat: add ENGINEERING_FAILED so a failed cycle names the step that died` — the change, test-first.

Each sub-section's own "Commit" line is a record of what that slice contributes to one of those two; it is not a third commit.

**Goal.** A cycle sitting in a failed state says which step died. `PLANNING_FAILED` and `QA_FAILED` already exist; `ENGINEERING_FAILED` is the missing third. This is an **add-and-narrow, not a rename** — `FAILED` stays load-bearing for the genuinely generic cases (A§6 Phase 4b), and the risk to design against is the half-migration where new code writes the new status while an old check still tests `FAILED`.

The three generic `FAILED` producers that must keep producing `FAILED`:

- Max iterations reached — `index.js:2107` (`approveCycle`, `request_changes`, `stage: 'max_iterations_reached'`).
- Deploy-fix exhaustion — `continueDeployIteration` (`index.js:3133+`) and `retryDeploy` (`:3365`).
- QA-stage failures inside `routeAfterEngineering` — `index.js:241` (QA invocation threw) and `:302` (`qaResult.success === false`).

And the `/continue` branch that keys on `status === FAILED && pullRequest?.number` to skip the build and route straight to QA — `index.js:778` and `:874`. Both must keep matching `FAILED` and must **also** be considered for `ENGINEERING_FAILED`; see Task 4b.5.

### Task 4b.1 — add the constant and the classification sets

**Files**
- `backend/common/cycleStatuses.js` (MOD)

**Tests first** — `backend/__tests__/cycleStatusInvariants.test.js` (MOD). This file already encodes exactly the invariants that break, so extend it rather than writing a new suite. Note its `APPROVABLE` / `CANCELLABLE` / `ITERABLE` / `RETRY_PLANNING` lists (`:26-71`) are **deliberate mirrors** of the inline lists in `index.js` (docblock `:18-25`), so the test change and the handler change in Task 4b.5 must land together or the mirror drifts.

New tests:

```
describe('engineering failures are distinguishable')
  it('exports ENGINEERING_FAILED as a lowercase status string')
      → expect(cycleStatuses.ENGINEERING_FAILED).toBe('engineering_failed')
  it('treats an engineering failure as needing human attention, like the other two step failures')
      → expect(ATTENTION_STATUSES).toContain(ENGINEERING_FAILED)
      → expect(ATTENTION_STATUSES).toContain(PLANNING_FAILED)   // regression anchor
      → expect(ATTENTION_STATUSES).toContain(QA_FAILED)
  it('does not make an engineering failure look like work in progress')
      → expect(ACTIVE_STATUSES).not.toContain(ENGINEERING_FAILED)
  it('does not poll a failed engineering step')
      → expect(POLLED_STATUSES).not.toContain(ENGINEERING_FAILED)
  it('does not make an engineering failure terminal — it is retryable')
      → expect(TERMINAL_STATUSES).not.toContain(ENGINEERING_FAILED)
  it('does not treat an engineering failure as stallable — there is nothing running to stall')
      → expect(UNWATCHED_STATUSES).not.toContain(ENGINEERING_FAILED)
      → expect(isStalled({status: ENGINEERING_FAILED, updatedAt: <2 hours ago>})).toBe(false)
```

The five existing `describe('status classification is unambiguous')` invariants (`:77-105`) then cover the overlap cases for free — they iterate the arrays, so they will fail automatically if `ENGINEERING_FAILED` lands in two of them.

**Implementation.** Mirror `PLANNING_FAILED` exactly:

- `const ENGINEERING_FAILED = 'engineering_failed';` next to `ENGINEERING` (around `:13`).
- Add to `ATTENTION_STATUSES` (`:67-80`), alongside `PLANNING_FAILED`, with a one-line comment saying engineering failures are human-retryable like the other two.
- Add to the `module.exports` constant list (`:174-196`).
- Do **not** add to `ACTIVE_STATUSES`, `POLLED_STATUSES`, `TERMINAL_STATUSES`, or `UNWATCHED_STATUSES`.

**Acceptance.** The Go tests for the new constant and its classification sets pass, and the ported transition engine's parity tests stay green. **The old warning here no longer applies:** revision 2 said *"do not push this slice alone"* because adding the status to `backend/common/cycleStatuses.js` turned `cycleStatusInvariants.test.js`'s affordance invariant red until 4b.4 landed. Defining it in Go removes that coupling — the JS suite never sees the status, so this slice is independently green and independently pushable.

**Contributes to commit 2.** `feat: add ENGINEERING_FAILED so a failed cycle names the step that died`

### Task 4b.2 — teach the transition table and the step graph about it

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/sdlcEngine.js` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/sdlcStepGraph.js` (MOD)

This task is absent from A§6 Phase 4b's touch list and A§5.3 calls both files untouched. Both are wrong — see §1.1.

**Tests first** — `backend/__tests__/sdlcEngine.test.js` (MOD, 38 KB, already the transition-table suite).

```
describe('ENGINEERING_FAILED transitions')
  it('does not throw when targeted, which an unknown status would')
      → expect(() => applyTransition({status: ENGINEERING}, ENGINEERING_FAILED)).not.toThrow()
  it('is reachable from every engineering-step status')
      → for from of [ENGINEERING, BUILDING, ITERATING]:
          expect(isLegalTransition(from, ENGINEERING_FAILED)).toBe(true)
  it('is reachable from a planning approval, because a session can fail before its first event')
      → expect(isLegalTransition(PLANNING_REVIEW, ENGINEERING_FAILED)).toBe(true)
  it('lets a human retry engineering from it')
      → expect(isLegalTransition(ENGINEERING_FAILED, ENGINEERING)).toBe(true)
      → expect(isLegalTransition(ENGINEERING_FAILED, ITERATING)).toBe(true)
  it('is cancellable, like every non-terminal status')
      → expect(isLegalTransition(ENGINEERING_FAILED, CANCELLED)).toBe(true)
  it('is not terminal')
      → expect(TERMINAL).not.toContain(ENGINEERING_FAILED)
  it('explains itself as an engineering-step status rather than a step-less outcome')
      → expect(stepForStatus(ENGINEERING_FAILED)).toBe(STEP.ENGINEERING)
      → expect(explainTransition(ENGINEERING, ENGINEERING_FAILED).reason).toMatch(/within step "engineering"/)
  it('keeps FAILED reachable from anywhere, unchanged')
      → expect(UNIVERSAL_TARGETS).toEqual([CANCELLED, FAILED])
```

That last one is the narrowing guard: `ENGINEERING_FAILED` must **not** join `UNIVERSAL_TARGETS` (`sdlcEngine.js:298`). `FAILED` is universal because 24 of 54 call sites are error handlers that must always be able to reach it (docblock `:519-531`); `ENGINEERING_FAILED` is a specific edge and should be declared as one, so an illegal write to it is reported rather than blessed.

**Implementation.**

`sdlcEngine.js`:
1. New `TRANSITIONS` key `[cycleStatuses.ENGINEERING_FAILED]: [cycleStatuses.ENGINEERING, cycleStatuses.ITERATING]` placed in the `// --- Engineering step ---` section after `[ITERATING]` (`:100-103`), with a provenance comment naming the writer (the `sdlcEventConsumer`'s per-stage `error` mapping, `DEP-P1-6`) — every entry in that table carries provenance, keep the convention.
2. Add `cycleStatuses.ENGINEERING_FAILED` to the outbound arrays of `[ENGINEERING]` (`:81-87`), `[BUILDING]` (`:91-97`), `[ITERATING]` (`:100-103`), and `[PLANNING_REVIEW]` (`:59-62`).
3. Do not touch `UNIVERSAL_TARGETS`, `TERMINAL`, or `applyTransition`.

`sdlcStepGraph.js`:
4. Add `cycleStatuses.ENGINEERING_FAILED` to `STEPS[STEP.ENGINEERING].statuses` (`:82-86`), matching how `PLANNING_FAILED` sits in `STEPS[STEP.PLANNING].statuses` (`:69`) and `QA_FAILED` in `STEPS[STEP.QA].statuses` (`:103`). Without this, `STATUS_TO_STEP` (`:212-216`) has no entry, `stepForStatus` returns `null` (`:223`), and `explainTransition` renders `step "engineering" → "null"` in every illegal-transition log line.
5. Do **not** add it to `OUTCOME_STATUSES` (`:191-196`) — that set is `COMPLETED`/`FAILED`/`REJECTED`/`CANCELLED`, deliberately step-less.

**Acceptance.** `npx jest backend/__tests__/sdlcEngine.test.js` green, and `node -e "const e=require('./backend/lambda_handlers/agentDrivenOrchestrator/sdlcEngine');e.applyTransition({status:'engineering'},'engineering_failed')"` exits 0 (it throws today).

**Commit.** `feat: declare ENGINEERING_FAILED edges in the transition table and step graph`

### Task 4b.3 — render it in the UI

**Files**
- `frontend/src/components/cycles/cycleStatusConfig.js` (MOD)
- `frontend/src/components/CycleProgress.jsx` (MOD)

**Not TDD — there is no frontend test suite** (§2.1). Enforce the contract from the backend instead.

**Test first** — `backend/__tests__/cycleStatusInvariants.test.js` (MOD), a static guard in the `frontendContractGuard.test.js` style:

```
describe('every attention status is renderable')
  it('has a STATUS_CONFIG entry for each attention status')
      → read frontend/src/components/cycles/cycleStatusConfig.js as text
      → for each ATTENTION_STATUSES value, assert the file contains `[${CONSTANT_NAME}]:`
        (the config keys by computed constant, so match the destructured name, not the string)
      → assert ENGINEERING_FAILED appears in the destructuring block at the top of the file
      // ASSERT A LABEL, NOT AN ACTION. An earlier revision of this guard required
      // "a label and a primary action for each attention status" and would have
      // FAILED ON ARRIVAL: cycleStatusConfig.js has 24 `label:` entries against 9
      // `primaryAction:` entries, and POST_DEPLOY_QA_FAILED (:164-168) has had a
      // label, colour and prompt but no primaryAction since #707. ENGINEERING_FAILED
      // deliberately follows that same pattern, so the assertion was wrong, not the
      // code. Operator affordances are enforced backend-side by
      // cycleStatusInvariants.test.js's APPROVABLE/CANCELLABLE/ITERABLE lists — a
      // status can be actionable without STATUS_CONFIG naming a primary button.
  it('maps every attention status to a pipeline stage rather than falling through to -1')
      → read frontend/src/components/CycleProgress.jsx as text
      → assert it contains `engineering_failed:`
```

Derive `CONSTANT_NAME` from the status value by upper-snaking it, so the test needs no hand-maintained list and cannot rot.

**Implementation.**

`cycleStatusConfig.js`:
1. Add `ENGINEERING_FAILED` to the destructuring block (`:3-18`) — it imports from the Vite alias `'common/cycleStatuses'`, which `frontend/vite.config.js:19` maps to `../backend/common/cycleStatuses.js`, so the constant is available the moment Task 4b.1 lands.
2. Add a `STATUS_CONFIG` entry copying the `[PLANNING_FAILED]` shape verbatim (`:88-94`):

```js
[ENGINEERING_FAILED]: {
  label: 'Engineering Failed',
  color: 'error',
  prompt: 'The engineering step failed',
  primaryAction: 'retry',
  primaryLabel: 'Retry',
},
```

`primaryAction: 'retry'` routes through `AttentionCard.handleAction` (`:19-37`), which normalises hyphens to underscores and dispatches `retry` to `onRetry`, wired in `ApplicationDetail.jsx` to `handleRetryCycle` (`:327`). **That handler branches on `cycle.status === PLANNING_FAILED` to pick `/retry-planning` vs `/retry-qa`** — neither is right for an engineering failure. See Task 4b.4.

`CycleProgress.jsx`:
3. Add `engineering_failed: 2` to `mapStatusToStageIndex`'s map (`:32-56`), next to `engineering: 2` and `iterating: 2`. Without it the status falls to `?? -1` (`:57`) and the pipeline renders as if nothing happened. Note `planning_failed: 0` is the precedent — a step failure maps to its own step's index, not to `-1`.
4. **Do not touch the activity-feed line while you are in this file.** By the time this task runs, `:75` reads through `selectActivities(cycle)` from `backend/common/cycleActivities.js` (Part 1 Task 0.5), which is length-aware and merges `progressLog` with `activities` — the naive `||` chain is broken in both directions (R§1.1, and `progressTracker.js:190` for the mirror case). This task changes the stage map and nothing else.

**Acceptance.** The new static guards pass; `npm run build:frontend` succeeds; a hand-forced cycle record with `status: 'engineering_failed'` renders an "Engineering Failed" red chip with a Retry button (screenshot, §4.x verification below).

**Commit.** `feat: render ENGINEERING_FAILED in the attention card and pipeline`

### Task 4b.4 — give it a working retry

**Files**
- `frontend/src/pages/ApplicationDetail.jsx` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)

`ATTENTION_STATUSES` carries an invariant (`cycleStatuses.js:62-66`): *"every status listed here must have at least one operator affordance… A status that asks for human action while offering none is an inescapable dead end."* `cycleStatusInvariants.test.js:109-120` enforces it. So this task is mandatory, not polish.

**Which affordance.** An engineering failure means the agent did not produce a usable commit. The right action is to re-enter engineering — which is what `retryQA` cannot do and `retryPlanning` must not do. The closest existing affordance is `continueIteration` (`index.js:3467`), reached by `POST /v1/applications/{id}/cycles/{cycleId}/iterate` and already the QA-failure loop-back.

**Decision (recorded, not invented):** `ENGINEERING_FAILED` gets `iterate` (re-enter engineering) and `cancel`. It does **not** get `approve`, because there is nothing to approve. This mirrors how `QA_FAILED` sits in `ITERABLE` and `CANCELLABLE` but reaches `PENDING_APPROVAL` only through an explicit operator over-ride, which an engineering failure has no equivalent of.

**Tests first** — `backend/__tests__/cycleStatusInvariants.test.js` (MOD):

```
describe('an engineering failure is escapable')
  it('can be cancelled')      → expect(CANCELLABLE).toContain(ENGINEERING_FAILED)
  it('can be iterated')       → expect(ITERABLE).toContain(ENGINEERING_FAILED)
  it('cannot be approved — there is nothing to approve')
                              → expect(APPROVABLE).not.toContain(ENGINEERING_FAILED)
  it('is not a planning retry')
                              → expect(RETRY_PLANNING).not.toContain(ENGINEERING_FAILED)
```

Plus a wiring guard in the `runtimeDispatch.test.js:104-131` style:

```
describe('the engineering-failed affordances reach a handler')
  it('accepts ENGINEERING_FAILED in cancelCycle')
      → read index.js as text; assert the cancellableStatuses array literal
        (:2323-2333) mentions cycleStatuses.ENGINEERING_FAILED
  it('accepts ENGINEERING_FAILED in continueIteration')
      → assert continueIteration's status gate mentions cycleStatuses.ENGINEERING_FAILED
```

**Implementation.**
1. `index.js:2323-2333` — add `cycleStatuses.ENGINEERING_FAILED` to `cancellableStatuses`, with a comment in the style of the `PLANNING_FAILED` one on `:2328`.
2. `continueIteration` (`index.js:3467`) — locate its status gate and add `ENGINEERING_FAILED` to the accepted set. Read the gate before editing; it is not on the lines A§ cites and the exact shape must be matched.
3. `cycleStatusInvariants.test.js` — mirror both into `CANCELLABLE` and `ITERABLE`.
4. `ApplicationDetail.jsx:327-341` — `handleRetryCycle` currently picks between `/retry-planning` and `/retry-qa`. Add a third branch: `cycle.status === ENGINEERING_FAILED` → `/iterate` with an empty feedback body.

   **This task must not depend on the uncommitted `cyclePath` helper.** An earlier revision said *"use the `cyclePath` helper introduced in the uncommitted diff at `:272-273`"* — that diff is **not on `origin/develop`**, so the instruction is unexecutable on a clean branch and blocked real work. Specify it here so the task stands alone. If the helper already exists, use it; otherwise add it:

   ```js
   // Cycle ids carry a literal 'CYCLE#' prefix, and the orchestrator
   // decodeURIComponent()s the path param. Left unencoded the '#' opens a URL
   // fragment, so the request is sent to /cycles/CYCLE and never reaches the route.
   const cyclePath = (cycle, suffix = '') =>
     `/v1/applications/${id}/cycles/${encodeURIComponent(cycle.cycleId)}${suffix}`;
   ```

   Then call `cyclePath(cycle, '/iterate')`. **Adding the helper does not fix the other call sites** — see the defect below; converting them is not this task's job and must not be smuggled into it.

**Two facts recorded here because they change what this task can achieve.**

**(a) 4b.4's backend half is hard-blocked on the orchestrator port, with no compliant route.** `cancellableStatuses` (`index.js:2323-2333`) and `continueIteration`'s status gate are inline `cycleStatuses.*` array literals inside `agentDrivenOrchestrator/index.js`. Adding `ENGINEERING_FAILED` to them needs **both** a forbidden edit to `backend/common/cycleStatuses.js` (read-only under the Go mandate — the definition now lives in Go, §0.15) **and** an in-place JS edit to a Lambda (forbidden under §13.0 rule 2 — an updated Lambda is ported first). There is no order of operations that satisfies both. The constraint is pinned executably in `backend/__tests__/fixtures/goOnlyStatuses.js`: **nothing may write `ENGINEERING_FAILED` until those gates are Go.**

This **narrows §3.2's existing gate rather than adding one.** That gate said "Phase 4b merged blocks Phase 3's error mapping"; the real shape is tighter and more specific — *the affordance gates must be Go before anything writes the status*, which means part of the orchestrator port precedes the rest of 4b. Frontend steps 1–4 above are unblocked and can land independently; steps 1–3 of the Implementation section (the backend lists) cannot.

**(b) Every cycle affordance in the UI is already non-functional on `develop`, and this is bigger than 4b.** `cycle.cycleId` is `` `CYCLE#${Date.now()}` `` (`index.js:1058`), and **11** cycle-endpoint interpolations in `ApplicationDetail.jsx` pass it raw — 10 using `cycle.cycleId` (`:273, :289, :305, :323, :324, :339, :353, :376, :395, :406`) and one using a local `cycleId` (`:513`). The `#` truncates the path in the browser, so the request never leaves for the intended route. **Reconciling an earlier miscount: this document previously said "eight handlers were broken."** That was counting hunks in the uncommitted diff, not call sites. The verified figure is 11 interpolations across 10 handlers.

**This is a pre-existing defect that predates and outlives Phase 4b, and it needs an owner outside this document.** It is not a 4b problem to solve and 4b must not absorb it — but it does make one of 4b's exit criteria unsatisfiable, which is recorded below.

**Acceptance.** `npx jest backend/__tests__/cycleStatusInvariants.test.js` fully green, including `offers at least one affordance for each attention status` and `has no remaining known affordance gaps`. `KNOWN_AFFORDANCE_GAPS` (`:74`) stays `[]`. **The live half of this task's acceptance is deferred** — see Phase 4b's exit criterion 3.

**Commit.** `feat: let an engineering failure be cancelled or iterated`

### Task 4b.5 — narrow nothing else; prove the generic producers still produce FAILED

**Files**
- `backend/__tests__/cycleStatusInvariants.test.js` (MOD) — tests only, no production change

The risk A§6 Phase 4b names is the half-migration. This task is the net that catches it.

**Tests first**, static source guards:

```
describe('FAILED stays load-bearing for the generic cases')
  it('still fails a cycle that exhausted its iterations')
      → assert index.js contains "stage: 'max_iterations_reached'" on a line whose
        applyTransition target is cycleStatuses.FAILED
  it('still fails a cycle whose QA stage errored')
      → assert routeAfterEngineering's two QA error paths target cycleStatuses.FAILED
        (index.js:241 and :302)
  it('still routes a FAILED cycle with a PR straight to QA on /continue')
      → assert index.js contains a guard matching
        /status === cycleStatuses\.FAILED[\s\S]{0,120}pullRequest\?\.number/
        at two places (:778, :874)
```

**Revision 2 — one test cut.** Revision 1 had a fourth assertion here, *"does not silently accept both statuses where only one is meant"*, explicitly *"loose by design"*: a regex over `index.js` hunting for `status === FAILED || status === ENGINEERING_FAILED`. R§5.2 is right to cut it — it will produce false positives on legitimate compound conditions and miss the real cases, which are `.includes()` calls on array literals rather than `===` chains. The three assertions above are load-bearing; that one was theatre. Deleted.

Also add, to `backend/__tests__/stalledCycleDetection.test.js`:

```
describe('a failed engineering step is not a stalled one')
  it('is not in UNWATCHED_STATUSES, so retry-stalled refuses it')
      → expect(UNWATCHED_STATUSES).not.toContain(ENGINEERING_FAILED)
```

**Migration.** Existing records already in `FAILED` stay there. Do not write a migration; treat pre-4b `FAILED` as "failed, step unknown". Document that in the `ENGINEERING_FAILED` constant's comment.

**Acceptance.** Full `npm run test:backend` green.

**Commit.** `fix: assert the generic FAILED producers survive the ENGINEERING_FAILED split`

### Phase 4b exit criteria

1. `npm run test:backend` green.
2. `node -e` forcing `applyTransition(cycle, 'engineering_failed')` from `engineering`, `building`, `iterating` and `planning_review` does not throw and does not log `SDLC_ILLEGAL_TRANSITION`.
3. A cycle record hand-written into `engineering_failed` (AWS console or `aws dynamodb put-item --profile testing-tooling`) renders in the attention card with the label "Engineering Failed". **Verified by** a screenshot.

   **The "working Retry that returns 200" half of this criterion is unsatisfiable and is struck.** Not because of anything 4b does: **every** cycle affordance in the UI is already non-functional on `develop`, because `cycle.cycleId` is `CYCLE#<timestamp>` and 11 interpolations in `ApplicationDetail.jsx` pass it raw, truncating the path at the `#` before the request leaves the browser (Task 4b.4 fact (b)). A Retry button for `ENGINEERING_FAILED` will be exactly as broken as Retry for `PLANNING_FAILED` is today, and no worse.

   So: **verify the button renders and dispatches the right action**, and verify the endpoint itself with `curl` against a correctly-encoded path. Defer the browser-to-200 round trip to whoever owns the encoding defect. Gating 4b on a pre-existing repo-wide bug would block a correct change behind an unrelated one.
4. The three generic producers still write `FAILED` — verified by the static guards in 4b.5, not by a live run.
5. `stepForStatus('engineering_failed') === 'engineering'`.

---

## 5. Phase 4 — engineering step cutover

**Goal (A§6 Phase 4).** One application with `agentConfig.buildMode = 'runtime'` runs its engineering step on a runtime session, end to end, producing a draft PR whose head SHA equals the session's reported `commitSha`. Every other application is untouched.

**The feature flag already exists.** `index.js:1628`:

```js
if (application?.agentConfig?.buildMode === 'runtime') {
```

`buildMode` is a free-form string on `application.agentConfig` (the other observed value is `'ssm'`, branched at `engineeringAgent/orchestratorHandler.js:378`). Absent `buildMode` keeps the SQS path. Enabling is a single DynamoDB field on one application record; **rollback is removing it** (§9).

`cycle.runtimeSessionStubbed` (`index.js:1638`) is the second half of the flag: it marks a cycle whose session id is synthetic (`stub-<uuid>`, `runtimeDispatch.js:68`). Part 1 Phase 2 replaces `createRuntimeSession`'s body; this phase must treat `runtimeSessionStubbed === true` as "do not expect events" so a half-migrated environment does not look stalled.

### What already exists on `develop` — build on it, do not duplicate

Two commits landed the seam:

- **`5eafe042` "Add runtime dispatch branch for engineering step (stubbed)" (#761)** — `runtimeDispatch.js` (76 lines: `buildRuntimeSessionRequest` `:35-50`, stub `createRuntimeSession` `:63-71`), the `buildMode === 'runtime'` branch at `index.js:1620-1652`, the `runtimeSessionId`/`runtimeSessionStubbed` write at `:1636-1641`, and `backend/__tests__/runtimeDispatch.test.js`.
- **`546408af` "Map runtime session events to cycle state (consumer half)" (#763)** — `runtimeEvents.js` (152 lines: `mapSessionEvent` `:47-101`, `buildCompletionEffect` `:113-133`), `applyRuntimeEffect` at `index.js:140-168` with its four effect arms, `exports.applyRuntimeEffect` at `:4982`, and `backend/__tests__/runtimeEvents.test.js` + `applyRuntimeEffect.test.js`.

So the mapper, the effect executor, the dispatch branch and their tests exist. **What does not exist** and is this phase's work:

- Any per-step session id (§1.4).
- Any idempotency guard on session create (A§3.2).
- The engineering-completion sequence in the consumer — `applyRuntimeEffect`'s `complete` arm (`index.js:152-164`) calls `routeAfterEngineering` and **nothing else**, so on the runtime path no PR is created, `cycle.branch`/`githubUrl`/`buildHeadSha` are never set, and three of four `sdlcType` branches never persist (A§3.3).
- Any session release (A§3.8).
- Any session-aware stall handling (A§3.9).
- `signalPmItem` on the runtime branch — the SQS path calls it at `index.js:1668-1669`; the runtime branch returns at `:1647-1651` without it, so Linear is never told work started on a runtime cycle.

### Task 4.0 — make `stuckCycleDetector` deployable and put it on the status write path

**Prerequisite for the whole phase.** See §1.5.

**Files**
- `.github/workflows/deploy-dev.yml` (MOD)
- `backend/lambda_handlers/stuckCycleDetector/index.js` (MOD)
- `backend/lambda_handlers/stuckCycleDetector/package.json` (MOD)
- `infrastructure/lambdas.tf` (MOD)

**Tests first** — `backend/__tests__/stalledCycleDetection.test.js` (MOD). Extend the existing `describe('the retry-stalled route reaches dev')` block (`:150-177`), which is already the deploy-target guard pattern in this repo:

```
describe('the stuck-cycle detector is actually deployed')
  it('is declared in Terraform')
      → expect(tf).toContain('resource "aws_lambda_function" "stuck_cycle_detector"')
      → expect(tf).toContain('resource "aws_cloudwatch_event_rule" "stuck_cycle_detector_schedule"')
  it('is in the deploy -target list, or a code change can never reach AWS')
      → expect(workflow).toContain('-target="aws_lambda_function.stuck_cycle_detector"')
      → expect(workflow).toContain('-target="aws_cloudwatch_log_group.stuck_cycle_detector_logs"')
      → expect(workflow).toContain('-target="aws_cloudwatch_event_rule.stuck_cycle_detector_schedule"')
      → expect(workflow).toContain('-target="aws_cloudwatch_event_target.stuck_cycle_detector_target"')
      → expect(workflow).toContain('-target="aws_lambda_permission.stuck_cycle_detector_eventbridge"')
  it('is packaged under the name Terraform expects')
      → read scripts/deployment/package-lambdas.sh
      → expect(sh).toContain('maybe_package "stuckCycleDetector"')
      → expect(tf).toContain('filename         = "stuckCycleDetector.zip"')

describe('the detector writes status the same way the orchestrator does')
  it('imports the shared status constants instead of hardcoding strings')
      → read stuckCycleDetector/index.js
      → expect(src).toContain("require('../../common/cycleStatuses')")
      → expect(src).not.toMatch(/':failed':\s*'failed'/)
  it('clears the polling index when it fails a polled cycle')
      → expect(src).toContain('statusUpdateFragments')   // or writeStatus, per implementation
  it('does not overwrite a status that changed since the scan')
      → expect(src).toContain('ConditionExpression')
```

**Implementation.**

1. `deploy-dev.yml` — add the five `-target=` entries to the **Core** list (`:427-586`), grouped together, 12-space indent, matching the format at `:519-523`. Place them near the other cycle-pipeline Lambdas.
2. `stuckCycleDetector/index.js` — replace the raw `UpdateCommand` at `:174-197` with a status write that: imports `cycleStatuses` and `statusUpdateFragments` from `common/pollingIndex`; sets the status via the shared fragments so `GSI4PK`/`GSI4SK` are removed on a non-polled target; and carries `ConditionExpression: 'attribute_exists(PK) AND #status = :expected'` with `:expected` bound to the status read during the scan, treating `ConditionalCheckFailedException` as "already moved on, skip". This is the same pattern `processPlanGeneration` uses (`index.js:1258-1296`). Replace the three hardcoded status literal lists (`:23-29`, `:32-38`, `:110-117`) with references to `cycleStatuses.*`, and drive the scan's `ExpressionAttributeValues` from `IN_PROGRESS_STATUSES` programmatically so adding a status is one edit rather than four.
   - `stuckCycleDetector` cannot `require` `agentDrivenOrchestrator/index.js`'s `writeStatus` (separate Lambdas, separate zips). Either duplicate the small fragment composition using `common/pollingIndex` (which is already shared for exactly this reason — see `sdlcEngine.js:26-29`), or take `DEP-P1-8`'s `backend/common/cycleEffects.js` if Part 1 has landed it. **Prefer the latter if it exists**; it removes the duplication permanently.
3. `stuckCycleDetector/package.json` — no new dependencies. Node 20 (`lambdas.tf:2273`) has global `fetch`, so Task 4.5's HTTP call needs nothing added.
4. `infrastructure/lambdas.tf:2275` — raise `timeout` from 60 to 300. Task 4.5 adds one `GET /sessions/{id}` per stale cycle; 60 seconds is too tight once the detector makes network calls, and the schedule is `rate(5 minutes)` so 300 is safe.

**Acceptance.** `npx jest backend/__tests__/stalledCycleDetection.test.js` green, and after merge to `develop` the deploy run's "Terraform Apply: Core" log shows `aws_lambda_function.stuck_cycle_detector: Modifying...`. **Not TDD for the deploy half — verified by** `aws lambda get-function-configuration --function-name command-center-stuck-cycle-detector-testing --profile testing-tooling --query 'Timeout'` returning `300`.

**Commit.** `fix: deploy stuckCycleDetector and put its status write on the shared path`

### Task 4.1 — per-step session ids on the cycle record

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/common/cycleStatuses.js` — no change
- `backend/__tests__/runtimeDispatch.test.js` (MOD)

**Why.** `runtimeSessionId` is one field and Phases 5–6 need three concurrently (§1.4). A§3.2 and A§3.9 already assume the three names.

**Tests first** — new `backend/__tests__/runtimeSessionFields.test.js`, pure-function style. Extract the field-name mapping into a small pure helper so it is testable without the orchestrator:

New module `backend/common/runtimeSessionFields.js`:

```js
// stage -> the cycle attribute holding that step's current runtime session id
function sessionFieldForStage(stage) { … }
```

```
describe('sessionFieldForStage')
  it('maps each SDLC stage to its own session-id attribute')
      → planning     -> 'planSessionId'
      → engineering  -> 'engineeringSessionId'
      → qa           -> 'qaSessionId'
  it('refuses an unknown stage rather than guessing a field name')
      → expect(() => sessionFieldForStage('deploy')).toThrow(/unknown stage/i)
  it('covers exactly the correlation stages the transport carries')
      → expect(Object.keys(STAGE_SESSION_FIELDS).sort()).toEqual(['engineering','planning','qa'])
```

That last test is the coupling guard against `DEP-P1-5`. If Part 1 adds a fourth `correlation.stage` value, this test fails, which is the desired outcome.

Then extend `runtimeDispatch.test.js`'s wiring block (`:104-131`):

```
describe('orchestrator wiring — per-step session ids')
  it('persists the engineering session id under its own attribute')
      → expect(src).toContain('engineeringSessionId')
  it('keeps writing runtimeSessionId for backward compatibility with in-flight cycles')
      → expect(src).toContain('runtimeSessionId: sessionId')
```

**Implementation.**
1. New `backend/common/runtimeSessionFields.js` exporting `STAGE_SESSION_FIELDS` and `sessionFieldForStage(stage)`. Follow `progressLogger.js`'s shape: one small module, no `process.env`, a docblock saying why the mapping is explicit rather than string-concatenated (`${stage}SessionId` would produce `qaSessionId` but also `engineeringSessionId` only by luck and `planningSessionId` ≠ `planSessionId`).
2. `index.js:1636-1641` — write **both** `engineeringSessionId: sessionId` and `runtimeSessionId: sessionId`. Keeping `runtimeSessionId` is not cosmetic: cycles already in flight when this deploys carry only that field, and Task 4.5 must still find their sessions. Add a comment saying `runtimeSessionId` is a compatibility alias for the *current* session of whatever step is running, and that the three per-step fields are authoritative.
3. Do not migrate existing records.

**Acceptance.** `npx jest backend/__tests__/runtimeSessionFields.test.js backend/__tests__/runtimeDispatch.test.js` green. A runtime cycle's record shows both `engineeringSessionId` and `runtimeSessionId` set to the same value.

**Commit.** `feat: give each SDLC step its own runtime session id on the cycle record`

### Task 4.2 — idempotency guard on session create

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)

**Why (A§3.2).** `createRuntimeSession` followed by `updateItem` (`index.js:1629-1641`) is two operations, and `approvePlan` is reachable from an SQS-triggered retry. A retry landing after the session was created but before its id was persisted starts a **second session on the same branch** — two agents, one branch, which is the collision workspace release exists to prevent.

**Revision 2 — the field change from Phase 2 to Phase 4 opens a retry window. Close it.** R§4(j): Part 1 Task 2.8 deliberately writes the guard against **`runtimeSessionId`** and defers the three per-step fields to Task 4.1. That is the right call for Phase 2, but it leaves a window. Cycles created in Phases 2–3 carry `runtimeSessionId` and **not** `engineeringSessionId`. After the Phase 4 deploy, an SQS retry on one of those passes `attribute_not_exists(engineeringSessionId)` and starts a second session on the same branch — which is the exact in-flight case Task 4.1 preserves `runtimeSessionId` for.

**The condition asserts both absent, and the read-guard checks both:**

```js
if (cycle.engineeringSessionId || cycle.runtimeSessionId) return alreadyDispatched(cycle);
// …
ConditionExpression: 'attribute_not_exists(engineeringSessionId) AND attribute_not_exists(runtimeSessionId)'
```

The window is narrow — it closes as soon as every Phase 2/3-era cycle reaches a terminal status — but it costs one clause and the failure it prevents is two agents on one branch. Keep both clauses until Phase 7 removes `runtimeSessionId`, and say so in a comment naming Task 7.7.

**Tests first** — `backend/__tests__/runtimeSessionIdempotency.test.js` (NEW), orchestrator style (`applyRuntimeEffect.test.js` exemplar: mock the SDKs, `jest.isolateModules`, capture `send` payloads, reuse `trackCommandArgs`).

```
describe('runtime session dispatch is idempotent')
  it('does not create a second session when one is already recorded for the step')
      → seed the mocked GetCommand to return a cycle with engineeringSessionId set
      → call approvePlan
      → expect the createRuntimeSession spy call count to be 0
  it('does not create a second session for a cycle dispatched before the per-step fields existed')
      → seed a cycle with runtimeSessionId set and engineeringSessionId absent
      → expect the createRuntimeSession spy call count to be 0
      // The Phase 2 -> Phase 4 window, R§4(j). Without this the guard misses
      // every cycle in flight across the Phase 4 deploy.
  it('persists the session id under a condition that BOTH attributes are still absent')
      → capture the UpdateCommand payload
      → expect(payload.ConditionExpression).toMatch(/attribute_not_exists\(engineeringSessionId\)/)
      → expect(payload.ConditionExpression).toMatch(/attribute_not_exists\(runtimeSessionId\)/)
  it('treats a lost race as already-dispatched rather than an error')
      → make send reject with { name: 'ConditionalCheckFailedException' }
      → expect approvePlan to resolve with a 200-shaped response, not throw
  it('does not swallow any other DynamoDB error')
      → make send reject with { name: 'ProvisionedThroughputExceededException' }
      → expect approvePlan to reject or return 500
```

**Implementation.** In `index.js`'s runtime branch (`:1628-1652`), before `createRuntimeSession`:

```js
const existing = cycle.engineeringSessionId || cycle.runtimeSessionId;
if (existing) {
  console.log(`[AgentOrchestrator] Engineering session ${existing} already dispatched for ${sk} — retry is a duplicate`);
  return buildResponse(200, { success: true, cycle: { cycleId, status: cycle.status, stage: cycle.stage }, runtimeSessionId: existing });
}
```

and replace the bare `updateItem` with a raw `UpdateCommand` carrying `ConditionExpression: 'attribute_not_exists(engineeringSessionId) AND attribute_not_exists(runtimeSessionId)'`, catching `ConditionalCheckFailedException` and returning the same already-dispatched response.

**Losing the race means releasing the session you just created.** Part 1 Task 2.8's second behavioural test says this and it is right: if the conditional write fails, a session exists that nobody will ever consume, holding a workspace and a concurrency slot. Call `releaseSession` on it before returning the already-dispatched response, best-effort with a warning on failure. `updateItem` (`dynamoHelpers.js:380`) takes an `options` argument that spreads into the command, so `updateItem(pk, sk, fields, { ConditionExpression: … })` also works — read `:380-423` and use whichever it actually supports.

The read-guard alone is not enough and the `ConditionExpression` alone is not enough: the guard avoids a wasted session create on the common sequential retry, the condition closes the genuine concurrent race. Both.

**Acceptance.** `npx jest backend/__tests__/runtimeSessionIdempotency.test.js` green. Invoking `approvePlan` twice in quick succession against the testing environment produces exactly one session id in the cycle record and one session in `GET /v1/sdlc/sessions` — **verified by** `aws dynamodb get-item … --query 'Item.engineeringSessionId'` matching the single session id.

**Commit.** `fix: guard runtime session dispatch against SQS retry starting a second agent`

### Task 4.3 — extract the engineering-completion sequence

**Files**
- `backend/common/cycleEffects.js` (MOD — created by Part 1 Phase 3; see `DEP-P1-8`)
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)

**Why (A§3.3).** `routeAfterEngineering` does not open a PR — it contains zero `pulls.create` calls, branches on `sdlcType`, sets a status, and **only persists on the `ai-qa` branch** (the `PutCommand` at `index.js:198-204` sits inside that branch). By the time this task runs, `routeAfterEngineering` itself lives in `backend/common/cycleEffects.js` (Part 1 Task 3.3, whose acceptance check is `grep -c "^async function routeAfterEngineering" index.js` returning `0`). The hundred lines that actually run on engineering completion are still in `processSQSCycleExecution`, ahead of the call:

| Step | Lines | What |
|---|---|---|
| 0 | **`:1839`** | `cycle.updatedAt = new Date().toISOString();` — **the first statement of the sequence** |
| 1 | `:1842-1847` | Validate `branchName` and `commitSha`; missing → `FAILED` with an explicit message naming which field |
| 2 | `:1849` | `cycle.branch = engineeringResult.branchName` — **preserve the extraction faithfully, then fix it in a separate commit. Revision 8: this line is command-center #854.** It overwrites the branch the dispatch *asked for* with the one the session *reported*, with no comparison; combined with the runtime's unlogged clone fallback (nevado-sherpa-tui #103) an agent that landed on `develop` and honestly said so sets `cycle["branch"] = "develop"`, after which the draft PR at step 5 is `develop → develop`, step 6's `buildHeadSha` poll watches `develop`'s CI, and the iteration loop appends to `develop`. **As of PR #857 the requested branch is on the record before the session starts** (`cyclerecord.WorkingBranch`, §1.13), so the comparison is local and needs no runtime fetch. Add it as its own commit with its own test — a mismatch must refuse or flag, never write through — and do not fold it into the extraction, or the extraction stops being a faithful move |
| 3 | `:1851-1853` | Progress line with `engineeringResult.changes?.files?.length` |
| 4 | `:1855` | `cycle.githubUrl = \`${repoUrl}/tree/${encodeURIComponent(branchName)}\`` |
| 5 | `:1858-1913` | Draft PR, guarded by `!cycle.pullRequest?.number` (**`:1858`**); `pulls.create` `:1863`; 422 fallback adopting an existing open PR `:1881-1901`; hard `FAILED` if still none `:1903-1908`; else-branch updating `headSha` on retry `:1910-1912` |
| 6 | `:1916-1928` | Route: `no-qa`/`manual-review` → `routeAfterEngineering`; otherwise `BUILDING` + `buildHeadSha` + `pollingStartedAt` |
| 6b | `:1930-1933` | The `else` for `engineeringResult.success === false`: `FAILED` + `cycle.error` |
| 7 | `:1936-1943` | `PutCommand` with `ConditionExpression: 'attribute_not_exists(#status) OR #status <> :cancelled'` |
| — | `:1944-1951` | `catch (condErr)` — **stays in the caller.** See below |

**Extract, do not reimplement.** The 422 fallback and both `ConditionExpression`s are hard-won retry behaviour; a fresh implementation will omit them. That is the whole reason this is a task of its own.

**Revision 2 — two corrections to the boundaries, one of them structural** (§1.9, R§4(i)).

Revision 1 gave the range as `:1841-1941` and the PR guard as `:1857`; Part 1 gave `:1839-1945` and `:1858`. **The start is `:1839` and the guard is `:1858`** — Part 1 is right on both, and `:1839` matters because `cycle.updatedAt` is the statement that makes the whole block's write meaningful.

The structural correction: **`:1948`'s `continue` is loop control**, belonging to `for (const record of event.Records)` at `:1692`. Part 1's `:1839-1945` range cuts mid-`catch` and, taken literally, moves a `continue` into a function where it is a syntax error. So:

- **Extract `:1839-1943`** — the decision logic *and* the conditional `PutCommand`, because that `ConditionExpression` must not be duplicated across two callers.
- **`completeEngineering` does not catch.** `ConditionalCheckFailedException` from the `PutCommand` propagates to the caller. This is option (i) in §1.9's table: the exception path is untouched, so the move is provably behaviour-preserving.
- **Each caller owns its own disposition.** The SQS loop keeps its existing `catch (condErr)` → `results.push({success: false, cycleId, status: 'cancelled'}); continue;` verbatim, with `continue` in the loop where it is legal. The consumer catches the same exception and returns cleanly, treating it as "the cycle was cancelled underneath us" — which is what Part 1 Task 3.3's `it('abandons the ai-qa branch when the cycle was cancelled underneath it')` already establishes as the convention.

Locate every boundary **by content, not by line number** — the file will have moved by the time this task runs — and assert the first and last moved statements by name in the test (below).

**Tests first** — `backend/__tests__/engineeringCompletion.test.js` (NEW). The extracted function takes its collaborators as parameters (§2.2), which makes it testable without the orchestrator. **Signature follows Part 1 §0.2's `Queries.*` convention — `(client, tableName, …)` first:**

```js
completeEngineering(docClient, tableName, {
  cycle, application, pk, sk, engineeringResult, task, applicationId,
}, {
  octokit, addProgressLog, applyTransition, routeAfterEngineering, extractOwnerRepo,
})  // -> void. ConditionalCheckFailedException propagates; the caller disposes of it.
```

Two boundary tests come first, because a one-line-off extraction silently drops the first or last statement and nothing else would catch it:

```
describe('completeEngineering — the extraction is complete')
  it('stamps updatedAt, which is the first statement of the moved block')
      → cycle.updatedAt is a fresh ISO string after the call
      // index.js:1839. Omitting it means every completion writes a stale updatedAt,
      // which feeds isStalled (cycleStatuses.js:157) and stuckCycleDetector.
  it('issues the conditional persist, which is the last statement of the moved block')
      → a PutCommand was sent whose ConditionExpression matches
        /attribute_not_exists\(#status\) OR #status <> :cancelled/
  it('lets a failed condition propagate, because the caller owns the disposition')
      → send rejects ConditionalCheckFailedException → the call rejects with it
      // Option (i), §1.9. The SQS loop's `continue` and the consumer's return are
      // loop/function control that cannot live inside a shared function, and
      // converting the exception to a return value would be a control-flow change
      // in a commit whose contract is "behaviour must be identical."
  it('lets any other DynamoDB error propagate too, so a transient failure retries')
      → send rejects ProvisionedThroughputExceededException → rejects
```

```
describe('completeEngineering — validation')
  it('fails the cycle when a successful session reported no branch')
      → engineeringResult {success:true, commitSha:'abc'}
      → expect(cycle.status).toBe(FAILED)
      → expect(cycle.error).toMatch(/no branchName or commitSha/)
      → expect(octokit.rest.pulls.create).not.toHaveBeenCalled()
  it('fails the cycle when a successful session reported no commit')
  it('names which field was missing in the progress line')
      → detail contains 'branchName' but not 'commitSha' when only the branch is absent

describe('completeEngineering — the record it writes')
  it('sets branch and githubUrl from the reported branch')
      → cycle.branch === 'cycle/12'
      → cycle.githubUrl === 'https://github.com/o/r/tree/cycle%2F12'   // encodeURIComponent
  it('reports the file count from changes.files, which is the shape the consumer must build')
      → engineeringResult.changes.files = [{path:'a'},{path:'b'}]
      → progress detail contains '2 files'
      → and: the same call with {filesChanged: 2} and no `changes` reports 0 files
        (this is the trap A§3.4 names — two engineeringResult shapes)

describe('completeEngineering — PR creation')
  it('opens a draft PR against the application default branch')
      → pulls.create called with {draft: true, base: 'develop', head: 'cycle/12'}
  it('does not open a second PR when the cycle already has one')
      → cycle.pullRequest = {number: 7}
      → pulls.create not called; cycle.pullRequest.headSha updated to the new sha
  it('adopts an existing open PR when GitHub 422s on a duplicate head')
      → pulls.create rejects {status: 422}; pulls.list returns [{number: 9, ...}]
      → cycle.pullRequest.number === 9; cycle.status is not FAILED
  it('fails the cycle when the 422 fallback finds no PR either')
      → pulls.list returns []
      → cycle.status === FAILED; error matches /Draft PR creation failed/
  it('fails the cycle when listing PRs also throws, rather than silently continuing')

describe('completeEngineering — routing')
  it('routes no-qa and manual-review straight to routeAfterEngineering')
      → sdlcConfig.type 'no-qa' | 'manual-review' → routeAfterEngineering called once
      → BUILDING not written
  it('holds ai-qa and ci-only at BUILDING so CI gates before QA')
      → sdlcConfig.type 'ai-qa' | 'ci-only'
      → cycle.status === BUILDING
      → cycle.buildHeadSha === engineeringResult.commitSha
      → cycle.pollingStartedAt is an ISO string
      → routeAfterEngineering not called
  it('lets cycle.skipQA override the application sdlcType to no-qa')
  it('does not route at all when PR creation already failed the cycle')

describe('completeEngineering — persistence')
  it('persists under a condition that the cycle was not cancelled')
      → PutCommand.ConditionExpression matches /#status <> :cancelled/
  it('reports a cancelled cycle as skipped rather than throwing')
```

`buildHeadSha` deserves its own emphasis: without it a runtime cycle enters `BUILDING` and `testResultPoller` has nothing to query (A§3.3), so the cycle sits in `BUILDING` until `stuckCycleDetector` notices. The test above is the only thing that catches its absence before a live run.

**Implementation.**
1. Move `index.js:1839-1943` into `backend/common/cycleEffects.js` as `completeEngineering(docClient, tableName, args, deps)`. It catches nothing (§1.9 option (i)). Part 1 Task 3.3 leaves a comment in that file naming this task as the destination; put it there, not somewhere else.
2. In `index.js`, replace those lines with a call, passing the module-level `docClient`, `TABLE_NAME`, the `addProgressLog` wrapper (`:79-81`), and `await getOctokit()`. Keep the `catch`/`continue` structure by branching on the returned string. **The behaviour must be byte-identical**; this commit must not change what the SQS path does.
3. `extractOwnerRepo` (`index.js:717-740`) is also needed by the consumer. Move it to `backend/common/cycleEffects.js` (or `backend/common/githubAppClient.js`, which already owns `getOctokit`) and re-export from `index.js` so its existing call sites do not change.
4. **No `process.env` inside `backend/common/`** (§2.2, Part 1 §0.2). `octokit` and the table name arrive as arguments.

**Acceptance.** `npx jest backend/__tests__/engineeringCompletion.test.js` green **and** the full existing suite unchanged — in particular `backend/__tests__/stepAuditEvidence.test.js`, `cycleCheckpoints.test.js` and `planGenerationWrite.test.js`, which exercise the SQS path around these lines. A deliberate diff review confirming `index.js`'s remaining call passes exactly the same arguments the inline code used.

**LEGACY GATE 2 (§9.3).** This task changes the code path every non-runtime application takes today, guarded by no flag. **Before merging anything that depends on it, run one non-runtime cycle end to end on testing** — plan → approve → engineering → draft PR → CI → QA → approve → merge — and confirm the PR is opened, `buildHeadSha` is set, and the poller picks it up. R§3.2: Phase 6 has this check as an exit criterion and Phases 3 and 4 did not, which is where it matters more.

**Commit.** `fix: extract the engineering-completion sequence so both transports share one implementation`

### Task 4.4 — wire the consumer's `complete` arm to the completion sequence

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js` (MOD)
- `backend/common/cycleEffects.js` (MOD)
- `backend/__tests__/runtimeEvents.test.js` (MOD)

**Why.** `applyRuntimeEffect`'s `complete` arm (`index.js:152-164`) calls `routeAfterEngineering` and nothing else. On the runtime path as it stands, **no PR is ever created** and three of four `sdlcType` branches never persist (A§3.3).

**Depends on** `DEP-P1-5` (stage from `correlation`), `DEP-P1-11`, and Part 1's `completion` arm in `mapSessionEvent`. If Part 1 has already added the `completion` arm, this task only changes what the `engineering` branch's effect *is*; if not, add the arm here and tell Part 1.

**Revision 2 — the effect vocabulary must fail closed, and that is a Part 1 change this task depends on.** R§1.4 found a silent-data-loss path created by the split between the two documents: `applyRuntimeEffect`'s `default` arm is `console.warn` (`index.js:164`), and `sdlcEventConsumer` does not report a batch item failure for an unrecognised effect, so **the SQS message is deleted**. Between Phase 3 and Phase 5, a `completion` carrying `kind: 'plan'` would be mapped to an effect nothing handles, warned about, and dropped; the cycle sits in `PLANNING` until `stuckCycleDetector` notices twenty minutes later. Same for `kind: 'review'` between Phase 3 and Phase 6. **Neither phase's exit criteria would detect it, because neither runs a plan or QA session.**

Two things close it, both `DEP-P1-11`:

1. **`applyRuntimeEffect`'s `default` arm throws.** Under at-least-once delivery a silently-dropped effect is the failure class this design cannot afford, and one `throw` buys a DLQ entry and an alarm. Do this regardless of the ordering question.
2. **Phase 3's `completion` arm handles `engineering` only**, mapping `plan` and `review` to an explicit unsupported-stage outcome the consumer reports as a batch item failure. Tasks 5.1 and 6.5 replace those placeholders.

If Part 1 has not landed (1) by the time this task starts, land it here — it is three lines and this task is the first one whose correctness depends on it.

**Tests first** — `backend/__tests__/runtimeEvents.test.js` (MOD), pure-function style. Follow the existing `only(...)` helper (`:13-16`).

```
describe('mapSessionEvent — engineering completion')
  it('builds the engineeringResult shape completeEngineering consumes, not the persisted projection')
      → event {type:'completion', payload:{kind:'engineering', outcome:'success',
          branch:'cycle/12', commitSha:'abc', filesModified:['a.js','b.js'],
          summary:'s', verification:'npm test passed', pushed:true,
          model:'us.anthropic...', usage:{inputTokens:1,outputTokens:2}}}
      → effect.kind === 'complete'
      → effect.engineeringResult.changes.files === [{path:'a.js'},{path:'b.js'}]
      → effect.engineeringResult.branchName === 'cycle/12'   // note: branchName, not branch
      → effect.engineeringResult.commitSha === 'abc'
      → effect.engineeringResult.success === true
  it('carries the summary and verification forward for the PR body')
      → effect.engineeringResult.summary / .verification present
  it('carries the model and usage forward for cost attribution')
      → effect.engineeringResult.model, .usage present
  it('treats a blocked outcome as an engineering failure with the stated reason')
      → payload {kind:'engineering', outcome:'blocked', blockedReason:'no test harness'}
      → effect.kind === 'status'
      → effect.to === cycleStatuses.ENGINEERING_FAILED
      → effect.error contains 'no test harness'
  it('does not reach PR creation on a blocked outcome')
      → no 'complete' effect in the returned array
  it('fails loudly when the payload kind does not match the correlation stage')
      → stage 'engineering', payload {kind:'plan'}
      → expect(() => mapSessionEvent(...)).toThrow(/kind 'plan' on stage 'engineering'/)

describe('mapSessionEvent — buildCompletionEffect remains the fallback')
  it('still builds a completion from the session record for a session that ended without the tool')
      → unchanged existing tests at :111-121 stay green
  it('prefers the tool payload over the session record when both are available')
```

The `branch` → `branchName` rename in that first test is the single most likely silent bug in this phase. `EngineeringCompletion` (A§2.6) names the field `branch`; every CC consumer reads `branchName` (`index.js:1843`, `:1849`, `:1858`, `:1873`, `runtimeEvents.js:128`). Assert it.

The stage/kind mismatch throwing is deliberate (A§2.4: *"a mismatch means CC granted the wrong tool and is worth failing loudly rather than coercing"*). A throw here becomes a DLQ message and an alarm, which is the right outcome for a contract violation.

**Implementation.**
1. `runtimeEvents.js` — add the `completion` case. Dispatch on `payload.kind`; for `kind: 'engineering'`, map to `{kind: 'complete', engineeringResult: {…}}` using `buildCompletionEffect`'s existing shape (`:124-132`) as the template, but sourced from the payload rather than the session record. Keep `buildCompletionEffect(session)` exported and keep it as the no-tool fallback — it already returns `success: false` with a usable message (`:114-122`).
2. `applyRuntimeEffect`'s `complete` arm — change the call from `routeAfterEngineering` to `completeEngineering` from Task 4.3. Pass `application` (already in `ctx`, `index.js:141`) and `task: cycle.task` (already passed, `:161`).
3. The PR body: `completeEngineering` currently uses `cycle.task || task` for both title and body (`index.js:1866-1867`). A§3.4 says the body should come from `EngineeringCompletion.summary` + `verification`. Do that here, falling back to `cycle.task` when `summary` is absent (the SQS path supplies no summary). Keep the title as `(cycle.task || task).substring(0, 70)` — unchanged, and GitHub's title limit is why.

**Acceptance.** `npx jest backend/__tests__/runtimeEvents.test.js backend/__tests__/engineeringCompletion.test.js` green. Replaying the same `completion` message twice produces one PR (`index.js:1857`'s guard, now inside `completeEngineering`) and does not move the cycle backwards (`DEP-P1-7`'s conditional status write).

**Commit.** `feat: drive PR creation and routing from the engineering completion payload`

### Task 4.5 — session-aware stall handling in `stuckCycleDetector`

**Files**
- `backend/lambda_handlers/stuckCycleDetector/index.js` (MOD)
- `infrastructure/lambdas.tf` (MOD — env vars)
- `.github/workflows/deploy-dev.yml` — already covered by Task 4.0

**Why it is a Phase 4 prerequisite and not a follow-up (A§3.9).** `TIMEOUT_THRESHOLDS.engineering` is 15 minutes (`stuckCycleDetector/index.js:25`), calibrated to a Lambda-bound agent — `testResultPoller/index.js:45` says "13min agent + buffer" verbatim. A runtime session routinely exceeds that; removing the ceiling is the point of the migration. Ship Phase 4 without this and every runtime cycle is marked `failed` at fifteen minutes while its agent keeps working, orphaned, still holding a workspace and a concurrency slot. **Phase 4 exit criterion 3 cannot pass without it.**

Note the second-order reason elapsed time stops meaning failure: A§2.5's publish cadence deliberately emits nothing on an empty 10-second window, and `addProgressLog` is what refreshes `updatedAt` (`progressLogger.js:43`). A session thinking, or running one long command, writes nothing. `cycleStatuses.js:140-142` asserts the opposite — *"a working agent keeps the record fresh; on a real cycle the largest observed gap between writes was 64 seconds"* — and that observation is about the SSM agent, not a runtime session. Update that comment in this task so the next reader is not misled.

**Tests first** — `backend/__tests__/stallSessionCheck.test.js` (NEW). `stuckCycleDetector/index.js` exports **only** `exports.handler` (`:43`); `findStuckCycles`, `isStuck`, `handleStuckCycle` and `sendSlackNotification` are module-private and untestable. So extract the decision into a pure function first and test that:

New module `backend/common/stallDecision.js`:

```js
// Given a stale cycle and the result of asking the runtime about its session,
// decide what to do. Pure: no AWS, no fetch.
function decideStallAction(cycle, sessionProbe) -> { action, status?, error?, reason }
//   sessionProbe: { ok: true, session: {status, reason?} } | { ok: false, httpStatus: number }
//   action: 'touch' | 'fail' | 'complete' | 'noop' | 'legacy'
```

```
describe('decideStallAction — legacy cycles keep today’s behaviour')
  it('fails a stale cycle that has no runtime session id at all')
      → cycle {status:'engineering'}, no session fields
      → { action: 'legacy' }
  it('treats a stubbed session as legacy, because nothing publishes events for it')
      → cycle {engineeringSessionId:'stub-…', runtimeSessionStubbed:true}
      → { action: 'legacy' }

describe('decideStallAction — an active session is not a stalled cycle')
  it('only touches updatedAt when the session is still active')
      → probe {ok:true, session:{status:'active'}}
      → { action: 'touch' }
  it('only touches updatedAt when the session is queued behind the concurrency cap')
      → probe {ok:true, session:{status:'queued'}}
      → { action: 'touch', reason: /queued/ }

describe('decideStallAction — a stopped session is a failed cycle')
  it('fails the cycle with the pause reason when the session gave up')
      → probe {ok:true, session:{status:'paused', reason:'max_turns'}}
      → { action:'fail', status: ENGINEERING_FAILED, error: /max_turns/ }
  it('names the step in the failure status')
      → stage planning -> PLANNING_FAILED; engineering -> ENGINEERING_FAILED; qa -> QA_FAILED
  it('runs the completion path when the session finished but the terminal event was lost')
      → probe {ok:true, session:{status:'completed'}}
      → { action: 'complete' }
  it('fails the cycle when the session is gone')
      → probe {ok:false, httpStatus:404}
      → { action:'fail', status: ENGINEERING_FAILED, error: /session not found/ }

describe('decideStallAction — never fail a cycle because the runtime was unreachable')
  it('does nothing on a 5xx')       → probe {ok:false, httpStatus:503} → {action:'noop'}
  it('does nothing on a timeout')   → probe {ok:false, httpStatus:0}   → {action:'noop'}
  it('does nothing on a 401 or 403, which is a CC config problem not a cycle problem')
      → probe {ok:false, httpStatus:403} → {action:'noop'}
```

That last one matters: an expired M2M secret would otherwise fail every in-flight cycle on the next sweep.

Then a wiring guard in `backend/__tests__/stalledCycleDetection.test.js`:

```
describe('the detector asks the session before failing a cycle')
  it('requires the stall decision module')
      → expect(src).toContain("require('../../common/stallDecision')")
  it('reads the runtime base URL from the environment')
      → expect(src).toContain('SDLC_RUNTIME_API_BASE')
  it('is given the runtime base URL and M2M credentials in Terraform')
      → expect(tf).toMatch(/stuck_cycle_detector[\s\S]{0,900}SDLC_RUNTIME_API_BASE/)
```

**Implementation.**
1. `backend/common/stallDecision.js` (NEW) — the pure function above. It takes the stage from `cycle.stage`? **No** — `cycle.stage` is the fine-grained field (`code_generation`, `human_approval`, `qa_execution`, …), not the coarse `correlation.stage`. Derive the coarse stage from which session-id attribute is populated for the cycle's current `status`, via `runtimeSessionFields.js` from Task 4.1. Document that, because conflating the two `stage` vocabularies is the exact confusion A§7 open question 2 warns about.
2. `stuckCycleDetector/index.js` — in `handleStuckCycle` (`:163`), branch: resolve the session id; if none or stubbed, keep today's path unchanged; otherwise `await getSession(id)` (`DEP-P1-2`) wrapped so any network failure becomes `{ok:false, httpStatus}` rather than a throw, call `decideStallAction`, and execute the verdict:
   - `touch` → one `UpdateCommand` setting only `updatedAt`, so the next sweep measures from now. Nothing else. Log at `info`.
   - `fail` → the Task 4.0 conditional status write with the decided status and error.
   - `complete` → `buildCompletionEffect(session)` (`runtimeEvents.js:113`) + `completeEngineering` from Task 4.3. **This is the one place the detector needs the completion path**, and it is why Task 4.3's extraction into `backend/common/` is a hard dependency rather than a nicety.
   - `noop` → log and return; do not count it in the Slack summary.
   - `legacy` → today's behaviour.
3. Update the docblock at `:1-11` and the `TIMEOUT_THRESHOLDS` comment at `:22` to say the numbers are **sweep intervals, not deadlines** for session-backed cycles.
4. `cycleStatuses.js:140-142` — correct the "64 seconds" claim to note it predates runtime sessions, which are deliberately quiet (A§2.5).
5. `infrastructure/lambdas.tf:2282-2289` — add `SDLC_RUNTIME_API_BASE`, `COGNITO_TOKEN_ENDPOINT`, `SDLC_M2M_CLIENT_ID`, `SDLC_M2M_CLIENT_SECRET_ARN` to the `environment.variables` block, matching whatever names Part 1 Phase 1 settled on. If Part 1 put the token fetch in a shared module, require it here rather than duplicating.
6. IAM: `secretsmanager:GetSecretValue` on the M2M secret ARN, added to `aws_iam_role_policy.lambda_policy` (already targeted at `deploy-dev.yml:428`).

**Acceptance.** `npx jest backend/__tests__/stallSessionCheck.test.js backend/__tests__/stalledCycleDetection.test.js` green. **Not TDD for the live half — verified by** letting one testing cycle run past 20 minutes and checking:

```
aws logs tail /aws/lambda/command-center-stuck-cycle-detector-testing --since 10m --profile testing-tooling
```

expecting a line naming the cycle and the session with the verdict `touch`, and `aws dynamodb get-item … --query 'Item.status.S'` still reading `engineering`.

**Commit.** `fix: ask the runtime session before failing a quiet cycle`

### Task 4.6 — `retryStalled` consults the session

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)

**Why (A§3.9).** *"Restarting a session that is still running is the worst outcome available — two agents committing to one branch."* `retryStalled` (`index.js:3931`) already refuses unless `cycleStatuses.isStalled(cycle)` (`:3952-3961`, returning 409 with the elapsed seconds) and already closes the concurrency race with `ConditionExpression: '#status = :expected AND updatedAt = :lastSeen'` (`:3989+`). What it does **not** do is ask the runtime.

**Tests first** — extend `backend/__tests__/stallSessionCheck.test.js`:

```
describe('retryStalled refuses to restart a live session')
  it('reuses the same decision function the detector uses')
      → static guard: index.js requires common/stallDecision
  it('returns 409 when the session is still active')
      → probe {ok:true, session:{status:'active'}} -> action 'touch'
      → assert the handler maps 'touch' to a 409 with a hint naming the session
  it('allows the retry when the session is gone')
      → probe {ok:false, httpStatus:404} -> action 'fail'
      → assert the handler proceeds past the gate
  it('refuses on an unreachable runtime rather than retrying blind')
      → probe {ok:false, httpStatus:503} -> action 'noop' -> 503 to the caller
```

That last one is a deliberate behaviour choice: an operator clicking Retry while the runtime is unreachable should get "try again", not a second agent.

**Implementation.** After the `isStalled` gate (`index.js:3961`) and before `getApplication`, add: if the cycle has a non-stubbed session id for its current step, probe the session and call `decideStallAction`. Map `touch` → 409 with `hint: 'The runtime session is still active; retry refused'` plus the session id, `noop` → 503, `fail`/`complete`/`legacy` → proceed. Reuse the existing 409 response shape at `:3954-3960` so the frontend's error rendering is unchanged.

**Acceptance.** `npx jest backend/__tests__/stallSessionCheck.test.js` green; `POST /retry-stalled` against a cycle with a live session returns 409 naming the session — **verified by** `curl` and the response body.

**Commit.** `fix: refuse a stalled-cycle retry while its runtime session is still active`

### Task 4.7 — supersede a session: create first, then release

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/common/cycleEffects.js` (MOD)

**Depends on** `DEP-P1-1`, Task 4.1.

**Revision 2 — the ordering is reversed, because its justification is false.** A§3.2 and revision 1 of this document both said: *"Release BEFORE create. The outgoing session holds the branch checked out, and git refuses the same branch in two worktrees."* R§1.2 shows the premise does not hold, and §1.10 verifies it: **the runtime provisions one independent shallow clone per session** — `git clone --depth 1 [-b <branch>] -- <repoUrl> <baseDir>/<sessionId>` (`nevado-sherpa-tui apps/runtime/src/agent/workspace.ts:33-47`), torn down with `fs.rm` (`:83-86`), with **zero occurrences of `worktree` anywhere in `apps/runtime/src/`**.

Git's "branch is already checked out" refusal applies to worktrees of one repository. **Two separate clones of the same remote can both have `cycle/12` checked out with no interaction whatsoever.** So there is no collision to order around, and release-first is now strictly worse:

- **Release-first can lose both sessions.** It destroys the outgoing workspace before the replacement exists. If the create then fails — a concurrency rejection, a `503`, a transient network error — the cycle holds nothing and the iteration is gone.
- **Release does not even free a slot in time to help.** The concurrency slot is released asynchronously in `startSession`'s `finally` (`worker.ts:228-232`), *after* `DELETE` has returned `200`. Ordering release first to make room for the create does not reliably make room.
- **Create-then-release is recoverable in both directions.** If the create fails, the outgoing session is untouched and the cycle can retry. If the release fails, the new session is already working, so the cycle can still make progress while the leak is surfaced.

**Therefore: create, persist the new session id, then release the outgoing one.**

**But release failure is not a warning. Revision 2 said it was, and that was wrong on a fact.** It reasoned that *"a leaked workspace is a disk problem the TTL sweep handles (A§3.8)"*. **There is no TTL sweep** — 86 of 89 `/workspaces` directories on the live instance carry a `.git`, the oldest from `2026-06-08`, and nothing prunes them (§1.10). And the leak is worse than disk: `release()` is what **revokes the GitHub push credential**. `GitHubTokenManager.release` only calls `stopRefresh()` + `deleteHostsFile()` when `activeRepos.size` reaches zero (`github-token-manager.ts:36-49`), so a skipped release leaves a **live, self-refreshing installation token** on the box — re-minted every ~50 minutes, indefinitely, for every repo any active session touches.

That is observed behaviour, not a hypothetical: `hosts.yml` was being rewritten on the testing instance with `activeSessions: 0`, and `doRefresh()` returns early on an empty `activeRepos` (`:63-64`), so the rewrite can only be a leaked acquisition — one that had been re-minting for two days.

**So the severity is: retry once, then surface it on the cycle.** Not "fail the cycle" — the replacement session is already running and killing a working cycle to report a credential leak is the wrong trade. But not a `console.warn` either, because nothing else will ever notice. Concretely:

1. Retry the release once.
2. If it still fails, write a **progress-log line at `error` level** naming the session id and the repo, and set `cycle.releaseLeak = {sessionId, repo, at}` on the record so the leak is queryable rather than buried in CloudWatch.
3. Let the cycle continue.
4. Emit the CloudWatch metric Task 4.7's acceptance names, so a leak is alarmable without reading logs.

The cycle keeps its progress and the leak stops being invisible. Revision 1's *"fail the cycle"* was justified by a branch collision that does not exist; revision 2's *"warning"* was justified by a sweep that does not exist. This is the position neither of those facts undermines.

**Tests first** — `backend/__tests__/sessionSupersede.test.js` (NEW). Extract the ordering into a helper so it is testable without the orchestrator.

New in `backend/common/cycleEffects.js`:

```js
async function supersedeSession(docClient, tableName, { cycle, pk, sk, stage, spec }, { createSession, releaseSession })
//   -> { sessionId }
```

```
describe('supersedeSession')
  it('creates the replacement before releasing the outgoing session')
      → assert the call order: createSession THEN releaseSession
      // Reversed from revision 1. R§1.2: the runtime has no worktrees, so there is
      // no branch collision, and release-first can lose both sessions.
  it('creates without releasing when the step has no current session')
  it('persists the new session id under the step’s own attribute before releasing')
      → the conditional write lands first; a release that fails afterwards cannot
        strand the cycle with an unrecorded session
  it('retries a failed release exactly once before giving up')
      → releaseSession rejects twice → called 2 times
  it('does not fail the cycle when the release ultimately fails')
      → resolves with the new sessionId; the replacement session is already running
        and killing a working cycle to report a leak is the wrong trade
  it('records the leak on the cycle so it is queryable, not just logged')
      → cycle.releaseLeak === {sessionId, repo, at}
      → and an error-level progress line names the session and the repo
      // §1.10: release is what revokes the GitHub push credential. A skipped
      // release leaves a live, self-refreshing installation token on the box —
      // observed re-minting every ~50 minutes for two days with zero active
      // sessions. There is no TTL sweep to clean up after it.
  it('treats a 404 from release as already-released')
      → releaseSession rejects {httpStatus: 404} → resolves, no warning escalation
  it('leaves the outgoing session alone when the create fails')
      → createSession rejects → releaseSession not called; the error propagates
      // The property release-first cannot provide.
  it('accepts a 200 from release, not only a 204')
      → the runtime returns 200 {ok:true} (sessions.ts:438); DEP-P1-1
```

**Implementation.**
1. `supersedeSession` in `backend/common/cycleEffects.js` as above, mapping `stage` through `runtimeSessionFields.js` (Task 4.1). It knows nothing about steps beyond the `stage` string.
2. Call it from the runtime branch in `approvePlan` (`index.js:1628-1652`), with Task 4.2's idempotency guard in front of it.
3. **Cycle-termination release** (A§3.8) — `COMPLETED`, `CANCELLED`, `REJECTED`, `PLANNING_REJECTED`. Add `releaseAllSessions(cycle, {releaseSession})` releasing whichever of the three attributes is set. Call it from `cancelCycle` (`index.js:2306`), `mergePullRequest`'s completion path (`:4757`), and `approvePlan`'s reject branch.

   **This is where the credential leak bites hardest, so it gets the same treatment as step 1–4 above:** retry once, then record `cycle.releaseLeak` and log at `error`. A terminal cycle is the last moment anything will ever look at these session ids — if the release is dropped here it is dropped for good, and the token keeps refreshing. "Best-effort" as revision 2 phrased it means "silently never" in practice.

   Do **not** call it from the failure paths — a human may want to inspect the workspace, and a `FAILED`/`ENGINEERING_FAILED` cycle is retryable. **But that is a deliberate, bounded credential exposure, so bound it:** note in the runbook that an abandoned failed cycle holds a push credential until someone cancels it, which is what makes `cancelCycle` the operator's cleanup action rather than an optional tidy-up.
4. **Use `DELETE` (release), never `cancel`, on termination — and this is not interchangeable.** §1.10 verifies that `cancelSession` (`worker.ts:235-248`) aborts the controller and sweeps pending approvals but **does not remove the parked resolver from the queue** (`:41`). When a slot frees, `:231`'s `queue.shift()?.()` resolves it and the aborted turn proceeds into `startSession`'s body. So cancelling a cycle whose session is *queued* would later start an agent on a cancelled cycle's branch. `DELETE` deletes the session record, which is what makes the resumed turn harmless. Add a test:

```
describe('releaseAllSessions')
  it('releases rather than cancels, because a cancelled queued session still runs')
      → releaseSession called for each set attribute; cancelSession never called
      // worker.ts:41,231,235-248 — the parked resolver survives cancelSession.
      // And cancelSession does NOT touch the token manager at all, so it would
      // leave the push credential live even when it does stop the agent.
  it('releases every session the cycle holds, not just the current step’s')
  it('continues after one release fails, and records each failure separately')
      → two of three releases reject → the third is still attempted
      → cycle.releaseLeak lists both failures, not just the last
  it('records a leak on a terminal cycle, which is the last chance to notice')
      → status COMPLETED, release rejects → cycle.releaseLeak written before the
        terminal status is persisted
```

**Acceptance.** `npx jest backend/__tests__/sessionSupersede.test.js` green. **Not TDD for the live half — verified by** driving one testing cycle through a QA failure into a second engineering session and confirming, in order: the second session is `active`, then `GET /v1/sdlc/sessions/{firstId}` returns `404`. Observing that order is the point — the reverse would mean the ordering did not change.

**Then verify the credential actually went away**, because the `404` only proves the session record is gone and the token lives in a different place:

```
# after the cycle reaches a terminal status, with no other session active
curl -s <runtime>/health          # expect activeSessions: 0, queuedSessions: 0
# then, via SSM on the instance:
stat -c '%y' ~/.config/gh/hosts.yml 2>&1   # expect "No such file or directory"
```

An existing `hosts.yml` with zero active sessions **is** the leak — `doRefresh()` returns early on an empty `activeRepos` (`github-token-manager.ts:63-64`), so the file can only persist if a release was skipped. This is the check that would have caught the leak already sitting on the testing box (§1.10), and it is worth running once before Phase 4 starts so a pre-existing leak is not mistaken for one this task introduced. **Clear it first** (`shutdown()` runs on restart, or delete the file) so the baseline is clean.

**Commit.** `fix: create a replacement runtime session before releasing the one it supersedes`

### Task 4.7b — `dispatchEngineeringSession`, with three callers from the start

**Files**
- `backend/common/cycleEffects.js` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/__tests__/engineeringDispatch.test.js` (NEW)

**Depends on** Tasks 4.1, 4.2, 4.7.

**Revision 2 — this task is new, and it is the fix for Phase 4's own exit criterion.** Revision 1 deferred the CI-build-failure loop to a proposed Phase 6b (OD-3) while Task 4.9 turned it into an unrecoverable `ENGINEERING_FAILED`. R§1.5 shows that makes Phase 4 exit criterion 1 — *"ten consecutive cycles reach `PENDING_APPROVAL` or `QA_TESTING` with a draft PR"* — unachievable except by luck:

- For an `ai-qa` or `ci-only` cycle the path runs through `BUILDING` and requires CI on the pushed commit to pass (A§3.3, and Task 4.3 step 6).
- **An agent's first commit failing CI is the common case, not an edge case.**
- Verified route: `testResultPoller/index.js:523` and `:642` dispatch `action: 'continueBuildIteration'`; `index.js:792-793` routes it to `continueBuildIteration` (`:2951`), whose status gate is `building`/`pr_checks_pending` (`:2958`) and which calls `invokeEngineeringAgent` at `:3056`.

So under revision 1, every cycle whose first commit failed CI died and needed manual repair, and the phase gated on ten consecutive clean runs.

**Decision (R§7.4): migrate `continueBuildIteration` in Phase 4. Split the two loops.** The deploy-failure loop (`continueDeployIteration:3223`) is post-merge, genuinely rarer, and carries the infra-versus-code classification and the three-attempt cap that should not be disturbed under schedule pressure — it goes to **Phase 6b** (§8). `approveCycle`'s `request_changes` (`:2132`) follows the deploy loop's timing.

And pull the extraction forward: revision 1 created `dispatchEngineeringSession` inside Task 6.6 for one caller. **Build it here, for three** — `approvePlan`, `continueBuildIteration`, and (in Phase 6) `continueIteration` — which is what OD-3 itself recommended: *"cheaper to shape it for three callers than to retrofit."*

**Tests first** — `backend/__tests__/engineeringDispatch.test.js` (NEW):

```
describe('dispatchEngineeringSession')
  it('checks out the cycle branch, not a commit')
      → spec.branch === cycle.branch; spec has no commitSha
      // Revision 8: cycle.branch is DERIVED and already on the record by the time
      // this runs — cyclerecord.WorkingBranch(sk) = "cycle/" + SK minus "CYCLE#",
      // set by approvedispatch.ensureWorkingBranch and recorded on the claim write
      // (§1.13). This test must NOT seed it by hand; seeding is what hid #851 for
      // a whole iteration. Derive the expected value from the fixture's SK.
  it('refuses a cycle whose branch is missing rather than substituting one')
      → DispatchEngineering's refusal, naming the cycle and the attribute.
        An SK with no cycle number derives "" — never the bare "cycle/", which
        is not a ref git accepts (§1.13)
  it('sends a branch inside the cycle/** glob the provisioned CI workflow triggers on')
      → /^cycle\/.+/ — a WIRE constraint, not a naming preference: the workflow
        the provisioner writes into every customer repo triggers only on
        push: branches: ['cycle/**'] (githubClient.js:1670-1687), so any other
        prefix produces NO workflow run at all and the cycle hangs in BUILDING
  it('correlates the session at the engineering stage')
  it('persists the id under engineeringSessionId')
  it('supersedes whatever engineering session the cycle already had')
      → createSession before releaseSession (Task 4.7's order)
  it('takes its first prompt from the caller, not from the cycle')
      → the same cycle produces different prompts for an approved plan, a build
        failure and a QA failure — the composition is the caller’s
  it('names the session within the runtime limit')
      → name.length <= 50   // SESSION_NAME_MAX_LENGTH, protocol/session.ts:165
  it('does not dispatch when the caller supplies no prompt')
      → throws rather than starting an agent with an empty first message

describe('continueBuildIteration on the runtime path')
  it('starts an engineering session instead of invoking the engineering Lambda')
      → buildMode 'runtime' → dispatchEngineeringSession called; invokeEngineeringAgent not
  it('carries the build errors into the first prompt')
      → prompt contains the failure classification and the log tail the legacy
        payload passed as `buildErrors` (index.js:793, :3056)
  it('leaves the cycle in ENGINEERING for the consumer to resolve')
  it('respects the existing status gate')
      → a cycle not in building/pr_checks_pending is skipped, as today (:2958)
  it('respects maxIterations, failing to FAILED with the existing message')
      → the generic FAILED producer Phase 4b preserves
  it('keeps the legacy synchronous path for non-runtime applications')
      → no buildMode → invokeEngineeringAgent called
  it('does not carry a conversation history')
      → ssmBuildRunner returns conversationHistory: null (:978) and a runtime
        session has no conversation to carry (A§7 open question 3)
```

**Implementation.**
1. `dispatchEngineeringSession(docClient, tableName, {cycle, application, pk, sk, prompt}, deps)` in `backend/common/cycleEffects.js`, built on `supersedeSession` with `stage: 'engineering'`.
2. Retarget `approvePlan`'s runtime branch (Task 4.2/4.7) to call it, so there is one implementation from the first commit that has two callers.
3. In `continueBuildIteration` (`index.js:2951`), branch on `buildMode === 'runtime'` before the `invokeEngineeringAgent` call at `:3056`. Compose the prompt from the same material the legacy payload carried — `buildErrors`, `cycle.expandedRequirements`, `cycle.branch` — per A§2.7's "Build/deploy failure" row. **Leave the classification logic alone**; deciding whether a failure is worth an agent is orchestration and stays in CC (A§3.7).
4. Do **not** touch `continueDeployIteration` or `approveCycle`'s `request_changes` — Phase 6b.

**Depends on `DEP-P1-15`.** This task's whole point is a session that commits and pushes a fix, and `tokenManager` is an **optional** parameter of the runtime's session-route registration — omit it and the session runs to completion with no credential, failing only at the push, after the model has done all the work. If the first run of this task ends with an authentication failure on `git push`, check that wiring before debugging anything CC-side.

**Acceptance.** `npx jest backend/__tests__/engineeringDispatch.test.js` green. **Not TDD for the live half — verified by** pushing a deliberately failing test to a runtime cycle's branch so CI fails, then confirming the cycle returns to `engineering` with a new `engineeringSessionId`, the previous session `404`s, and the new session's first message names the failing test. Then confirming the cycle reaches a draft PR with a passing build.

**Commit.** `feat: run the CI-build-failure loop as a runtime engineering session`

### Task 4.8 — behaviour parity on the runtime branch

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)

**Why.** The SQS branch of `approvePlan` calls `signalPmItem(cycle, 'engineering_started', …)` at `index.js:1668-1669`, best-effort. The runtime branch returns at `:1647-1651` without it, so a runtime cycle never tells Linear that work started. Small, silent, and it will be blamed on Linear.

**Tests first** — extend `backend/__tests__/runtimeDispatch.test.js`'s wiring block:

```
describe('orchestrator wiring — parity with the SQS branch')
  it('signals the PM item on the runtime branch, as the SQS branch does')
      → static guard: the runtime branch's source region between
        "buildMode === 'runtime'" and its closing return contains "signalPmItem"
  it('signals engineering_started, not some other action name')
```

Prefer a behavioural test if the orchestrator-style harness can reach `approvePlan` — `applyRuntimeEffect.test.js`'s setup can, with `getApplication` stubbed. Use a source guard only as the fallback.

**Implementation.** Add the `signalPmItem` call to the runtime branch before the `return`, with the same best-effort wrapping the SQS branch uses. Read `:1666-1670` and mirror it exactly.

While in there, check the rest of the SQS branch for anything else the runtime branch skips and either mirror it or comment why not. From reading `:1653-1683`: the SQS branch's `sqsMessage` construction and `SendMessageCommand` are correctly skipped; the response shape differs (`:1647-1651` vs `:1677-1684`) and the runtime one omits `message`. Add an equivalent message so the UI's snackbar says something.

**Acceptance.** `npx jest backend/__tests__/runtimeDispatch.test.js` green; a runtime cycle's Linear issue receives the "engineering started" comment — **verified by** the Linear issue's activity feed on one testing cycle.

**Commit.** `fix: signal the PM item when a runtime engineering session starts`

### Task 4.9 — make the two remaining un-migrated dispatch sites fail loudly

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)

**Revision 2 — two sites, not four, and the write paths are specified rather than left to the implementer.** Revision 1 guarded all four un-migrated `invokeEngineeringAgent` sites and told the implementer to *"match each enclosing function's existing persistence and response style — they differ"*. R§5.1 is right that four hand-replicated write paths across four functions is a day-plus of underspecified work, and R§1.5 removes half of it: Task 4.7b migrates `continueBuildIteration:3056`, and Phase 6 Task 6.6 migrates `continueIteration:3613`.

**Remaining, both deferred to Phase 6b (§8):**

| Line | Enclosing function | Why deferred |
|---|---|---|
| `index.js:2132` | `approveCycle` (`:1973`), `request_changes` branch (`:2101`) | Operator-initiated; follows the deploy loop's timing |
| `index.js:3223` | `continueDeployIteration` (`:3133`) | Post-merge, rarer, and carries the infra-versus-code classification and the three-attempt cap that must not be disturbed (A§3.7) |

Neither may run silently on a `buildMode: 'runtime'` application: both would fall through to the JSON-blob Bedrock path, producing a cycle half-executed on a session and half on the legacy path with two prompt-assembly paths live (risk R6) and no signal.

**Tests first** — `backend/__tests__/runtimeDispatchCoverage.test.js` (NEW), static guard:

```
describe('every engineering dispatch site is accounted for')
  it('finds exactly five invokeEngineeringAgent call sites')
      → count matches of /invokeEngineeringAgent\(/ in index.js, minus the declaration
      → expect(count).toBe(5)
      // :1779, :2132, :3056, :3223, :3613. If this fails, a new dispatch site
      // appeared and must be triaged rather than silently inheriting a branch.
  it('routes three of them through dispatchEngineeringSession on the runtime path')
      → :1779 (via approvePlan/completeEngineering), :3056 (Task 4.7b), :3613 (Task 6.6)
      → for each, assert /buildMode === 'runtime'/ appears within the preceding 40 lines
  it('guards the remaining two against an unmigrated runtime dispatch')
      → :2132 and :3223
      → for each, assert the guard reaches ENGINEERING_FAILED with an error naming
        the enclosing function
  it('names Phase 6b in each guard, so the guard is removable by search')
      → each guard's comment contains 'Phase 6b'
```

**Implementation.** The two guards differ only in their persistence, and both write paths already exist in the enclosing functions — **use them, do not invent a third**:

- **`approveCycle:2132`.** The enclosing `request_changes` branch already has the exact pattern at `:2107-2122`: `applyTransition(cycle, FAILED, {stage, now: timestamp})`, `cycle.error = …`, then a `PutCommand` with `ConditionExpression: '#status <> :cancelled'`. Copy that block, substituting `ENGINEERING_FAILED` and the message, and return the branch's existing response shape.
- **`continueDeployIteration:3223`.** Follow the same function's own cancellation-guarded save; read it before editing and mirror it.

Both guards:

```js
if (application?.agentConfig?.buildMode === 'runtime') {
  // Phase 6b migrates this path. Until then a runtime cycle must not fall
  // through to the JSON-blob Bedrock path — see Part 2 Task 4.9.
  await addProgressLog(pk, sk, 'error', 'Runtime build mode not supported on this path',
    '<functionName> has not been migrated to runtime sessions');
  applyTransition(cycle, cycleStatuses.ENGINEERING_FAILED, { stage: 'code_generation' });
  cycle.error = 'Runtime build mode is not yet supported from <functionName>';
  // …then the enclosing function's existing persist and response
}
```

`ENGINEERING_FAILED` is an `ATTENTION_STATUS` with `iterate` and `cancel` affordances (Task 4b.4), so an operator who hits one of these has a way out. That is the whole reason Phase 4b ships first.

**Acceptance.** `npx jest backend/__tests__/runtimeDispatchCoverage.test.js` green. On a `buildMode: 'runtime'` cycle, `PUT` with `approvalStatus: 'request_changes'` produces `engineering_failed` with an explicit message rather than a legacy Bedrock run — **verified by** `curl` plus the cycle's `progressLog`.

**Commit.** `fix: fail loudly on the two dispatch paths not yet migrated to runtime sessions`

### Task 4.10 — end-to-end verification on testing

**Not TDD.** There is no way to unit-test a live cycle on a live runtime. This task is the phase's evidence.

**Nothing in this task has been performed. One end-to-end run exists and it was LOCAL** — it proved dispatch through completion against a runtime on a laptop, and that is the entire body of evidence. **No cycle has been verified running on AWS.** Read every check below as a check to perform.

#### Preconditions — an ORDERED sequence, and revision 8 added it because its absence was the document's worst omission

Revisions 1–7 of this task opened by setting `buildMode` on an application. **That is step three of three**, and performed first it produces a cycle whose dispatch cannot reach the runtime at all. Do not start here. The three steps and the reason each must precede the next:

**(a) Apply Terraform with `sdlc_ingress_mode = "token"`.** This is a `workflow_dispatch` of `deploy-testing.yml` with the input set — a full untargeted apply, deliberately human-initiated, because one must not happen against testing's live ALB merely because someone merged.

**There is no human step inside it.** Terraform *generates* the bearer token and writes it: `random_password.sdlc_auth_token` → `aws_secretsmanager_secret.sdlc_auth_token` (`${var.project_name}/sdlc-auth-token/${var.environment}`, i.e. `command-center/sdlc-auth-token/prod`) → `aws_secretsmanager_secret_version.sdlc_auth_token` (`infrastructure/sherpa-sdlc-ingress.tf:488-546`). **No CA, no certificate, no trust store, no manual secret population, no leaf to rotate and no expiry alarm nobody automated.** The same apply adds `aws_lb_listener_rule.sherpa_sdlc_token_forward` on the **existing :443 listener at priority 6** (`:600-640`), matching `/v1/sdlc/*` **AND** an `Authorization: Bearer *` header — two conditions the ALB ANDs — above the unconditional **priority-8 deny** `aws_lb_listener_rule.sherpa_sdlc_deny_443` (`:394-398`). Read the `sdlc_auth_token_secret_arn` output for the value step (b) needs.

`mtls` and `header` remain implemented and inactive. **Never write that mTLS was infeasible or needed an unexecutable manual step.** The trust store does need a CA bundle in S3, but nothing requires a *human* to produce it: AWS Private CA self-signs a root entirely in Terraform (`aws_acmpca_certificate_authority` type `ROOT` → `aws_acmpca_certificate` under `RootCACertificate/V1` → `..._certificate_authority_certificate`, then an `aws_s3_object` for the CA cert), and the `tls` provider can play CA for nothing. **The unautomated step is ours, not mTLS's** — this file points a trust store at an S3 key nothing in it writes, and leaves `aws_secretsmanager_secret.sdlc_client_cert` as a container with no version. Both were **declined on cost**, which is a smaller and different claim than "mTLS could not be switched on". `sherpa-sdlc-ingress.tf:25-52` is the account of record; note that `runtimeclient`'s `transport.go:14-19` and `secrets.go:55-58` still carry the older false framing and must not be cited for it.

> **⚠ STEP (a) DOES NOT HOLD TODAY. Command Center issue #858 destroys it on the next merge.**
>
> `sdlc_ingress_mode` exists **only** as a `workflow_dispatch` input and is persisted **nowhere** — not a tfvars file, not a repository variable, not the Terraform default. `deploy-testing.yml:157` resolves it as `${{ inputs.sdlc_ingress_mode || 'none' }}`, and a **push-triggered run has no inputs**. So every merge to `develop` re-applies with `none`, and `random_password.sdlc_auth_token`, `aws_secretsmanager_secret.sdlc_auth_token`, its `_version`, and `aws_lb_listener_rule.sherpa_sdlc_token_forward` all go to `count = 0` and are **destroyed**.
>
> Observed live on **2026-09-26**: applied at **13:12** (secret created, priority-6 rule live, both verified against AWS); an unrelated merge at 13:38 started a push-triggered run; the downstream apply at **13:40** ran with `-var="sdlc_ingress_mode=none"` and destroyed all four. **The reverting apply reported success.** At 13:45 the runtime was redeployed with `SDLC_AUTH_TOKEN_ARN` set, read the ARN, got `ResourceNotFoundException … staging label: AWSCURRENT`, and withheld every `/v1/sdlc` route. The runtime's fail-closed behaviour is correct and worked; the defect is upstream of it.
>
> The asymmetry is the point, and it is why this is a defect rather than a trade-off: the `none` default guards against accidental *enablement* and **guarantees** accidental *teardown*. Nobody chose the second. There is no signal either — the apply that tears it down is green.
>
> The timing is worth recording because it is the same window as everything else in this section: **the very merge that closed #851 (PR #857, merged 13:38) is the class of merge that tears the ingress down.**
>
> **Consequence for this task: until #858 is fixed the ingress cannot stay on, so Task 4.10 cannot be completed — a ten-cycle run spans merges.** Do not work around it by re-dispatching the apply between cycles; that is not a run, it is ten separate runs with an outage in each gap. Fix #858 first. Its own suggested shape is to persist the intent (`${{ inputs.sdlc_ingress_mode || vars.SDLC_INGRESS_MODE || 'none' }}`), so a human sets it once and a merge neither enables nor disables it; whatever the mechanism, the requirement is that **turning the ingress on survives a merge, and turning it off is as deliberate as turning it on was.** Compounded by #856 — the concurrency group guards the dispatcher, not the apply.

**(b) Deploy the runtime artifact AND set `SDLC_AUTH_TOKEN_ARN` on its systemd unit IN THE SAME RESTART WINDOW.** These cannot be two restarts, and the reason is not tidiness. `user_data_replace_on_change = false` and the cloud-init heredoc that writes the systemd unit **runs on first boot only** (`infrastructure/sherpa-ec2.tf:409-437`), so adding the Terraform variable reaches the *next* instance and no existing one. Getting it onto the live box is a manual step: rewrite the unit's `Environment` block over SSM, `daemon-reload`, restart.

So: the ARN is inert without a build that carries the bearer check, and the build is unauthenticated without the ARN. **An unset variable removes the authorization decision entirely** — in token mode the ALB filters on the *shape* of the `Authorization` header and cannot judge its value, so the instance's own check is the only thing deciding. One window, both halves. `SDLC_EVENTS_QUEUE_URL` sits in exactly the same position and belongs in the same window, since the publisher is inert without it.

**A restart kills every live session**, human included — `SIGTERM` does not drain agent sessions and the concurrency queue is in-memory — so run the all-zeros `/health` pre-flight first. Two failure modes worth distinguishing before debugging: a blank env var on the unit means this half did not reach the instance; an `AccessDenied` from SQS means the `SdlcEventPublish` statement did not reach the role.

**(c) Only now set `agentConfig.buildMode = 'runtime'`, on ONE application.**

```
aws dynamodb update-item --profile testing-tooling \
  --table-name <command-center-data-testing> \
  --key '{"PK":{"S":"APPLICATION"},"SK":{"S":"APP#<appId>"}}' \
  --update-expression 'SET agentConfig.buildMode = :m' \
  --expression-attribute-values '{":m":{"S":"runtime"}}'
```

Confirm the field landed (`aws dynamodb get-item … --query 'Item.agentConfig.M.buildMode.S'` → `"runtime"`), then start ten cycles through the UI.

**And pick a HUMAN-APPROVED application.** §1.13's last part: PM auto-approval still routes through the JavaScript gate and its stub dispatcher, so a PM-linked application would produce ten `stub-` session ids and no events, and criterion 5 below would be the only thing that noticed. Confirm the chosen application is not PM-linked, or that its cycles will be approved by a human, before starting.

**The four exit criteria (A§6 Phase 4), each with its check:**

1. **Ten consecutive cycles reach `PENDING_APPROVAL` or `QA_TESTING` with a draft PR whose head SHA equals the session's reported `commitSha`.**

   **Revision 2 — "consecutive" now means what it says, because the build loop works.** Under revision 1's sequencing this criterion was unachievable except by luck (R§1.5): an `ai-qa` or `ci-only` cycle passes through `BUILDING`, an agent's first commit failing CI is the common case, and Task 4.9 turned that into an unrecoverable `ENGINEERING_FAILED`. Task 4.7b migrates `continueBuildIteration`, so a cycle whose first commit fails CI now loops back into a fresh engineering session and can still reach the criterion. **A cycle that required one or more build iterations still counts** — the criterion is about reaching the destination, not about reaching it first try. Record the iteration count for each of the ten; if the median is above two, that is a prompt-quality signal worth acting on before Phase 5, not a Phase 4 failure.
   ```
   aws dynamodb query --profile testing-tooling --table-name <table> \
     --key-condition-expression 'PK = :pk AND begins_with(SK, :sk)' \
     --expression-attribute-values '{":pk":{"S":"APP#<appId>"},":sk":{"S":"CYCLE#"}}' \
     --query 'Items[].{s:status.S,pr:pullRequest.M.number.N,sha:pullRequest.M.headSha.S,bsha:buildHeadSha.S}'
   ```
   Expect ten rows, each `status` in `{pending_approval, qa_testing, building}`, each with a `pr` number, and `sha === bsha`. Cross-check one against GitHub: `gh pr view <n> --repo <org/repo> --json headRefOid,isDraft`.

2. **Every one shows a non-empty activity feed in the UI.** This is Part 1 Task 0.5's `selectActivities(cycle)` holding under the real write path (A§1.5, risk R0). **Verified by** a screenshot of the expanded Activity panel on each of the ten, plus `--query 'Items[].progressLog.L | length(@)'` returning > 0 for all ten. If the feed is empty while `progressLog` is populated, Part 1 Task 0.5 did not ship, or something is reading the raw field instead of the selector — stop and fix that before continuing.

   **And run the legacy half of the same check**, because R§1.1's defect blanks the *other* path: one non-runtime cycle must also show a non-empty feed, from `activities`. Both surfaces, both sources — `CycleProgress.jsx` and whatever renders `progressTracker.js`'s payload — must agree for the same cycle. This is cheap and it is the only check that catches a selector that fixed one direction and broke the other.

3. **At least one cycle runs longer than 15 minutes without being failed by `stuckCycleDetector`** — the proof for Task 4.5. **Verified by** `aws logs tail /aws/lambda/command-center-stuck-cycle-detector-testing --since 30m --profile testing-tooling` showing a `touch` verdict for that cycle and no `Marking cycle … as failed` line for it, while the cycle's `status` stays `engineering`.

4. **`cycle.buildHeadSha` is set on every `ai-qa`/`ci-only` cycle and the poller picks each one up** — the `BUILDING` hop still works. **Verified by** `aws logs tail /aws/lambda/command-center-test-result-poller-testing --since 30m --profile testing-tooling | grep -c '<cycleId>'` being non-zero for each `building` cycle.

Two extra checks worth making because they are cheap and their absence is silent:

5. `runtimeSessionStubbed` is `false` on all ten. A `true` means Part 1 Phase 2's real `createRuntimeSession` did not ship and you have been testing the stub.
5b. **No `cycle.releaseLeak` on any of the ten**, and after the tenth cycle reaches a terminal status with `/health` reporting `activeSessions: 0`, `~/.config/gh/hosts.yml` **does not exist** (Task 4.7's acceptance check). Clear any pre-existing leak before starting so the baseline is clean — there is one on the testing box today (§1.10). This is the criterion that proves ten cycles did not leave ten live push credentials behind.
5c. All ten pushed successfully, which depends on Part 1 wiring `tokenManager` into the SDLC create route (`DEP-P1-15`). An authentication failure at `git push` on cycle one is that wiring, not a CC bug.
5d. **`cycle.branch` on all ten equals `cycle/` plus the cycle number from the SK, and every one produced a CI workflow run.** New in revision 8 (§1.13). Add `b:branch.S` to criterion 1's `--query` and check it against each row's own `SK` — the branch is a pure function of the sort key, so a row where they disagree is #854 firing: the agent reported a branch the clone fell back to and `engcomplete.go:345` wrote it through unchecked. **A `branch` of `develop` on any of the ten is a stop-the-run finding**, not a cosmetic one, because the iteration loop will then append engineering work to `develop`. Cross-check the CI side too — `gh run list --repo <org/repo> --branch cycle/<n>` non-empty for each — since a branch outside `cycle/**` produces **no workflow run at all** and criterion 4 would read as a poller bug rather than a branch bug.
6. `aws sqs get-queue-attributes` on the SDLC events DLQ returns `ApproximateNumberOfMessagesVisible: 0`.

**Commit.** None — this is a verification task. Record the evidence in the PR that closes Phase 4.

### Phase 4 exit criteria

All four of A§6 Phase 4's, as measured in Task 4.10, plus:

7. `npm run test:backend` green.
8. `stuckCycleDetector`'s Terraform is in the Core `-target=` list and the deploy log shows it being applied (Task 4.0).
9. A replayed terminal message does not open a second PR and does not move the cycle backwards — **verified by** redriving one message from the DLQ **after** the 5-minute dedup window (Part 1 Phase 3 exit criterion 4; re-confirm it here against the real completion sequence, because Part 1 tested it before `completeEngineering` existed).
10. **LEGACY GATE 2 passed** (Task 4.3): one non-runtime cycle completed end to end after the completion-sequence extraction merged. §9.3 — this is the criterion that protects every application not on the flag.
11. At least one of the ten cycles looped back through `continueBuildIteration` on a real CI failure and still reached a draft PR (Task 4.7b). If none of the ten failed CI, force one by pushing a failing test — an untested loop-back is an untested loop-back.
12. `applyRuntimeEffect`'s `default` arm throws, and a deliberately malformed effect reaches the DLQ and fires the alarm rather than being warned about and dropped (`DEP-P1-11`, R§1.4).
13. Setting `buildMode` back to absent on the application returns the next cycle to the SQS path with no code change — **but read §9.3 before treating that as a rollback.** By this point five shared-code refactors are live on every application and the flag bounds none of them.

---

## 6. Phase 5 — planning as a session

**Goal (A§6 Phase 5).** A plan session produces the structured `expandedRequirements` the approval gate reads; the dual-planner divergence ends; plan revision works through a session; and the approval gate has exactly one path — a fresh `do` session created from the persisted plan, superseding the plan session.

**Revision 3 — `submitPlan` exists again, its payload is opaque, and CC's schema is enforced *inside* the session.** Two revisions moved on this; here is the settled position.

A§2.6 specified `submitPlan` as a typed tool whose `inputSchema` **is** CC's `expandedRequirements` shape, living in the runtime's protocol package, and claimed the runtime would validate against it at the tool-call boundary. **That claim is false against the deployed code:** `inputSchema` is assembled into Bedrock's `toolConfig` (`4.0.1 bedrock-client.ts:159-165`) and never read again, and there is no schema validator in either repo. So the typed design bought no enforcement, while costing an outage-gated release per field change — and the runtime's two most recently added tools (`saveLearnedPattern`, `updateTechDebt`) demonstrate that a declared schema and a hand-written handler in that package **do** silently drift, with no test noticing.

Revision 2 then over-corrected: it collapsed the tool names into one generic `submitResult` and accepted that a malformed plan costs a whole session. Collapsing the names was never necessary — the boundary objection is to the plan schema's *contents*, not to the word "plan", which the runtime already knows generically. And accepting the lost enforcement was avoidable.

**Settled (D§ verdict):**

- **`submitPlan` is a named tool taking `{outcome, summary, payload: unknown}`.** The `completion` event's `kind` is `'plan'`, so Part 1's `kind` ↔ `correlation.stage` cross-check can tell a plan from a review — which revision 2's single generic tool could not (D§5(b)).
- **The plan schema lives in Command Center, once, as data**: `PLAN_PAYLOAD_SCHEMA` in `backend/common/sdlcContract.js`. The same object is serialised into `instructions` (the prompt contract), shipped as `resultSchema` at session create (the in-session gate), and run by the consumer's validator (authoritative acceptance). **One object, three uses, no possibility of drift** — which is the property both earlier designs lacked.
- **Enforcement comes back.** The runtime runs `payload` through `resultSchema` generically; a failure is a `toolError` naming the failing paths and the agent corrects **inside the same turn** (`DEP-P1-13`). `toolError` → `status: 'error'` → appended to `messages` → the turn loop continues; verified end to end in 4.0.1.
- **The consumer's validator stays and is not redundant.** It catches a session created without a schema, the cross-field rules JSON Schema cannot express (`verdict: 'fail'` requires non-empty `findings`; `technicalApproach` must be *rejected*, not merely unrequired), CC's real DynamoDB budget, and the standing rule that CC never trusts the wire.
- **Nothing here needs an SDK release or a deploy window.** All of it rides Part 1 Task 2.1's 4.0.2 release.

**What is true today, verified.**

- Planning is enqueued through SQS: `startDevelopmentCycle` (`index.js:990`) writes the cycle at `PLANNING` (`:1068-1109`, `initialStatus` `:1064`) and a later SQS message with `action: 'generatePlan'` routes to `processPlanGeneration` (`:1170`, dispatched at `:1699-1703`).
- `processPlanGeneration` tries `agentRouter.generatePlan` (`:1215-1219`) and **falls back to `expandBusinessGoals`** on any throw (`:1220-1223`).
- The plan is persisted in **two** `UpdateCommand`s because DynamoDB rejects both single-expression alternatives — the docblock at `:1225-1248` is explicit and worth reading before touching it. Step 1 seeds `iterations` with `if_not_exists` (`:1251-1270`); step 2 writes `expandedRequirements`, `status = PLANNING_REVIEW`, `stage = 'awaiting_approval'`, `updatedAt` and `iterations[0].planningResult` atomically (`:1272-1289`). Both carry `ConditionExpression: 'attribute_exists(PK) AND #status <> :cancelled'`.
- `recordCheckpoint` fires only after the plan is durably stored (`:1291-1317`), with evidence counting `requirements.length`, `filesToCreate.length`, `filesToModify.length`, `estimatedEffort` and `generatedBy`. `backend/__tests__/stepAuditEvidence.test.js:183-188` asserts on that shape.
- `generatePlan`'s output schema (`engineeringAgent/orchestratorHandler.js:298-321`) has `summary`, **`approach`**, `requirements[]` (with `id`, `title`, `description`, `acceptanceCriteria[]`, `complexity`, `dependencies[]`, `category`, `files[]`), `filesToModify[]`, `filesToCreate[]`, `assumptions[]`, `risks[]`, `estimatedEffort` — then `generatedAt` and `generatedBy = 'nevado'` stamped at `:345-346`.
- `expandBusinessGoals`' output schema (`index.js:1474-1494`) has `summary`, `requirements[]`, `assumptions[]`, `risks[]`, `estimatedEffort` and **nothing else** — no `approach`, no `filesToCreate`, no `filesToModify`, no `files` on requirements, no `generatedBy`. A§1.4 and A§3.1 are correct about the divergence.
- `ApplicationDetail.jsx:1750,1753` read `expandedRequirements.technicalApproach`, which **nothing in the repo writes** (verified: exactly two hits, both reads). That panel has never rendered.

### Task 5.1 — validate a plan payload and persist it as `expandedRequirements`

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js` (MOD)
- `backend/common/cycleEffects.js` (MOD)
- `backend/__tests__/runtimeEvents.test.js` (MOD)
- `backend/__tests__/planCompletionWrite.test.js` (NEW)

**Depends on** `DEP-P1-3` (answered: no — CC stamps them), `DEP-P1-5`, `DEP-P1-7`, `DEP-P1-11`, `DEP-P1-12`, `DEP-P1-13`, `DEP-P1-14`.

**This task owns the plan schema, and it is one object with three consumers.** Create `backend/common/sdlcContract.js` exporting `PLAN_PAYLOAD_SCHEMA` (a JSON Schema literal, in the bounded keyword subset `DEP-P1-13` permits) and `validatePlanPayload(payload)` returning `{ok: true, plan} | {ok: false, field, reason}`. The **same** `PLAN_PAYLOAD_SCHEMA` object is:

1. serialised into the `instructions` Task 5.2 composes — so the prose contract cannot drift from the validator;
2. sent as `resultSchema` on session create — so the runtime enforces it in-session and the agent corrects in-turn;
3. run by `validatePlanPayload` in the consumer — the authoritative acceptance, plus the cross-field rules a schema cannot state.

**Required set, re-derived from the live consumers (§1.12).** Revision 2 justified this list partly by "the plan-review dialog consumes it", and D§5(a) is right that the dialog it meant is dead at `HEAD`. Corrected sources:

| Field | Why required |
|---|---|
| `summary` | `PlanReview.jsx:118` (live primary surface) and the attention-card dialog |
| `requirements[]`, **`{type: 'array'}`** | `PlanReview.jsx:19-20` — and the array constraint is load-bearing: a truthy non-array makes `tasks.reduce` throw and **the approval gate fails to render** (§1.12(4)) |
| `requirements[].{id, title, description, complexity, acceptanceCriteria, dependencies}` | `PlanReview.jsx:125-178`. **`dependencies` is new in revision 3** — the live surface renders it and no prior revision required it |
| `assumptions[]`, `risks[]` | `PlanReview.jsx:187-208` |
| `filesToCreate[]`, `filesToModify[]` | `recordCheckpoint`'s evidence counts (`index.js:1306-1313`), which Phase 5 exit criterion 2 measures |
| `estimatedEffort` | `recordCheckpoint`'s evidence (`:1314`) |
| `approach` | **Four readers, and the strongest is not a dialog at all.** `cursorAgent/devHandler.js:98` is `requirements: plan?.approach \|\| task` — so under `application.devAgent: 'cursor'` the plan's `approach` **is the dev agent's entire requirements input**, and a missing one silently degrades it to the one-line task. `devHandler.js:150` interpolates it unguarded (`**Approach:** ${plan.approach}`), so an absent value renders the literal `undefined` into the prompt. `integrationAgent/orchestratorHandler.js:112` feeds it to knowledge-pack detection. And `recordCheckpoint`'s evidence reads it (`index.js:1315`, asserted by `stepAuditEvidence.test.js:187`). The attention-card dialog is the fifth, after Task 5.3. **Not** `PlanReview.jsx`, which never reads it — §12.4 |

**The `tasks` alias is a different producer's schema, not a synonym. The schema is single-keyed on `requirements`, and CC does not normalise.** This reverses the instruction revision 3 first wrote here ("tolerate the alias on read"), which rested on an inference I did not check: that `tasks` was a legacy synonym of the same plan shape. It is not.

Verified at `cursorAgent/cursorClient.js:286-294`, `generatePlan` for the **cursor** backend returns:

```js
{ approach, tasks, filesToModify, filesToCreate, complexity, generatedAt, generatedBy: 'cursor' }
```

— no `requirements`, no `summary`, no `risks`, no `assumptions`, no `estimatedEffort`. So `PlanReview.jsx:19`'s `plan?.requirements || plan?.tasks` exists to render **cursor-produced** plans, whose whole shape differs. It is a frontend compatibility detail for a different agent backend and it is not Phase 5's business.

Three consequences:

1. **`PLAN_PAYLOAD_SCHEMA` requires `requirements` and does not mention `tasks`**, with `additionalProperties: false`. A runtime plan session emitting `tasks` is not using a synonym — it is mimicking a different backend's schema, which is a real defect worth surfacing.
2. **`resultSchema` rejects it in-session**, so the agent gets a `toolError` and corrects at the cost of one model turn. That is a better outcome than silent normalisation, which would hide the misconfiguration until someone wondered why a runtime plan looked like a cursor plan.
3. **`validatePlanPayload` does not normalise either.** A `tasks`-only payload arriving with `resultSchema` suppressed is a `PLANNING_FAILED` naming `requirements` — diagnosable, retryable, and honest about what went wrong.

Leave `PlanReview.jsx:19` alone. The alias is load-bearing for cursor and harmless for runtime plans, which will always carry `requirements`.

**Do not accept `technicalApproach`** — nothing in the repo has ever written it (§1.6), and accepting it would re-create the divergence Phase 0 closes. This is a cross-field rule JSON Schema cannot express as a rejection, so it lives in `validatePlanPayload`, not in `PLAN_PAYLOAD_SCHEMA`.

Validator tests come first and are cheap:

```
describe('validatePlanPayload')
  it('accepts a payload matching generatePlan’s schema')
      → the fixture is generatePlan's own output-format block, verbatim
        (engineeringAgent/orchestratorHandler.js:298-321)
  it('names the missing field rather than failing generically')
      → {summary, requirements} with no approach → {ok:false, field:'approach'}
      // The whole value of a CC-side schema is a diagnosable failure. A boolean
      // reject strands an operator with "plan invalid" and nothing else.
  it('rejects technicalApproach, which nothing in the repo writes')
  it('rejects a requirements array whose entries are strings, not objects')
      → the likeliest malformation from a model
  it('rejects a requirements value that is a string, not an array')
      → {requirements: 'REQ-001: add the endpoint'}
      → {ok:false, field:'requirements'}
      // §1.12(4): PlanReview.jsx:19-20 does tasks.reduce(...) on this value, so a
      // truthy non-array THROWS the approval gate before render. This is the one
      // assertion in the module protecting a live crash rather than a data shape.
  it('rejects a payload that supplies tasks instead of requirements')
      → {tasks:[…]} with no `requirements` → {ok:false, field:'requirements'}
      // `tasks` is the CURSOR backend's plan shape (cursorClient.js:286-294,
      // generatedBy:'cursor'), not a synonym. A runtime session emitting it is
      // mimicking another backend and that should surface, not be normalised
      // away. PlanReview.jsx:19's alias exists for cursor plans and stays.
  it('requires dependencies on each requirement, which the live dialog renders')
      → PlanReview.jsx:125-178. New in revision 3.
  it('accepts an empty filesToCreate but not a missing one')
      → recordCheckpoint counts .length; null and [] are different evidence
  it('does not mutate the payload it validates')
  it('rejects a payload too large for the cycle item’s remaining budget')
      → the record also carries progressLog and iterations; a 300KB plan fails the
        write with a ValidationException an operator cannot act on
  it('expresses that budget as maxLength/maxItems inside PLAN_PAYLOAD_SCHEMA')
      → PLAN_PAYLOAD_SCHEMA carries maxItems on requirements/filesToCreate/
        filesToModify/assumptions/risks and maxLength on summary/approach
      → so the SAME limit the consumer enforces is also enforced in-session
      // Not a fixed byte gate. The runtime's own size check is sized against SQS's
      // 256 KB message limit, while the binding constraint is the 400 KB DynamoDB
      // item shared with progressLog/iterations/activities — so a fixed runtime
      // gate leaves a band of payloads it forwards happily and CC cannot persist,
      // failing AFTER the session is gone as a lost completion rather than a
      // correctable toolError (D§2 item 6, DEP-P1-13).
  it('derives the budget from what the cycle record already holds, not a constant')
      → a cycle with a large progressLog yields tighter limits than a fresh one
      // This is why the bound cannot live in the runtime: it is per-cycle.
```

**Tests first.**

`runtimeEvents.test.js` (MOD), pure:

```
describe('mapSessionEvent — plan completion')
  it('maps a submitPlan completion on the planning stage to a plan-completion effect')
      → stage 'planning'
      → event {type:'completion', kind:'plan', outcome:'success', summary:'…',
          payload:{summary:'s', approach:'a', requirements:[…],
                   filesToCreate:['x'], filesToModify:['y'],
                   assumptions:[], risks:[], estimatedEffort:'2d'},
          model:'…', usage:{inputTokens:1,outputTokens:2}}
      → effect.kind === 'planComplete'
      → effect.payload.approach === 'a'        // NOT technicalApproach
  it('does not validate the payload — that is the consumer’s job')
      → a `completion` with kind:'plan' and payload:{} does NOT throw and does NOT
        produce a status effect; it produces planComplete with an empty payload
      // Part 1's mapper-purity test. The mapper decides protocol disposition; the
      // consumer decides model-failure disposition (§A.4.6). Revision 2 put
      // validation here and conflicted with this assertion.
  it('reports a protocol violation when kind does not match the stage')
      → stage 'planning' with kind:'review' → an unsupported-kind failure the
        consumer reports as a batch item failure (DLQ + alarm)
      // Meaningful only because revision 3 restored three named tools: under one
      // generic tool both planning and QA produced the same kind, so a
      // planning/QA stage mislabel was undetectable at the boundary (D§5(b)).
  it('fails the cycle when the agent reports outcome blocked')
      → {kind:'plan', outcome:'blocked', summary:'repo has no package.json'}
      → PLANNING_FAILED with the summary as the error
  it('does not carry the model or usage into expandedRequirements')
      → effect.payload has no `model` / `usage` keys; they go to effect.usage
      // Rationale: expandedRequirements is the artifact a human reviews and the
      // frontend renders; polluting it with billing fields changes an item the
      // plan-review dialog iterates.
  it('fails a planning-stage error to PLANNING_FAILED, not FAILED')
      → stage 'planning', event {type:'error', error:'boom'}
      → effect.to === cycleStatuses.PLANNING_FAILED
  it('maps planCreated to a progress activity, not to the plan itself')
      → event {type:'planCreated', filePath:'context/plans/x.md', content:'# …'}
      → effect.kind === 'activity'; the content is NOT persisted as expandedRequirements
      // A§3.1: "Do not ask the planner to write markdown and then parse it back."
```

`planCompletionWrite.test.js` (NEW) — mirror `backend/__tests__/planGenerationWrite.test.js`, which already tests the two-step write and is the exemplar:

```
describe('the consumer validates before persisting')
  it('fails the cycle to PLANNING_FAILED when the payload does not validate')
      → payload missing `approach`
      → status PLANNING_FAILED, error names the field: /approach/
      → no expandedRequirements write, no checkpoint
      // Consumer-side, not mapper-side. PLANNING_FAILED is an ATTENTION_STATUS with
      // a working retry (retryPlanning, index.js:3789). Never generic FAILED, and
      // never a throw — §A.4.6's question: a redrive re-reads the same bytes.
  it('consumes the message rather than DLQ-ing it on a validation failure')
      → the consumer reports no batch item failure for this record
  it('logs the raw payload, so a human can see what the model produced')

describe('persistPlan')
  it('seeds iterations with if_not_exists before writing into iterations[0]')
      → two UpdateCommands, first UpdateExpression matches /if_not_exists\(iterations/
  it('writes expandedRequirements, status and stage in one atomic update')
      → second UpdateExpression contains expandedRequirements, #status, stage,
        iterations[0].planningResult
      → ExpressionAttributeValues[':status'] === cycleStatuses.PLANNING_REVIEW
      → [':stage'] === 'awaiting_approval'
  it('guards both writes against a cancelled or deleted cycle')
      → both ConditionExpressions match /attribute_exists\(PK\) AND #status <> :cancelled/
  it('stops without writing PLANNING_REVIEW when the cycle was cancelled mid-plan')
      → send rejects ConditionalCheckFailedException on step 2
      → resolves without throwing; no checkpoint recorded
  it('stamps generatedBy and generatedAt, which the tool does not supply')
      → persisted plan.generatedBy === 'nevado'
      → plan.generatedAt is an ISO string
  it('records non-zero evidence counts for a plan that names files')
      → checkpoint evidence {requirementCount: 3, filesToCreate: 2, filesToModify: 1,
          estimatedEffort: '2d', generatedBy: 'nevado'}
  it('does not double-apply on a replayed completion message')
      → second call with the same payload leaves status at PLANNING_REVIEW and
        does not append a second checkpoint
  it('keeps the first plan when a session submits twice with different payloads')
      → two planComplete effects, different payloads, same session
      → expandedRequirements holds the FIRST; the second is logged and discarded
      // DEP-P1-14: the terminal tool is not terminal. The engine loops until
      // end_turn (4.0.1 agent-engine.ts:608-610), so a model CAN call submitPlan
      // twice, and the second envelope carries a fresh dedup id so SQS delivers it.
      // First-write-wins must be a decision, not a side effect of Task 3.4's
      // ConditionExpression — assert it here and let DEP-P1-14's runtime-side
      // "result already submitted" toolError make it rare (D§5(e)).
```

The `generatedBy` test is the one §1.6 exists for: `stepAuditEvidence.test.js:187` already asserts `generatedBy: 'nevado'`, so omitting the stamp breaks an existing test — good.

**Implementation.**

1. `runtimeEvents.js` — add the `kind: 'plan'` arm of the `completion` case, producing `{kind: 'planComplete', payload, usage}`. Strip `model` and `usage` out of the object destined for `expandedRequirements`.

   **The mapper does not validate the payload, and must not throw on one.** Two corrections here, both to text that survived a revision:

   - **Revision 2's step 1 still said to validate `approach`/`technicalApproach` *"throwing on violation"*.** That contradicts §A.4.6, §1.7 and §12.2 — three sections of two documents that exist to forbid exactly this — and D§5(c) flags it as a live spec defect. An implementer works from the task body and would have built the ~9-minute frozen-message-group path. **Deleted.** A malformed payload becomes a `PLANNING_FAILED` status effect naming the field; nothing throws.
   - **Validation moves out of the mapper into the consumer.** Part 1's mapper-purity test (`it('does not validate the payload — that is the consumer's job')`) is right and revision 2's placement conflicted with it. The mapper decides **protocol** disposition — is this a `kind` the stage permits? — and the consumer decides **model-failure** disposition by calling `validatePlanPayload`. Keeping the mapper pure is what makes the §A.4.6 split implementable.

2. **Cross-check `kind` against `correlation.stage` in the mapper.** `kind: 'plan'` on `stage: 'engineering'` is a protocol violation — CC granted the wrong tool — and is the one failure here that **does** belong in the DLQ, because a redrive after a CC deploy replays it correctly. This check is meaningful again only because revision 3 restored three named tools (`DEP-P1-12`, D§5(b)).
2. `backend/common/cycleEffects.js` — new `persistPlan({pk, sk, plan, usage, attempt, startedAt}, deps)`. **Move** `index.js:1250-1317` into it, unchanged in behaviour, so both the SQS path and the consumer share one implementation. Add the `generatedBy = 'nevado'` / `generatedAt` stamp at the top. Preserve the two-step write, both `ConditionExpression`s, the shared `catch (condErr)` that treats `ConditionalCheckFailedException` as "cancelled, stop", and the post-write `recordCheckpoint`. Do not collapse the two writes; the docblock at `index.js:1225-1248` explains in detail why DynamoDB rejects both alternatives.
3. `applyRuntimeEffect` — add the `planComplete` arm calling `persistPlan`, then `addProgressLog(pk, sk, 'planning', 'Plan generated — awaiting review')` to match `index.js:1319`.
4. `processPlanGeneration` — replace its inline write block with a call to `persistPlan`. This commit must not change SQS-path behaviour.

**Acceptance.** `npx jest backend/__tests__/planCompletionWrite.test.js backend/__tests__/planGenerationWrite.test.js backend/__tests__/stepAuditEvidence.test.js backend/__tests__/runtimeEvents.test.js` green.

**Commit.** `feat: validate a plan payload and persist it as the cycle's expandedRequirements`

### Task 5.2 — dispatch a plan session

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/__tests__/planSessionDispatch.test.js` (NEW)

**Depends on** `DEP-P1-4`, Task 4.1, Task 4.2.

**Tests first** — orchestrator style:

```
describe('plan dispatch on buildMode runtime')
  it('starts a plan-mode session instead of enqueueing generatePlan')
      → application.agentConfig.buildMode = 'runtime'
      → createSession called with mode: 'plan'
      → SendMessageCommand NOT sent
  it('keeps the SQS generatePlan enqueue as the default')
      → no buildMode → SendMessageCommand sent, createSession not called
  it('correlates the session to the cycle at the planning stage')
      → correlation === {applicationId, cycleId, stage: 'planning'}
  it('persists the session id under planSessionId')
  it('does not dispatch a second plan session on retry')
      → cycle.planSessionId already set → createSession not called
  it('names the session within the runtime limit')
      → name.length <= 50            // SESSION_NAME_MAX_LENGTH, protocol/session.ts:165
      → name matches /^cycle\/.* · planning$/   // A§7 open question 5
  it('leaves the cycle in PLANNING, not PLANNING_REVIEW')
      → the gate transition happens only on plan completion (Task 5.1)
  it('sends PLAN_PAYLOAD_SCHEMA as resultSchema, so the session enforces it')
      → spec.resultSchema === PLAN_PAYLOAD_SCHEMA   (identity, not a copy)
      // DEP-P1-13. Identity matters: a copy is a second artifact that can drift.
  it('embeds the same schema object in the instructions it composes')
      → the composed instructions contain the serialised PLAN_PAYLOAD_SCHEMA
      → and it is the SAME object the resultSchema field carries
      // D§5(d): revision 2's QA half shared a contract constant between prompt and
      // validator and its plan half shared nothing, so the plan schema — the one a
      // human approves — was the half where prompt and validator could diverge.
  it('still dispatches when resultSchema is omitted by config')
      → with the schema suppressed, create still succeeds and the session runs
      // resultSchema is optional so CC can disable in-session validation with no
      // runtime deploy (DEP-P1-13). Prove the degraded path works.
  it('fails the cycle to PLANNING_FAILED when session creation throws')
```

**Revision 2 — the insertion point is now verified, and there are three of them.** Revision 1 said *"read `index.js:1120-1168` and find it"*, which R§5.1 correctly calls out as the one task in this document with no verified insertion point. Found and pinned: `action: 'generatePlan'` is enqueued at **three** sites, all with the same message shape.

| Site | Enclosing function | Trigger |
|---|---|---|
| `index.js:1131-1144` | `startDevelopmentCycle` (`:990`) | Cycle creation. `planningMessage` built `:1131-1139`, `SendMessageCommand` `:1141-1144` |
| `index.js:3872-3883` | `retryPlanning` (`:3789`) | Operator retry from `PLANNING_FAILED`. Comment at `:3870-3871` says the shape is deliberately identical *"so `processPlanGeneration` handles this identically to a first attempt"* |
| `index.js:4027-4036` | `retryStalled` (`:3931`) | Stall recovery, in the `cycle.status === PLANNING` arm of a ternary (`:4027`) |

**All three must branch, or a runtime application's retry paths silently fall back to the Bedrock planner** — which is the same class of half-migration Task 4.9 exists to prevent, arriving through planning instead of engineering. Revision 1 implied one site and would have left two.

**Implementation.**
1. Extract `dispatchPlanSession(docClient, tableName, {cycle, application, pk, sk, prompt}, deps)` into `backend/common/cycleEffects.js`, built on `supersedeSession` (Task 4.7) with `stage: 'planning'`. For the first plan session there is nothing to supersede, which that function already handles.
2. Branch at all three sites on `application?.agentConfig?.buildMode === 'runtime'`, in the same shape as `approvePlan`'s branch (`:1628`). Three call sites, one implementation — the same reason Task 4.7b builds `dispatchEngineeringSession` for three callers.
3. Keep the SQS enqueue as the `else` at each site. Do not restructure the messages.

Add to the test list:

```
  it('branches all three generatePlan enqueue sites, not just cycle creation')
      → static guard: for each of the three SendMessageCommand sites carrying
        action: 'generatePlan', assert /buildMode === 'runtime'/ within the
        preceding 30 lines
  it('finds exactly three generatePlan enqueue sites')
      → count /action: 'generatePlan'/ in index.js, excluding the consumer's
        dispatch check at :1700
      → expect(count).toBe(3)
      // A fourth appearing later must be triaged, not silently inherit the branch.
```

Instructions and first message are Part 1's composition (A§2.7). This task passes them through; it does not author them. The message content is *"the business goal, attached documents, chat transcript, RAG context, and the plan output contract"* — sourced from `cycle.task`, `cycle.contextOptions` (`:1076-1080`), and `expandBusinessGoals`' existing chat-transcript and RAG loading (`index.js:1420-1449`). That loading logic survives Phase 7's delete of `expandBusinessGoals` only if it is moved out first — see Task 7.5.

Session name: `cycle/<shortId> · planning`, truncated to 50. `runtimeDispatch.js:48`'s `task.slice(0, 80)` is a live bug Part 1 Phase 0 fixes; do not reintroduce it.

**Acceptance.** `npx jest backend/__tests__/planSessionDispatch.test.js` green. A runtime cycle created through the UI reaches `PLANNING_REVIEW` with a populated `expandedRequirements` and no SQS `generatePlan` message — **verified by** `aws sqs get-queue-attributes` on the cycle queue showing no messages sent, and the cycle record's `planSessionId` being a real (non-`stub-`) id.

**Commit.** `feat: run planning as a runtime plan-mode session`

### Task 5.3 — fix the attention-card dialog to read `approach`, and guard both surfaces

**Files**
- `frontend/src/pages/ApplicationDetail.jsx` (MOD)
- `backend/__tests__/frontendContractGuard.test.js` (MOD) or a new `planFieldContract.test.js`

**This is Part 1 Phase 0's item** (A§6 Phase 0: *"Fix `ApplicationDetail.jsx:1750,1753` — read `expandedRequirements.approach`"*). Verified still unfixed at `develop` plus the uncommitted diff. If Part 1 has landed it, this task is the guard only.

**Revision 3 — the framing changes, the fix stays.** D§5(a) argues this task *"lands in code no user reaches"*, because at committed `HEAD` the dialog containing `:1750`/`:1753` is unreachable — all four `setPlanningReviewData(` sites pass `null`. **Verified true of `HEAD`, and not true of the tree that will ship:** the uncommitted `onReviewPlan={setPlanningReviewData}` at `ApplicationDetail.jsx:618`, wired to `AttentionCard.jsx:25-26`'s `review_plan` action and `cycleStatusConfig.js`'s new `primaryAction: 'review-plan'`, exists precisely to make that dialog reachable from the attention card. §1.12 has the evidence; §12.4 states the disagreement.

So the fix is worth keeping, on a corrected justification: it repairs a panel that is *about to become* reachable, rather than one that has been silently broken for months. **And the guard's scope widens** — there are two live plan-review surfaces and the previous version of this task only knew about one.

**Not TDD — no frontend test suite.** Guard from the backend.

**Test first:**

```
describe('the plan-review surfaces read fields the backend actually writes')
  it('reads approach, not technicalApproach, in the attention-card dialog')
      → read frontend/src/pages/ApplicationDetail.jsx
      → expect(src).not.toContain('technicalApproach')
      → expect(src).toMatch(/expandedRequirements\.approach/)
  it('covers the primary surface too, which reads a different field set')
      → read frontend/src/components/PlanReview.jsx
      → collect /plan\??\.(\w+)/g and /\bt\.(\w+)|req\.(\w+)/g
      → assert each is in PLAN_PAYLOAD_SCHEMA's property set, with `tasks` and
        `complexity` exempted as the CURSOR backend's plan keys
      // §1.12: PlanReview.jsx is what actually renders at PLANNING_REVIEW, via
      // ActiveCycleHero.jsx:51-62. It reads `dependencies`, which no revision
      // before 3 required, and never reads `approach`. It also reads `tasks` —
      // but that is cursorClient.js:286-294's shape, not ours, so the exemption
      // list is the right place for it, NOT PLAN_PAYLOAD_SCHEMA (Task 5.1).
      // Keep the exemption list short and commented, or it becomes a dumping
      // ground that stops the guard catching real drift.
  it('does not lose the dialog’s reachability wiring')
      → expect(src).toContain('onReviewPlan={setPlanningReviewData}')
      // Without this line the dialog is dead again and this task's fix is moot.
      // Pin it, because it arrives as uncommitted work and could be dropped in a
      // rebase without anything failing.
  it('reads only fields the plan schema defines')
      → collect /expandedRequirements\.(\w+)/g from the jsx
      → assert each is in the set generatePlan's schema produces (which is also
        what PLAN_PAYLOAD_SCHEMA in backend/common/sdlcContract.js requires,
        Task 5.1):
        summary, approach, requirements, filesToCreate, filesToModify,
        assumptions, risks, estimatedEffort, generatedBy, generatedAt
      → this is the test that would have caught technicalApproach four months ago
```

Build the allowed set by parsing `engineeringAgent/orchestratorHandler.js:298-321`'s JSON block? No — that is brittle. Hardcode the list with a comment naming its source, and add a second assertion that `orchestratorHandler.js` contains each name, so the two cannot drift silently.

**Implementation.** `ApplicationDetail.jsx:1750` and `:1753` — `technicalApproach` → `approach`. One word, twice. Keep the heading text.

**Acceptance.** The guards pass; `npm run build:frontend` succeeds. Then **both** surfaces, because they are reached differently and only one was ever checked:

- **Primary** — a runtime cycle at `PLANNING_REVIEW` that is the application's *active* cycle: `PlanReview.jsx` renders via `ActiveCycleHero.jsx:51-62`, showing summary, requirement cards with complexity chips, assumptions and risks. Screenshot.
- **Attention card** — the same cycle reached by clicking "Review Plan" on the attention card: the `ApplicationDetail.jsx` dialog opens and the Technical Approach panel is populated. Screenshot. **If the dialog does not open, the uncommitted `onReviewPlan` wiring was lost** — fix that before concluding the field rename failed.

**Commit.** `fix: read the plan's approach field the backend actually writes`

### Task 5.4 — end the dual-planner divergence

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/agentRouter.js` (MOD)
- `backend/__tests__/planFieldContract.test.js` (MOD)

**Why (A§1.4, A§3.1).** `expandBusinessGoals` is the fallback when `agentRouter.generatePlan` throws (`index.js:1220-1223`), and it emits a narrower object — so `recordCheckpoint`'s `filesToCreate`/`filesToModify` counts were *"zero or meaningful depending on which planner ran"*. Collapsing to one planner fixes that; taking the union would preserve it.

**Note the fallback is still load-bearing for non-runtime applications.** Deleting it in Phase 7 removes the safety net for every SSM/legacy application too. So this task does not delete it — it makes the divergence visible and scopes the removal to the runtime path.

**Tests first:**

```
describe('plan schema is single-sourced')
  it('asserts generatePlan emits every field the checkpoint evidence counts')
      → read engineeringAgent/orchestratorHandler.js
      → for each of ['summary','approach','requirements','filesToCreate',
          'filesToModify','assumptions','risks','estimatedEffort']
        assert the output-format block mentions it
  it('records the fallback planner’s narrower schema as a known divergence')
      → read index.js's expandBusinessGoals prompt
      → assert it does NOT mention filesToCreate / filesToModify / approach
      → and assert a comment near the fallback call site names this
      // This test documents the divergence rather than pretending it is gone; it
      // flips to an equality assertion when Phase 7 deletes the fallback.

describe('the runtime path has no fallback planner')
  it('does not fall back to expandBusinessGoals when buildMode is runtime')
      → static guard: the fallback catch block at index.js:1220-1223 is gated on
        buildMode !== 'runtime', or the runtime path never reaches processPlanGeneration
```

**Implementation.**
1. `index.js:1220-1223` — gate the fallback: when `application?.agentConfig?.buildMode === 'runtime'`, re-throw rather than falling back, so a runtime cycle fails to `PLANNING_FAILED` (an `ATTENTION_STATUS`, retryable via `retryPlanning`, `:3789`) instead of silently producing a narrower plan. Add a comment naming the divergence and pointing at Phase 7.
2. Add a comment at the fallback call site recording exactly which fields it omits, so the next reader does not have to diff two prompts.
3. `agentRouter.js` — A§5.2 asks for `runtime` as a dev/qa backend and for `generatePlan` (`:147-158`) to point at the plan session. **Defer both.** Justification: verified at `index.js:1215`, `agentRouter.generatePlan` is the *only* router use on the cycle path — `invokeEngineeringAgent` (`:2488`) and `invokeQAAgent` (`:2516`) invoke ARNs directly and do not consult the router at all (A§5.2 notes this). Adding a `runtime` backend to the router therefore changes planning only, and Task 5.2 already branches on `buildMode` before the router is reached. Registering `runtime` in `AGENT_ARNS` (`:14-27`) would be decorative. Record this as a deliberate deferral in the commit message, not a rejection. `supportsPlanning` for copilot (`:151`, `false` at `:48`) keeps rejecting, unchanged.

**Acceptance.** `npx jest backend/__tests__/planFieldContract.test.js` green. A runtime cycle whose plan session errors lands in `planning_failed` with a retry button, not in `planning_review` with a half-plan — **verified by** forcing a session failure (e.g. an unreachable `repoUrl`) and reading the cycle record.

**Commit.** `fix: stop the narrow fallback planner from serving runtime cycles`

### Task 5.5a — harvest the context loaders out of `expandBusinessGoals`

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/common/planContext.js` (NEW)
- `backend/__tests__/planContext.test.js` (NEW)

**Revision 2 — split out of revision 1's Task 5.5, which R§5.1 correctly identifies as four tasks wearing one label.** Doing this first de-risks everything after it, because it is the piece whose omission is silent.

**Why.** `expandBusinessGoals` (`index.js:1412-1534`) is not only a prompt. Before it builds one it loads, from S3:

- the **chat transcript** for `contextOptions.chatTranscriptSessionId`, flattening `chatData.history` into a `User:` / `AI:` transcript (`:1420-1449`);
- the **RAG context** via `knowledgeBaseLoader`;
- and it folds in `contextOptions.attachedDocuments` by name.

All three are inputs the plan session's first message needs (A§2.7's "Session create (`plan`)" row: *"The business goal, attached documents, chat transcript, RAG context, and the plan output contract"*). **Task 7.5 deletes `expandBusinessGoals`. If this harvest has not happened first, that delete silently drops the chat transcript and the attached documents from every plan** — with no error, no failing test, and a plan that simply knows less than it used to. That is the worst kind of regression to ship in a delete commit.

**Tests first** — `backend/__tests__/planContext.test.js` (NEW), pure except for a stubbed S3 client:

```
describe('loadPlanContext')
  it('flattens a chat transcript into the User/AI form the prompt expects')
      → history [{role:'user',content:'a'},{role:'assistant',content:'b'}]
      → 'User: a\nAI: b'     // matches index.js:1441-1444 exactly
  it('returns an empty transcript when no session id was supplied')
  it('does not fail the plan when the transcript object is missing')
      → S3 rejects NoSuchKey → resolves with an empty transcript and a warning
      // Matches the existing catch at :1446-1448. A missing transcript must never
      // fail a cycle.
  it('does not fail the plan when the transcript body is not JSON')
  it('lists attached documents by name')
  it('takes the S3 client and bucket as arguments, reading no environment')
      → source grep for process.env returns zero  // backend/common/ rule, §2.2
```

**Implementation.** Move `:1420-1449`'s loaders into `backend/common/planContext.js` as `loadPlanContext(s3Client, bucket, {task, contextOptions})` returning `{chatContext, ragContext, attachedDocuments}`. Call it from `expandBusinessGoals` (which still exists until Phase 7) **and** from Task 5.2's plan-session prompt composition. Two consumers, one implementation — which is also mitigation for risk R6.

**Acceptance.** `npx jest backend/__tests__/planContext.test.js` green; `expandBusinessGoals` still produces byte-identical prompts for a fixture cycle (compare before/after by logging the composed prompt, once, on a branch).

**Commit.** `refactor: extract the plan context loaders so a plan session can reuse them`

### Task 5.5b — plan revision as a session

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/sdlcEngine.js` (MOD)
- `backend/common/cycleEffects.js` (MOD)
- `backend/__tests__/planRevision.test.js` (NEW)

**Depends on** Task 5.5a, Task 5.1, Task 5.2.

**Why (A§5.1).** `expandBusinessGoals`' second caller is `handleRequestRevision` (`index.js:4871`, calling it at `:4903-4906`). *"Deleting the function without replacing that call breaks plan revision."* Phase 7 task 7.5 cannot ship until this does.

**The conversion is synchronous → asynchronous** (§1.8) and the architecture does not say so. Today `handleRequestRevision` returns `{success, message, plan: revisedPlan, addressedComments}` (`:4943-4948`). A plan session cannot produce `plan` in the request.

**What must move to the consumer's plan-completion arm:**
- `planComments.storePlanVersion(applicationId, cycleId, revisedPlan, devAgent, revisionContext)` (`:4914-4920`)
- `planComments.markCommentsAddressed(applicationId, cycleId, commentIds)` (`:4923-4927`)
- The `PLANNING_REVIEW` + `expandedRequirements` write (`:4930-4941`)

**What stays in the request handler:**
- `planComments.getOpenComments` (`:4874`) and the 400 on zero comments (`:4876-4880`)
- `planComments.formatCommentsForAgent` (`:4884`) — this becomes the session's first message, appended to `cycle.task`
- The cycle fetch and 404 (`:4888-4898`)

**Tests first** — `backend/__tests__/planRevision.test.js` (NEW):

```
describe('handleRequestRevision — dispatch')
  it('still refuses when there are no open comments')
      → 400, message matches /No open comments/
  it('starts a plan session carrying the task plus the formatted comments')
      → createSession called with mode 'plan'
      → prompt contains cycle.task AND the formatted comment text
  it('records what the revision is for, so the consumer can finish the job')
      → cycle gains planRevision = {commentIds:[…], revisionContext:'…', requestedAt}
  it('moves the cycle out of PLANNING_REVIEW so the UI stops offering the stale plan')
      → status === cycleStatuses.PLANNING
      → stage === 'requirements_expansion'
  it('returns 202 without a plan body, because the plan does not exist yet')
      → response.statusCode === 202
      → response body has no `plan` key
  it('releases the previous plan session before starting the revision session')
      → releaseSession called with the old planSessionId, before createSession
  it('does not start a second revision session on a double click')
      → cycle.planRevision already pending → 409

describe('persistPlan — revision mode')
  it('stores a new plan version through planComments.storePlanVersion')
      → called with (applicationId, cycleId, plan, devAgent, revisionContext)
  it('marks exactly the comments the request recorded as addressed')
      → markCommentsAddressed called with cycle.planRevision.commentIds
  it('clears planRevision so a later plan completion is not mistaken for a revision')
  it('does not touch planComments on a first-time plan')
      → cycle.planRevision absent → storePlanVersion not called
```

That last pair is the important one: `persistPlan` now has two modes and must not confuse them.

**Implementation.**
1. `handleRequestRevision` — keep the comment fetch and formatting; replace the `expandBusinessGoals` call and everything after it with: record `cycle.planRevision = {commentIds, revisionContext, requestedAt}`, `applyTransition(cycle, cycleStatuses.PLANNING, {stage: 'requirements_expansion'})`, `supersedeSession` with `stage: 'planning'`, persist under the existing `ConditionExpression: '#status <> :cancelled'`, return 202. **Keep the non-runtime path unchanged** — branch on `buildMode === 'runtime'` exactly as elsewhere, so legacy applications keep the synchronous behaviour until Phase 7.
   - `PLANNING_REVIEW → PLANNING` is not in `TRANSITIONS` (`sdlcEngine.js:59-62` allows only `ENGINEERING` and `PLANNING_REJECTED`). **Add it in this task**, with a provenance comment naming `handleRequestRevision`. Otherwise every revision logs `SDLC_ILLEGAL_TRANSITION`.
2. `persistPlan` (Task 5.1) — when `cycle.planRevision` is present, additionally call `storePlanVersion` and `markCommentsAddressed`, and `REMOVE planRevision` in the same update as the status write. Order matters: store the version *before* writing `PLANNING_REVIEW`, so a reviewer who refreshes the instant the status flips sees a version row.
3. `devAgent` for `storePlanVersion` — today `application.devAgent || 'nevado'` (`:4913`). Keep that; a runtime plan session is still the `nevado` agent as far as the plan-version audit is concerned, and `planComments.js:290,316` only stores the string.
4. Frontend: `PlanReview.jsx:43` already discards the response and calls `onRefreshCycles()` — **no frontend change needed**, verified. But the cycle now leaves `PLANNING_REVIEW`, so `ActiveCycleHero.jsx:51-61` stops rendering `PlanReview` and falls to the generic active card (`:74-144`) showing "Planning". That is the correct behaviour and it is worth a screenshot in the acceptance evidence, because it is a visible change nobody asked for.

**Acceptance.** `npx jest backend/__tests__/planRevision.test.js` green. On testing: add a plan comment, click Request Revision, observe the cycle go `planning_review → planning → planning_review` with a new plan version — **verified by** `aws dynamodb query … --expression-attribute-values '{":pk":{"S":"APP#<id>"},":sk":{"S":"<cycleId>#PLANVERSION"}}'` (read `planComments.js:290-320` for the real key shape) returning two rows.

**Acceptance addendum.** The `PLANNING_REVIEW → PLANNING` edge added to `TRANSITIONS` must not log `SDLC_ILLEGAL_TRANSITION` — check the orchestrator's logs for that string across the revision round trip. And confirm the UI change is the intended one: with the cycle out of `PLANNING_REVIEW`, `ActiveCycleHero.jsx:51-61` stops rendering `PlanReview` and falls to the generic active card. **Screenshot it.** It is a visible change nobody asked for and an operator seeing a plan vanish mid-review will file a bug unless it is documented.

**Commit.** `feat: revise a plan through a runtime plan session`

### Task 5.6 — one path across the approval gate

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/__tests__/sessionSupersede.test.js` (MOD)

**Why (A§3.2).** *"Every step entry is a new session. There is no resume path."* The engineering session starts from the **persisted plan**, not from the planner's conversation or working tree. What that buys: no resume-versus-fresh branching anywhere, no `modeSwitch` API, and the gate's unbounded wall-clock stops mattering — a multi-day approval is identical to a one-minute one.

By this point Tasks 4.2, 4.7 and 5.2 have already built the mechanism. This task is the assertion that there is exactly one path and no second one crept in.

**Tests first** — extend `sessionSupersede.test.js`:

```
describe('the approval gate has exactly one path')
  it('creates the engineering session before releasing the plan session')
      → call order: createSession(mode:'do') then releaseSession(planSessionId)
      // Reversed from revision 1 — R§1.2, Task 4.7. The plan and engineering
      // sessions want different branches anyway, so this case never needed an
      // ordering rule; it follows the one rule that holds for all cases.
  it('builds the engineering prompt from the persisted plan, not from the plan session')
      → createSession's prompt contains cycle.expandedRequirements content
      → createSession spec has no `resumeSessionId` / `sessionId` field
  it('has no resume branch at all')
      → static guard: index.js contains no /resumeSession|resume: true|modeSwitch/
  it('takes the identical path for a one-minute and a multi-day approval')
      → two runs with approvedAt differing by 3 days produce identical createSession specs
        (modulo timestamps)
  it('does not carry a consumer cursor across the gate')
      → static guard: no `seq` or cursor field is read from the cycle record
```

**Implementation.** No new mechanism — verify and delete. Concretely:
1. Confirm the runtime branch in `approvePlan` (`:1628`) goes through `supersedeSession` with `stage: 'engineering'`, releasing `cycle.planSessionId`.
2. Confirm the engineering prompt is composed from `cycle.expandedRequirements` + `cycle.approvedPlan` (A§2.7's "Plan approved" row), not from anything session-shaped. `approvePlan` already writes `cycle.approvedPlan` at `:1591-1606`.
3. Delete `runtimeDispatch.js`'s original `expandedRequirements` string check if Part 1 Phase 0 has not — `:42-44` does `typeof expandedRequirements === 'string' ? expandedRequirements : (task || '')`, and `expandedRequirements` is **always** an object (`index.js:1300-1301`'s comment says so, written after someone got it wrong), so the approved plan is silently dropped and the agent receives only the raw one-line task. If Part 1 has fixed it, add the regression test here anyway.

**Acceptance.** `npx jest backend/__tests__/sessionSupersede.test.js` green. A cycle approved 24 hours after its plan completes produces an engineering session whose first message contains the plan's requirement titles — **verified by** `GET /v1/sdlc/sessions/{engineeringSessionId}` and reading the first user message.

**Commit.** `fix: start engineering from the persisted plan on a fresh session`

### Phase 5 exit criteria

A§6 Phase 5's three, plus two:

1. **The plan session's payload populates every field both live surfaces read** — measured against `PLAN_PAYLOAD_SCHEMA` (Task 5.1), whose provenance is `generatePlan`'s output-format block (`engineeringAgent/orchestratorHandler.js:298-321`) plus the two live consumers enumerated in §1.12. **Verified by** two screenshots (`PlanReview.jsx` and the attention-card dialog, per Task 5.3) plus `--query 'Item.expandedRequirements.M | keys(@)'` listing every required property.
1b. **A plan that would have crashed the approval gate is caught in-session, not at the gate.** Force it: send a `resultSchema` run against a payload whose `requirements` is a string, and confirm the agent receives a `toolError` and retries rather than the payload reaching CC. This is the one criterion that proves `resultSchema` is doing work — without it, §1.12(4)'s `tasks.reduce` crash is still live (D§5(b) item 1).
1c. **Suppressing `resultSchema` degrades cleanly.** With the schema omitted by config, a plan session still completes and CC's own validator still rejects a malformed payload to `PLANNING_FAILED`. Proves the kill switch works without a runtime deploy (`DEP-P1-13`).
2. **`recordCheckpoint`'s evidence counts are non-zero** — `filesToCreate` and `filesToModify` populated. **Verified by** `aws dynamodb query … begins_with(SK, '<cycleId>#step#planning')` and reading `evidence.filesToCreate`.
3. **Plan revision produces a new plan version through `planComments.storePlanVersion`**, as today. Tasks 5.5a and 5.5b.
3b. **`cycle.planVersion` increments and the primary surface shows it.** `PlanReview.jsx:17,71` renders "Plan v2" once `planComments.storePlanVersion` has stamped it (`planComments.js:304-305`). A first-pass plan correctly shows no version badge. Neither prior revision accounted for this field (§1.12(3)).
4. `technicalApproach` appears nowhere in `frontend/src/` (Task 5.3's guard).
5. A runtime plan-session failure lands in `planning_failed`, not in `planning_review` with a narrow plan (Task 5.4).
6. **A session that submits a plan twice leaves the first one persisted**, and the second is logged and discarded rather than overwriting (`DEP-P1-14`). Force it if the model does not do it spontaneously.
7. **No SDK release and no runtime deploy were needed for this phase.** If one was, something in Task 5.1 or 5.2 put CC vocabulary on the wrong side of the boundary — raise it rather than scheduling a window.

No long-gate test is needed: nothing crosses the gate any more (A§3.2). A 24-hour approval and a one-minute approval take the identical code path, and Task 5.6's test asserts it.

---

## 7. Phase 6 — QA as a session

**Goal (A§6 Phase 6, A§3.5).** QA becomes a **`mode: 'plan'`** session in its own clone, pinned to the engineering commit with `baseBranch` fetched, with instructions scoped to review rather than implement, and with **no write tools**.

> **Corrected 2026-09-25.** This line previously said `mode: 'do'`, which contradicted **Task 6.7 in this same document** — that task argues at length that a reviewer able to edit the code under review will fix findings instead of reporting them. `mode: 'plan'` is what the implementation does (`runtimesession.ModeForStage(StageQA)`), on Phase 2's verification against the deployed runtime that plan mode drops `editFile` and `writeFile` while keeping `useMcpTool`, so the terminal tool still works. Plan mode is therefore the mechanism that delivers "no write tools" — it is not an alternative to it. Note that the per-role tool allowlist §A.7.3 describes is **not expressible** in the deployed `sherpa-core@4.0.1`: `buildToolConfig` is subtractive with no allowlist parameter, so the session mode plus a provider that advertises exactly one terminal tool are the controls that actually exist. A review payload → `cycle.qaResult` → CC validates findings against the real diff → `pulls.createReview`.

**Revision 2 — three changes, and one of them is a hazard revision 1 did not see.**

1. **`submitReview` is a named tool whose `payload` is `unknown`; `ReviewCompletion` is not a protocol type.** Revision 2 collapsed it into a generic `submitResult`; revision 3 restores the name and keeps the payload opaque (§0.2, D§ verdict). The QA payload shape is a **Command-Center-side schema** — `QA_PAYLOAD_SCHEMA` in `backend/common/sdlcContract.js`, the same object used for the prompt contract, the `resultSchema` sent at create, and the consumer's validator. Task 6.1 owns all three uses. The `completion` event's `kind` is `'review'`, which restores the `kind` ↔ `correlation.stage` cross-check that the generic tool had drained (D§5(b)).
1b. **In-session enforcement is back.** `resultSchema` (`DEP-P1-13`) means a malformed review is a `toolError` the agent corrects in-turn, not a dead session. That matters more for QA than for planning, because D§2 enumerates four *current* silent corruptions a schema catches: a non-array `findings` throws out of `routeAfterEngineering` (`index.js:278`, `qaAgent:503`, neither in a try/catch); a string `line` silently demotes every finding from an inline comment to a body bullet (`qaAgent:321`, a `Set<number>` membership test); an unknown `severity` silently becomes 💡 (`:312-317`); and a string `requirements` char-iterates in the engineering prompt.
2. **QA's clone is not detached, and `branch` is returned.** §1.11a. Revision 1's Task 6.3 acceptance check would have failed on correct behaviour.
3. **A bad `commitSha` silently serves the branch tip.** §1.11b — `checkoutCommit` returns `false` and the session proceeds against whatever the clone has. **This is the worst failure mode in this document**: QA returns a confident verdict on code the cycle never asked it to review. Task 6.3 carries the CC-side half of the mitigation; `DEP-P1-10` carries Part 1's.

**This is the largest phase.** Unlike engineering, where `routeAfterEngineering` was transport-agnostic, QA's entire result-handling is inline and synchronous (§1.2). Read this before starting:

- `routeAfterEngineering`'s `ai-qa` branch (`index.js:192-305`) sets `QA_TESTING`, persists (`:198-204`), builds `qaPayload` (`:206-232`), calls `invokeQAAgent` **synchronously at `:237`**, and then processes the result across `:247-305`.
- `qaAgent/orchestratorHandler.handle` (`:45`) reassigns the module-level `let agentContext = ''` (`:40`) at `:49` — so a warm container leaks the previous cycle's bootstrap prompt if `:49` throws. Pre-existing; fixed on the way out in Phase 7.
- `analyzeCodeChanges` (`:161-355`) fetches the diff via `octokit.paginate(pulls.listFiles)` at `:188` and reviews it **blind** — it cannot open a file the diff does not touch, cannot run the tests, cannot check whether a changed function has other callers. A session can. *"This is the single largest quality improvement in the migration and it is a side effect, not the goal."*
- Diff-line validation is at `:291-303` (building `validDiffLines` from each file's `patch` hunks) and `:318-323` (the `validLines?.has(f.line)` test with a fallback into the review body). GitHub 422s the whole review if any comment line is not in the diff — this is load-bearing.
- `pulls.createReview` is at `:337-344`, with `event: approved ? 'APPROVE' : 'COMMENT'`.
- `updateCycleWithQAResult` (`:453-562`) reads `qaResult.success` and `qaResult.requirementsMet` (`:480-490`) to pick between `PENDING_APPROVAL` and `QA_FAILED`.

### Task 6.1 — the QA payload schema and its adapter onto `qaResult`

**Files**
- `backend/common/qaResultAdapter.js` (NEW)
- `backend/__tests__/qaResultAdapter.test.js` (NEW)

**Why.** §1.7. The review payload CC asks for and the `qaResult` shape CC's consumers read are different, and revision 1 specified an adapter between a protocol type and a consumer. Since `submitReview.payload` is opaque, the upstream half is CC's too, so this module owns **three** ends: it defines `QA_PAYLOAD_SCHEMA` (which Task 6.3's instructions embed **and** send as `resultSchema`), validates what arrives, and adapts it onto `qaResult`.

**`QA_PAYLOAD_SCHEMA` carries its own `maxItems`/`maxLength`**, for the same per-cycle reason as the plan schema (`DEP-P1-13`): `qaResult` is written into `cycle.iterations[n]`, so a review with 200 findings competes for the same 400 KB item as `progressLog` and every prior iteration. Cap `findings` with `maxItems` and each `body` with `maxLength`, derived from the cycle's remaining room rather than from a constant.

**Put the schema in `backend/common/sdlcContract.js` alongside `PLAN_PAYLOAD_SCHEMA`** (Task 5.1), not in a QA-specific module. D§6.3: one file holding both schemas as data, with `instructions`, `resultSchema` and the consumer's validator all reading the same objects, is what makes drift structurally impossible. `qaResultAdapter.js` keeps the *adapter* and `QA_DIFF_COMMAND`; the schema and its validator move to `sdlcContract.js`.

**That is a net simplification, not extra work.** One module, one fixture set, no version skew between what a protocol package says a review is and what CC reads. And the wire shape can now be chosen to minimise the adapter rather than to satisfy a protocol type — so where revision 1 had to collapse a four-level severity onto three and leave `suggestion` and `context` unsourced, **the payload schema simply asks for the fields CC renders.** The mapping table below keeps the four-level severity because it is the better input (it lets CC set the verdict threshold independently of the emoji) but adds `suggestion` and `context`, which the agent naturally produces and the PR comment is better for.

**Field mapping, as verified against every consumer:**

| payload (CC-defined) | `qaResult` | Consumer | Note |
|---|---|---|---|
| `verdict: 'pass'` | `success: true`, `requirementsMet: true`, `allTestsPassed: true` | `index.js:256`, `:269`, `qaAgent:480-490` | |
| `verdict: 'fail'` | `success: true`, `requirementsMet: false`, `allTestsPassed: false` | as above | `success` means "the review ran", not "the code passed" — see below |
| `summary` | `summary` | `index.js:299`, `qaAgent:331` | Becomes the PR review body |
| `findings[].path` | `findings[].file` | `qaAgent:318,320`, `index.js:280-283` | **Rename** |
| `findings[].line` | `findings[].line` | same | Must be a NEW-file diff position |
| `findings[].body` | `findings[].issue` | same | **Rename** |
| `findings[].severity` (`blocker\|major\|minor\|nit`) | `findings[].severity` (`error\|warning\|info`) | `qaAgent:307` | **Enum collapse**: blocker/major → `error`, minor → `warning`, nit → `info` |
| `findings[].suggestion?` | `findings[].suggestion` | `qaAgent:311`, `index.js:282` | **Now asked for.** Optional; consumers already guard with `f.suggestion ? … : null` |
| `findings[].context?` | `findings[].context` | `qaAgent:310` | **Now asked for.** Optional, guarded at `:310` |
| `testsRun?` | `testPlan: {summary: testsRun}` | `index.js:299` fallback | |
| — | `overallQuality` | `qaAgent:286` (`!== 'needs-work'`) | **Derived** from `verdict`. Not asked for — the same judgement in two vocabularies invites the model to contradict itself |
| — | `testsGenerated`, `testBranch`, `testFiles`, `githubActionsRequired` | `index.js:258-266` | **No source.** QA-as-review generates no tests; see OD-2 |
| — | `recommendations[]` | `index.js:288-292`, `qaAgent:333` | **No source.** Leave empty |
| `model`, `usage` | stored alongside, not inside `qaResult` | | Cost attribution, A§2.3 |

**This task closes a live QA safety hole. Record it as a deliberate fix, not an incidental one** (D§5(f)), because the next person to "restore parity with the old QA agent" would reopen it.

Today a cycle's QA pass/fail comes from `validateRequirements().met`, and **`requirementsMet` defaults to `true` when the requirements argument is falsy.** Verified: `qaAgent/orchestratorHandler.js:86` is `let requirementsMet = true;` and it is only overridden inside `if (requirements)` at `:89-92`. And the argument CC passes is **`requirements: cycle.task`** (`index.js:220`) — the one-line business goal, not the plan. So **a cycle whose `task` is empty passes QA regardless of what the review found.**

Separately, the GitHub review verdict comes from entirely different fields: `approved = result.passed === true && result.overallQuality !== 'needs-work'` (`:286`). Two deciders, two sources, no coupling between them — so the PR could be approved while the cycle records a failure, or the reverse.

Task 6.1 derives **everything** from one `verdict`, which closes both halves. Add the assertions so the property is pinned rather than emergent:

```
describe('toQaResult — one verdict decides everything')
  it('never passes a review that reported a fail verdict, whatever else is absent')
      → {verdict:'fail', findings:[…]} with no requirements-like field anywhere
      → requirementsMet === false
      // The live defect: requirementsMet defaults to true when its input is falsy
      // (qaAgent:86-92) and its input is cycle.task (index.js:220), so an empty
      // task passed QA regardless of findings. Do not reintroduce a default-true.
  it('derives the PR review verdict from the same verdict as the cycle status')
      → the value driving `approved` and the value driving requirementsMet agree
        for both 'pass' and 'fail'
```

`success` is the trap. `qaAgent/orchestratorHandler.js:480` reads `qaResult.success` as "the QA step completed" and then `requirementsMet` as the verdict — but `routeAfterEngineering:301-304` treats `!qaResult.success` as `FAILED` with `cycle.error = qaResult.error`. So a failing **review** must be `{success: true, requirementsMet: false}`, and only a failing **session** is `{success: false, error}`. Get this backwards and every QA rejection becomes a generic `FAILED` cycle with no findings.

**Tests first:**

```
describe('toQaResult — verdict')
  it('maps a pass to a successful review with requirements met')
      → {verdict:'pass'} → {success:true, requirementsMet:true, allTestsPassed:true}
  it('maps a fail to a successful review with requirements unmet, not to a failed step')
      → {verdict:'fail'} → {success:true, requirementsMet:false, allTestsPassed:false}
      // The distinction routeAfterEngineering:301 depends on.
  it('derives overallQuality so the GitHub review verdict is computable')
      → pass → overallQuality !== 'needs-work'; fail → overallQuality === 'needs-work'

describe('toQaResult — findings')
  it('renames path to file and body to issue, which is what CC reads')
      → {path:'src/a.js', line:12, severity:'major', body:'unchecked null'}
      → {file:'src/a.js', line:12, severity:'error', issue:'unchecked null'}
  it('collapses the four-level severity onto the three CC renders')
      → blocker → 'error'; major → 'error'; minor → 'warning'; nit → 'info'
  it('does not invent a suggestion or a context')
      → result.findings[0].suggestion === undefined
      → result.findings[0].context === undefined
  it('preserves finding order, because the review body lists them in order')
  it('drops nothing — every finding in, every finding out')
      → 7 findings in, 7 out

describe('toQaResult — fields with no source in ReviewCompletion')
  it('reports no generated tests, because a review generates none')
      → testsGenerated === 0; testBranch undefined; githubActionsRequired === false
      // githubActionsRequired must be falsy or routeAfterEngineering:265 parks the
      // cycle at QA_WAITING_FOR_TESTS waiting for CI that will never report.
  it('carries testsRun into testPlan.summary so it reaches the review body')
  it('leaves recommendations empty rather than duplicating findings')

describe('validateQaPayload — rejection')
  it('refuses a fail verdict with no findings')
      → {verdict:'fail', findings:[]} → {ok:false, field:'findings'}
      // A§2.6 put this in the tool's inputSchema, which enforces nothing
      // (bedrock-client.ts:159-165). Under resultSchema the runtime DOES enforce it
      // in-session, but CC must still check: resultSchema is optional, so the
      // validates nothing, so CC is the only check there is.
  it('refuses an unknown verdict rather than defaulting to pass')
      // Defaulting to pass on a malformed payload would merge unreviewed code.
      // This is the single most consequential rejection in the module.
  it('refuses an unknown severity rather than defaulting to error')
  it('refuses a finding with no path, because it cannot be placed or rendered')
  it('names the offending field, so a failed review is diagnosable from the record')
  it('accepts a pass verdict with no findings')
      → a clean review is legitimate and must not be rejected
```

The `githubActionsRequired` test is the one most likely to save a day: `routeAfterEngineering:264-267` checks it first and parks the cycle at `QA_WAITING_FOR_TESTS`, which is a `POLLED_STATUS` with a 30-minute poller timeout, waiting for a CI run nobody triggered.

**Implementation.** Two modules, because the schema is shared and the adapter is not:

- **`backend/common/sdlcContract.js`** (created by Task 5.1) gains `QA_PAYLOAD_SCHEMA` and `validateQaPayload(payload)` returning `{ok: true, payload} | {ok: false, field, reason}`. The schema literal is the single source for the prompt contract, the `resultSchema` Task 6.3 sends, and this validator.
- **`backend/common/qaResultAdapter.js`** keeps `toQaResult(payload)` and `QA_DIFF_COMMAND`. Pure, no AWS, no `process.env`; follow `progressLogger.js`'s module shape.

**Return a result object, do not throw, for validation.** Part 1 §A.4.6's question decides it: **would a redrive after a code deploy succeed?** For a malformed payload, no — the same bytes fail the same way, so DLQ-ing it would freeze the session's FIFO group for ~9 minutes (A§2.4) to reach the conclusion the first attempt already had. It becomes a **terminal cycle status with the message consumed**, and Task 6.5 maps it to `QA_FAILED` with the field name. Reserve throwing for the failures a redrive *would* fix — an unrecognised effect kind means the mapper and the executor have drifted, which a deploy repairs (`DEP-P1-11`).

**Acceptance.** `npx jest backend/__tests__/qaResultAdapter.test.js` green, with a test for every row of the table above.

**Commit.** `feat: define the QA payload schema and adapt it onto qaResult`

### Task 6.2 — move the GitHub review post into a shared CC action

**Files**
- `backend/common/qaReview.js` (NEW)
- `backend/lambda_handlers/qaAgent/orchestratorHandler.js` (MOD)
- `backend/__tests__/qaReviewPost.test.js` (NEW)

**Why (A§3.5).** *"Posting the GitHub review stays in CC… That validation logic is load-bearing and it is an API call with no working tree — by decision 7's principle it is an orchestrator action."* Today it lives inside `analyzeCodeChanges`, which Phase 7 deletes. Extract it now, while both callers exist.

**Tests first** — pure except for a stubbed `octokit`:

```
describe('buildValidDiffLines')
  it('collects new-file line numbers from a unified diff patch')
      → one file, patch with '@@ -1,3 +1,5 @@' and two '+' lines
      → set contains the added line numbers and the context line numbers
  it('does not count removed lines, which have no new-file position')
  it('ignores the +++ and --- headers')
  it('handles a file with no patch (binary or too large) as having no valid lines')
  it('resets the counter at each hunk header')
      → two hunks with a gap; the second hunk's lines use its own start

describe('postQaReview')
  it('posts findings whose line is in the diff as inline comments')
  it('moves findings outside the diff into the review body instead of dropping them')
      → finding on a line not in validDiffLines
      → createReview.comments does not include it
      → createReview.body contains the finding text
  it('moves findings with no file or line into the review body')
  it('approves when the verdict passed')
      → createReview called with event: 'APPROVE'
  it('comments rather than requesting changes, as today')
      → createReview called with event: 'COMMENT'
      // Commit 1 pins existing behaviour. Commit 2 flips this assertion to
      // event: 'REQUEST_CHANGES'. See below — two commits, deliberately.
  it('sends every inline comment with side RIGHT')
  it('does not throw when GitHub rejects the review')
      → createReview rejects → resolves, logs a warning
      // Preserves today's non-fatal behaviour (qaAgent:346-348).
  it('reports whether the post succeeded, so the caller can record it')
      → returns {posted: boolean, inlineCount, fallbackCount}
```

**The `REQUEST_CHANGES` change ships as its own commit, not as a line in this one's PR description.** `qaAgent/orchestratorHandler.js:341` sends `event: approved ? 'APPROVE' : 'COMMENT'` while the log line at `:345` says `REQUEST_CHANGES` — so QA has never actually requested changes on GitHub, only commented.

R§3.2 is right that revision 1 under-handled this. Fixing it means **a failing QA review starts blocking merges on GitHub for every application, immediately, on the legacy path** — it is a user-visible behaviour change to a shipped integration, and revision 1 buried it inside a commit whose message is `fix: extract the QA review post…`. Flagging it for the PR description is not the same as not smuggling it.

**So: two commits.**

1. `fix: extract the QA review post so both transports share the diff validation` — pure extraction, behaviour identical, `event: approved ? 'APPROVE' : 'COMMENT'` unchanged. The test `it('comments rather than requesting changes, as today')` pins the current behaviour.
2. `fix: request changes on a failing QA review instead of only commenting` — one line, its own PR, its own description explaining the effect on merge gating. The test above flips to `it('requests changes when the verdict failed')`.

If the reviewer prefers to keep `COMMENT`, commit (2) never lands and the log line at `:345` is corrected instead — but do not leave the code and the log disagreeing.

**Implementation.**
1. `backend/common/qaReview.js` — export `buildValidDiffLines(prFiles)` and `postQaReview({owner, repo, prNumber, qaResult, approved}, {octokit})`. **Move** `qaAgent/orchestratorHandler.js:288-348` verbatim, then apply the `REQUEST_CHANGES` fix and the emoji/body composition as-is (`:306-336`).
2. `qaAgent/orchestratorHandler.js` — replace that block with a call. Behaviour must be identical apart from the review event.
3. The caller needs the PR's file list. Today `analyzeCodeChanges` already has `fetchedPrFiles` from `:188`. The consumer path (Task 6.4) must fetch it itself: `octokit.paginate(octokit.rest.pulls.listFiles, {owner, repo, pull_number})`. Put that fetch in `qaReview.js` as `fetchPrFiles({owner, repo, prNumber}, {octokit})` so both callers share it.

**Acceptance.** `npx jest backend/__tests__/qaReviewPost.test.js` green; the existing QA path still posts a review on a legacy cycle — **verified by** one non-runtime cycle reaching QA and the PR showing a review.

**Commit.** `fix: extract the QA review post so both transports share the diff validation`

### Task 6.3 — dispatch a QA session

**Files**
- `backend/common/cycleEffects.js` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/__tests__/qaSessionDispatch.test.js` (NEW)

**Depends on** `DEP-P1-4` (`commitSha` + `baseBranch` on create), Task 4.1, Task 4.7.

**Why the commit and not the branch (A§3.5).** *"A branch is a moving reference — engineering's session may still be alive and may push again on an iteration, so 'review the branch' means reviewing whatever the tip happens to be at read time."* It matters concretely at the end of the step: CC validates inline comments against the real diff of a **specific commit** before `pulls.createReview`. Review a different commit than the PR head and the positions are wrong.

**Why `baseBranch` is required (A§3.5), and which diff command to use.** A separate clone has only what it fetched. Verified: the only fetch in the SDK is `git fetch --depth 1 origin -- <sha>` for the pinned commit (`4.0.1 git-checkout.ts:44`), and `cloneWorkspace` takes no base ref (`nevado-sherpa-tui apps/runtime/src/agent/workspace.ts:33`) — so **there is no base-branch fetch anywhere today.** Part 1 Task 2.3 builds it.

**The diff form is two-dot, and this is derived rather than guessed** (§1.11c). `git diff A...B` is defined as `git diff $(git merge-base A B) B`, and `git-checkout.ts:33-35` rejects ancestry reasoning over a `--depth 1` graft in terms that apply exactly here: *"a grafted `--depth 1` commit has no known ancestry, so ancestor reasoning is unsound."* Two independent shallow grafts may have no merge base at all.

**Therefore the QA instructions use `git diff origin/<baseBranch> HEAD`.** R§4(g) is right that Part 1 leaving this as "write the working command back into §A.2.1 during implementation" will not survive an unsupervised implementer, so this task makes it a **named deliverable**: the command lives in `QA_DIFF_COMMAND` in `backend/common/qaResultAdapter.js` (alongside `QA_PAYLOAD_CONTRACT`, Task 6.1) and the instruction composer imports it. One place to change if Part 1 Task 2.3's empirical test contradicts the reasoning — in which case Part 1 wins, being authoritative on runtime facts.

**Tests first:**

```
describe('dispatchQaSession')
  it('pins the engineering commit rather than following the branch')
      → spec.commitSha === cycle.pullRequest.headSha
      → spec has no `branch` field
      // "Pins", not "checks out detached" — §1.11a. HEAD stays on the cloned
      // branch with that ref moved to the commit.
  it('requests the application default branch as the diff target')
      → spec.baseBranch === application.github.defaultBranch
  it('falls back to develop when the application declares no default branch')
      → matches index.js:1871's `application.github.defaultBranch || 'develop'`
  it('correlates the session at the qa stage')
      → correlation.stage === 'qa'
  it('puts the PR number in the prompt, not in the API payload')
      → spec.prompt contains `#${prNumber}`
      → spec has no `prNumber` / `pullRequest` field
      // A§3.5: putting it in the API would hand the runtime an SDLC concept.
  it('creates the QA session before releasing the engineering session')
      → createSession then releaseSession(engineeringSessionId)
      // Task 4.7's order. If the QA create fails, the engineering session and its
      // workspace are still there and the step can retry.
  it('records a leak when the engineering release fails, rather than warning')
      → releaseSession rejects twice → cycle.releaseLeak written, error-level
        progress line, QA still proceeds
      // Task 4.7's severity. Release revokes the push credential (§1.10, R14).
  it('persists the session id under qaSessionId')
  it('does not dispatch twice for one iteration')
      → cycle.qaSessionId already set for this iteration → no create
  it('does not tolerate a missing head SHA')
      → cycle.pullRequest absent → fails the cycle rather than reviewing HEAD
  it('verifies the created session actually landed on the requested commit')
      → createSession resolves with a session whose commitSha differs from the request
      → the cycle fails to QA_FAILED and no review is posted
      // §1.11b: checkoutCommit returns false and the session proceeds on the branch
      // tip. Without this check QA returns a confident verdict on the wrong code.
  it('falls back to GET /sessions/{id} when create does not report the commit')
      → DEP-P1-2. A 200 that carries {status:'unshared'} is "unknown", not "match".
  it('does not assume branch is absent on a commitSha session')
      → a response carrying a real `branch` is accepted, not treated as an error
      // Revision 1 asserted `branch` was null/absent here and would have failed
      // on correct behaviour — §1.11a, R§4(a).
  it('names the session within the runtime limit')
      → name.length <= 50; matches /^cycle\/.* · qa$/
  it('sends QA_PAYLOAD_SCHEMA as resultSchema')
      → spec.resultSchema === QA_PAYLOAD_SCHEMA   (identity, not a copy)
  it('embeds the same schema object in the instructions, alongside QA_DIFF_COMMAND')
      → the composed instructions contain the serialised QA_PAYLOAD_SCHEMA and the
        two-dot diff command, and both are the same objects the module exports
```

The head-SHA test matters because `cycle.buildHeadSha` and `cycle.pullRequest.headSha` are both set by `completeEngineering` (Task 4.3) and they are the same value — but only on the happy path. Assert the source explicitly rather than reading whichever happens to be present.

**Implementation.** `dispatchQaSession({cycle, application, pk, sk}, deps)` in `backend/common/cycleEffects.js`, built on `supersedeSession` with `stage: 'qa'`. Instructions and prompt are Part 1's composition (A§2.7); this function assembles the spec and the correlation.

**Release ordering, and the revision-1 note it replaces.** Revision 1 said *"the engineering session is released **before** the QA session is created… QA holds a detached clone (harmless)"*. Both halves are wrong. The ordering is **create then release** (Task 4.7, R§1.2), and **QA's clone is not detached** — §1.11a: `git-checkout.ts:29-32` deliberately rejected detaching, so HEAD stays on the cloned branch with that ref moved to the engineering commit. There is no collision either way, because the runtime uses independent clones; the ordering follows the one rule that holds everywhere.

**Acceptance.** `npx jest backend/__tests__/qaSessionDispatch.test.js` green. A runtime cycle entering QA produces a session pinned to the engineering commit — **verified by** the create response's (or `GET /v1/sdlc/sessions/{id}`'s) `commitSha` matching `cycle.pullRequest.headSha`. **`branch` will be a real ref, not `null`** (§1.11a); asserting otherwise is the revision-1 error. Then the negative: create one QA session with a deliberately unfetchable-but-well-formed sha (40 hex characters from an unrelated repo) and confirm the cycle fails to `qa_failed` with an explicit mismatch message and **no review is posted** — that is the §1.11b mitigation actually working, and it is the single most important check in this phase.

**Commit.** `feat: run QA as a runtime session on its own clone of the engineering commit`

### Task 6.4 — make `routeAfterEngineering`'s `ai-qa` branch asynchronous

**Files**
- `backend/common/cycleEffects.js` (MOD) — **not `index.js`**
- `backend/__tests__/cycleEffects.test.js` (MOD) and `backend/__tests__/routeAfterEngineeringQa.test.js` (NEW)

**This is the structural change A§5.3 says is not needed.** See §1.2.

**Revision 2 — retargeted. `routeAfterEngineering` is not in `index.js` any more by the time this task runs.** R§3.1: Part 1 Task 3.3 moves it to `backend/common/cycleEffects.js` in Phase 3, and that task's own acceptance check is `grep -c "^async function routeAfterEngineering" index.js` returning `0`. Revision 1's file list, `describe` blocks and implementation notes all targeted `index.js:192`, which would have sent an implementer hunting through a 4,984-line file for a function that had moved. Every line reference below is to the **pre-move** source, for identification only; find the code in `cycleEffects.js` by content.

Two consequences beyond the file path:

- **The injected-dependency signature applies.** `routeAfterEngineering` in `cycleEffects.js` takes `(client, tableName, …)` and receives `invokeQAAgent`, `getOctokit`, `addProgressLog` and `applyTransition` as parameters (Part 1 Task 3.3). `dispatchQaSession` joins that dependency set rather than being imported directly — otherwise `cycleEffects.js` would need a runtime HTTP client, and `backend/common/` may not read `process.env` for the base URL.
- **Part 1's tests already pin the four branches.** `cycleEffects.test.js` has `it('routes an ai-qa cycle to qa_testing and invokes QA')` and `it('persists only on the ai-qa branch, as today')`. This task must **amend** the first (it becomes conditional on `buildMode`) and leave the second alone. Extending Part 1's suite is cheaper and safer than a parallel one; use the new file only for the runtime-path cases.

**Tests first** — orchestrator style (`applyRuntimeEffect.test.js` harness):

```
describe('routeAfterEngineering — ai-qa on the runtime path')
  it('starts a QA session and stops, rather than invoking the QA Lambda')
      → application.agentConfig.buildMode = 'runtime'
      → dispatchQaSession called once; invokeQAAgent not called
  it('leaves the cycle at QA_TESTING for the consumer to resolve')
      → cycle.status === QA_TESTING; stage === 'qa'
  it('persists before returning, so a lost invocation does not lose the dispatch')
      → a PutCommand is sent with the QA_TESTING status and the qaSessionId
  it('writes nothing about tests, because none have run yet')
      → cycle.testBranch / testsGenerated / testMetadata untouched

describe('routeAfterEngineering — the other three branches are unchanged')
  it('sends no-qa straight to PENDING_APPROVAL')
  it('sends manual-review straight to PENDING_APPROVAL')
  it('parks ci-only at QA_WAITING_FOR_TESTS')
  it('lets cycle.skipQA force no-qa')
  it('keeps the legacy synchronous QA invoke for non-runtime applications')
      → no buildMode → invokeQAAgent called; dispatchQaSession not called
```

That last one is the safety net: this task must not change behaviour for any application not on `buildMode: 'runtime'`.

**Implementation.** In `cycleEffects.js`'s `routeAfterEngineering`, in the `ai-qa` else-branch (pre-move `index.js:192`), after `applyTransition(cycle, QA_TESTING, {stage: 'qa'})` (pre-move `:195`) and the `PutCommand` (pre-move `:197-204`), branch:

```js
if (application?.agentConfig?.buildMode === 'runtime') {
  await deps.dispatchQaSession(client, tableName, { cycle, application, pk, sk });
  cycle.updatedAt = new Date().toISOString();
  return;   // the consumer resolves the step from the review payload
}
// legacy: synchronous invokeQAAgent, unchanged
```

The existing `PutCommand` already persists the `QA_TESTING` status before QA runs, which is exactly what the async path needs — do not move it. Note that this `PutCommand` is the **only** persistence in `routeAfterEngineering` and it is inside this branch; the other three branches mutate `cycle` in memory and rely on the caller to write it (A§3.3). Do not "fix" that here; `completeEngineering`'s step 7 is the caller's write.

`dispatchQaSession` needs the session-id write to survive, so it must happen inside the same `PutCommand` or in its own conditional update. Prefer its own update with `ConditionExpression: 'attribute_not_exists(qaSessionId)'` (Task 4.2's pattern) rather than folding it into the existing put — the put writes the whole item and would clobber a concurrent write.

**Acceptance.** `npx jest backend/__tests__/routeAfterEngineeringQa.test.js` green, and the existing `backend/__tests__/postDeployQaPolicy.test.js` and `cycleActionWiring.test.js` unchanged. A legacy cycle still runs the synchronous QA path — **verified by** one non-runtime cycle reaching `qa_testing` and then `pending_approval` within a single orchestrator invocation (check the Lambda's log for one request id covering both).

**Commit.** `feat: dispatch QA asynchronously on the runtime path`

### Task 6.5 — resolve the QA step from the review payload

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js` (MOD)
- `backend/common/cycleEffects.js` (MOD)
- `backend/__tests__/runtimeEvents.test.js` (MOD)
- `backend/__tests__/qaCompletion.test.js` (NEW)

**Depends on** Tasks 6.1, 6.2, `DEP-P1-5`, `DEP-P1-7`.

**Tests first.**

`runtimeEvents.test.js` (MOD):

```
describe('mapSessionEvent — QA completion')
  it('maps a submitReview completion on the qa stage to a QA-completion effect')
      → stage 'qa'
      → event {type:'completion', kind:'review', outcome:'success', summary:'…',
          payload:{verdict:'fail', summary:'…',
                   findings:[{path:'a.js',line:3,severity:'blocker',body:'…'}],
                   testsRun:'npm test: 2 failures'},
          model:'…', usage:{…}}
      → effect.kind === 'qaComplete'
      → effect.payload.findings[0].path === 'a.js'
      // The adapter runs in the consumer, not the mapper — so the effect carries
      // the raw payload and `file` does not exist yet at this point.
  it('does not validate the payload — that is the consumer’s job')
      → kind:'review' with payload:{} produces qaComplete, not a status effect,
        and does not throw
      // Part 1's mapper-purity test. Revision 2 validated here and conflicted.
  it('reports a protocol violation when kind does not match the stage')
      → stage 'qa' with kind:'plan' → unsupported-kind failure → batch item
        failure → DLQ + alarm
      // Restored in revision 3: under one generic tool this mislabel was
      // undetectable, and a review stamped stage:'planning' would have been
      // misrouted to the plan validator and landed as PLANNING_FAILED on a QA
      // step — misdiagnosed and wrongly named (D§5(b)).
  it('fails the cycle when the agent reports outcome blocked')
      → {outcome:'blocked', summary:'could not fetch the base ref'}
      → QA_FAILED with the summary as the error
  it('fails a qa-stage error event to QA_FAILED, not FAILED')
      → stage 'qa', {type:'error'} → effect.to === cycleStatuses.QA_FAILED
```

`qaCompletion.test.js` (NEW):

```
describe('completeQa — the record it writes')
  it('writes qaResult onto the current iteration, not onto the cycle root')
      → cycle.iterations[n].qaResult set; cycle.qaResult absent
      // Matches index.js:249-252 / qaAgent:475-476.
  it('stamps qaCompletedAt and completedAt on the iteration')
  it('promotes nothing about generated tests, because a review generates none')
      → cycle.testBranch / testsGenerated remain undefined

describe('completeQa — verdict routing')
  it('sends a pass to PENDING_APPROVAL with the human_approval stage')
      → status PENDING_APPROVAL; stage 'human_approval'
  it('sends a fail to QA_FAILED with file:line-scoped failures for engineering')
      → status QA_FAILED
      → cycle.qaFailures[0] === 'a.js:3 — unchecked null'
      // The exact format index.js:280-284 builds, so the engineering prompt is
      // unchanged from today's.
  it('carries the summary as qaFeedback')
      → cycle.qaFeedback contains the review summary
  it('does not append recommendations when there are none')

describe('completeQa — posting the review')
  it('fetches the PR files and posts the review before transitioning')
      → fetchPrFiles then postQaReview then the status write
  it('still transitions when the review post fails')
      → postQaReview resolves {posted:false} → status still written
      → a progress line records that the review was not posted
  it('does not post a review when the cycle has no PR')
      → cycle.pullRequest absent → postQaReview not called; cycle fails with a reason

describe('completeQa — idempotency')
  it('does not double-post a review on a replayed message')
      → second call with the same payload: postQaReview not called again
      → guard on cycle.iterations[n].qaCompletedAt
  it('does not move a cycle already past QA backwards')
      → cycle.status PENDING_APPROVAL, replayed fail verdict
      → the conditional status write fails the condition; status unchanged
```

The replay guards are not optional. A§2.4 is explicit that duplicate delivery must be assumed and that `applyTransition` does not prevent double-application — it takes `strict = false` and assigns anyway (`sdlcEngine.js:544`). A replayed `fail` on a cycle already at `PENDING_APPROVAL` would move it backwards to `QA_FAILED`, and `PENDING_APPROVAL → QA_FAILED` is not even in `TRANSITIONS` (`sdlcEngine.js:145-155`), so it would log a violation and do it anyway.

**Implementation.**
1. `runtimeEvents.js` — the `kind: 'review'` arm of the `completion` case, cross-checked against `correlation.stage === 'qa'`, returning `{kind: 'qaComplete', payload, usage}`. **It does not validate and it does not adapt** — both move to the consumer, per Part 1's mapper-purity test. This **replaces** the placeholder unsupported-stage arm Part 1 Phase 3 installed (`DEP-P1-11`); delete that arm in the same commit rather than leaving two paths for one stage.
1b. **The consumer** calls `validateQaPayload` then `toQaResult`, and on a validation failure writes `QA_FAILED` naming the field and **consumes the message** (§A.4.6: a redrive re-reads the same bytes). Add the test that a second `submitReview` in one session does not overwrite the first (`DEP-P1-14`), mirroring Task 5.1's.
2. `backend/common/cycleEffects.js` — `completeQa({cycle, application, pk, sk, qaResult, usage}, deps)`. Port the verdict routing from `index.js:254-305` and `qaAgent/orchestratorHandler.js:478-500`, dropping the test-generation branches (`index.js:258-267`) which have no source in a review. Keep the `qaFailures` formatting at `:280-284` byte-identical — it is what the engineering loop-back prompt is built from.
3. `applyRuntimeEffect` — the `qaComplete` arm.
4. Use `DEP-P1-7`'s conditional status write, asserting the cycle is still `QA_TESTING`.

**Acceptance.** `npx jest backend/__tests__/qaCompletion.test.js backend/__tests__/runtimeEvents.test.js` green. A runtime cycle's QA failure produces a GitHub review with inline comments and lands at `qa_failed` — **verified by** `gh pr view <n> --repo <org/repo> --json reviews --jq '.reviews[-1]'` showing the review and its state, plus the cycle record's `qaFailures`.

**Commit.** `feat: resolve the QA step from a review payload`

### Task 6.6 — a failed review starts a new engineering session

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/common/cycleEffects.js` (MOD)
- `backend/__tests__/qaIterationLoop.test.js` (NEW)

**Why (A§3.5, A§3.2).** *"QA verdict fail → `QA_FAILED`, then the iteration loop creates a **new engineering session** whose first prompt carries the findings, and supersedes the previous one."* The loop-back today is `continueIteration` (`index.js:3467`), which calls `invokeEngineeringAgent` at `:3613` with a payload carrying `qaFailures`, `testMetadata`, `feedback` and `conversationHistory` (`:3593-3611`).

**Revision 2 — this task shrinks. `dispatchEngineeringSession` already exists.** Revision 1 created it here, for one caller. Task 4.7b builds it in Phase 4 for three callers (`approvePlan`, `continueBuildIteration`, and this one) because deferring the build loop made Phase 4's exit criterion unachievable (R§1.5, R§7.4). So this task is now **the third caller of an existing function**, not an extraction: wire `continueIteration` to it, compose the prompt from the QA findings, and handle the double release. That is the whole task.

**Note `conversationHistory` is dead on this path.** `ssmBuildRunner` returns `conversationHistory: null` (`:978`), and a runtime session has no conversation to carry (A§7 open question 3 — *"dissolved"*). `loadConversationHistory` (`:3591`) still runs and its result still goes into the payload; on the runtime path, drop it.

**Tests first:**

```
describe('continueIteration on the runtime path')
  it('starts a new engineering session instead of invoking the QA-fix Lambda')
      → dispatchEngineeringSession called; invokeEngineeringAgent not called
  it('creates the engineering session before releasing the QA session')
      → createSession then releaseSession(qaSessionId)
      // Task 4.7's order.
  it('releases the previous engineering session too, since both want the branch')
      → releaseSession called for engineeringSessionId as well
      // This is the case A§3.2 says create-then-release deadlocks on.
  it('carries the findings into the first prompt, file and line scoped')
      → prompt contains 'a.js:3' and the finding text
  it('carries the review summary as feedback')
  it('does not carry a conversation history, because there is none to carry')
      → spec.prompt / spec has no conversationHistory
  it('checks out the cycle branch, not a commit')
      → spec.branch === cycle.branch; spec has no commitSha
  it('increments the iteration and respects maxIterations')
      → currentIteration 5 of 5 → FAILED with 'Maximum iterations reached',
        matching index.js:2107's message
  it('keeps the legacy path for non-runtime applications')
```

**Implementation.**
1. **`dispatchEngineeringSession` already exists** (Task 4.7b, in `backend/common/cycleEffects.js`, built on `supersedeSession` with `stage: 'engineering'`). Do not create a second one.
2. In `continueIteration`, call it. There is no Task 4.9 guard to remove here — Task 4.9 was narrowed to two sites and this is not one of them.
3. **Two sessions must be released, and both matter more than revision 2 said.** `supersedeSession` handles `engineeringSessionId`; add an explicit `releaseSession(cycle.qaSessionId)` **after** the create, alongside it, with Task 4.7's severity — retry once, then record `cycle.releaseLeak` and log at `error`.

   Revision 1 said the QA clone was "detached (harmless)" while the engineering session "holds the branch (fatal)"; revision 2 corrected both to "neither is urgent, they cost disk and a concurrency slot". **Revision 4 corrects the correction:** they cost disk, a concurrency slot, **and a live self-refreshing GitHub push credential** (§1.10, R14). This is the densest release point in the whole design — a QA-failure iteration releases two sessions at once, so a bug here leaks two credentials per iteration on a cycle that may iterate five times.
3. Keep the max-iterations check and its `FAILED` transition unchanged — it is one of Phase 4b's three generic `FAILED` producers.
4. Prompt composition is Part 1's (A§2.7's "QA feedback" row: *"The QA findings, file+line scoped, plus 'address these and call `reportCompletion` again'"*). Source it from `cycle.qaFailures` and `cycle.qaFeedback`, which Task 6.5 writes in exactly the format the existing prompt builder consumes.

**Acceptance.** `npx jest backend/__tests__/qaIterationLoop.test.js` green. On testing, drive one cycle through a QA failure into a second engineering session and confirm: the first engineering session and the QA session both 404 on `GET`, the second engineering session is active on the cycle's branch, and its first message contains the findings — **verified by** the three `GET`s and the session transcript.

**Commit.** `feat: start a new engineering session from a failed QA review`

### Task 6.7 — QA has no write tools

**Files** — runtime-side. This is Part 1's contract to build; this task is CC's verification of it.

**Why (A§4.3).** *"A reviewer that can edit the code under review will fix findings instead of reporting them, and the independence that justified running the review is gone."* The `qa` allowlist is `readFile`, `listFiles`, `searchCode`, `runCommand`, `updateTaskTracker`, `dispatchSubAgents`, and **`submitReview`** — not `editFile`, `writeFile`, or `reportCompletion`.

**Revision 3 — the terminal tool is `submitReview` again.** Revision 2 renamed it `submitResult`; revision 3 restores the three named tools (§0.2, D§ verdict). ~~`submitResult`~~ in the allowlist above is corrected accordingly. A§4.3's own allowlist table remains wrong on the other axis: it lists `reportCompletion` for the `qa` role, which would still be wrong, though **revision 8 narrows *why*.** `reportCompletion` does not itself commit and push — it verifies a commit and push the agent performed with `runCommand` (§0.2) — so the objection is not "it hands write access to the reviewer". It is that a QA session has nothing to verify: the empty-tree rule rejects every honest QA completion, so granting it produces a reviewer that cannot report at all. Part 1 caught this and corrects it (R§4(h)); follow Part 1.

**Revision 8 — this is now enforced at create, which converts most of this task from a verification into an assertion.** `profile.tools` must name **exactly one** terminal tool or the create is a `400` (`nevado-sherpa-tui apps/runtime/src/routes/sdlc.ts:337-353`). So a QA `profile` naming `submitReview` **and** `reportCompletion` does not produce an over-privileged session — it produces no session, with a message naming both tools it found. **That closes the specific hole A§4.3's table would have opened**, and it closes it at the cheapest possible moment.

Two things it does **not** close, which is why the negative test below stays:

- **It says nothing about `editFile` / `writeFile`.** Those are governed by session mode, and the QA session runs `mode: 'plan'` (`runtimesession.ModeForStage(StageQA)`, §7's header note), which is subtractive: plan mode *drops* `editFile` and `writeFile` while keeping `useMcpTool`, so the terminal tool still works. **Plan mode is the mechanism that delivers "no write tools"** — the one-terminal-tool rule is not a substitute for it.
- **The per-role allowlist A§4.3 describes is still not expressible** in the deployed `sherpa-core@4.0.1`: `buildToolConfig` is subtractive with no allowlist parameter. So the controls that actually exist are (1) the session mode and (2) a provider advertising exactly one terminal tool. Treat the "allowlist" above as a description of intent, not of a mechanism.

Revision 1 marked this "⚠ VERIFY BEFORE IMPLEMENTING" on the belief that the runtime repo was absent. It is not (§1.10), but the conclusion is unchanged: **the allowlist is enforced runtime-side, so this task is a verification, not an implementation.** It stays here because it is a Phase 6 exit criterion and the failure mode is silent.

**Not TDD — verified by** a live negative test on testing: give a QA session a task that invites an edit ("fix the null check you find"), and confirm the session reports that it cannot edit files rather than editing them. Concretely:

```
# after the QA session completes
gh pr view <n> --repo <org/repo> --json commits --jq '.commits | length'
```

must be unchanged from before the QA session ran, and the session's transcript must contain a refusal or a finding rather than an `editFile` tool call. Also confirm `git log origin/<cycle-branch>` shows no commit authored during the QA session's window.

**Acceptance.** The negative test above, plus `submitReview` being the only terminal tool the QA session was granted and `reportCompletion` being absent from its allowlist. If the session *can* edit, stop — Phase 6 does not ship, and it is a Part 1 defect.

**Commit.** None, unless the verification finds a gap. If it does, the fix is in the **runtime**, not the SDK — `nevado-sherpa-tui apps/runtime/src/sdlc/terminal-tools.ts` and `routes/sdlc.ts` — and belongs to Part 1. §1.14: nothing here routes through an SDK version.

### Phase 6 exit criteria

A§6 Phase 6's four, each with its check:

1. **QA findings cite file+line and survive the diff validation before posting**, matching today's behaviour (`qaAgent/orchestratorHandler.js:291-323`). **Verified by** `gh pr view <n> --json reviews` showing inline comments at the expected paths and lines, with no 422 in the orchestrator log, plus the unit tests in Tasks 6.1 and 6.2.
2. **QA reviews the engineering commit, not the branch tip** — *"verified by pushing to the branch mid-review and confirming the review still targets the original `commitSha`."* **Verified by** doing exactly that: `git push` an unrelated commit to the cycle branch while the QA session is active, then confirm the posted review's `commit_id` equals `cycle.pullRequest.headSha` and not the new tip. **This is also the outer net for §1.11b** — it is the one check that catches a silently-wrong checkout even if Task 6.3's verification and `DEP-P1-10`'s `400` both fail.
2b. **A well-formed but unfetchable `commitSha` fails the cycle before any review is posted** (§1.11b, Task 6.3's acceptance). Forced, not waited for.
3. **A `fail` verdict creates a new engineering session carrying the findings, and supersedes the previous one — create first, then release** (Task 4.7's order). Task 6.6's acceptance.
4. **QA's session has no write tools** — an attempt to edit a file under review fails rather than quietly fixing the finding. Task 6.7.

Plus:

5. `npm run test:backend` green.
6. A legacy (non-`buildMode: 'runtime'`) application still runs the synchronous QA path with identical behaviour — **verified by** one legacy cycle reaching `pending_approval` through QA, because Phase 7's deletes depend on nothing having silently broken here.
7. The GitHub review event change shipped as **its own commit** with its own PR description, or the decision to keep `COMMENT` is recorded and the log line at `qaAgent/orchestratorHandler.js:345` corrected instead (Task 6.2, R§3.2).
8. A malformed review payload lands in `qa_failed` naming the offending field, and does **not** DLQ or freeze the session's FIFO group (Tasks 6.1, 6.5).
9. **`resultSchema` catches a malformed review in-session.** Force one — a `findings` value that is a string, or a `line` that is a string — and confirm the agent receives a `toolError` and retries rather than the payload reaching CC. This is the criterion that proves the four silent corruptions D§2 enumerates are actually closed; without it they are merely documented.
10. **A `kind`/`stage` mismatch reaches the DLQ and fires the alarm** rather than being misrouted to the wrong validator. Force it by stamping `stage: 'planning'` on a QA dispatch in a throwaway branch — this is the check revision 2's generic tool could not perform (D§5(b)).
11. **One verdict decides both the cycle status and the PR review event.** Confirm on one pass and one fail cycle that `requirementsMet`, the cycle status and the GitHub review event agree — the live defect at `qaAgent:86-92` + `index.js:220` is closed and must stay closed (Task 6.1, D§5(f)).
12. **No SDK release and no runtime deploy were needed for this phase.**

---

## 7b. Phase 6b — the deploy-failure loop and the request-changes path

**New in revision 2.** Revision 1 floated a "Phase 6b" inside open decision OD-3 without specifying it, while Task 4.9 guarded four dispatch sites. R§7.4 splits the two loops: the CI-build loop moves into Phase 4 (Task 4.7b), and **the deploy loop gets a real phase here**. It is small, it is a hard gate on Phase 7 Task 7.7, and leaving it as a footnote in an open decision is how it gets forgotten.

**Scope: the two sites Task 4.9 guards.**

| Site | Enclosing function | What it does today |
|---|---|---|
| `index.js:3223` | `continueDeployIteration` (`:3133`) | A post-merge deploy failed. Classifies the failure, caps at three attempts, invokes engineering off the **default branch** (the cycle's branch is merged and gone), opens a **second PR** within the cycle (`:3252`), and enables auto-merge via GraphQL (`:3267`) with a direct-merge fallback (`:3277`) |
| `index.js:2132` | `approveCycle` (`:1973`), `request_changes` branch (`:2101`) | An operator asked for changes at the PR gate. Increments `currentIteration` (`:2120`), or fails with `'Maximum iterations reached'` (`:2107`) |

**What must not change.** A§3.7 is explicit and it is right: *"`continueDeployIteration` keeps its classification logic: the infra-versus-code check and the three-attempt cap stay in CC, because deciding whether a failure is worth an agent at all is orchestration, not agent work."* R§7.4 adds that these are *"the part you do not want to disturb under schedule pressure"*. **This phase changes how engineering is dispatched and nothing else.** No reclassification, no cap change, no PR-flow change.

Also unchanged: the second PR within one cycle (`:3252`) is a pre-existing stretch of the one-cycle-one-PR model, noted in A§3.4 and not fixed here; and the pre-merge CI-failure path sets `PR_CHECKS_FAILED` and parks for a human (`testResultPoller/index.js:662,748,763`) with no agent loop, which stays out of scope.

### Task 6b.1 — run the deploy-failure loop as a runtime session

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/__tests__/deployIterationDispatch.test.js` (NEW)

**Depends on** Task 4.7b (`dispatchEngineeringSession` — fourth caller), Task 4.1.

**Tests first:**

```
describe('continueDeployIteration on the runtime path')
  it('starts an engineering session instead of invoking the engineering Lambda')
      → buildMode 'runtime' → dispatchEngineeringSession called; invokeEngineeringAgent not
  it('branches off the default branch, not the cycle branch')
      → spec.baseBranch === application.github.defaultBranch
      → spec.branch is a NEW fix branch, not cycle.branch
      // A§3.7: by this point the cycle's branch is merged and gone. Reusing it
      // would clone a branch that no longer exists and fail at create time.
      → and it is STILL INSIDE cycle/** — assert /^cycle\/.+/
      // Revision 8, §1.13. "A new fix branch" is the only place in this document
      // that mints a branch name rather than deriving one, and it is therefore the
      // one place the wire constraint can be broken by accident. The provisioned CI
      // workflow triggers ONLY on push: branches: ['cycle/**'], so a tidier name
      // like fix/<cycleId> produces no workflow run at all: the deploy-failure fix
      // would push, nothing would build, and the cycle would sit in BUILDING until
      // the stall detector reaped it. Derive this name too — cycle/<n>-deployfix or
      // similar — so it stays inside the glob. Do not invent a new prefix.
  it('carries the deploy failure classification into the first prompt')
      → prompt contains the classification and the log tail continueDeployIteration
        already computes — the same material the legacy payload passed
  it('leaves the classification logic untouched')
      → the infra-versus-code decision runs before the dispatch branch and its
        result is unchanged by it
  it('still caps at three attempts, failing to FAILED as today')
      → deployRetryCount at the cap → FAILED, one of Phase 4b's generic producers
  it('still opens the second PR and still enables auto-merge')
      → the PR flow at :3252-3281 is reached unchanged on the runtime path
  it('keeps the legacy synchronous path for non-runtime applications')

describe('approveCycle request_changes on the runtime path')
  it('starts an engineering session carrying the operator feedback')
      → prompt contains the `feedback` body parameter
  it('increments currentIteration exactly once')
      → :2120's behaviour preserved
  it('still fails to FAILED with the max-iterations message at the cap')
      → :2107's 'Maximum iterations reached', a generic FAILED producer (Phase 4b)
  it('keeps the legacy synchronous path for non-runtime applications')
```

**Implementation.**
1. In `continueDeployIteration`, replace Task 4.9's guard with a `dispatchEngineeringSession` call **after** the classification and cap checks, so their behaviour is provably unchanged. The fix branch name is whatever the function already computes; pass it as `spec.branch` with `baseBranch` = the application default.
2. In `approveCycle`'s `request_changes` branch, the same, after the `currentIteration >= maxIterations` check at `:2106`.
3. Delete both Task 4.9 guards in the same commit. They are searchable by the `Phase 6b` comment Task 4.9 requires.
4. **`dispatchEngineeringSession` supersedes, so this phase releases too.** It inherits Task 4.7's create-then-release ordering and its release-failure severity — retry once, then `cycle.releaseLeak` plus an error-level line (R14). The deploy loop is the one place where the superseded session's repo may differ from the new one's (the cycle branch is merged and gone, so the fix branches off the default), which makes it the most likely place for `activeRepos` to hold two repos at once — see R15 on the shared `hosts.yml`.

**Acceptance.** `npx jest backend/__tests__/deployIterationDispatch.test.js backend/__tests__/runtimeDispatchCoverage.test.js` green — the coverage guard's *"guards the remaining two"* assertion must be updated to *"routes all five through `dispatchEngineeringSession`"* in this commit, or it fails. **Not TDD for the live half — verified by** forcing a deploy failure on a runtime cycle (break the deploy workflow on the merged commit) and confirming the cycle reaches `deploy_fix_engineering`, then a new engineering session, then a second PR with auto-merge enabled.

**Commit.** `feat: run the deploy-failure and request-changes loops as runtime sessions`

### Phase 6b exit criteria

1. `npm run test:backend` green.
2. `runtimeDispatchCoverage.test.js` asserts **five** `invokeEngineeringAgent` sites and that all five route through `dispatchEngineeringSession` on the runtime path. No `ENGINEERING_FAILED` guards remain; `grep -c 'Phase 6b' backend/lambda_handlers/agentDrivenOrchestrator/index.js` returns `0`.
3. One runtime cycle survives a real post-merge deploy failure, gets a fix from a session, and completes.
4. One runtime cycle survives an operator `request_changes` at the PR gate and reaches `pending_approval` again.
5. The three-attempt cap still fires — **verified by** a cycle whose deploy fails four times landing in `failed` with the existing message, not looping forever.

---

## 8. Phase 7 — deletes

**Gate: only after `buildMode: 'runtime'` is unconditional.** Concretely: every application record in every environment carries `agentConfig.buildMode === 'runtime'`, and the `buildMode !== 'runtime'` branches have not been exercised for a full release cycle.

**The test is the safety net.** For every delete, the sequence is: (1) prove the code is dead, (2) land a static guard that keeps it dead, (3) delete. Never the other way round — a delete that is merged before its guard can be silently re-introduced by the next person who needs a similar function.

### 8.1 What "prove it dead" means here

Three levels of proof, in decreasing strength. Say which one you used for each delete.

| Level | Method | Sufficient for |
|---|---|---|
| **P1 — static: no references** | `grep -rn '<symbol>' backend frontend scripts infrastructure .github` (excluding `node_modules`), zero non-documentation hits | Module-level deletes with no dynamic dispatch. Used for `codingAgentAdapter.js` |
| **P2 — static: only reachable through a branch that cannot be taken** | The single call site is inside `if (payload.buildMode === 'ssm')` or equivalent, and no record in any environment has that value | `ssmBuildRunner.js`, the SSM branch, `generateCodeWithClaude` and friends |
| **P3 — observational: no invocation in N days** | CloudWatch Logs Insights over the Lambda's log group, N ≥ 14 | `handleEngineeringTask`, which is reachable from API Gateway and whose caller is outside this repo |

For P2, the value-absence proof is a scan, not an assumption:

```
aws dynamodb scan --profile testing-tooling --table-name <table> \
  --filter-expression 'entityType = :t' \
  --expression-attribute-values '{":t":{"S":"application"}}' \
  --projection-expression 'SK, agentConfig' \
  --query 'Items[].{app:SK.S, mode:agentConfig.M.buildMode.S}'
```

Every row must read `runtime`. Run it against **every** environment, not just testing. Read `index.js:1068`'s `entityType` values and `Keys.application` in `dynamoHelpers.js` to get the filter right — the shape above is illustrative.

For P3:

```
aws logs start-query --profile testing-tooling \
  --log-group-name /aws/lambda/command-center-engineering-agent-<env> \
  --start-time $(( $(date +%s) - 14*86400 )) --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /handleEngineeringTask|AI Engineering Task/ | stats count()'
```

Expect `0`.

### Task 7.1 — delete `codingAgentAdapter.js`

**Proof level: P1, already complete.** Verified: `backend/common/codingAgentAdapter.js` is 279 lines with `module.exports` at `:265`, and a repo-wide grep for `codingAgentAdapter` returns **six hits, all documentation** — `sdlc-runtime-execution-architecture.md:1113,1117,1172,1218` and `docs/agent-architecture-plan.md:237,375`. Zero `require`, zero `import`.

**This is Part 1 Phase 0's item** (A§6 Phase 0). If Part 1 shipped it, skip. Recorded here because A§5.1 lists it under Phase 7's table and it is independent of every gate.

**Files**
- `backend/common/codingAgentAdapter.js` (DELETE)
- `docs/agent-architecture-plan.md` (MOD — the two references become stale)
- `backend/__tests__/deadCodeGuard.test.js` (NEW)

**Test first** — `backend/__tests__/deadCodeGuard.test.js` (NEW). This file is the Phase 7 net and every subsequent task extends it. Model it on `frontendContractGuard.test.js`'s file-reading approach.

```
describe('deleted modules stay deleted')
  it.each([['backend/common/codingAgentAdapter.js']])
    ('%s does not exist', p => expect(fs.existsSync(path.join(ROOT, p))).toBe(false))
  it.each([['codingAgentAdapter']])
    ('nothing requires %s', sym => {
       // walk backend/, frontend/src/, scripts/ for .js/.jsx
       // assert no file contains require('…<sym>') or from '…<sym>'
    })
```

Write the walker once, generically, taking a list of `{path, symbols}`. Every later task adds a row.

**Do not delete `agentInterface.js`.** Verified live: `backend/lambda_handlers/cursorAgent/devHandler.js:8` is `const { DevAgent } = require('../agentDrivenOrchestrator/agentInterface');` and `qaHandler.js:8` is the `QAAgent` equivalent, both verbatim, and `cursor_agent` is a deployed Lambda (`infrastructure/lambdas.tf:1474`, `handler = "devHandler.handler"`). A§5.1 is right to call this out; conflating the two cases is how a dead-code sweep takes out a live dependency. Add a positive assertion to the guard:

```
describe('live modules stay live')
  it('keeps agentInterface, which cursorAgent’s two handlers extend')
      → expect(fs.existsSync('backend/lambda_handlers/agentDrivenOrchestrator/agentInterface.js')).toBe(true)
      → expect(devHandler).toContain("require('../agentDrivenOrchestrator/agentInterface')")
```

**Acceptance.** `npx jest backend/__tests__/deadCodeGuard.test.js` green; `npm run test:backend` green; `npm run lint:backend` clean.

**Commit.** `fix: delete the unreferenced coding agent adapter`

### Task 7.2 — delete `ssmBuildRunner.js` and the SSM/EC2 dependencies

**Proof level: P2.**

The single caller is `engineeringAgent/orchestratorHandler.js:378-397`, inside `if (payload.buildMode === 'ssm')`. Nothing else requires it: the only `require` is `orchestratorHandler.js:19`. Run the application scan above and confirm no record carries `buildMode: 'ssm'`.

**Files**
- `backend/lambda_handlers/engineeringAgent/ssmBuildRunner.js` (DELETE — 1,002 lines)
- `backend/lambda_handlers/engineeringAgent/orchestratorHandler.js` (MOD — remove `:19` and `:378-397`)
- `backend/lambda_handlers/engineeringAgent/package.json` (MOD — drop `@aws-sdk/client-ssm` and `@aws-sdk/client-ec2`)
- `backend/__tests__/deadCodeGuard.test.js` (MOD)

What goes with it, all verified present: the 7 tool definitions (`:56-172`), `runOnInstance` and the SSM transport (`:181-214`, `SendCommandCommand` `:182`, `GetCommandInvocationCommand` `:197`), `handleToolCall` (`:221`), `discoverBuildInstance` (`:354`), `checkInstanceHealth` (`:370`), `setupWorkspace` (`:401`), `cleanupWorkspace` (`:425`), `gitPushWithRetry` (`:438`), the parallel/sequential executor (`:461-552`), `agenticBuildLoop` (`:555-835`), `handleSSMBuild` (`:840`), the truncation-recovery machinery, and the IMDSv2-hop-limit-as-sandbox comment at `:252`.

**Tests first** — extend `deadCodeGuard.test.js`:

```
describe('the SSM build path is gone')
  it('deletes ssmBuildRunner.js')
  it('leaves no require of it')
  it('leaves no ssm buildMode branch in the engineering orchestrator')
      → expect(src).not.toMatch(/buildMode === 'ssm'/)
  it('leaves no SSM or EC2 SDK import in the engineering agent')
      → expect(src).not.toContain('@aws-sdk/client-ssm')
      → expect(pkg.dependencies['@aws-sdk/client-ssm']).toBeUndefined()
      → expect(pkg.dependencies['@aws-sdk/client-ec2']).toBeUndefined()
  it('keeps SSM in the devops handlers, which are out of scope')
      → devopsAlertDelivery and devopsDiagnosisEngine still reference @aws-sdk/client-ssm
```

That last assertion is the one that stops an over-eager sweep: `ssmBuildRunner.js` is the only SDLC-path SSM caller, but `devopsAlertDelivery` and `devopsDiagnosisEngine` keep theirs and are explicitly out of scope (A§5.2). Verify their exact module paths before writing the assertion.

**Before deleting, harvest two things:**

1. ~~**`gitPushWithRetry` (`:438-459`)** retries three times, and A§2.6 says that behaviour is *"worth preserving"* in `reportCompletion` … copy the retry logic and its backoff into the Part 1 issue as reference, or the knowledge is lost with the file.~~

   **Revision 8 — this harvest is WITHDRAWN. Do not copy the retry logic anywhere; there is nothing left to preserve it in.** A§2.6's *"stage, commit and push with three retries"* was not built, and the reason is a hard constraint rather than a shortcut. The terminal tools reach the deployed `sherpa-core@4.0.1` through its `McpToolProvider`, invoked from `useMcpTool`, whose `TOOL_METADATA` is `{timeoutMs: 30_000, timeoutExempt: false}` — **and metadata is keyed on the DISPATCHER name, so an injected tool cannot be exempted.** The engine's `withTimeout` then resolves a `toolError` at 30 s **without cancelling the underlying promise**. A push to a slow remote would therefore complete while the agent was told it failed, and the agent would retry — producing either a duplicate commit or a cycle that **records failure over landed work.**

   So the division of labour inverted, and the result is better than what this task was trying to rescue: **the agent commits and pushes itself with `runCommand`** — `{timeoutMs: 0, timeoutExempt: true}`, already auto-approved on an unattended session — and **`reportCompletion` VERIFIES that it happened**, read-only (`nevado-sherpa-tui apps/runtime/src/sdlc/git-state.ts`, `terminal-tools.ts:306+`). Two properties follow. A push failure reaches the agent directly as `runCommand` stderr, **which is exactly what A§2.6's "the final failure must reach the agent, not be swallowed" asked for** and is strictly better than any retry loop. And the runtime never reports a sha it has not confirmed was pushed, where a retry loop would have trusted its own last word. Delete `gitPushWithRetry` with the rest of the file and record only this paragraph.
2. **`COMMAND_TIMEOUT_SECONDS = 300` (`:23`)** versus the runtime's 120-second cap (`node-platform-adapter.ts:186`). That is risk **R1** and it becomes unrecoverable once this file is gone: nobody will remember that the SSM path allowed 300s. Record the number in the Phase 7 PR description.

**Acceptance.** `npm run test:backend` green; `npm run lint:backend` clean; `npm run package-lambdas` succeeds and `infrastructure/engineering-agent.zip` shrinks; a deploy to testing followed by one runtime cycle completing normally.

**Commit.** `fix: delete the SSM build runner and its EC2 dependencies`

### Task 7.3 — collapse `engineeringAgent/orchestratorHandler.js`

**Proof level: P2** for each function, individually. This is the largest delete and the one most likely to take something live with it, so the task is a checklist, not a sweep.

**Files**
- `backend/lambda_handlers/engineeringAgent/orchestratorHandler.js` (MOD — 1,369 lines → ~250)
- `backend/__tests__/deadCodeGuard.test.js` (MOD)

**Deletes, each verified present at the line given:**

| Symbol | Lines | Why dead |
|---|---|---|
| `convertSystemBlocks` | `:36-46` | Bedrock message shaping; no Bedrock call remains |
| `convertMessages` | `:48-60` | same |
| `reportProgress` | `:89-117` | **Must not move** — see below |
| `selfVerifyCode` | `:132-212` | Bedrock self-verification. Its `MAX_RETRIES = 2` loop never iterates — every branch returns on attempt 1 |
| `applySlidingWindow` | `:734-742` | Context-window management; the session owns context now |
| `generateCodeWithClaudeStateful` | `:748-761` | JSON-blob generation |
| `generateCodeWithConversation` | `:767-804` | same |
| `generateCodeWithClaude` | `:808-901` | same, including the 190K-token pre-flight |
| `getApplicationRepo` | `:906-910` | A `TODO` stub returning `null`. **Check `:383`'s use first** — `repository \|\| await getApplicationRepo(applicationId)`, which always yields `repository` |
| `parseGitHubUrl` | `:915-927` | Only used by `createGitHubBranch` |
| `validateJavaScriptSyntax` | `:930-970` | Validated generated blobs |
| `validateCodeSyntax` | `:973-1012` | same |
| `createGitHubBranch` | `:1015-1146` | The agent commits and pushes now |
| `estimateTokens` / `truncateContent` | `:1197-1221` | Token budgeting |
| `loadApplicationCodebase` | `:1224-1369` | The agent reads the repo itself |

**Survives, transformed:**

| Symbol | Lines | Disposition |
|---|---|---|
| `exports.generatePlan` | `:217-355` | Becomes the plan session's instructions + first message (Part 1's composition). The **output-format block at `:298-321` is the plan payload schema's source of truth** — keep it, or move it into `backend/common/planSchema.js` (Task 5.1) as the single definition, but do not let it disappear. With an opaque `payload` there is no tool schema to fall back on (§1.7): if this block goes, CC's plan contract goes with it. `plan.generatedBy = 'nevado'` at `:346` is what Task 5.1 reproduces |
| `buildEngineeringPrompt` | `:550-731` | Becomes the engineering session's instructions + first message, **minus the token-budget machinery at `:559-585`** |
| `loadEngineeringDocumentation` | `:1149-1194` | Unchanged |
| The bootstrap assembly | `:367-377` | Unchanged; it authors the `instructions` field now (A§2.7) |
| `checkCancellation` | `:63-87` | **Moves** to `sdlcEventConsumer` |

**`reportProgress` deletes — it must not move.** A§5.2 is emphatic and correct. Verified: `reportProgress` (`:89-117`) writes `activities = list_append(…)` plus `currentStage` and `lastActivityAt` (`:101-106`); the consumer writes `progressLog` via `addProgressLog` (`progressLogger.js:43`). Moving it would leave two writers of two different fields with the UI reading one of them — which is exactly the split A§1.5 exists to close. **One writer, one field: `addProgressLog`.**

But note the consequence, and it is not in the architecture: `currentStage` and `lastActivityAt` are written **only** by `reportProgress`. Deleting it stops writing both.

- `lastActivityAt` — grep before deleting. If anything reads it, replace the read with `updatedAt`, which every write path maintains (`cycleStatuses.js:150-151`).
- `currentStage` — A§1.4 says `currentStage` *"appears once in the orchestrator, at `index.js:4241`, and only as an error-response key"*, and then flags *"But see §1.5 — the engineering agent writes a real `currentStage`."* Confirm the only reader is that error-response key before deleting, and if the frontend reads it anywhere, switch that read to `cycle.stage`.

**Tests first** — extend `deadCodeGuard.test.js`:

```
describe('the engineering orchestrator handler is collapsed')
  it.each([...the 15 deleted symbols...])('no longer defines %s', sym => {
      expect(src).not.toMatch(new RegExp(`function ${sym}\\b|${sym}\\s*=\\s*async`));
  })
  it.each([...])('is not referenced anywhere in backend/', …)
  it('still defines the survivors')
      → generatePlan, buildEngineeringPrompt, loadEngineeringDocumentation
  it('keeps the plan output schema, which sdlcContract.js is derived from')
      → expect(src).toContain('"filesToCreate"')
      → expect(src).toContain('"approach"')
  it('no longer writes the activities field, so there is one progress writer')
      // Writers, not readers. The three READERS were unified behind
      // selectActivities(cycle) in backend/common/cycleActivities.js by Part 1
      // Task 0.5; this assertion is about collapsing the two write paths onto
      // progressLogger.js. Do not delete the selector when the second writer goes —
      // it still merges historical `activities` on pre-cutover cycle records.
      → expect(src).not.toContain('activities = list_append')
      → and assert progressLogger.js is the only file in backend/ containing
        'progressLog = list_append'
  it('has no BedrockRuntimeClient left')
      → expect(src).not.toContain('BedrockRuntimeClient')
  it('does not touch the 17 out-of-scope Bedrock callers')
      → count files under backend/ containing 'BedrockRuntimeClient'
      → expect(count).toBe(<the number after this phase>)
      // Pin the number. A§ says 21 files hold one today and 17 are out of scope;
      // measure it before and after rather than trusting the count.
```

That last test is the guard against an over-broad Bedrock sweep. Measure the count on the current tree first (`grep -rln BedrockRuntimeClient backend | wc -l`) and pin the expected post-phase number.

**Acceptance.** `npm run test:backend` green; the file is ≤ 300 lines; `npm run lint:backend` clean with no unused imports; one runtime cycle completes on testing after deploy.

**Commit.** `fix: collapse the engineering orchestrator handler onto session-based execution`

### Task 7.4 — collapse `qaAgent/orchestratorHandler.js`

**Proof level: P2.** The caller is `invokeQAAgent` (`index.js:2516`), reached only from `routeAfterEngineering:237`, which Task 6.4 gates on `buildMode !== 'runtime'`.

**Files**
- `backend/lambda_handlers/qaAgent/orchestratorHandler.js` (MOD — 632 → ~280)
- `backend/__tests__/deadCodeGuard.test.js` (MOD)

**Deletes:** `analyzeCodeChanges` (`:161-355`, the Bedrock call and the `octokit.paginate(pulls.listFiles)` at `:188`), `parseDiffLines` (`:358-387`), `validateRequirements` (`:567-632`).

**Survives:** `getApplication` (`:390-400`), `updateCycleStatus` (`:403-450`), `updateCycleWithQAResult` (`:453-562`).

**Also fix on the way out, as A§5.2 notes:** the module-level `let agentContext = ''` (`:40`) is reassigned per invocation at `:49`, so a warm container leaks the previous cycle's bootstrap prompt if `:49` throws. Make it a local inside `handle` (`:45`).

**Note the extraction dependency.** Task 6.2 already moved `:288-348` (diff validation + `createReview`) into `backend/common/qaReview.js`. If Task 6.2 has not shipped, **stop** — deleting `analyzeCodeChanges` before that extraction loses the diff-line hunk parser, which is ~15 lines of correct-but-fiddly code nobody will want to rewrite.

**Tests first** — extend `deadCodeGuard.test.js`:

```
describe('the QA orchestrator handler is collapsed')
  it.each(['analyzeCodeChanges','parseDiffLines','validateRequirements'])
    ('no longer defines %s', …)
  it('keeps the cycle-result writers the legacy path still needs')
      → getApplication, updateCycleStatus, updateCycleWithQAResult present
  it('no longer holds agent context at module scope')
      → expect(src).not.toMatch(/^let agentContext/m)
  it('kept the diff-line validation, in backend/common/qaReview.js')
      → expect(fs.existsSync('backend/common/qaReview.js')).toBe(true)
      → expect(qaReviewSrc).toContain('@@ -')   // the hunk-header regex survived
  it('has no BedrockRuntimeClient left')
```

**Acceptance.** `npm run test:backend` green; the file is ≤ 300 lines; one runtime cycle completes through QA on testing.

**Commit.** `fix: collapse the QA orchestrator handler onto review-session results`

### Task 7.5 — delete `expandBusinessGoals` and its three sites

**Proof level: P1 after Phase 5.** Gated on **Tasks 5.5a and 5.5b** (§1.8): plan revision must already run through a session, **and** the chat-transcript / attached-document loaders must already have been harvested out (Task 5.5a) — otherwise this delete silently drops both from every plan.

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD — delete `:1412-1534`, the fallback at `:1220-1223`, and the revision call path)
- `backend/scripts/verify-converse.js` (MOD — delete case 11, `:1093-1141`)
- `backend/__tests__/deadCodeGuard.test.js` (MOD)
- `backend/__tests__/planFieldContract.test.js` (MOD)

**Three sites, not two** (§1.8). The third is `verify-converse.js:1093-1141`, which reproduces the prompt inline as verification case 11. It will not break — it will silently keep verifying a prompt for a function that no longer exists. Delete the case and renumber the ones after it, or the verification script lies.

**Harvest before deleting.** `expandBusinessGoals` (`:1412-1534`) is not only a prompt. `:1420-1449` loads the RAG context and the chat transcript from S3 (`chatTranscriptSessionId` → `chatData.history` → a `User:`/`AI:` transcript), and `:1449+` folds in `attachedDocuments`. Task 5.2's plan-session prompt needs all of that. **If it has not already been moved out, moving it is part of this task, not a follow-up** — otherwise deleting the function silently drops the chat transcript and attached documents from every plan.

Grep for the loading logic before you delete, and confirm Task 5.2's dispatch reads it from wherever it now lives.

**Tests first:**

```
describe('the fallback planner is gone')
  it('no longer defines expandBusinessGoals')
  it('has no caller anywhere in backend/')
  it('no longer verifies a prompt for a deleted function')
      → expect(verifyConverseSrc).not.toContain('expandBusinessGoals')
  it('still loads the chat transcript and attached documents for a plan session')
      → assert the S3 chat-transcript load survives somewhere reachable from the
        plan-session dispatch (name the module once Task 5.2 settles it)

describe('plan schema is now single-sourced')
  it('has exactly one plan schema in the repo')
      → the Task 5.4 divergence test flips: assert no second, narrower plan schema
        exists (no prompt mentioning requirements[] without filesToCreate)
```

That last one is the Task 5.4 test inverting, which is the point of having written it as a documented-divergence test rather than a skip.

**Acceptance.** `npm run test:backend` green; `node backend/scripts/verify-converse.js` runs to completion with the renumbered cases; one runtime cycle's plan contains content traceable to an attached document — **verified by** creating a cycle with a document attached and checking the plan references it.

**Commit.** `fix: delete the fallback planner now that planning is a session`

### Task 7.6 — delete `handleEngineeringTask` and the dead Bedrock agent import

**Proof level: P3 for `handleEngineeringTask`, P1 for the import.**

**Files**
- `backend/lambda_handlers/engineeringAgent/index.js` (MOD — 380 lines → ~132)
- `backend/__tests__/deadCodeGuard.test.js` (MOD)
- `infrastructure/` — check for an API Gateway route targeting this handler and remove it plus its `-target=` entry if present

**`handleEngineeringTask` (`:133-380`)** is the whole rest of the file after the router. Verified: it is reachable **only** via HTTP — `exports.handler` (`:74-131`) branches at `:79` on `!event.requestContext && !event.httpMethod` for direct/orchestrator invokes (routing to `orchestratorHandler.generatePlan` at `:84-87` or `orchestratorHandler.handle` at `:89`), and only an HTTP `POST` reaches it (`:106-108`). It creates a branch, commits `docs/ai-tasks/task-<ts>.md` from `placeholderContent` (`:236`, used `:283`), and opens a PR (`:290`) whose body says implementation *"will be completed by the engineering team"*. One of the five `pulls.create` sites and the only one nothing real depends on.

**Run the P3 query** (§8.1) before deleting, and also check whether an API Gateway route targets it:

```
grep -rn 'engineering_agent' infrastructure/*.tf | grep -i 'route\|integration'
```

If a route exists, it must be removed together with its `-target=` entry (§2.4), and `routeCoverageGuard.test.js` / `frontendContractGuard.test.js` must be re-run — the latter reconciles `apiService.js` against route keys and will fail if the frontend still calls it.

**The dead import (§1.9).** `:4` requires `@aws-sdk/client-bedrock-agent-runtime`, destructuring `BedrockAgentRuntimeClient` and `InvokeAgentCommand`. `InvokeAgentCommand` is never used. `bedrockClient` (`:12`), `BEDROCK_AGENT_ID` (`:16`) and `BEDROCK_AGENT_ALIAS_ID` (`:17`) are never used. **And `@aws-sdk/client-bedrock-agent-runtime` is not in `engineeringAgent/package.json`** — the handler's dependencies list `@aws-sdk/client-bedrock-runtime`, a different package. So either the common Lambda layer happens to supply it, or this Lambda throws on cold start for every HTTP request. Check before deleting, because the answer changes whether this is a cleanup or a latent outage:

```
aws lambda get-layer-version --profile testing-tooling \
  --layer-name <common-layer> --version-number <n> --query 'Content.Location'
# download and list node_modules/@aws-sdk/
```

Either way, deleting `:4`, `:12`, `:16`, `:17` is correct. `:5-10` are live requires (dynamodb, lib-dynamodb, githubAppClient, corsHelpers, responseHelpers) — **A§5.1's `:4-10` range is wrong; delete only `:4`.**

**Tests first:**

```
describe('the placeholder engineering task path is gone')
  it('no longer defines handleEngineeringTask')
  it('has no pulls.create left in the engineering agent')
      → expect(src).not.toContain('pulls.create')
  it('no longer imports the Bedrock agent runtime client')
      → expect(src).not.toContain('@aws-sdk/client-bedrock-agent-runtime')
  it('keeps the router that reaches the real orchestrator handler')
      → expect(src).toContain("require('./orchestratorHandler')")
      → expect(src).toMatch(/action === 'generatePlan'/)
  it('returns 405 rather than 500 for the removed POST')
      → assert the handler's method switch has an explicit branch

describe('PR creation sites are accounted for')
  it('has exactly four pulls.create sites left, all in the orchestrator')
      → count matches of /pulls\.create\(/ across backend/
      → expect(count).toBe(4)
      // Down from five. This is the guard for risk R7: adding a sixth, or
      // deleting one by accident, fails the build.
```

**Acceptance.** `npm run test:backend` green including `routeCoverageGuard` and `frontendContractGuard`; `npm run lint:backend` clean; `POST` to the removed route returns 405 (or 404 if the route was removed) — **verified by** `curl`.

**Commit.** `fix: delete the placeholder engineering task handler and its dead imports`

### Task 7.7 — retire the legacy dispatch branches

**Files**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD)
- `backend/__tests__/deadCodeGuard.test.js` (MOD)

The last step: `buildMode` stops being a branch and becomes an assumption.

**Deletes:**
1. The SQS engineering dispatch in `approvePlan` (`index.js:1653-1684`) — the `sqsMessage` construction and `SendMessageCommand`, now that the runtime branch is the only path. Keep `signalPmItem` (Task 4.8 moved it into the runtime branch).
2. `processSQSCycleExecution`'s engineering arm (`index.js:1688-1962`) — everything except the `action: 'generatePlan'` dispatch at `:1699-1703`, which Task 5.2 also retires. **If both go, the SQS event source mapping `aws_lambda_event_source_mapping.agent_orchestrator_sqs` (`infrastructure/lambdas.tf:1561-1566`) and `aws_sqs_queue.bedrock_queue` (`infrastructure/main.tf:435`) become dead infrastructure.** Do not delete them in the same commit — removing a queue while messages may be in flight loses work. Leave them, add a comment, and open a follow-up. Note `bedrock_queue` has **no DLQ** (`main.tf:435-447`), so any in-flight message lost here is lost silently.
3. `invokeEngineeringAgent` (`:2488-2511`) and `invokeQAAgent` (`:2516-2539`) once their five and one call sites are gone — and with them `ENGINEERING_AGENT_ARN` / `QA_AGENT_ARN` from the orchestrator's Terraform env block.
4. Task 4.9's loud-failure guards, now that the paths they guard are migrated or deleted.

**Do not delete:** the four surviving `pulls.create` sites (`:3252`, `:3687`, `:4313`, and `completeEngineering`'s, formerly `:1863`), the two 422-fallback blocks, or the three merge sites. A§3.4 is explicit: *"This design does not consolidate them… touching four PR-create paths while also moving execution is how a migration turns into a rewrite."* Risk **R7** stays open, guarded by Task 7.6's count test.

**Tests first** — extend `deadCodeGuard.test.js`:

```
describe('buildMode is no longer a branch')
  it('has no buildMode comparison left in the orchestrator')
      → expect(src).not.toMatch(/buildMode === 'runtime'/)
      → expect(src).not.toMatch(/buildMode === 'ssm'/)
  it('no longer invokes the engineering or QA Lambdas')
      → expect(src).not.toContain('invokeEngineeringAgent')
      → expect(src).not.toContain('invokeQAAgent')
  it('keeps the four PR-create sites and the three merge sites')
      → count /pulls\.create\(/ === 4 ; count /pulls\.merge\(/ === 2
        plus the GraphQL enablePullRequestAutoMerge
      // Verified counts today: create at :1863, :3252, :3687, :4313 (five with
      // engineeringAgent/index.js:290, which Task 7.6 removes); merge at :3277,
      // :4521, :4692.
  it('keeps both 422 fallbacks, which are idempotency not dead code')
      → expect(src).toMatch(/status === 422/g) with length >= 3
  it('leaves the cycle queue and its event source mapping in place for now')
      → expect(tf).toContain('aws_lambda_event_source_mapping.agent_orchestrator_sqs')
      // Deliberate: see the note above.
```

**Acceptance.** `npm run test:backend` green. Ten consecutive cycles complete end to end on testing with no `buildMode` field set on the application at all — which is the real proof that the branch is gone.

**Commit.** `fix: retire the legacy engineering and QA dispatch branches`

### Phase 7 exit criteria

1. `npm run test:backend` green, `npm run lint:backend` clean with zero unused imports.
2. `backend/__tests__/deadCodeGuard.test.js` covers every deleted symbol and every kept-alive counterpart (`agentInterface.js`, the devops SSM callers, the four PR-create sites, the 422 fallbacks).
3. Net deletion measured, not estimated: `git diff --stat main...HEAD -- backend/` on the Phase 7 branch. A§5.4 predicts ~2,920 lines; record the actual.
4. Ten consecutive cycles complete end to end on testing with no `buildMode` field on the application.
5. `grep -rln BedrockRuntimeClient backend | wc -l` equals the pinned post-phase number from Task 7.3, proving no out-of-scope Bedrock caller was touched.
6. `npm run package-lambdas` succeeds and the `engineering-agent` and `qa-agent` zips are measurably smaller.
7. Two follow-ups opened and referenced, not silently dropped: the dead `bedrock_queue` + event source mapping (Task 7.7), and PR-create consolidation (risk R7, A§3.4).

---

## 9. Cutover and rollback

### 9.1 The flag that already exists

`application.agentConfig.buildMode` — a free-form string on the application record, read at `index.js:1628` (`=== 'runtime'`) and forwarded to the engineering agent at `:1788`, `:2143`, `:3032`, `:3217`, `:3608` where `engineeringAgent/orchestratorHandler.js:378` branches on `=== 'ssm'`. Absent means the default SQS/Bedrock path.

No new flag is needed, and no new flag should be added. One switch, one record, per application.

`cycle.runtimeSessionStubbed` (`index.js:1638`) is the second signal: `true` means the session id is synthetic (`stub-<uuid>`, `runtimeDispatch.js:68`) and nothing will publish events for it. Every consumer of a session id in this spec must treat it as "not a real session" — Task 4.5's `decideStallAction` does, and any new reader must too.

### 9.2 Enabling

**Revision 8 — the field edit is the LAST of three steps, and on its own it does nothing but misroute cycles.** Task 4.10's precondition block is the authoritative sequence and is not repeated here. In short: **(a)** apply Terraform with `sdlc_ingress_mode = "token"`, which generates and writes the bearer-token secret with **no human step** and puts the priority-6 `/v1/sdlc/*` + `Authorization: Bearer *` rule on the existing :443 listener; **(b)** deploy the runtime artifact and set `SDLC_AUTH_TOKEN_ARN` on its systemd unit **in one restart window**, because cloud-init writes that unit on first boot only; **(c)** then, and only then, the edit below.

**Step (a) does not survive a merge today — command-center #858.** `sdlc_ingress_mode` is a `workflow_dispatch` input persisted nowhere, so every merge to `develop` re-applies with `none` and destroys the secret, its version and the ALB rule, with a green apply and no signal. Observed live on 2026-09-26. **Until #858 is fixed, enabling is not durable and this section cannot be followed as written.**

```
aws dynamodb update-item --profile testing-tooling \
  --table-name <table> \
  --key '{"PK":{"S":"APPLICATION"},"SK":{"S":"APP#<appId>"}}' \
  --update-expression 'SET agentConfig.buildMode = :m' \
  --condition-expression 'attribute_exists(agentConfig)' \
  --expression-attribute-values '{":m":{"S":"runtime"}}'
```

The `condition-expression` matters: without `agentConfig` present the `SET` on a nested path fails with a `ValidationException` that reads like a permissions error. Check the real key shape against `dynamoHelpers.js`'s `Keys.application` before running it.

Rollout order: one **human-approved** application in testing → the same application for ten cycles (Phase 4 exit criterion 1) → remaining testing applications → production applications one at a time.

**PM-linked applications go last, and not because of risk appetite.** §1.13: PM auto-approval still calls the JavaScript `approvePlan` and its `runtimeDispatch.js` stub, and the Go `planning.autoApprove` has no `cmd/` caller, so setting `buildMode` on a PM-linked application produces `stub-` session ids and no events — the flag is *read* and the runtime is never *reached*. That is a pre-existing cutover gap rather than a regression, but it means this field is only a real switch for the human-approval path. Wiring one of the two auto-approval paths is a prerequisite for a PM-linked application, not a rollout step.

### 9.3 Rolling back — and the point at which it stops being a field edit

**Revision 2 — revision 1's claim here was false and it was the most consequential error in the document.** It said: *"**Remove the field.** … No deploy, no code change, no restart"* and *"The irreversible line is Phase 7."*

That is true of the **dispatch decision** and false of the **code**. R§3.2 is right, and this section is rewritten to say so.

#### The five shared-code changes the flag does not bound

`buildMode` decides which *dispatch* a cycle takes. It bounds none of the following, every one of which is a hand refactor of live orchestration code that every non-runtime application also runs:

| # | Task | Change | Blast radius |
|---|---|---|---|
| 1 | **Part 1 Task 3.3** | `applyRuntimeEffect`, `routeAfterEngineering`, `writeStatus` and the `addProgressLog` wrapper move out of `index.js` into `backend/common/cycleEffects.js` | Every cycle, every application, every environment. **5 `routeAfterEngineering` call sites** (`index.js:1921`, `:2197`, `:2926`, `:3743`, and via `applyRuntimeEffect`) and **52 `addProgressLog` sites** |
| 2 | **Part 1 Task 3.4** | `writeStatus` gains `expectedFrom` | Shared, but **additive and correctly optional** — an omitted `expectedFrom` behaves exactly as today. The lowest-risk of the five |
| 3 | **Task 4.3** | ~100 lines of engineering-completion sequence move out of `processSQSCycleExecution` | The legacy SQS path, i.e. every application today |
| 4 | **Task 5.1** | `processPlanGeneration`'s two-step plan write is replaced by `persistPlan` | Every plan, every application |
| 5 | **Task 6.2** | QA diff validation + `pulls.createReview` move to `qaReview.js` — **and, in a separate commit, the review event changes from `COMMENT` to `REQUEST_CHANGES`** | Every QA review, every application. The event change is user-visible **immediately**: a failing review starts blocking merges on GitHub |

Each is labelled "behaviour must be identical" and each is a refactor by hand of a 4,984-line file.

#### So: where per-cycle rollback stops working

**The line is Part 1 Task 3.3**, not Phase 7.

- **Before Task 3.3 merges** — removing `agentConfig.buildMode` is a genuine, complete rollback. One `aws dynamodb update-item`, no deploy, and the next cycle is byte-identical to a pre-project cycle.
- **After Task 3.3 merges** — removing the field still redirects dispatch, but the shared code is already different for everyone. Rollback of *the refactor* is a code revert, and it gets progressively more expensive:

| Rolling back… | Costs |
|---|---|
| Task 3.3 alone, before Phase 4 | A clean revert of one commit (or three, if split per R§5.1). Part 1 calls it *"the riskiest revert in Part 1"* because it touches five live call paths |
| Task 3.3 after Phase 4 has merged | **Tasks 4.3 and 4.4 revert with it** — they add to the module Task 3.3 created. A partial revert leaves `cycleEffects.js` importable but incomplete |
| Task 5.1 or 6.2 | Independent of each other, but each is a revert of a live write path. Task 6.2's second commit (the review event) reverts trivially and independently — which is exactly why it is its own commit |
| Anything, after Phase 7 | **Not available.** The legacy branches are gone |

**Honest statement, for the Phase 4 PR description:** *the first unguarded blast-radius change is Part 1 Task 3.3, and from that point a rollback is a code revert of a refactor that later phases build on — not a DynamoDB edit. The point of no return is Phase 7.*

#### What can be made safer, and what cannot

Asked directly: can any of the five be flag-guarded or made behaviour-preserving instead?

| # | Can it be guarded? | Answer |
|---|---|---|
| 1 | **No, and a flag would make it worse.** | The whole point is that `sdlcEventConsumer` — a separate Lambda — must be able to `require` these functions, and `package_lambda_with_common` cannot reach a handler directory (Part 1 §0.2). A flag would mean *two copies*, which is the drift R§5.1 warns about for Part 1 Task 2.2. **Mitigate by splitting, not by flagging:** R§5.1's three PRs — (a) `writeStatus` + the `addProgressLog` wrapper + `extractOwnerRepo`, mechanical; (b) `applyRuntimeEffect` alone; (c) `routeAfterEngineering` last, on its own, with the full suite plus a live legacy cycle as the gate |
| 2 | **Already is.** | `expectedFrom` is optional and absent means today's behaviour. Nothing to do |
| 3 | **Yes, and it is free.** | `completeEngineering` is called from exactly two places. Keep the call in `processSQSCycleExecution` unconditional and **prove equivalence by test rather than by flag**: the existing suites (`stepAuditEvidence`, `cycleCheckpoints`, `planGenerationWrite`) exercise this region, and Task 4.3's boundary tests pin the first and last moved statements. A flag here would mean two copies of the 422 fallback, which is worse than the risk it removes |
| 4 | **Yes, and it is free.** | Same reasoning. `persistPlan` has two callers; keep `processPlanGeneration`'s unconditional and lean on `planGenerationWrite.test.js`, which already tests the two-step write |
| 5 | **Partly, and the part that matters is already split.** | The extraction cannot sensibly be flagged (two copies of the hunk parser). The **behaviour change** is separable and is now its own commit, revertible in one line without touching the extraction. That is the whole mitigation and it is sufficient |

**Conclusion: do not add flags. Add gates.** Two, and they are new in revision 2 because R§3.2 is right that Phase 6 had this check and Phases 3 and 4 — where it matters more — did not:

- **LEGACY GATE 1 — after Part 1 Task 3.3, before Phase 4 starts.** One **non-runtime** cycle end to end on testing: plan → approve → engineering → draft PR → CI → QA → approve → merge → deploy → complete. Not a unit test; a cycle.
- **LEGACY GATE 2 — after Task 4.3, before Phases 5 and 6 start.** The same, again.

Each costs under an hour and each is the only thing that catches a behaviour-preserving refactor that did not preserve behaviour. Record the cycle id in the PR.

#### Scope of removing the flag, per cycle state

Unchanged from revision 1 and still correct — this is the *dispatch* half, which does roll back cleanly:

| State | Effect of removing the flag |
|---|---|
| Cycle not yet started | Fully legacy. Clean. |
| Cycle in `PLANNING` with a live plan session | The session keeps running and publishing; the consumer keeps consuming, because it branches on the cycle record and not on the flag. Completes normally. **Do not also cancel the session** — that strands the cycle until `stuckCycleDetector` notices, and §1.10 shows `cancelSession` on a *queued* session does not even stop it. |
| Cycle at `PLANNING_REVIEW` | Approving it now takes the **legacy** engineering path, reading the same `expandedRequirements`. This works, and it is the single most useful property of the design. |
| Cycle in `ENGINEERING` or `QA_TESTING` with a live session | As above — let it finish. |
| Any state, after Phase 7 | No rollback. |

### 9.4 What "done" looks like for an end-to-end cycle on the runtime

One cycle, one application, no human intervention beyond the two gates:

1. `POST /v1/applications/{id}/cycles` → cycle at `planning`, `planSessionId` set to a non-`stub-` id.
2. Activity feed fills at roughly 10-second granularity while the plan session works.
3. `planning_review` with `expandedRequirements` populated on all eight schema fields; the plan-review dialog renders all four panels including Technical Approach.
4. **A HUMAN approves** — and revision 8 makes that word load-bearing rather than incidental: PM auto-approval still routes through the JavaScript gate and its stub dispatcher (§1.13), so this walkthrough describes the human path only. `planSessionId`'s session 404s; `engineeringSessionId` is a new, active session on the cycle's branch, and `cycle.branch` is **already set** — `cycle/<cycleNumber>`, derived from the sort key at approve-dispatch and recorded on the engineering claim write, not chosen by the agent and not seeded by hand (§1.13).
5. Activity feed fills. The cycle stays in `engineering` past fifteen minutes without being failed.
6. `reportCompletion` → `building`, with `cycle.githubUrl`, `cycle.buildHeadSha`, and a **draft PR** whose `headRefOid` equals the session's `commitSha` and whose body is the session's `summary` + `verification`. **`cycle.branch` is still the derived name at this point** — the reported branch matched it. If it does not match, that is #854 and the run is not "done": the cycle has silently moved to whatever the clone fell back to (§1.13). The agent pushed the commit itself with `runCommand`; `reportCompletion` verified it (§0.2).
7. `testResultPoller` picks it up on `buildHeadSha`. If CI fails, the cycle loops back through `continueBuildIteration` into a fresh engineering session (Task 4.7b) and tries again. When CI passes → `qa_testing`, with `qaSessionId` set to a session **pinned** to the engineering commit (a real `branch` ref, not detached — §1.11a), whose reported `commitSha` CC has verified against `cycle.pullRequest.headSha`; `engineeringSessionId`'s session 404s.
8. `submitReview` with a payload that passes both `resultSchema` in-session and CC's validator on receipt → a GitHub review with inline comments at real diff positions, and either `pending_approval` (pass) or `qa_failed` with `qaFailures` (fail). On fail, a new `engineeringSessionId` appears and the old one 404s.
9. A human approves. PR undrafted and merged → `deploying` → `post_deploy_qa` → `completed`.
10. Every session the cycle held 404s. The SDLC events DLQ is empty. No `SDLC_ILLEGAL_TRANSITION` in the logs.

That list is the Phase 6 demo and the Phase 7 gate.

---

## 10. Risks, stall handling, and the approval race

### 10.1 The risks that land in these phases

A§7's register, narrowed to what Phases 4–7 own, with the current state of each.

| Risk | Lands in | State and mitigation |
|---|---|---|
| **R0 — the activity feed goes blank at cutover** | Phase 4 | **Still open. Nothing is fixed — Phase 0 is reverted** (`cycleActivities.js` is gone, and `frontend/vite.config.js` carries only the pre-existing `cycleStatuses` alias). And the fix moved sides: **`backend/lambda_handlers/sherpa/tools/cc_cycles.py` must not be touched**, so a reader-side selector cannot reach every reader and is dead. **The fix is write-path convergence onto one canonical field** — Part 1's to specify. See §10.1a below for the field-choice derivation, which is a CC-side fact and therefore this document's |
| **R1 — `runCommand` caps at 120s where SSM allowed 300s** | Phase 4 | Verified live: `node-platform-adapter.ts:186` `timeoutMs: 120_000`, exit code 124 at `:188`; `ssmBuildRunner.js:23` `COMMAND_TIMEOUT_SECONDS = 300`. Deliberate deferral (A§7): *"deal with it if a cycle hits it."* The failure is unambiguous. **Record the 300 in the Phase 7 PR before `ssmBuildRunner.js` is deleted**, or the number is lost. Fix if it bites: a per-session `commandTimeoutMs`, runtime-side, a day's work |
| **R5b — the status write is not idempotent** | Phases 4, 5, 6 | Verified: `applyTransition` takes `strict = false` and `cycle.status = toStatus` runs unconditionally (`sdlcEngine.js:544`), deliberately, because 24 of 54 call sites target `FAILED` from error handlers. So a replayed `ENGINEERING → QA_TESTING` on a cycle already past QA logs a violation and moves it backwards. **The fix is `DEP-P1-7`'s conditional status write**, and *"it is easy to skip, because nothing fails visibly without it until a redrive happens in production."* Every completion task here (4.4, 5.1, 6.5) has an explicit replay test — those tests are the only thing that catches its absence |
| **R6 — two prompt-assembly paths during the transition** | Phases 4, 5, 6 | `AgentBootstrap` feeds both Bedrock system blocks and the `instructions` field. Both live through Phases 4–6. Mitigation A§ names: *"one composition function, two consumers."* Enforce it: Phase 7 Task 7.3 keeps `generatePlan`'s output schema and `buildEngineeringPrompt` as the single source, and Task 7.3's test asserts the schema block survives. Divergence is a silent quality regression, so it will not show up in any test — the only real mitigation is that Phase 7 deletes one consumer |
| **R7 — five PR-create sites, none consolidated** | Phases 4, 7 | Verified: `index.js:1863`, `:3252`, `:3687`, `:4313`, `engineeringAgent/index.js:290`. Task 7.6 deletes the fifth; the other four stay. Each must produce a body from `EngineeringCompletion.summary` rather than the old JSON blob — Task 4.4 does the first; `:3252`, `:3687` and `:4313` are on paths this spec does not migrate (**OD-3**) and keep their current bodies. *"Missing one leaves a path that produces an empty PR body and nobody notices for weeks."* Mitigation: Task 7.6's count test pins the number at four |
| **R4 — one runtime, ten slots, shared by humans and machines** | Phase 4 onward | **One arithmetic correction, and it is not the one revision 2 made.** The runtime's *default* is **5**, not 10 (`nevado-sherpa-tui apps/runtime/src/config.ts:49`); the deployed value is 10 via cloud-init, and **the live cap is exactly 10.** Revision 2 claimed 11 by reading the `>` gate as off-by-one. It is not: `worker.ts:127` reserves into `activeSessions` *before* the gate at `:134`, so the starter is already counted, and `:130-133` states the intent — *"hence `>` (not `>=`) to keep the original admission of exactly maxConcurrentSessions running at once."* The tenth starter sees `10 > 10` false and runs; the eleventh parks. A burst of cycles starves human sessions with no warning. v1 mitigation is `'queued'` surfaced to CC, which Task 4.5's `decideStallAction` handles (`touch`, logged, `reason: /queued/`). No reservation mechanism exists; the real fix is out of scope. Note also that releasing a session does **not** free its slot synchronously (`worker.ts:228-232`), so release-then-create would not have relieved pressure either — one of the reasons Task 4.7 reverses the ordering |
| **R13 — a bad `commitSha` produces a confident review of the wrong code** | Phase 6 | **New in revision 2, and it is the most dangerous risk in this document** (§1.11b). `checkoutCommit` returns `false` rather than throwing, and the session proceeds against the branch tip (`4.0.1 git-checkout.ts:56-66`, docblock `:49-54`). A check that silently checks the wrong thing is worse than a check that fails. Three layered mitigations: `DEP-P1-10`'s create-time `400`; Task 6.3's independent `commitSha` verification before any review is posted; and Phase 6 exit criterion 2's comparison of the posted review's `commit_id` against `cycle.pullRequest.headSha`. **Do not drop to one** — the first depends on Part 1 having built it, the second on the runtime reporting the commit, the third catches both escaping |
| **R14 — a skipped session release leaks a live push credential, and one is already leaked** | Phase 4 onward | **New in revision 4, and it is observed rather than predicted.** `release()` is what revokes the GitHub App installation token: `GitHubTokenManager` only calls `stopRefresh()` + `deleteHostsFile()` when `activeRepos.size` hits zero (`github-token-manager.ts:36-49`). A skipped release leaves a self-refreshing token on the box — and one is there now, re-minting every ~50 minutes for two days with `activeSessions: 0`. **There is no TTL sweep to catch it** (86 of 89 `/workspaces` dirs carry a `.git`, oldest `2026-06-08`), so A§3.8's backstop does not exist. Mitigations, all in Task 4.7: create-then-release so a create failure cannot strand a release; retry once; record `cycle.releaseLeak` and log at `error` so the leak is queryable; a CloudWatch metric so it is alarmable; and the `hosts.yml` baseline check before Phase 4 starts. **Do not restore revision 2's "logged warning"** — nothing else in the system will ever notice |
| **R15 — one `hosts.yml` serves every concurrent session, across repositories** | Phase 4 onward | Each `acquire` re-mints a token covering **all** of `activeRepos` (`github-token-manager.ts:63,66`) into a single file. So two concurrent SDLC sessions on different customer repositories each hold a credential valid for the other's repository — and the same is true alongside human sessions. Not introduced by this project (it is true of human sessions today) but this project is what makes concurrent multi-repo sessions routine, so the blast radius changes in practice even though the mechanism does not. Out of scope to fix here; **record it in the runbook alongside the blanket-auto-approve posture (R2)**, because the two compound: a session that can run arbitrary commands also holds push credentials for repositories it was never asked to touch |
| ~~**R5c — the `seq` counter is invented state with a silent failure mode**~~ | — | **Deleted in revision 2.** `seq` is no longer durable state: the FIFO `MessageDeduplicationId` is a per-envelope UUID (§0.2, R§2.4, R§7.2). A lost UUID produces a harmless duplicate the consumer is already idempotent against; a lost counter produced a reused id SQS silently dropped. The risk is gone, not mitigated |
| **R9 — `sherpaSessions.js` drift** | Phase 4 onward | `backend/common/sherpaSessions.js` documents that it is a *port* of the SDK's S3 key rules, not a dependency, and that drift silently empties the session list. Machine sessions increase the volume through that port. **Keep `backend/__tests__/sherpaSdkDriftGuard.test.js` green** — add it to every phase's acceptance |
| **R2 — the auto-approve allowlist reads like a security control** | All phases | `auto-approve.ts:4` calls itself a security boundary; `:38` splits on `&&`, `\|\|`, `;` but **not** a single `\|`, and `:33` is a `startsWith` test. Verified. So `curl … \| sh` is auto-approved. SDLC sessions run with blanket auto-approve (A§4.4 position (a)). **Say so in the runbook**, because someone will cite `autoApprovedCommands` as evidence the runtime is sandboxed. Containment is the host, not the string |
| **R11 / R12 — the shared bearer credential, and the identity it cannot carry** | Part 1 Phase 1 | Not this document's, but **restated in revision 8 because the mechanism is now known.** Authentication is one bearer token, generated by Terraform, read once at boot from `command-center/sdlc-auth-token/prod` and compared by the runtime itself with `timingSafeEqual` (`nevado-sherpa-tui apps/runtime/src/sdlc/auth-token.ts`). Two consequences. **(1)** It is a *shared* secret: every M2M caller presents the same bytes, so a leak is total and **rotation needs a runtime restart**, because the verifier is built at boot precisely so a request never pays for a secret read. Task 4.5 adds a second consumer of it (`stuckCycleDetector`), widening the blast radius by one Lambda. **(2)** Revision 2's *"forgeable client-id header"* is superseded — there is no client-id header. A shared token carries **no per-client identity at all**, which is why §0.2's ACL scopes on `application === 'sdlc'` and a `createdBy` predicate is unavailable rather than deferred (§1.14). Note both; do not design around either here |
| **R16 — the token ingress does not survive a merge, so any multi-cycle run on AWS is interrupted by an unrelated PR** | Phase 4 onward | **New in revision 8, and observed rather than predicted.** `sdlc_ingress_mode` is persisted nowhere, so every merge to `develop` re-applies with `none` and destroys the secret, its version and the priority-6 ALB rule — green apply, no signal (**command-center #858**; live on 2026-09-26, applied 13:12, destroyed 13:40 by an unrelated merge). The runtime fails closed correctly and withholds every `/v1/sdlc` route, so the symptom is a total dispatch outage that looks like a CC bug. **This is a hard blocker on Task 4.10 specifically**, because a ten-cycle run spans merges; it is not mitigable from inside this document. Compounded by #856, whose concurrency group guards the dispatcher rather than the apply, which is why three applies raced in the same window. `DEP-P1-18` |
| **R17 — a reported branch overwrites the requested one, so a clone fallback can put engineering work on `develop`** | Phase 4 onward | **New in revision 8.** `engcomplete.go:345` writes the reported `branchName` onto the cycle with no comparison against the derived one, and the runtime substitutes a clone candidate without logging that it differed (command-center **#854**, nevado-sherpa-tui **#103**). An agent that honestly reports the branch it actually landed on therefore sets `cycle["branch"] = "develop"`, after which the draft PR is `develop → develop`, `buildpoll` polls the wrong CI, and the **iteration loop appends to `develop`**. Newly reachable *and* newly checkable as of PR #857, since the requested branch is now recorded before the session starts. Mitigation is local and cheap — compare and refuse — and belongs in whichever task writes the completion (§1.13). **Rank this above R16 in severity:** R16 stops cycles, which is loud; R17 commits to the wrong branch, which is not |

### 10.1a R0, restated: three readers, three different fields, and only one possible canonical choice

The reader-side selector is dead because one of the three readers is off-limits. That constraint does not merely change the mechanism — **it determines which field the writers must converge onto**, and the reasoning is short enough to be checkable.

What the three readers read today, verified on `develop`:

| Reader | Reads | Sees a cycle that wrote only `activities` | Sees a cycle that wrote only `progressLog` |
|---|---|---|---|
| `frontend/src/components/CycleProgress.jsx:75` | `cycle?.activityLog \|\| cycle?.activities` — and `activityLog` exists nowhere in `backend/`, so effectively **`activities` only** | yes | **no** |
| `agentDrivenOrchestrator/progressTracker.js:190` | `cycle.activities \|\| cycle.progressLog \|\| []` — **both**, `activities` winning | yes | yes |
| `sherpa/tools/cc_cycles.py:99` | `item.get("progressLog", [])[-10:]` — **`progressLog` only** | **no** | yes |

And the writers: `engineeringAgent/orchestratorHandler.js:89-117` (`reportProgress`) and `progressTracker.js` write `activities`; `common/progressLogger.js:43` (`addProgressLog`) writes `progressLog`.

**Therefore the canonical field must be `progressLog`.** `cc_cycles.py` reads only `progressLog` and cannot be modified, so any convergence onto `activities` blinds it permanently. Converging onto `progressLog` leaves exactly one reader needing a change — `CycleProgress.jsx`, which is frontend and freely touchable. That is a one-way door and it is worth stating as a conclusion rather than leaving the field choice open.

Three consequences for this document:

1. **`applyRuntimeEffect`'s `activity` arm is already correct and needs no change.** It calls `addProgressLog` (`index.js:145`), which writes `progressLog` — the canonical field. So the runtime path this project introduces is canonical-by-construction, and **convergence can land before the cutover rather than with it.** It should: landing it first means Phase 4 inherits a feed that already works, instead of proving the fix and the cutover simultaneously. Tasks 4.4, 5.1 and 6.5 all route activity through this arm and none of them changes as a result.
2. **The legacy path is what moves.** Convergence means `reportProgress` and `progressTracker.js` stop writing `activities`. Under the Go mandate that is a port, not an edit (§13), and it overlaps Task 7.3 — see §13.3.
3. **Historical records stay split, and one reader already under-reports them.** A cycle written before convergence may have entries in either field or both. `cc_cycles.py` has therefore **always** been blind to legacy `activities`-only cycles — that is a pre-existing gap, not one convergence introduces, and it is unfixable without touching Python. What convergence fixes is every cycle from that point on. For the ones already written: terminal cycles do not matter (nobody reads a completed cycle's feed), and the frontend can merge both fields at read time because the frontend is touchable. **The case to actually handle is a cycle in flight across the convergence deploy**, whose entries land in `activities` before and `progressLog` after; a frontend read-time merge covers it, and that is the only place a merge is still needed.

### 10.2 Stall and timeout handling — the whole picture

There are **four** independent timeout mechanisms and it is worth having them in one table, because A§3.9 discusses one and the others quietly interact with it.

| Mechanism | Where | Threshold | What it does | After cutover |
|---|---|---|---|---|
| `stuckCycleDetector` | `stuckCycleDetector/index.js:23-29`, `rate(5 minutes)` (`lambdas.tf:2314`) | planning 5, engineering 15, qa_testing 15, iterating 20, qa_waiting_for_tests 30 (minutes) | Sets `status = 'failed'` raw, plus a `progressLog` entry and a Slack summary | **Task 4.5.** Thresholds become sweep intervals, not deadlines. Asks `GET /sessions/{id}` and acts on the answer |
| `cycleStatuses.isStalled` + `retryStalled` | `cycleStatuses.js:157-164`, `index.js:3931` | `STALL_THRESHOLD_MS = 20 min` (`:144`), over `UNWATCHED_STATUSES = [planning, engineering, iterating]` (`:126-130`) | Gates the operator's Retry button; 409 if not stalled | **Task 4.6.** Also consults the session; 409 while it is active |
| `testResultPoller` | `testResultPoller/index.js:38-48`, six `findCyclesByStatus` sites (`:348, 427, 656, 774, 1302, 1343`) | `DEPLOY_NO_RUN` 10 min, `DEPLOY_FIX` 20 min, `POST_DEPLOY_QA` 20 min | Resolves polled statuses from CI/deploy runs | **Unchanged.** It keys on `buildHeadSha` and `pollingStartedAt`, which `completeEngineering` sets (Task 4.3). The `BUILDING` hop is the one place a runtime cycle still depends on the 5-minute poller (A§3.3) |
| `ENGINEERING_DEADLINE_MS` | `index.js:64` = 13 min, forwarded as `deadlineMs` at all five dispatch sites | 13 min | The in-Lambda agent's own budget, inside a 900s Lambda | **Deleted with the legacy path** in Phase 7. Do not extend it to a runtime session; the point of the migration is that no invocation spans a session |

**The interaction that will bite.** A§2.5 deliberately publishes nothing when a 10-second window is empty, and `addProgressLog` is what refreshes `updatedAt` (`progressLogger.js:43`). A session thinking, or running one long command, writes nothing — so `updatedAt` goes stale on a perfectly healthy cycle, `isStalled` returns true, and the detector fires. **That is not a flaw to fix with heartbeats.** It is exactly why elapsed time must trigger a *question* rather than a verdict.

`cycleStatuses.js:140-142` currently claims the opposite — *"a working agent keeps the record fresh; on a real cycle the largest observed gap between writes was 64 seconds."* That observation is about the SSM agent. Task 4.5 corrects the comment; leaving it is how the next person concludes the detector is safe as-is.

**FIFO group-blocking adds a fifth clock.** A§2.4: a message the consumer cannot process blocks its whole `messageGroupId` — the session — for 3 receives × 180s visibility ≈ **9 minutes**, against the 20-minute stall threshold. It fits, but the margin is the design, not an accident. If Part 1 changes `visibility_timeout_seconds` or `maxReceiveCount`, re-check that budget. Add it to the Phase 4 review checklist.

**Which is why failure disposition is a question, not a case list.** Part 1 §A.4.6: **would a redrive after a code deploy succeed?** If yes — a drifted effect kind, an unparseable envelope, a stage no phase supports yet — the DLQ is right, and the ~9-minute group freeze is the price of a replayable fix. If no — a malformed terminal-tool payload, which fails identically on every redelivery — the message is consumed and the cycle takes a terminal attention status. Tasks 5.1, 6.1 and 6.5 all sit on the second side of that line. An implementer who memorises the cases instead of the question will collapse them the first time a new failure mode appears.

**Revision 8 — two shipped facts that sharpen Task 4.5's `decideStallAction`, both of which read as bugs if you meet them without warning.**

- **The sweep cannot fail a paused session, and the reason is a read-model gap rather than a policy choice (#816).** `stall.Decide` has no pause *reason* to judge, because `AgentSession` — the HTTP read model `GET /sessions/{id}` returns — carries no `reason` field. The reason only ever arrives on the SQS `sessionEnd` envelope. So the two signals are not interchangeable and a `touch` on a paused session is the correct conservative verdict from the HTTP side, not an oversight. Do not "fix" it by inferring a reason from the status; §1.14 is the general form of why that cannot work.
- **A runtime restart orphans in-flight sessions as `active`, and the touch re-arms itself (#838).** The worker's `activeSessions` map is in-memory, so a restart — which every runtime deploy is — leaves the *records* saying `active` with nothing running. The sweep asks, gets `active`, touches, and does so again five minutes later, indefinitely. That is the mechanism by which a cycle can sit past every threshold in the table above without ever being failed. Task 4.5 must not treat a `touch` verdict as self-limiting; bound the number of consecutive touches or the total elapsed time, or an orphan never surfaces.

### 10.3 The human approval gate's read-then-check race

A§1.2 and A§3.6 both flag it and neither fixes it. Stated precisely, from the code:

`approvePlan` (`index.js:1544`) and `approveCycle` (`index.js:1973`) read the cycle, check the status in application code (`approveCycle:1991-1993`: `const approvableStatuses = [PENDING_APPROVAL, QA_FAILED]; if (!approvableStatuses.includes(cycle.status)) return 400`), then write. There is no `ConditionExpression` on the status they checked. **Two concurrent approvals can both pass the check and both write.**

The pattern is available and used elsewhere in the same file: `processPlanGeneration` (`index.js:1258-1296`) carries `ConditionExpression: 'attribute_exists(PK) AND #status <> :cancelled'` on both of its writes, and `retryStalled` (`:3989+`) goes further with `'#status = :expected AND updatedAt = :lastSeen'`.

**Why it gets worse after cutover, and why that is not a reason to fix it here.**

Worse: a runtime cycle moves faster through the states either side of the gate, and `approvePlan` now starts a *session* rather than enqueueing a message — so a doubled approval starts **two engineering sessions on one branch**, which is precisely the collision A§3.8's release-on-supersede exists to prevent.

Not here: Task 4.2 already closes the operational consequence. Its guard is `if (cycle.engineeringSessionId) return` plus `ConditionExpression: 'attribute_not_exists(engineeringSessionId)'` on the persist. A doubled approval therefore produces one session; the second approval finds the attribute present (or loses the conditional write) and returns the already-dispatched response. The cycle record may record the second approval's `approvedBy`/`approvedAt`, which is an audit inaccuracy, not a correctness failure.

**Recommendation, recorded as a decision:** do not widen Phase 4 to fix `approvePlan`/`approveCycle`. Task 4.2 removes the sharp edge. Open a separate issue to add `ConditionExpression: '#status = :expected'` to both approval writes, referencing `processPlanGeneration:1258` as the pattern — A§7 R8 says both inherited defects *"are worth their own issues"* and that is the right call. **But add one test** to Task 4.2's suite making the property explicit, so the next person does not discover it by accident:

```
it('starts one engineering session even if the plan is approved twice concurrently')
    → two interleaved approvePlan calls, the second's conditional write rejected
    → createSession called exactly once
```

The second inherited defect A§3.6 names — the webhook-driven transition that 401s because generated repos post without the signature header (`common/webhookAuth.js`), leaving the 5-minute poller as the only working mechanism — is untouched by these phases and needs no mitigation here. It is already the reason `testResultPoller` is load-bearing, which Phase 4 exit criterion 4 depends on.

### 10.4 Open decisions

Resolved with evidence where possible; flagged as blocking where not, with a recommended default and the cost of guessing wrong.

**OD-1 — `ENGINEERING_FAILED`'s operator affordance. RESOLVED.** Task 4b.4 gives it `iterate` (re-enter engineering) and `cancel`, not `approve`. Evidence: `ATTENTION_STATUSES` carries a hard invariant that every member has an affordance (`cycleStatuses.js:62-66`, enforced at `cycleStatusInvariants.test.js:109-120`); `QA_FAILED` is the closest analogue and sits in `ITERABLE` + `CANCELLABLE`; `continueIteration` (`index.js:3467`) is the existing engineering re-entry. Cost of getting it wrong: an inescapable dead end, which the test catches at build time.

**OD-2 — the QA payload's unsourced fields. RESOLVED, and the resolution changed shape twice.** Revision 1 framed this as fields a protocol type lacked, and picked defaults. Since `submitReview.payload` is opaque and its schema is CC's own (§0.2, §1.7), most of the question dissolves: **the payload simply asks for the fields CC renders**, and `resultSchema` enforces that in-session. Implemented in Task 6.1:

- `findings[].suggestion` and `findings[].context` — **now asked for**, optional. The agent naturally produces them and the PR comment is better for them. Revision 1 left them undefined because a protocol type could not carry them.
- `overallQuality` — **derived** from `verdict`, not asked for. Asking the same judgement in two vocabularies invites the model to contradict itself, and its only consumer is the `approved` expression at `qaAgent/orchestratorHandler.js:286`.
- `testsGenerated: 0`, `testBranch: undefined`, `githubActionsRequired: false`. **`githubActionsRequired` must be falsy** or `routeAfterEngineering`'s `:264-267` parks the cycle at `QA_WAITING_FOR_TESTS` waiting for a CI run nobody triggered — a polled status with a 30-minute timeout. This is the single most valuable detail in Phase 6 and it is tested in Task 6.1.
- `recommendations: []` — left empty rather than duplicating findings. Both consumers guard on `?.length`.
- `severity` stays the four-level enum (`blocker|major|minor|nit`) collapsing to CC's three (`error|warning|info`). The four-level input is better: it lets CC set the verdict threshold independently of the emoji switch at `:312`.

**OD-3 — do the build-failure and deploy-failure loops move to sessions? RESOLVED, and split.**

Revision 1 flagged this BLOCKING with a recommended "Phase 6b" for both. R§1.5 and R§7.4 take the decision, and the reasoning is one revision 1 missed: **Task 4.9's guard turned the CI-build-failure loop into an unrecoverable `ENGINEERING_FAILED`, and an agent's first commit failing CI is the common case, so Phase 4's own "ten consecutive cycles" exit criterion became a matter of luck.**

- **`continueBuildIteration` (`index.js:3056`) moves in Phase 4** — Task 4.7b, which also pulls `dispatchEngineeringSession` forward and gives it three callers from the start, as revision 1's own reasoning recommended.
- **`continueDeployIteration` (`:3223`) and `approveCycle`'s `request_changes` (`:2132`) move in Phase 6b** — §7b, now a real phase with tasks and exit criteria rather than a footnote. Post-merge deploy failure is genuinely rarer, and its infra-versus-code classification and three-attempt cap are the part not to disturb under schedule pressure (A§3.7).
- **Task 4.9 guards two sites, not four**, and Phase 6b removes both guards.

**OD-4 — does the runtime fetch `baseBranch` for a `commitSha` checkout? RESOLVED — see §1.11.**

Revision 1 declared this BLOCKING and unverifiable because it could not find the runtime repo. It can now (§1.10), and the answer is three facts, all worse than the question:

- **No base-branch fetch exists**, and `commitSha`/`baseBranch` are unreachable from the create path at all (`nevado-sherpa-tui apps/runtime/src/agent/workspace.ts:33`). Part 1 Task 2.3 builds it.
- **The checkout is not detached** and `branch` is returned (`4.0.1 git-checkout.ts:29-32`) — revision 1's Task 6.3 acceptance check would have failed on correct behaviour.
- **`checkoutCommit` never throws**, so a bad sha silently serves the branch tip (`:56-66`). This is risk R13 and it needs three layered mitigations, not one.

And the diff form is **two-dot**, derived rather than guessed: `git diff A...B` needs a merge base, and `git-checkout.ts:33-35` states that ancestry reasoning is unsound over a `--depth 1` graft. `git diff origin/<baseBranch> HEAD`, shipped as a named constant rather than as prose (§1.11c, Task 6.3).

**OD-5 — does `planCreated` reach the publisher? NOT BLOCKING; already routed around.**

A§7 U2. Verified in `sherpa-sdk`: `onPlanCreated` exists in `EngineCallbacks` (`packages/core/src/types.ts:110`) and `EngineEvents` (`engine/engine-types.ts:32`), is fired by `executeWritePlan` (`workspace-tools.ts:219`), and `planCreated` is a real `ServerMessage` carrying `{sessionId, filePath, content}` (`messages.ts:22`). **Revision 8 — it does forward it.** The publisher maps `planCreated` to a `{type: 'planCreated', filePath}` envelope (`apps/runtime/src/sdlc/publisher.ts:433-436`), reaching it through the `Broadcaster` tap rather than a second transport (§0.2). Note it carries **`filePath` only, not `content`** — deliberately, and it is the right call: the plan's substance belongs on the `submitPlan` payload, not duplicated down the progress channel. **It does not matter:** Task 5.1 explicitly maps `planCreated` to a progress activity and **not** to `expandedRequirements` — the contract is the `submitPlan` payload (A§3.1: *"Do not ask the planner to write markdown and then parse it back"*). If `planCreated` never arrives, the only loss is one activity line.

**OD-6 — `usage` / `model` storage location. RESOLVED, low stakes.** A§2.3 says cumulative session totals ride on the completion payload and are *"written to the cycle record alongside `engineeringResult`."* `TokenUsage` is `{inputTokens, outputTokens}` only (`4.0.1 packages/protocol/src/session.ts:5-8`, verified) — **cache read/write counts do not survive the move**, which `bedrockConverse.js:168-193` captures today. That is a real regression and A§2.3 accepts it.

Decision, implemented in Tasks 4.4 / 5.1 / 6.5: store as `cycle.iterations[n].<step>Usage = {inputTokens, outputTokens, model}` next to the step's result, **not** on the cycle root and **not** inside `expandedRequirements` (which the plan-review dialog iterates). Cost of getting it wrong: cost attribution is coarser than intended, or the plan dialog renders a billing field. Both cheap to correct later.

### 10.5 What is still open after revision 2

Two items, both outside this document's authority, both worth naming so they are not mistaken for settled. **Revision 8 adds a third, and it is the only one of the three that blocks work today.**

**OQ-3 — `sdlc_ingress_mode` is not persisted, so the machine ingress does not survive a merge (command-center #858).** Unlike OQ-1 and OQ-2 this is not a posture question: it is a defect with a live consequence, it is inside Command Center's own authority, and **Task 4.10 cannot be completed while it stands** (R16, `DEP-P1-18`). Observed live 2026-09-26 — applied 13:12, destroyed 13:40 by an unrelated merge, reverting apply green. Fix it before scheduling any multi-cycle run on AWS.

**OQ-1 — does the runtime get a deploy pipeline and a session drain before Phase 1?** R§3.3 recommends it and this document endorses the recommendation, because the cost lands squarely on Phases 4–6: counting the runtime-side merges across both documents there are **eight to ten manual SSM deploys**, each of which stops the service, `rm -rf`s `/opt/sherpa`, and loses every in-flight session — including human ones — with an advisory-only pre-flight and no drain (`nevado-sherpa-tui docs/AWS_DEPLOYMENT.md:553-607`; `SIGTERM` does not drain agent sessions, `apps/runtime/src/index.ts:131-137`). Batching gets that to four or five windows.

The decision is the runtime owner's, not this document's. **What this document depends on:** if the answer is no, then the opaque-`payload` decision (§0.2, §1.7) moves from "decided" to "load-bearing", because it becomes the only mechanism that keeps plan-schema and QA-schema iteration out of outage windows. It is already decided, so this is not a blocker — but if anyone proposes reverting to typed `submitPlan`/`submitReview`, the pipeline question must be answered first.

**Revision 8 — OQ-1 is still open, and the shipped code raised its stakes rather than lowering them.** The terminal tools, `resultSchema` and its validator were built in `nevado-sherpa-tui apps/runtime/src/sdlc/` rather than in the SDK (§1.14), so the *only* delivery vehicle for a protocol change is a runtime deploy — the expensive one. There is no npm-release escape hatch, and there never was: §0.2's "publish 4.0.2" plan is struck. That makes the opaque payload the thing actually holding schema iteration out of outage windows, exactly as the paragraph above anticipated. Treat a proposal to type either payload as a proposal to put every future schema change inside an outage window that kills live human sessions.

**OQ-2 — the two inherited defects at the approval gate** (§10.3, A§7 R8). The read-then-check race in `approvePlan`/`approveCycle` and the signature-less webhook that 401s. Neither is worsened by these phases, Task 4.2 removes the sharp edge of the first, and both should have their own issues. Named here so that "we know about it" is written down rather than assumed.

---

## 11. Appendix — file-by-file change map

Every file Phases 4b–7 touch, with the task that touches it. Use this as the pre-PR checklist. **Revised for revision 2**: Phase 6b added, Task 5.5 split into 5.5a/5.5b, Task 6.4 retargeted off `index.js`, three new modules.

### Modified

| File | Tasks |
|---|---|
| `backend/common/cycleStatuses.js` | 4b.1, 4.5 |
| `backend/lambda_handlers/agentDrivenOrchestrator/sdlcEngine.js` | 4b.2, 5.5b |
| `backend/lambda_handlers/agentDrivenOrchestrator/sdlcStepGraph.js` | 4b.2 |
| `backend/lambda_handlers/agentDrivenOrchestrator/index.js` | 4.1, 4.2, 4.3, 4.6, 4.7, 4.7b, 4.8, 4.9, 5.1, 5.2, 5.4, 5.5a, 5.5b, 5.6, 6.3, 6.6, 6b.1, 7.5, 7.7 |
| `backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js` | 4.4, 5.1, 6.5 — **serialise these three; they touch the same `completion` dispatch** |
| `backend/lambda_handlers/agentDrivenOrchestrator/agentRouter.js` | 5.4 (deferral comment only) |
| `backend/lambda_handlers/stuckCycleDetector/index.js` | 4.0, 4.5 |
| `backend/lambda_handlers/stuckCycleDetector/package.json` | 4.0 |
| `backend/lambda_handlers/engineeringAgent/orchestratorHandler.js` | 7.2, 7.3 |
| `backend/lambda_handlers/engineeringAgent/index.js` | 7.6 |
| `backend/lambda_handlers/engineeringAgent/package.json` | 7.2 |
| `backend/lambda_handlers/qaAgent/orchestratorHandler.js` | 6.2 (two commits), 7.4 |
| `backend/scripts/verify-converse.js` | 7.5 |
| `frontend/src/components/cycles/cycleStatusConfig.js` | 4b.3 |
| `frontend/src/components/CycleProgress.jsx` | 4b.3 |
| `frontend/src/pages/ApplicationDetail.jsx` | 4b.4, 5.3 |
| `infrastructure/lambdas.tf` | 4.0, 4.5 |
| `.github/workflows/deploy-dev.yml` | 4.0 |
| `docs/agent-architecture-plan.md` | 7.1 |

**Owned by Part 1, extended here — coordinate before editing:**

| File | Tasks in this document |
|---|---|
| `backend/common/cycleEffects.js` | 4.3 (`completeEngineering`), 4.7 (`supersedeSession`, `releaseAllSessions`), 4.7b (`dispatchEngineeringSession`), 5.1 (`persistPlan`), 5.2 (`dispatchPlanSession`), 6.3 (`dispatchQaSession`), 6.4 (`routeAfterEngineering`'s ai-qa branch), 6.5 (`completeQa`) |
| `backend/common/runtimeClient.js` | Consumed only. Created by Part 1 Task 2.6 |

Note `backend/common/cycleEffects.js` is edited by eight tasks across three phases. It is the single busiest file in this document and the one most likely to produce merge conflicts; sequence the phases rather than parallelising within it.

### New

| File | Task |
|---|---|
| `backend/common/runtimeSessionFields.js` | 4.1 |
| `backend/common/stallDecision.js` | 4.5 |
| `backend/common/sdlcContract.js` | 5.1 creates it with `PLAN_PAYLOAD_SCHEMA` + `validatePlanPayload`; 6.1 adds `QA_PAYLOAD_SCHEMA` + `validateQaPayload`. **Revision 3 replaces revision 2's `planSchema.js`** — one file holding both schemas as data, read by `instructions`, by `resultSchema` at create, and by the consumer's validator, so the three cannot drift (D§6.3) |
| `backend/common/planContext.js` | 5.5a — **new in revision 2**, the harvest out of `expandBusinessGoals` |
| `backend/common/qaResultAdapter.js` | 6.1 — holds `toQaResult` and `QA_DIFF_COMMAND`. The QA **schema and validator** live in `sdlcContract.js`, not here |
| `backend/common/qaReview.js` | 6.2 |

### Deleted

| File | Task | Lines |
|---|---|---|
| `backend/common/codingAgentAdapter.js` | 7.1 | 279 |
| `backend/lambda_handlers/engineeringAgent/ssmBuildRunner.js` | 7.2 | 1,002 |

### Test files

| File | Tasks |
|---|---|
| `backend/__tests__/cycleStatusInvariants.test.js` (MOD) | 4b.1, 4b.3, 4b.4, 4b.5 |
| `backend/__tests__/sdlcEngine.test.js` (MOD) | 4b.2, 5.5b |
| `backend/__tests__/stalledCycleDetection.test.js` (MOD) | 4b.5, 4.0, 4.5 |
| `backend/__tests__/runtimeDispatch.test.js` (MOD) | 4.1, 4.8 |
| `backend/__tests__/runtimeEvents.test.js` (MOD) | 4.4, 5.1, 6.5 |
| `backend/__tests__/cycleEffects.test.js` (MOD — Part 1's) | 6.4 amends `it('routes an ai-qa cycle to qa_testing and invokes QA')` |
| `backend/__tests__/runtimeSessionFields.test.js` (NEW) | 4.1 |
| `backend/__tests__/runtimeSessionIdempotency.test.js` (NEW) | 4.2, §10.3 |
| `backend/__tests__/engineeringCompletion.test.js` (NEW) | 4.3 |
| `backend/__tests__/stallSessionCheck.test.js` (NEW) | 4.5, 4.6 |
| `backend/__tests__/sessionSupersede.test.js` (NEW) | 4.7, 5.6 |
| `backend/__tests__/engineeringDispatch.test.js` (NEW) | 4.7b |
| `backend/__tests__/runtimeDispatchCoverage.test.js` (NEW) | 4.9, amended by 6b.1 |
| `backend/__tests__/planCompletionWrite.test.js` (NEW) | 5.1 |
| `backend/__tests__/planSessionDispatch.test.js` (NEW) | 5.2 |
| `backend/__tests__/planFieldContract.test.js` (NEW) | 5.3, 5.4, 7.5 |
| `backend/__tests__/planContext.test.js` (NEW) | 5.5a |
| `backend/__tests__/planRevision.test.js` (NEW) | 5.5b |
| `backend/__tests__/qaResultAdapter.test.js` (NEW) | 6.1 |
| `backend/__tests__/qaReviewPost.test.js` (NEW) | 6.2 |
| `backend/__tests__/qaSessionDispatch.test.js` (NEW) | 6.3 |
| `backend/__tests__/routeAfterEngineeringQa.test.js` (NEW) | 6.4 |
| `backend/__tests__/qaCompletion.test.js` (NEW) | 6.5 |
| `backend/__tests__/qaIterationLoop.test.js` (NEW) | 6.6 |
| `backend/__tests__/deployIterationDispatch.test.js` (NEW) | 6b.1 |
| `backend/__tests__/deadCodeGuard.test.js` (NEW) | 7.1 – 7.7 |

Existing suites that must stay green throughout and are the regression net for the SQS path: `planGenerationWrite.test.js`, `stepAuditEvidence.test.js`, `cycleCheckpoints.test.js`, `cycleChildCleanup.test.js`, `postDeployQaPolicy.test.js`, `cycleActionWiring.test.js`, `routeCoverageGuard.test.js`, `frontendContractGuard.test.js`, `sherpaSdkDriftGuard.test.js` (risk R9), `applyRuntimeEffect.test.js`, `pollingIndex.test.js`, `progressLogger.test.js`.

---

## 12. Where this document disagrees with the review, and why

Both reviewers asked that disagreements be argued with evidence rather than silently complied with. Four were raised across two revisions; **two are closed**, one is uncontested, and **one is new in revision 3**. Everything else in both reviews is adopted.

| # | Subject | Status |
|---|---|---|
| 12.1 | The extraction end boundary | **Closed.** Part 1 confirmed `:1943`. The follow-on question — what the extracted function does with the exception — was this document's to decide and is decided (option (i), §1.9) |
| 12.2 | Malformed payloads must not throw | **Closed, and promoted.** Part 1 §A.4.6 turned it into a principle better than the case-by-case rule proposed here. D§4 and D§6.6 both endorse keeping it verbatim |
| 12.3 | `agentRouter` registration stays deferred | **Open, uncontested** |
| 12.4 | Whether the plan-review dialog is dead code | **Partly disputed — new in revision 3.** True of committed `HEAD`, false of the tree that will ship |

### 12.1 The extraction range is `:1839-1943`, not `:1839-1945`, and the reason is structural

R§4(i) records that Part 1 says `index.js:1839-1945` and revision 1 of this document said `:1841-1941`, and instructs "find by content". Adopted on the method, and the start (`:1839`) and the PR guard (`:1858`) are Part 1's and correct.

**But the end boundary cannot be `:1945`, and it is not an off-by-two.** `:1944` opens `catch (condErr) {` and `:1948` is `continue;`, which belongs to `for (const record of event.Records)` at `:1692`. Moving `:1944-1945` into a function and leaving the rest behind splits a `catch` block; moving the whole `catch` makes `continue` a syntax error. Verified by reading `:1930-1956` and the indentation shift at `:1935`.

**Resolution:** extract `:1839-1943`, let `ConditionalCheckFailedException` propagate, and leave both the `catch` and the loop control with the caller (§1.9, Task 4.3). Part 1 has confirmed the `:1943` boundary; the range half of this disagreement is closed.

Recorded because the first attempt at it was wrong in a way worth remembering: revision 2 paired `:1943` with a `'persisted' | 'cancelled'` return, which is incoherent — a function ending at the `PutCommand` has no `catch` and cannot report `'cancelled'`. Converting the exception into a return value is the nicer shape and is the right eventual destination, but it is a control-flow change and must not ride a commit whose contract is behaviour preservation. §1.9's table records both options and why (i) wins, so a later reader can take (ii) deliberately rather than rediscovering the tension.

### 12.2 A malformed plan or review payload must not throw — accepted and promoted

R§1.4 recommends making `applyRuntimeEffect`'s `default` arm throw, *"regardless"*, so the effect vocabulary fails closed. **Adopted, and it is the right call** (`DEP-P1-11`).

**But it must not be generalised to payload validation**, and with an opaque `payload` that distinction becomes load-bearing. Once the runtime validates nothing, CC is the only validator, and malformed payloads stop being a contract breach between two codebases and become an ordinary model failure — a thing that will happen. Throwing on one costs a DLQ entry, an alarm, and ~9 minutes of a frozen `messageGroupId` (3 receives × 180s visibility, A§2.4) to reach a conclusion the first attempt already had, because a malformed payload will not validate on retry either.

So Tasks 5.1, 6.1 and 6.5 return a `PLANNING_FAILED` / `QA_FAILED` status effect naming the offending field. Both are `ATTENTION_STATUSES` with working retries.

**Resolved, and no longer a disagreement.** Part 1 accepted the point and generalised it into a principle rather than a case list, which is better than what this document proposed: **Part 1 §A.4.6 — "would a redrive after a code deploy succeed?" DLQ if yes; terminal cycle status with the message consumed if no.** That is the general form, it covers failure modes neither document has thought of yet, and this document's wording is aligned to it throughout (§1.7, Task 6.1, §10.2). Retained here only as provenance: the distinction exists because the review's own §2.1 decision created the category, and an implementer who finds the two rules in tension should read §A.4.6 rather than re-deriving them.

### 12.3 `agentRouter` stays deferred

A§5.2 asks for `runtime` to be registered as a dev/qa backend in `agentRouter.js`. Revision 1 deferred it; the review does not comment either way. **Still deferred, and the evidence is stronger than revision 1 stated.**

`agentRouter.generatePlan` is the router's **only** use on the cycle path (`index.js:1215`). `invokeEngineeringAgent` (`:2488`) and `invokeQAAgent` (`:2516`) invoke Lambda ARNs directly and never consult it — A§5.2 notices this and then asks for the registration anyway. Task 5.2 branches on `buildMode` at all three `generatePlan` enqueue sites *before* the router is reached, and Phase 7 deletes both direct invokers. So registering `runtime` in `AGENT_ARNS` (`agentRouter.js:14-27`) would add a backend nothing dispatches to.

Recorded as a deliberate deferral with a comment in `agentRouter.js`, not a rejection. It becomes worth doing the moment `application.devAgent` is meant to select between `runtime` and `cursor` on the execution path — which is a product decision nobody has taken.

### 12.4 The plan-review dialog is dead at `HEAD` and alive in the working tree — and the distinction decides two tasks

D§5(a) reports that both specs derive the plan schema's required set from a dead dialog, that `approach` therefore *"has no live consumer at all"*, and that Task 5.3 as written *"lands in code no user reaches"* so the panel *"will still never render."* The coordinator asked me to verify rather than take it on trust. **Verified: right about committed `HEAD`, and wrong about the tree that will ship.** Both halves matter, so both are recorded.

**Where the adjudication is right.** At `git show HEAD:frontend/src/pages/ApplicationDetail.jsx` the dialog at `:1670-1794` is genuinely unreachable: `planningReviewData` is `useState(null)` (`:75`) and the only four references are `setPlanningReviewData(null)` (`:1670`, `:1675`, `:1740`, `:1746`). Nothing opens it. And `PlanReview.jsx` — the surface that does render, via `ActiveCycleHero.jsx:51-62` — **never reads `approach` or `technicalApproach`**, which I confirmed by grep. Revision 2's justification for requiring `approach` ("the plan-review dialog consumes it") was therefore half dead code, and striking it is correct.

**Where the method misses the live wiring.** A grep for `setPlanningReviewData(` cannot find the site that matters, because the setter is passed **by reference, without a paren**:

```
ApplicationDetail.jsx:618   onReviewPlan={setPlanningReviewData}        (uncommitted)
AttentionCard.jsx:25-26     case 'approve_plan': onReviewPlan?.(cycle);  (uncommitted)
                            case 'review_plan':  onReviewPlan?.(cycle);
cycleStatusConfig.js        PLANNING_REVIEW.primaryAction: 'review-plan' (uncommitted)
```

Those three edits are one coherent piece of uncommitted work — present in `git diff` at session start, and already relied on by Task 4b.4, which routes `ENGINEERING_FAILED`'s retry through the same `cyclePath` helper from the same diff. Their sole purpose is to make that dialog reachable from the attention card, handed a full cycle object. So on the tree that ships there are **two** live plan-review surfaces reached by different routes (§1.12).

**What I changed as a result, and what I kept.**

- **Kept `approach` in the required set — and the contingency revision 3 flagged here is now closed.** Revision 3 said that if the uncommitted wiring were abandoned, `approach` would drop to evidence-only and need reconsidering. It would not. There are **four readers independent of any plan UI**, and the strongest is `cursorAgent/devHandler.js:98` — `requirements: plan?.approach || task` — which makes the plan's `approach` the dev agent's **entire requirements input** under `devAgent: 'cursor'`, degrading silently to the one-line task when absent. Plus the unguarded interpolation at `devHandler.js:150`, `integrationAgent/orchestratorHandler.js:112`, and `recordCheckpoint`'s evidence. §1.12(1) has the table. **`approach` is required regardless of what happens to the frontend work.**
- **Kept Task 5.3, reframed.** It is not fixing code nobody reaches; it is completing a change already in flight. Its guard now also pins `onReviewPlan={setPlanningReviewData}`, because that line arrives as uncommitted work and a rebase could drop it with nothing failing.
- **Adopted the part of the finding that was pure gain, and corrected my own over-reach in it.** The primary surface reads `requirements[].dependencies` (`:125-178`) and `cycle.planVersion` (`:17`), neither of which any revision had accounted for; both are now in Task 5.1's schema and Phase 5's exit criteria. Revision 3 also listed the **`tasks` alias** (`:19`) as a third and said to tolerate and normalise it — **that was wrong, and it was mine, not the adjudication's.** `tasks` is the cursor backend's plan key (`cursorClient.js:286-294`, `generatedBy: 'cursor'`, and no `requirements`/`summary`/`risks`/`assumptions`/`estimatedEffort` at all), so it is a different producer's schema rather than a synonym of ours. `PLAN_PAYLOAD_SCHEMA` stays single-keyed; the frontend alias stays for cursor plans; a runtime session emitting `tasks` is a defect that should surface as a `toolError` rather than be normalised away. And D§5(b)'s crash — `tasks.reduce` on a truthy non-array, unguarded where everything around it is optional-chained — is the strongest single argument for `resultSchema` in either document, because it turns a broken approval gate into a `toolError` and a retry.

**Why the distinction is worth this much space.** The adjudication's recommended action was *"fix Task 0.6 and Task 5.3 to target `PlanReview.jsx`, **or delete the dead dialog**."* Deleting it would have removed the surface the in-flight work is building, and retargeting `approach` to `PlanReview.jsx` would have retargeted it at a component that does not read it. Both follow correctly from `HEAD` and are wrong against the working tree. The lesson for anyone auditing this codebase: **it has four uncommitted frontend files that change the reachability of the cycle UI, and `git show HEAD:` is not the tree that ships.**

---

## 13. The Go mandate, task by task

§0.15 has the mandate and the ground state. This section is the disposition for all 39 tasks. **Nothing in Phases 4b–7 should be picked up before reading the row for it.**

### 13.0 The rule as narrowed, and what a task looks like under it

Three narrowings arrived after §0.15 was written, and together they make the work smaller and more structured than "port everything":

1. **Only *new and updated* Lambdas must be Go.** A Lambda this project leaves alone stays JS. A Lambda this project only *deletes* is never updated, so it is never ported — it stays JS until it is removed. That takes `engineeringAgent`, `qaAgent` and `testResultPoller` off the port list entirely (§13.3).
2. **An updated Lambda is ported first, as its own step, then changed.** Two commits, both test-first:
   - **Step A — the port.** JS → Go, behaviour identical. The tests are **parity tests**: Go tests that encode what the JS does today and fail until the Go reproduces it. No new behaviour, no cleanup, no opportunistic fixes — a port commit that also changes behaviour cannot be reviewed, because every line differs anyway.
   - **Step B — the change.** The task as originally written, against the Go code, with a test that **fails for the right reason** first.
   This is the same discipline §9.3's legacy gates exist for, applied at commit granularity: prove the refactor preserved behaviour before layering a change on it.
3. **Where a Go counterpart already exists, extend it instead of porting.** Checked, and the answer for these phases is uniform: **nothing here extends an existing Go handler.** `backend/go/cmd` has 31 handlers and none is the cycle orchestrator, `stuckCycleDetector` or `testResultPoller`. The one cycle-adjacent name, `sdlc-manager`, is **preset CRUD** — `SDLCTemplate`, `Stage`, `Config`, `PlanningConfig`, `QAConfig` (`internal/sdlc/handler.go:74-142`), 828 lines including tests, no cycle record and no status transition — and SDLC templates are out of scope (A§). So §13.2's **GO-PORT** rows all mean port-then-change, not extend.

**What parity means, concretely, and how it is shown.** Parity is not "the tests pass"; the JS has very few tests over the code being ported. It is: *for the same cycle record and the same inputs, the Go produces the same DynamoDB writes, the same GitHub calls, and the same status transitions.* Three ways to show it, in descending order of strength — use the strongest available per task:

| Evidence | Where it applies | How |
|---|---|---|
| **Golden-record parity** | Pure functions over a cycle record: the transition table, the step graph, `stallDecision`, the payload validators and adapters, `completeEngineering`'s decision logic | Capture real cycle records from testing as JSON fixtures, run the JS through them once to record outputs, commit those as golden files, and assert the Go reproduces them. This is the strongest form and it is available for most of §13.2 |
| **Write-shape parity** | Anything issuing DynamoDB commands | Assert the Go emits the same `UpdateExpression` / `ConditionExpression` / attribute sets. `applyRuntimeEffect.test.js`'s `trackCommandArgs` harness exists precisely to capture these in JS — run it once to record the shapes, then assert them in Go |
| **Live A/B on testing** | Whole-handler ports where neither of the above is tractable | Run one legacy cycle end to end before the port and one after, and diff the resulting cycle records field by field. This is §9.3's legacy gate, reused as port evidence |

**The ordering consequence, which is new:** §9.3's legacy gates were placed after Part 1 Task 3.3 and Task 4.3 because those were unguarded shared-code refactors. Under port-then-change **every** step A is an unguarded refactor of live orchestration code, so the gate generalises: **a legacy cycle must pass after each port, before its step B lands.** That is more gates but each is cheaper, because a port with golden-record parity has already proven most of what the gate would catch.

**Port granularity is `DEP-P1-17`, not this document's.** Whether the orchestrator ports as a whole handler, as the cycle path only, or module by module — and whether a partial port leaves a workable JS/Go boundary inside one Lambda — is Part 1's call. §13.2's rows are written so they resolve either way: each says what its change is and what parity means for it, and the granularity decision only affects how many step As there are and where their seams fall.

Dispositions used below:

| Code | Meaning |
|---|---|
| **GO-PORT** | The task's work survives but the artifact does not. Touching this Lambda means porting it to Go, test-first in Go. The JS task body below describes *what* to build; the *where* and *how* come from Part 1's shape choice (`DEP-P1-16`) |
| **NON-LAMBDA** | Not Lambda code. Survives as written, in the language it is already in |
| **DISSOLVES** | Under G2/G3 the JS code this task modifies is superseded rather than edited. The requirement is absorbed into the Go implementation and the task stops existing as a discrete unit |
| **PORT-DELETES** | The code was going to be deleted by a Phase 7 task, but a port removes it as it goes, so Phase 7 no longer owns it (§13.3) |
| **NO-PORT** | A Lambda this project only deletes, never updates — so under §13.0 rule 1 it is never ported and stays JS until removed |
| **BLOCKED** | Cannot be respec'd until a blocking question is answered |

### 13.1 Phase 4b — stopped, and blocked on `BQ-GO-1`

Phase 4b was mid-implementation in JS on `feat/sdlc-phase0-cleanups` and is **stopped**. It is also the one phase that cannot simply be re-pointed at Go, because its subject is a constant with four consumers in three languages.

| Task | Subject | Disposition |
|---|---|---|
| **4b.1** | `cycleStatuses.js` — add `ENGINEERING_FAILED`, `ATTENTION_STATUSES` | **BLOCKED on `BQ-GO-1`.** The constant must be defined in Go (the transition table's home), exposed to the frontend as a JS artifact via `vite.config.js:19`, and readable by `cc_cycles.py`. Whether that is one generated artifact, a hand-maintained copy behind a real conformance test, or an API is `BQ-GO-1`. **The task is one line of work and three languages of plumbing; the plumbing is the task now.** Tests first, in Go, on the Go definition |
| **4b.2** | `sdlcEngine.js` `TRANSITIONS` + `sdlcStepGraph.js` `STEPS` | **GO-PORT.** These are pure functions over data — the single most portable code in the document, and the natural first slice of a Go cycle path. Port `isLegalTransition`, `explainTransition`, `pollingIndexKey`, `statusUpdateFragments`, `applyTransition` and the step graph to Go with table-driven tests. **The `applyTransition`-throws finding (§1.1) transfers intact**: a Go table must contain `ENGINEERING_FAILED` before anything writes it, or the Go equivalent of the unknown-status error fires. Keep §1.1's reasoning; it is language-independent |
| **4b.3** | `cycleStatusConfig.js`, `CycleProgress.jsx` | **NON-LAMBDA.** Frontend React. Survives exactly as written — it consumes whatever artifact `BQ-GO-1` produces |
| **4b.4** | `ApplicationDetail.jsx` + `index.js` affordance lists | **SPLIT.** The `handleRetryCycle` branch is **NON-LAMBDA** and survives. The `cancellableStatuses` / `continueIteration` gate edits in `index.js` are **GO-PORT** — and note the invariant they satisfy (`cycleStatusInvariants.test.js`'s "every attention status has an affordance") must be re-expressed as a Go test over the Go affordance lists, which is a genuine improvement: today that test mirrors the handler's inline arrays by hand and the mirror can drift |
| **4b.5** | Tests asserting the generic `FAILED` producers survive | **GO-PORT, and mostly rewritten.** Its three assertions are source greps over `index.js`. Against a Go implementation they become behavioural table-driven tests on the real transition function — which is strictly better, and is what §0.15's TDD rule requires. The fourth assertion was already cut in revision 2 |

**Phase 4b's gate on Phase 3 is unchanged in substance:** nothing may write `ENGINEERING_FAILED` before the transition table knows it. Only the table's language changes.

### 13.2 Phases 4, 5, 6, 6b — near-total GO-PORT

**Read every GO-PORT row below as a port-then-change pair** (§13.0): step A ports the enclosing unit with parity tests, step B makes the change the row describes. Rows that say **DISSOLVES** have no step B — the requirement is absorbed into step A's Go implementation, and the row says what must survive into it. Because all of these tasks update the same Lambda, **they share step As**: the orchestrator is ported once (at whatever granularity `DEP-P1-17` sets), not once per task. Counting ports per task would triple the work for no benefit.

**Which means the phase's commit sequence inverts.** Revision 5 and earlier had ~30 change commits. Now it is: a small number of port commits with parity evidence, then the change commits — and the port commits are on the critical path for *every* task in the phase, so they land first and together rather than being interleaved. Sequence Phase 4 as: port the cycle path (step As, `DEP-P1-17`'s granularity) → legacy gate → then Tasks 4.1, 4.2, 4.4, 4.5, 4.6, 4.7, 4.7b as Go changes.

| Task | Subject | Disposition |
|---|---|---|
| **4.0** | `deploy-dev.yml` target list, `lambdas.tf`, `stuckCycleDetector/index.js`, its `package.json` | **SPLIT.** The Terraform and workflow work is **NON-LAMBDA** and survives — **but its field values change**: a Go `stuckCycleDetector` is `handler = "bootstrap"`, `runtime = "provided.al2023"`, moves from `maybe_package` to `GO_LAMBDA_NAMES`/`GO_LAMBDA_CMDS` (`package-lambdas.sh:420-421`), and drops its `package.json`. The finding that it is **in no `-target=` list** stands and is the reason this task exists. The handler edits are **GO-PORT** — and `stuckCycleDetector` is Wave 3 in the migration plan (`:151`, 291 lines, *"DynamoDB scan + status logic"*), so this is a scheduled port being pulled forward |
| **4.1** | `runtimeSessionFields.js` (NEW) + `index.js` | **GO-PORT.** A three-entry map and a lookup — trivial in Go, and it was only a separate module because `backend/common/` is how JS Lambdas share code. In Go it is a few lines in the orchestrator package |
| **4.2** | Session-create idempotency guard | **GO-PORT.** The finding survives unchanged and matters more, not less: two operations, an SQS-triggered retry, and a conditional write asserting **both** `engineeringSessionId` and `runtimeSessionId` absent (§4(j)'s window). Go's DynamoDB SDK expresses `ConditionExpression` the same way |
| **4.3** | Extract the engineering-completion sequence into `cycleEffects.js` | **DISSOLVES under G2/G3.** This task exists only because two *JS* callers must share one *JS* implementation. In Go the completion sequence is written once, in the orchestrator package, and there is no extraction. **What must not dissolve with it is the content** — §1.9's seven-step anatomy, the `:1858` PR guard, the 422 fallback, the two `ConditionExpression`s and the `buildHeadSha` write are the specification for the Go implementation. **Carry §1.9 forward as the Go acceptance criteria.** The `continue`-is-loop-control finding (§12.1) becomes moot: Go has no equivalent hazard, and the `'persisted' \| 'cancelled'` question answers itself with a normal Go error return |
| **4.4** | Consumer `complete` arm → completion sequence | **GO-PORT**, and the `default`-arm-must-throw requirement (`DEP-P1-11`) becomes "return an error", which Go makes harder to ignore than `console.warn` did |
| **4.5** | `stallDecision.js` (NEW) + `stuckCycleDetector` | **GO-PORT.** The pure decision function is table-driven-test-shaped and ports cleanly. The §1.5b finding — that the detector writes `status` raw, bypassing the shared write path, with no `ConditionExpression` and no GSI4 cleanup — **is the argument for porting it rather than patching it**, and a Go port that reuses `internal/dynamo` cannot casually reproduce it |
| **4.6** | `retryStalled` consults the session | **GO-PORT** |
| **4.7** | `supersedeSession`, `releaseAllSessions` | **GO-PORT.** All the substance survives: create-then-release (§1.10's clone-only verification), the credential-leak severity (R14), retry-once-then-record-`cycle.releaseLeak`, and release-not-cancel (`worker.ts:41,231,235-248`). None of it is language-dependent |
| **4.7b** | `dispatchEngineeringSession` + the CI-build-failure loop | **GO-PORT.** R§1.5's sequencing finding stands: without the build loop, Phase 4's "ten consecutive cycles" is unachievable. Three callers from the start still applies |
| **4.8** | `signalPmItem` parity on the runtime branch | **DISSOLVES under G3** — there is no "runtime branch" to bring to parity once the cycle path is Go; the Go implementation calls it once. Under G2 it is **GO-PORT**. Either way its **source-grep test is an illegitimate substitute** (§0.15) and becomes a behavioural test |
| **4.9** | Loud failure on the two un-migrated dispatch sites | **DISSOLVES under G3.** It is scaffolding for a half-migrated JS file. A Go cycle path has no `invokeEngineeringAgent` fall-throughs to guard. The **dispatch-site count guard survives as a deployment guard** while any JS dispatch site remains |
| **4.10** | End-to-end verification on testing | **NON-LAMBDA.** Language-agnostic: statuses, PRs, SHAs, feeds, DLQ depth, `hosts.yml`. Survives verbatim and becomes more important, because a port has more ways to differ from the original than an edit does |
| **5.1** | Plan payload schema + validator + persistence | **GO-PORT.** `sdlcContract.js` becomes a Go package holding `PLAN_PAYLOAD_SCHEMA` as data. The one-object-three-uses property (prompt contract, `resultSchema`, consumer validator) is preserved and is easier in Go, since the same struct/literal serialises to all three. §1.12's required-set derivation and the `approach` four-reader justification carry over unchanged. **`DEP-P1-13`'s `resultSchema` is JSON on the wire, so it is language-neutral** |
| **5.2** | Plan-session dispatch at three enqueue sites | **GO-PORT.** The three verified sites (`:1131-1144`, `:3872-3883`, `:4027-4036`) are the specification for what the Go path must cover |
| **5.3** | `technicalApproach` → `approach` | **NON-LAMBDA.** Frontend. Survives — and §1.12/§12.4's dead-vs-live dialog finding is unaffected by the mandate |
| **5.4** | End the dual-planner divergence | **GO-PORT**, and simplified: a Go planning path has no `expandBusinessGoals` fallback to gate, so the divergence ends by construction rather than by a branch |
| **5.5a** | Harvest the context loaders | **GO-PORT.** Still must happen before the `expandBusinessGoals` delete, or the chat transcript and attached documents are silently dropped from every plan. The hazard is unchanged; the language is not |
| **5.5b** | Plan revision as a session | **GO-PORT.** The sync→async conversion (§1.8) and the new `PLANNING_REVIEW → PLANNING` edge survive; the edge now lands in the Go transition table from 4b.2 |
| **5.6** | One path across the approval gate | **GO-PORT**, largely assertion-only. Its `runtimeDispatch.js:42-44` cleanup is already done (`5221cbb1`) |
| **6.1** | QA payload schema, validator, adapter | **GO-PORT.** The field-mapping table, the `githubActionsRequired`-must-be-falsy trap and the `requirementsMet` safety fix (D§5(f)) are all language-independent and all survive |
| **6.2** | Extract the QA review post | **GO-PORT**, and the extraction rationale weakens the same way 4.3's does — in Go it is written once. **The two-commit split for `COMMENT` → `REQUEST_CHANGES` still stands**, because that is a user-visible behaviour change and must not ride a port |
| **6.3** | QA session dispatch | **GO-PORT.** §1.11's findings — not-detached, `branch` returned, bad sha silently serves the branch tip, two-dot diff — are runtime-side facts and unaffected |
| **6.4** | `routeAfterEngineering`'s ai-qa branch async | **DISSOLVES under G3.** §1.2's finding — that this branch drives QA synchronously across 65 inline lines — becomes a specification for the Go implementation rather than a refactor of JS. Revision 3 retargeted this task to `cycleEffects.js`; under the mandate that target may not exist |
| **6.5** | Resolve the QA step from the review payload | **GO-PORT** |
| **6.6** | Failed review → new engineering session | **GO-PORT.** The double release and R14's credential severity survive |
| **6.7** | QA has no write tools | **NON-LAMBDA.** A verification of runtime-side (`nevado-sherpa-tui`) behaviour. Unaffected |
| **6b.1** | Deploy-failure loop + `request_changes` | **GO-PORT.** The instruction to leave the classification logic and three-attempt cap alone (A§3.7) becomes "reproduce them faithfully in Go", which is harder and worth calling out |

### 13.3 Phase 7 — what the port deletes, and what Phase 7 still deletes

Two forces now act on Phase 7 in opposite directions, and the net is that it **shrinks in one place and grows in another**.

**It shrinks because a port deletes as it goes.** If step A ports the cycle path to Go, the JS it replaces is removed *in that commit* — that is what a port is. So anything inside `agentDrivenOrchestrator/index.js` that Phase 7 was going to remove is **deleted by the port, not by Phase 7**, and specifying it in both places would have the same code deleted twice:

| Was Phase 7's | Now deleted by | Note |
|---|---|---|
| Task 7.7's SQS engineering dispatch (`index.js:1653-1684`) and `processSQSCycleExecution`'s engineering arm (`:1688-1962`) | **the port** | Both are cycle-path JS. A Go cycle path does not carry them forward |
| Task 7.7's `invokeEngineeringAgent` (`:2488-2511`) and `invokeQAAgent` (`:2516-2539`) | **the port** | Their Go equivalents either exist or the paths that called them dissolved |
| Task 7.7's `buildMode` branches | **the port** | There is no JS branch to retire once the JS path is gone |
| Task 7.5's `expandBusinessGoals` (`:1412-1534`) | **the port** | Cycle-path JS. But §13.2's 5.5a harvest still has to happen **before** the port, or the chat-transcript and attached-document loaders go with it |
| Task 4.9's loud-failure guards | **never written** | They were scaffolding for a half-migrated JS file |

**What Phase 7 still owns, because none of it is cycle-path JS the port touches:**

| Task | Still Phase 7's, because |
|---|---|
| **7.2** `ssmBuildRunner.js` | Lives in `engineeringAgent`, a Lambda the port does not touch. Still a delete, still needs its two harvests (`gitPushWithRetry`'s retry logic, and `COMMAND_TIMEOUT_SECONDS = 300` recorded against the runtime's 120s cap — R1) |
| **7.3 / 7.4** the two agent handlers | Not updated by this project, so not ported (§13.0 rule 1). They stay JS until deleted |
| **7.5**'s `verify-converse.js:1093-1141` | A Node script, not a Lambda. Case 11 still needs removing |
| **7.6** `handleEngineeringTask` | In `engineeringAgent/index.js`, untouched by the port |
| **7.7**'s Terraform | `aws_lambda_function.agent_orchestrator` + its `maybe_package` entry + the `bedrock_queue` question. Infrastructure, not code, and it retires when the JS handler does |
| **7.1** `codingAgentAdapter.js` | A shared module with zero references. Free delete, any time |

**So the honest restatement: Phase 7 is no longer "retire the legacy cycle path" — the port does that. Phase 7 is "remove the JS Lambdas the runtime made redundant."** That is a cleaner boundary than any previous revision had, and it removes the double-delete hazard the coordinator flagged.

Verified for completeness: **none** of `engineeringAgent`, `qaAgent`, `cursorAgent` or `integrationAgent` has a Go counterpart in `backend/go/cmd`, so nothing is already-dead by virtue of a Go replacement, and Phase 7's "prove it dead" levels (§8.1) are unchanged for all of them.

| Task | Disposition |
|---|---|
| **7.1** `codingAgentAdapter.js` | **NON-LAMBDA, and back to not-done.** It landed as `caebf0d1` and was reverted with the rest of Phase 0, so `backend/common/codingAgentAdapter.js` exists again with zero non-documentation references. A shared module, not a Lambda — no port, still a free delete at any time |
| **7.2** `ssmBuildRunner.js` | **Survives as a delete**, unchanged — but **revision 8 drops one of its two harvests.** `gitPushWithRetry`'s retry logic has nowhere to go: `reportCompletion` verifies rather than pushes, because `useMcpTool`'s non-overridable 30 s budget would report failure over a push that landed, so the agent pushes with `runCommand` and a failure reaches it as stderr (Task 7.2 item 1, §0.2). Delete it and keep the reasoning, not the code. The second harvest stands: record `COMMAND_TIMEOUT_SECONDS = 300` against the runtime's 120s cap (R1) before the file is gone |
| **7.3** Collapse `engineeringAgent/orchestratorHandler.js` | **No port — this handler is never updated, only deleted.** Under the narrowed rule (§13.0) only *new and updated* Lambdas must be Go, and `engineeringAgent` is not updated by any task here: Phases 4–6 route around it and Phase 7 removes it. So it stays JS until it is deleted, and the migration plan's Wave 4 slot (`:178`, 2,965 lines) is moot for this project. **Rescue the survivors first:** `generatePlan`'s output-format block (`:298-321`) is the plan schema's provenance and Task 5.1 needs it; `buildEngineeringPrompt` and `loadEngineeringDocumentation` author instructions.<br><br>**Its writer-collapse half shrinks to a verification, and must not be spec'd twice.** Revision 4 had this task achieve one-writer-one-field by deleting `reportProgress`. **Part 1 now owns write-path convergence** (§10.1a), which lands earlier and does it deliberately rather than as a delete's side effect. If convergence has landed, this task deletes a function that is already canonical: the assertion changes from *"there is now one progress writer"* to *"`progressLog` is still the only field written, and this delete did not reintroduce a second."* If convergence has **not** landed by Phase 7 the work is still here — but that ordering means the cutover ran on a split feed, which §10.1a argues against |
| **7.4** Collapse `qaAgent/orchestratorHandler.js` | **CHANGES SHAPE**, same reasoning — Wave 4 (`:174`). Task 6.2's extraction must have happened first or the diff-hunk parser is lost |
| **7.5** `expandBusinessGoals` + its three sites | **GO-PORT/DISSOLVES.** Gated on 5.5a and 5.5b regardless. `verify-converse.js:1093-1141` is a Node script, not a Lambda — **NON-LAMBDA**, and still needs its case 11 removed |
| **7.6** `handleEngineeringTask` | **Survives as a delete.** The dead `@aws-sdk/client-bedrock-agent-runtime` import at `:4` — absent from that handler's `package.json` — goes with it, and the four-PR-create-site count guard survives as a **deployment guard** |
| **7.7** Retire the legacy dispatch branches | **SPLIT by the port, and it is the task most changed.** Revision 5 called this GROWS; under port-then-change it mostly **moves**. The JS deletions it enumerated — the SQS engineering dispatch, `processSQSCycleExecution`'s engineering arm, `invokeEngineeringAgent`/`invokeQAAgent`, the `buildMode` branches — are all cycle-path JS and are **removed by the port**, not here (table above). What is left for Phase 7 is the **infrastructure retirement**: `aws_lambda_function.agent_orchestrator` (`lambdas.tf:1519-1528`), its `maybe_package` entry, and the `bedrock_queue` + event-source-mapping question this task already flags — *"do not delete a queue while messages may be in flight, and note `bedrock_queue` has no DLQ (`main.tf:435-447`), so anything lost there is lost silently."* **The four-PR-create-site count guard survives as a deployment guard**, and A§3.4's warning not to consolidate those sites during a migration applies with more force than ever, because a port already rewrites every line |

### 13.4 New blocking questions

**`BQ-GO-1` — what is the single source of truth for the cycle-status vocabulary and the activity selector, across Go, JS, Python and the browser?** **Blocks Phase 4b**, whose entire job is to add a status. Today: `cycleStatuses.js` authoritative in JS with a hand-maintained Go `AgentID` correspondence (`sdlcStepGraph.js:51` ↔ `internal/sdlc/handler.go:384-394`) and a Python consumer; `selectActivities` in JS with a hand port in `cc_cycles.py` (`e29f8264`), bound only by a **source-text guard** (`cycleProgressContract.test.js`) that cannot catch a logic divergence. Adding Go makes four. Recommended default: **generate the JS and Python artifacts from the Go definition**, because it is the only option with no hand-maintained duplicate, and because R0 is the proof that duplicated logic in this contract diverges invisibly. Cost of guessing wrong: the activity feed is the most-visible part of the cycle UI and its selector is already the one place a one-line mistake cost 7 of 11 tests.

**`DEP-P1-16` — which Go shape, and does the migration plan's Wave 5 become a prerequisite?** Part 1 owns it. This document needs only the answer, not the reasoning, and §13.1–13.3 resolve mechanically once it lands. Note the tension §0.15 records: the migration plan defers the orchestrator deliberately (*"until patterns are proven"*, `:15`) and this project needs the cycle path first. Also note `DEP-P1-8` is superseded — **`backend/common/cycleEffects.js` cannot be the shared home if the consumer is Go.** Part 1's Task 3.3 extraction either becomes a Go package or stops being an extraction at all; Tasks 4.3 and 6.4 above depend on which, and both dissolve under G3.

**`BQ-GO-2` — ANSWERED BY CODE in revision 8, and the answer is yes to both.** The question was whether the runtime-facing HTTP client and the SQS consumer become Go. They did, and they exist:

- the consumer is **`backend/go/cmd/sdlc-event-consumer`**, deployed as `${var.project_name}-sdlc-event-consumer-${var.environment}` with an event-source mapping off `aws_sqs_queue.sdlc_events` (`infrastructure/sdlc-events.tf:113`, `:204-206`);
- the client is **`backend/go/internal/runtimeclient`** (`transport.go`, `sessions.go`, `secrets.go`), bearer-token authenticated, with `cmd/cycle-approval` and `cmd/cycle-stall-sweeper` as its two callers. It defines four `/v1/sdlc` methods but **cancel has no production caller** — `runtimesession.go:39-44` argues *"RELEASE, NEVER CANCEL"*, for the reason §1.10's last row gives.

**So the consequence this question anticipated has landed: `runtimeEvents.mapSessionEvent` is Go**, as `backend/go/internal/orchestrator/runtimeevents`, ported from the JavaScript with its landed test suite as parity evidence. **The loss is smaller than feared, and the port is better than the original in one specific way** worth recording rather than rediscovering: the JavaScript read `sessionStatus: "completed"` as *"the session ended without calling its terminal tool, so resolve its result from the session record"*, and that arm is **deleted** (#815) on §1.14's reasoning — *a session's status is never its outcome, and no future status could be.* The port did not preserve it and must not. **Tasks 4.4, 5.1 and 6.5 extend the Go mapper**, not `runtimeEvents.js`.

**`DEP-P1-16` is likewise answered by what shipped, and the answer is nearest to G2.** Not G3: `agent-orchestrator` is still `nodejs20.x` running `agentDrivenOrchestrator.zip` (`infrastructure/lambdas.tf:1554-1560`), and cycle *creation* still runs in JavaScript. Not G1 either. What exists is **new Go Lambdas for slices of the cycle path** — `cmd/cycle-approval` (the human approval gate, API-Gateway-integrated at `lambdas.tf:3040-3122`), `cmd/cycle-stall-sweeper`, `cmd/sdlc-event-consumer` — over some thirty `backend/go/internal/orchestrator/*` packages, `planning`, `runtimesession`, `approvedispatch`, `engcomplete`, `buildpoll`, `iterate`, `cyclerecord`, `runtimeevents` and `cycleeffects` among them.

One shape that arrangement forced, because §13.2's task rows depend on it: **`approvedispatch` exists only to keep `planning` and `runtimesession` from importing each other.** A port with a recorded golden must not drag session concerns into its parity, and the session package must stay usable from the consumer, which has no approval gate. A third package importing both costs nothing and couples nothing. **`DEP-P1-8` is confirmed superseded:** `backend/common/cycleEffects.js` is not the shared home — `backend/go/internal/orchestrator/cycleeffects` is. Read `DEP-P1-8`'s row in §3.4 as historical.
