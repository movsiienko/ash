# Porting eve from Vercel to AWS — change inventory and phased plan

## Context

This repo is a hard fork of Vercel's `eve` (filesystem-first framework for durable
backend AI agents). The goal is to retarget it entirely to AWS and remove Vercel.
Decisions already made:

- eve keeps compiling to **Nitro**; the `aws-lambda` preset puts it on Lambda. Infra is CDK, solved separately.
- Durable execution moves from the Workflow DevKit (`@workflow/core`) to **AWS Lambda durable functions** (`@aws/durable-execution-sdk-js`).
- Default model provider becomes **Amazon Bedrock**.
- Sandbox becomes **AWS Lambda MicroVMs**.
- Hard fork: rip Vercel out, no dual-target compatibility.

This document is the "what needs to change" assessment plus an execution order.

---

## The finding that shapes everything

`packages/eve/src/internal/authored-directive-prologue.ts` **rejects** authored
`"use workflow"` / `"use step"` directives outright:

> Workflow directives are reserved for eve-generated workflow entrypoints.

Confirmed by grep: zero occurrences of either directive anywhere in `apps/`, `e2e/`,
or `docs/`. **The durable-execution layer is 100% framework-internal.** Swapping
`@workflow/core` for the AWS SDK is a pure internal refactor with **no authoring-API
break** — agent directories, tools, channels, skills, and schedules are untouched.

The corollary is better than a port: most of `src/internal/workflow-bundle/`
(~2,000+ lines) exists *only* to synthesize durable entrypoints out of directive-marked
module-scope functions. AWS's `context.step(name, fn)` takes an inline closure and
needs no stable module identity, so that machinery gets **deleted, not ported**:

| File | Why it exists today | Fate |
|---|---|---|
| `workflow-transformer.ts` | strips/parses `"use workflow"`/`"use step"` | delete |
| `workflow-core-shim.ts` | bridges workflow bodies to runtime via `Symbol.for("WORKFLOW_*")` globals | delete |
| `dynamic-tool-transform.ts` + `dynamic-tool-ast-references.ts` | hoists tool `execute` to module scope so it can be a step | delete |
| `builder.ts` (687 lines) + `builder-support.ts` | rolldown bundle of transformed workflow bodies | delete |
| `vercel-workflow-output.ts` (657 lines) | emits one `.func` per workflow into `.vercel/output` | replace with a single durable-handler entry |
| `authored-directive-prologue.ts` | guards the above | delete |

Replacing the workflow SDK is therefore a net **simplification**, not just a swap.

---

## Target architecture

```
EventBridge Scheduler ──┐
                        ▼
Client ── Function URL ─► HTTP Lambda (Nitro, aws-lambda preset)
             │             │  channels, routes, NDJSON tail
             │             ├─ async Invoke ──► Durable Lambda (withDurableExecution)
             │             │                     session/turn orchestration
             │             ├─ SendDurableExecutionCallbackSuccess (deliver/cancel/HITL)
             │             └─ tail events ◄── DynamoDB (event log, seq-numbered)
                                               DynamoDB (hookToken → callbackId)
                                               Lambda MicroVM (sandbox)
                                               Bedrock (models)
```

Two Lambda bundles from one compiled artifact set, instead of Vercel's N-functions-per-workflow.

---

## Semantic gaps to close (these are the real work)

The AWS SDK covers most of what eve needs (`step`, `wait`, `waitForCondition`,
`createCallback`/`waitForCallback`, `invoke`, `parallel`, `map`, `runInChildContext`,
1-year executions, free waits). Four things it does **not** give you:

### 1. Hook tokens vs. AWS-generated callback IDs — needs an index table
eve mints its own hook tokens (`"<completionToken>:inbox"`, `"<sessionId>:cancel"`,
connection-OAuth tokens) and resumes by token from an HTTP route
(`resumeHook(token, payload)` in `src/execution/workflow-runtime.ts`).
`createCallback()` returns an **AWS-generated** `callbackId`; names are observability-only
and are not a lookup key.

**Fix:** a DynamoDB `hooks` table keyed by eve token → `{callbackId, executionId, ttl}`.
The durable function writes the mapping inside a step immediately after `createCallback()`;
`deliver()` reads it and calls `SendDurableExecutionCallbackSuccess`. Contained entirely
within `src/execution/hook-ownership.ts` and `session-delivery-hook.ts`, whose interfaces
already isolate this.

### 2. No per-run event stream — needs an event log
eve's NDJSON tail is `getRun(id).getReadable({ startIndex })`
(`workflow-runtime.ts:224`, `durable-session-store.ts:138`), written by `getWritable()`
(`workflow-entry.ts:89`). Durable functions have no equivalent.

**Fix:** append-only DynamoDB event log `(sessionId, seq)`; the HTTP Lambda long-polls it.
The client protocol **already** carries `?startIndex=` for resumable reconnect
(`src/client/open-stream.ts`), so poll-tailing is a drop-in — no client or wire-format change.
Keep it behind a narrow interface so Kinesis/AppSync Events/Momento can replace it later.

### 3. No documented stop API — use a cancel callback
`cancelRun` has no direct equivalent. eve already has a dedicated `{sessionId}:cancel`
hook (`src/execution/turn-cancellation-control.ts`); make cancellation purely a
callback completion the turn races against. Cheaper than it sounds.

### 4. Lambda is request-scoped — long-lived-process assumptions break
Audit these; each currently assumes a process that outlives a request:

- **Nitro `scheduledTasks`** (`src/internal/nitro/host/schedule-task-routes.ts`) — the in-process cron scheduler cannot run on Lambda. → EventBridge Scheduler (below).
- **`event.waitUntil()`** (`src/internal/nitro/routes/channel-dispatch.ts`) — post-ack work is drained before the Lambda freezes. Slack-style "ack fast, work after" must instead async-invoke the durable function. This is a behavioral change worth calling out in docs.
- **Per-session MCP connection registry** (`src/runtime/connections/registry.ts`) — already per-session and `dispose()`d, so it is fine, but it now reconnects per step rather than per process. Watch OAuth token cache churn in `scoped-authorization.ts`.
- **Sandbox `shutdown()`** — must become *suspend*, not terminate (see below).
- **NDJSON response streaming** caps at the Lambda 15-minute limit; client reconnect with `startIndex` covers longer sessions.

---

## Component-by-component change map

### Durable execution — `packages/eve/src/execution/`
Rewrite `workflow-entry.ts`, `turn-workflow.ts`, and `workflow-steps.ts` to take an
explicit `DurableContext` and call `context.step(...)` / `context.createCallback(...)`
instead of directive functions. `workflow-runtime.ts` keeps its `Runtime` interface
(`run`/`deliver`/`getEventStream`/`cancelTurn`/`terminateSession`) — channels are
written against it and should not change — but its implementation swaps:

| Today | AWS |
|---|---|
| `start(workflowEntry, …)` | async `Invoke` of the qualified durable function (alias, not `$LATEST`) |
| `resumeHook(token, payload)` | token index lookup → `SendDurableExecutionCallbackSuccess` |
| `getRun(id).getReadable({startIndex})` | DynamoDB event-log tail |
| `cancelRun(id)` | complete the cancel callback |
| `shouldRouteToLatestDeployment()` (`VERCEL_ENV`) | Lambda alias routing — delete the function |

Keep `durable-session-store.ts` and `durable-session-migrations/` — the snapshot format
is platform-neutral and the versioning is worth preserving. Delete
`workflow-callback-url.ts` (Vercel protection-bypass) and
`src/internal/workflow/{validate-world,world-compatibility,local-world-data-directory,development-world-*}.ts`
— the World concept disappears entirely, along with `experimental.workflow.world`
in `agent.ts` and `resolveWorkflowWorldWiring()` in
`src/internal/application/compiled-artifacts.ts:246-345`.

Vendor `@aws/durable-execution-sdk-js` through the existing mechanism
(`packages/eve/scripts/vendor-compiled/index.mjs`) so the "one runtime dependency"
invariant holds.

### Local development — introduce a `DurableBackend` seam

Local iteration must not regress; today `eve dev` is a single process with a TUI, hot
reload of authored sources, and a dev-world RPC server
(`src/internal/workflow/development-world-{server,client,protocol,codec}.ts`) that keeps
run state alive across Nitro worker reloads.

**No SAM.** The local runner from `@aws/durable-execution-sdk-js-testing` is the local
runtime. Rather than binding `workflow-runtime.ts` directly to either SDK, put a thin
eve-owned **`DurableBackend`** interface behind it — `startExecution`, `completeCallback`,
`getExecution`, `appendEvents`, `tailEvents` — with two implementations:

- **`local`** — `LocalDurableTestRunner`. No AWS credentials, no containers, no deployment. Hot reload and the TUI are unchanged. Because it performs **real replay** against a simulated checkpoint server, determinism bugs surface on the laptop rather than in production.
- **`aws`** — `withDurableExecution` plus `SendDurableExecutionCallbackSuccess` / `GetDurableExecution` / `GetDurableExecutionHistory`.

The runner's surface maps onto what eve needs better than expected:

| eve need | Runner API |
|---|---|
| resume a hook by eve's own token | `createCallback(name, …)` with the token as `name`, then `getOperation(name).sendCallbackSuccess(result)` |
| cancel a turn | `sendCallbackFailure` on the cancel callback |
| subagent / child sessions | `registerDurableFunction(functionName, handler)` |
| `Runtime.resolveSession()` | `getStatus()` / `getOperations({ status })` |
| real (non-collapsed) waits in dev | `setupTestEnvironment({ skipTime: false })` — the default |

Note that `getOperation(name)` makes the callback **name** a lookup key locally, which is
exactly eve's token model. In the cloud it is not — names there are observability-only —
so the DynamoDB token index is still required for the `aws` backend. Same call shape,
different lookup; the seam absorbs the difference.

**Two constraints to design around, both verified:**

1. **The checkpoint store is in-memory and explicitly not pluggable.** State dies with the process, whereas today `.eve/.workflow-data` survives an `eve dev` restart. Mitigation is eve's *existing* pattern: host the runner in the `eve dev` parent process and give Nitro workers an RPC client, exactly as `development-world-server.ts` does now. Session state then survives hot reload and worker replacement — just not a full `eve dev` restart. That is an acceptable, nameable regression; snapshotting the checkpoint log to `.eve/` on shutdown can close it later if it bites.
2. **`run()` is documented as await-to-completion.** eve needs start-and-return-202, then stream. An async start surface is documented for .NET/Python/Java but **not for TypeScript**. This is the one genuine unknown and it should be a **spike at the top of Phase 1**: confirm a TS execution can be started without awaiting it (a floating promise held by the parent process is the likely answer).

If either constraint proves fatal, the fallback is an eve-owned `local` backend over a
disk-backed checkpoint log. The seam makes that a contained swap rather than a redesign —
which is the main reason to introduce it.

### HTTP host — `packages/eve/src/internal/nitro/host/`
`resolveProductionNitroPreset()` (`create-application-nitro.ts:84`) returns
`"aws-lambda"` unconditionally. Build emits two entries: the Nitro handler and the
durable handler. Replace `vercel-build-output-config.ts` / `build-vercel-agent-summary.ts`
with a **deploy manifest** (`.eve/aws-manifest.json`: function entries, cron expressions,
env requirements, table names, MicroVM image ids) — this is the CDK contract and is the
natural successor to the Vercel dashboard summary. Delete `cron-handler-route.ts`,
`vercel-build-prewarm.ts`.

### Models — Bedrock
Vendor `@ai-sdk/amazon-bedrock`. A bare string model id resolves to Bedrock
(`src/runtime/agent/resolve-model.ts`); credentials come from the Lambda execution role,
not an API key. Delete `src/internal/gateway.ts`, `src/internal/runtime-model.ts`'s
`formatLanguageModelGatewayId()`, and the entire gateway setup flow
(`src/setup/{ai-gateway-api-key,validate-gateway-key,gateway-models}.ts`,
`src/setup/boxes/{detect-ai-gateway,apply-ai-gateway-credential}.ts`); `WiringMode`
in `src/setup/state.ts:101` collapses to a single mode.
`DEFAULT_AGENT_MODEL_ID` → a Bedrock inference-profile id (`us.anthropic.claude-sonnet-5-*`).
**`src/compiler/model-catalog.ts` needs a decision**: it fetches the Gateway catalog at
build time to bake `contextWindowTokens`/`maxOutputTokens` into the manifest. Bedrock's
`ListFoundationModels` does not reliably expose context windows — bake a static catalog
and let `agent.ts` override.

### Sandbox — Lambda MicroVM
Maps cleanly onto the existing `SandboxBackend` interface
(`packages/eve/src/shared/sandbox-backend.ts`):

| `SandboxBackend` | MicroVM |
|---|---|
| `prewarm({templateKey, bootstrap, seedFiles})` | zip Dockerfile+seed → S3 → create MicroVM image (snapshot); `templateKey` → image id |
| `create({sessionKey, existingMetadata})` | `run-microvm` from snapshot, or `resume-microvm` when metadata holds a suspended id |
| `captureState()` | `{ microVmId, endpoint }` |
| `shutdown()` | **`suspend-microvm`**, not terminate — memory+disk preserved, sessions reattach |

**The one real gap:** `@vercel/sandbox` gave a command SDK (`runCommand`/`readFile`/`writeFile`);
MicroVM gives you a raw HTTPS endpoint with JWE auth. eve must ship a small **in-image
agent server** implementing the `SandboxSession` surface, baked into the image the root
`Dockerfile` produces. Budget real time for this; it is the sandbox long pole.
`defaultSandbox()` probe chain (`src/public/sandbox/backends/default.ts`) becomes
MicroVM (when running on Lambda) → Docker → microsandbox → just-bash; the local backends
stay and remain the dev path. Delete `src/execution/sandbox/bindings/vercel*.ts` (8 files)
and `src/public/sandbox/vercel-sandbox.ts`; drop `resolveVercelProjectIdFromEnvironment()`
from template-key derivation (`src/runtime/sandbox/keys.ts`) in favor of an account/function-scoped key.

### Schedules — EventBridge Scheduler
Discovery/compile/dispatch stay as-is. Only the trigger changes: `eve build` emits each
`defineSchedule` cron into the deploy manifest; CDK creates EventBridge Scheduler rules
that invoke the HTTP Lambda's cron route (keep the existing unguessable-path idea from
`cron-handler-route.ts`, or invoke the durable function directly). Nitro's in-process
`scheduledTasks` registration is removed. `eve dev`'s manual dispatch route
(`POST /eve/v1/dev/schedules/:scheduleId`) is unchanged.

### Auth
Delete `vercelOidc()` from `src/public/channels/auth.ts` and `src/public/agents/auth.ts`,
plus `src/runtime/governance/auth/vercel-oidc-project.ts` and `src/shared/vercel-project.ts`.
Add a **SigV4** authenticator for the framework default
(`src/runtime/framework-channels/index.ts:22` currently `[vercelOidc(), localDev()]`
→ `[sigv4(), localDev()]`), matching Function URL `AWS_IAM` auth; SigV4 signing also
covers agent→agent calls (`src/execution/remote-agent-dispatch.ts`). The existing
`oidc()`/`jwtEcdsa()`/`jwtHmac()`/`httpBasic()` strategies are already generic and cover Cognito.

### CLI and setup
`eve deploy` and `eve link` (`src/cli/commands/register-project-commands.ts`) are already
isolated. Delete them and the ~40-file Vercel setup surface
(`src/setup/primitives/run-vercel.ts`, `src/setup/vercel-*.ts`, `src/setup/flows/{deploy,install-vercel-cli}.ts`,
`src/setup/boxes/{deploy-project,link-project}.ts`, `src/cli/dev/tui/vercel-*.ts`,
`src/internal/vercel/`, `src/cli/vercel-service-output.ts`, `src/shared/vercel-output-directory.ts`).
`eve build` producing `.eve/aws-manifest.json` is the deploy contract; CDK owns the rest.
Optionally add `eve deploy` back later as a thin `cdk deploy` wrapper.

### Framework integrations and branding
`src/public/{next/vercel-output-config.ts, nuxt/vercel-services.ts, sveltekit/vercel-json.ts}`
(~1,100 lines) all write Vercel build output. Either rewrite them against the AWS manifest or —
**recommended** — drop them from phase 1 and keep only the standalone Nitro/Lambda deployment;
they are the least load-bearing surface and the most Vercel-shaped. `apps/frameworks/sveltekit`'s
`@sveltejs/adapter-vercel` goes with them.
Rename: `@vercel/eve-catalog` workspace package, `ghcr.io/vercel/eve` image, the
`vercel-sandbox` OS user in the root `Dockerfile`, and the repo URL in
`packages/eve/package.json`. Drop `@vercel/oidc`, `@vercel/sandbox`, `@vercel/sdk`,
`@vercel/detect-agent` (or vendor the last one's heuristic — it is trivial and useful),
and the `@vercel/connect` scaffolding in `src/setup/scaffold/**`.

### e2e
`.github/workflows/e2e-vercel.yml` and the shared-Vercel-project model in `e2e/README.md`
are replaced by a CDK stack deployed per CI run into a test account, with the 23 fixtures
under `e2e/fixtures/` deployed behind one Function URL each and torn down after.
`e2e/provision/tools-sandbox.ts` is rewritten against MicroVM images.
`e2e-local.yml` needs no Vercel and keeps working throughout.

---

## Phases

**Phase 0 — De-Vercel without changing behavior.** *Independently shippable.*
Delete `eve deploy`/`eve link`, the `src/setup/` Vercel surface, the framework
`vercel-*` output writers, `vercel-agent-summary`, and the `@vercel/*` deps.
Switch the Nitro preset to `aws-lambda`. Pin the sandbox to Docker and the workflow
world to `@workflow/world-local` so the tree stays green. Rebrand.
This is mostly deletion and lands fast — it makes every later phase smaller.

**Phase 1 — Durable execution on Lambda. *The long pole.***
Vendor `@aws/durable-execution-sdk-js`. **Spike first:** confirm `LocalDurableTestRunner`
can start a TypeScript execution without awaiting it to completion (see above) — this
gates the whole local-dev story and is a half-day answer. Then define the `DurableBackend`
seam and land the **`local` backend first** — it keeps `eve dev` and the test tiers
working throughout the rewrite, and it is the cheapest place to prove the design against a
fixed `Runtime` interface. Build the event log and
hook-token index behind that seam. Rewrite `workflow-entry.ts` / `turn-workflow.ts` /
`workflow-steps.ts` against `DurableContext`, keeping `workflow-runtime.ts`'s `Runtime`
interface fixed so channels and the harness are untouched. Delete
`src/internal/workflow-bundle/` and `src/internal/workflow/`. Add the `aws` backend last.
Everything downstream depends on this; nothing else should start until the `Runtime`
interface is proven against the local backend.

**Phase 2 — Bedrock.** *Parallelizable with Phase 3.*
Vendor `@ai-sdk/amazon-bedrock`, repoint bare-string resolution, static model catalog,
delete the gateway. Small and self-contained.

**Phase 3 — MicroVM sandbox.** *Parallelizable with Phase 2; contains the second long pole.*
In-image agent server first (it is the blocker), then the backend against the existing
`SandboxBackend` interface, then image build in `prewarm`, then suspend/resume lifecycle.

**Phase 4 — Schedules, auth, deploy manifest.**
EventBridge Scheduler, SigV4 authenticator, `.eve/aws-manifest.json`, and the CDK stack.

**Phase 5 — e2e and docs.**
Rewrite `docs/guides/deployment/**`, `docs/sandbox.mdx`, `docs/schedules.mdx`,
`docs/concepts/execution-model-and-durability.md`. New CI e2e workflow.

---

## Verification

- **Phase 0:** `pnpm build && pnpm typecheck && pnpm test` all green; `pnpm guard:invariants` passes; `grep -ri vercel packages/eve/src` returns only intentional leftovers.
- **Phase 1:** the tightest signal is `LocalDurableTestRunner`-backed integration tests over `workflow-runtime.ts`'s `Runtime` interface — start a session, deliver a message, tail NDJSON with `startIndex`, cancel a turn, resume after a simulated interruption. Then `eve dev` end-to-end against the weather fixture (`apps/fixtures/weather-fixture`), driving a real multi-turn conversation and confirming events stream and a HITL approval round-trips.
- **Phase 2:** run an agent against a real Bedrock model id; confirm the compiled manifest carries correct context-window limits and that compaction triggers at the right threshold.
- **Phase 3:** `e2e/fixtures/agent-tools-sandbox` locally against Docker, then a deployed MicroVM: create a session, write a file, let it idle into suspend, resume, confirm the file survived.
- **Phase 4/5:** deploy the CDK stack to a test account and run `eve eval` against the Function URL — the same shape as today's `e2e-vercel.yml`, different target.

## Open risks

1. **Durable-function payload/checkpoint costs.** eve checkpoints a full session snapshot per step (`durable-session-store.ts`). Metering is per-operation *and* per-payload-byte; large snapshots in a long session could get expensive. Measure early in Phase 1 and consider offloading snapshots to S3 with only a pointer checkpointed.
2. **Replay determinism.** eve's step bodies do model calls, tool calls, and clock reads. Everything non-deterministic must sit inside `context.step()`. `src/harness/tool-loop.ts` (~2,400 lines) is the file to audit hardest.
3. **Concurrent callback completion.** eve's turn-inbox hook can receive multiple deliveries; AWS callbacks appear to be single-completion. The inbox may need one callback per delivery rather than a reusable hook.
4. **MicroVM API maturity.** It is new; confirm the JS SDK surface, per-account MicroVM quotas, and image build times before committing Phase 3's schedule.
5. **The local runner is a test harness doing a dev-runtime job.** It is the right call — it is the only credential-free, container-free option — but it is not what AWS designed it for. Undocumented limits (max invocations per runner, concurrent executions, payload ceilings) may surface only under a real `eve dev` session. The Phase 1 spike and the `DurableBackend` seam exist to keep that discovery cheap.

---

## References

- [Lambda durable functions](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
- [Durable execution SDK](https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-sdk.html)
- [Callback operation API](https://docs.aws.amazon.com/durable-execution/sdk-reference/operations/callback/)
- [Testing runners](https://docs.aws.amazon.com/durable-execution/testing/runner/) and [testing API reference](https://docs.aws.amazon.com/durable-execution/testing/api-reference/)
- [Lambda MicroVMs](https://docs.aws.amazon.com/lambda/latest/dg/lambda-microvms-guide.html)
