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
or `docs/`. **The durable-execution layer is effectively framework-internal.** Swapping
`@workflow/core` for the AWS SDK touches no agent directory, tool, channel, skill, or
schedule.

**The wider public surface does break, though, and the changeset must enumerate it.** An
earlier draft called `experimental.workflow.world` "the only one"; that was wrong. Removed
or changed public exports include:

- `experimental.workflow.world` (`src/shared/agent-definition.ts:230`)
- `vercelOidc()` and `vercelSubject` from `eve/channels/auth`, and `vercelOidc()` from `eve/agents/auth`
- the `eve/sandbox/vercel` export and `ClientAuth.vercelOidc`
- `withEve` plus the Nuxt and SvelteKit integrations, if those are dropped rather than rewritten
- `eve deploy` / `eve link` CLI commands
- **`OutboundAuthFn`'s signature**, which SigV4 forces to widen (see Auth), and `AuthFn`'s if inbound SigV4 is supported

A `minor` changeset remains right, but it must describe every removed surface and ship
migration notes — not a one-line mention.

The corollary is better than a port: most of `src/internal/workflow-bundle/`
(**4,236 non-test lines**) exists *only* to synthesize durable entrypoints out of
directive-marked module-scope functions. AWS's `context.step(name, fn)` takes an inline
closure and needs no stable module identity, so that machinery gets **deleted, not ported**:

| File | Why it exists today | Fate |
|---|---|---|
| `workflow-transformer.ts` | strips/parses `"use workflow"`/`"use step"` | delete |
| `workflow-core-shim.ts` (181) | bridges workflow bodies to runtime via `Symbol.for("WORKFLOW_*")` globals | delete |
| `dynamic-tool-transform.ts` + `dynamic-tool-ast-references.ts` | hoists tool `execute` to module scope so it can be a step | delete |
| `builder.ts` (687) + `builder-support.ts` | rolldown bundle of transformed workflow bodies | delete |
| `workflow-builders.ts` (443) | `applyWorkflowTransform`, `createEveWorkflowQueueTrigger()` (`queue/v2beta`) | delete; queue triggers have no AWS analogue |
| `nitro-step-entry.ts` (210) | hosted step entrypoint for the Vercel step function | replace with the durable-handler entry |
| `eve-service-route-output.ts` (61) | `eve/__server.func` + `.well-known/workflow/` route prefixes | delete with the Build Output emitter |
| `vercel-workflow-output.ts` (657) | emits one `.func` per workflow into `.vercel/output` | replace with a single durable-handler entry |
| `build-queue.ts` | generic build serializer | delete with the directory (nothing else consumes it) |
| `authored-directive-prologue.ts` *(in `internal/`, not `workflow-bundle/`)* | guards the above | delete |

That is all 11 non-test files in `src/internal/workflow-bundle/`, plus the one guard that
lives outside it.

This is a net **simplification**, and it is the strongest argument that the user's
instinct here is right.

**But do not delete the naming machinery without replacing what it guaranteed.** The
claim above — "no stable module identity required" — is true of *module* identity and
false of *operation* identity. Replay works by matching each durable operation against the
checkpoint log; the AWS SDK derives that match from **invocation order within the
execution**, so any code path that changes the order or count of operations between the
original run and a replay produces divergence. Today the transform gave every step a
source-derived, stable name for free. After deletion, eve owns that contract.

Concretely, before `builder.ts` and `workflow-builders.ts` are removed, fix and document:

- **A stable naming scheme** for every `context.step` / `createCallback` / `runInChildContext` / `parallel` / `map` — derived from source location or a compiled-manifest identifier, not from a runtime value.
- **Ordering invariants for conditional and loop paths.** eve's turn loop iterates tool calls; the number of steps per turn depends on model output. That is fine *within* one execution, but any branch keyed on non-checkpointed state (wall clock, env, cache hits) will reorder operations on replay. This is the concrete form of open-risk #2.
- **A determinism test** in the integration tier: run a session, force a replay, assert the operation sequence is identical. This is the only cheap way to catch regressions here, and it should land with the first `DurableContext` rewrite rather than after.

---

## Target architecture

```text
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

**Default topology: two Lambda bundles** from one compiled artifact set — the HTTP handler
and the durable handler — instead of Vercel's N-functions-per-workflow. If route isolation
(see Auth) is implemented as separate handler entries rather than a surface-tagged
allowlist, that becomes three; the allowlist is the default precisely to keep it at two.

---

## Semantic gaps to close (these are the real work)

The AWS SDK covers most of what eve needs (`step`, `wait`, `waitForCondition`,
`createCallback`/`waitForCallback`, `invoke`, `parallel`, `map`, `runInChildContext`,
1-year executions, free waits). Seven things it does **not** give you:

### 1. Reusable hooks vs. single-completion callbacks — needs a durable inbox
eve mints its own hook tokens (`"<completionToken>:inbox"`, `"<sessionId>:cancel"`,
connection-OAuth tokens) and resumes by token from an HTTP route
(`resumeHook(token, payload)` in `src/execution/workflow-runtime.ts`).
`createCallback()` returns an **AWS-generated** `callbackId`; names are observability-only
and are not a lookup key.

**A token→callbackId index alone does not work.** `SessionDeliveryHook`
(`src/execution/session-delivery-hook.ts:22`) is a **reusable multi-message iterator** —
`consumeNext()` / `next(): Promise<IteratorResult<HookPayload>>`, constructed over
`bufferedDeliveries`, coalescing concurrent deliveries into one logical hook. An AWS
callback completes **once**. Map a stable eve token onto one callback and two simultaneous
deliveries race to complete it: one wins, the other errors or is silently lost. Retrying a
missing mapping does not fix this — the semantics are simply different.

**Fix: a durable inbox, with callbacks demoted to wake-up signals.**

- **Key: `(sessionId, seq)` — not `(sessionId, deliveryId)`.** `deliveryId` is an idempotency key with no ordering, but `bufferedDeliveries` (`session-delivery-hook.ts:43`) preserves **arrival order** today, and losing that is a user-visible semantic change.
- **Allocate `seq` inside the same transaction as the write — not with a separate `UpdateItem ADD`.** A standalone atomic increment followed by a separate transact opens a **sequence gap**: two concurrent writers can allocate N and N+1, and the N+1 item can become visible before N commits. A drain that advances its high-water mark past N+1 then **skips N permanently**. Use one `TransactWriteItems` containing (a) a **CAS on the counter** (`ConditionExpression` on the expected current value), (b) a `Put` on a separately keyed dedupe record `(sessionId, deliveryId)` with `attribute_not_exists`, and (c) the sequenced inbox item — retrying the whole transaction on conflict. Allocation and publication then commit together, so no `seq` is ever visible out of order.
- **Dedupe must live in that transaction, not in a condition expression on the item.** With `(sessionId, seq)` as the primary key, a `ConditionExpression` on non-key `deliveryId` evaluates only the item being written — DynamoDB cannot enforce uniqueness across a partition. A retried delivery would get a *new* `seq` and pass.
- After enqueuing, `deliver()` completes the current wake-up callback. A completion that loses a race is harmless: the message is already durable in the inbox.
- **Drain protocol** ("atomically drains" is not a DynamoDB primitive, so specify it): `Query` the session partition above the last-processed `seq` in order, process, then record the high-water `seq` in the checkpoint. **Do not delete-then-checkpoint**: DynamoDB and the checkpoint log are separate systems, so a failure between them either loses deliveries (deleted, never checkpointed) or strands them (checkpointed, never cleaned). Treat inbox items as **immutable until the checkpoint commits**, then garbage-collect below the checkpointed high-water mark — ideally via TTL, so cleanup needs no second write path.
- **Arm the next callback *before* the final empty check — otherwise deliveries are lost forever.** An earlier draft said mid-drain arrivals "wake the next callback," which is wrong: between the callback that woke the execution being consumed and a fresh one being armed, **no callback exists**. A delivery landing in that window writes to the inbox, finds nothing to signal, and parks indefinitely — the execution has already decided the inbox is empty and suspended. Correct sequence: arm a **generation-stamped** callback first, then re-`Query` above the high-water mark, then conditionally publish that generation. The re-query after arming is what closes the window.
- The token index (`token → {callbackId, executionId, ttl}`) still exists, but only to find the *current* wake-up callback — losing the race to a stale one no longer loses a message.

Use **`waitForCallback(name, submitter, config)`** rather than raw `createCallback()`: the
submitter runs as an SDK-managed step, so persisting the mapping gets retry semantics for
free instead of hand-rolled.

**Not every token needs the inbox.** Disposition each family explicitly, because the inbox
is the expensive option:

| Token family | Semantics | Mapping |
|---|---|---|
| Session delivery (public hook, rekeyed) | reusable, multi-message, ordered | durable inbox |
| Session **auth** (`{sessionId}:auth`, `workflow-entry.ts:166`) | **reusable iterator** — created before any turn so OAuth callbacks can resume repeatedly | durable inbox |
| **Turn inbox** (`{completionToken}:inbox`, `turn-workflow.ts:65`) | **reusable iterator** with a durable cursor shared between promise and iterator reads | durable inbox, per turn execution |
| **Turn control** (`turn-control-receiver.ts:27`) | **reusable iterator**, multi-message control channel | durable inbox or explicit redesign |
| `{sessionId}:cancel` | single-shot | plain callback (gap 3) |

**An earlier draft claimed only session delivery was reusable. That was wrong** — four of
the five families are `createHook(...)[Symbol.asyncIterator]()` multi-message protocols, and
each one hits the single-completion mismatch independently. In particular the auth hook is
*not* a 1:1 OAuth callback: it is long-lived and fires repeatedly across a session. Each
must be mapped onto a durable inbox with generational callbacks, or explicitly redesigned
out — and `turn-control-receiver.ts` deserves scrutiny for the latter, since its
buffered-delivery coupling to the delivery hook is exactly the kind of shared-cursor
protocol that does not survive the port unchanged.

**The token index needs two lifetimes.** `Runtime.resolveSession(continuationToken)`
(`workflow-runtime.ts:229`) resolves token → sessionId via `getHookByToken`, and it must
work **for the session's whole life** — including long-idle periods when no callback is
armed. A `ttl` scoped to the current callback would break it. Use two records (or two
tables): a session-lifetime `token → sessionId` mapping, and a short-lived
`token → {callbackId, executionId}` mapping for the currently armed callback.

**The startup race is worse than a 404 — it silently forks the session.** `deliver()`
translates `HookNotFoundError` into `RuntimeNoActiveSessionError`
(`workflow-runtime.ts:209`), and channels treat that as the **resume-or-start** signal: no
hook means *start a new session*. If all token-index persistence happens inside the durable
execution (in the `waitForCallback` submitter step), then between the HTTP-side async
`Invoke` and the execution's first checkpoint **no mapping exists** — and a delivery landing
in that window is indistinguishable from "no session." The channel starts a second one.
Execution-name idempotency (gap 7) does not save this: the duplicate start allocates a fresh
session id, hence a different execution name. And an earlier draft's "back off and retry"
is wrong twice over — there is no durable fact to retry against, and retrying
unconditionally would break legitimate first-message session starts.

**Fix: reserve the session synchronously on the HTTP side, via a state machine.** There is
no atomic write across DynamoDB and Lambda, so "before or atomically with the Invoke" is not
implementable as stated — record-first leaves a **phantom session** if the invoke fails,
invoke-first creates an **unowned execution**, and two concurrent first messages can both
start. Use an explicit ownership state machine on the token record:

`RESERVED` → (conditional put, `attribute_not_exists`; loser of a concurrent race joins the
winner's session) → async `Invoke` with a **deterministic execution name** derived from the
reserved session id → `STARTING` → execution's first checkpoint promotes to `ACTIVE`.

Recovery matters as much as the happy path: a record stuck in `RESERVED`/`STARTING` past a
timeout is swept and either re-invoked — safe, because the execution name is deterministic
and idempotent — or failed. Async invocation can also drop or duplicate events, so configure
a **DLQ** and reconcile from it rather than assuming delivery.

`deliver()` can then distinguish three states rather than two:

| State | Meaning | Action |
|---|---|---|
| No `token → sessionId` record | genuinely no session | start one (today's behavior) |
| Record exists, no callback armed | session starting, or mid-drain | enqueue to inbox, return accepted |
| Record exists, callback armed | steady state | enqueue, complete the callback |

State the owner and timing of each token-index record explicitly in the implementation —
this is precisely where the two-lifetime split (above) earns its keep.

### 2. No run streams — the client tail needs an event log
Workflow streams serve two jobs, and only one of them is load-bearing:

- **Client tail** — NDJSON events, `getRun(id).getReadable({ startIndex })` (`workflow-runtime.ts:224`), written by `getWritable()` (`workflow-entry.ts:89`). This is the real dependency.
- **Legacy session read** — `readDurableSession` (`durable-session-store.ts:135`) returns `state.snapshot` inline when present and only tails the namespaced `eve.session` stream for **states written before snapshots moved inline**. The code says so directly: *"New states carry the snapshot directly through Workflow step results. States without `snapshot` fall back to the legacy `eve.session` stream tail."*

A prior revision of this plan claimed the store "cannot be kept as-is" because its read path
went through a stream. **That was an overcorrection.** New sessions already carry the
snapshot through step results; under this repo's pre-1.0 no-legacy-fallback rule the stream
branch is simply deleted, and `durable-session-store.ts` largely survives.

**Fix A — externalize snapshots to S3, pointer-checkpointed. This is structurally required,
not a cost optimization.**

Three independent arguments, in increasing order of force:

1. *Cost* — eve checkpoints a full snapshot per step; metering bills per-operation **and** per-payload-byte.
2. *The 100 MB ceiling* — `DurableExecutionStorageWrittenBytes` is capped per execution, and inline snapshots consume it fastest.
3. *Strongest: separate executions cannot share checkpoint state.* Each durable execution has its own checkpoint log that other executions cannot read. Since every turn is a **separate** execution dispatched via `context.invoke` (see the versioning section), the turn cannot reach the driver's checkpoints — session state must travel in the invoke payload. **And that payload is capped by a documented limit, not an unknown one:** [`OperationUpdate`](https://docs.aws.amazon.com/lambda/latest/api/API_OperationUpdate.html) publishes a per-operation-type table — **1 MB for `CHAINED_INVOKE`**, which is what durable `context.invoke()` records, and for async `EXECUTION`; 6 MB for sync `EXECUTION`; and **256 KB for `CONTEXT`, `STEP`, `WAIT`, and `CALLBACK`**. An earlier draft of this plan called the `context.invoke()` ceiling unpublished. It is published; cite it.

**Shape:** write snapshots to **unique or content-addressed** S3 keys, then **conditionally
publish** the winner through a small DynamoDB **session head** record
`{latestSnapshotKey, seq, fenceToken}` with a CAS on the expected `seq`/fence. Small
intra-turn step results stay inline; only oversized ones spill. Invoke payloads carry
`{sessionId, snapshotKey, seq}`.

Note which ceiling actually binds where: the driver→turn payload has 1 MB to work with, but
**step results have only 256 KB**, so the `STEP` limit — not the invoke limit — is what sets
the inline-vs-spill threshold. The only thing left to measure is how much of each budget the
durable SDK's own framing consumes before eve's bytes; that is a sizing spike, not a
discovery one.

> An earlier draft proposed overwriting a **deterministic** key per `(sessionId, turn, step)`
> and claimed S3 PUT idempotency gave "exactly-once for free." **That is wrong on both
> counts.** A retried step is at-least-once and can compute *different* state, so the second
> PUT is not a duplicate of the first; and a delayed attempt can land *after* a newer one,
> silently overwriting the valid frontier with stale state. Content-addressed writes plus a
> conditional head update make the publish the single serialization point — the object write
> becomes harmless because an unreferenced object is simply garbage.

**Rejected: an S3 JSONL delta log.** It is the intuitive shape and it does not survive
contact. S3 objects are immutable — there is no append — so "JSONL" means one object per
delta plus a manifest, or multipart upload (disqualified: ≥5 MB parts, and the object is
unreadable until the upload completes). Its failure mode is **O(n) GET fan-out on
rehydration**: a session on day 60 with thousands of deltas needs thousands of reads to
reconstruct state, turning a ~50 ms resume into tens of seconds. The standard fix —
periodic compacted snapshots — *is* the design above, with a delta log bolted on. Note also
that JSONL-versus-snapshot is irrelevant to both ceilings: any externalization gets the
same relief.

**Replay invariant to hold:** rehydrate only the **frontier** snapshot — one GET at resume,
O(1). The SDK replays from the beginning on every wake, so dereferencing a pointer at every
replayed step would make resume O(steps) in S3 round-trips. Snapshot-pointer makes this
natural (stale pointers are simply never fetched); a delta log makes it structurally hard,
since "current state" is not an object and the tail must be folded every time.

**Latency is a non-issue here.** S3 is ~30–80 ms versus DynamoDB's ~5–10 ms, but this is
one round-trip *per turn*, in a loop dominated by multi-second model calls, with token
streaming on a separate path off the persistence boundary. Snapshots stay small in practice
because `shouldCompact` (`src/harness/compaction.ts:59`) already bounds history against the
model's context window — eve must compact for the model regardless.

**Fix B — the client tail becomes an event log.** This one *is* unavoidable. Append-only
DynamoDB, long-polled by the HTTP Lambda. Keep it behind a narrow interface so
Kinesis/AppSync Events/Momento can replace it later.

**Appends must be idempotent under replay *and* safe under concurrency.** Tail events are
written *mid-step*, token-by-token during a model call. A step that appends and then
crashes before checkpointing will re-run and append again — and a retried model call
streams *different* tokens, so this is divergence, not just duplication. Vercel's platform
owned this; eve now owns it. A naive append-with-counter **breaks under replay**, and a
read-then-increment counter also races when a parent execution and a child session write
concurrently.

The design must nail down five things, not just pick a scheme:

1. **Sequence allocation is atomic and gap-free.** Never read-then-write. As with the inbox, a bare `UpdateItem ADD` followed by a separate put lets a higher `seq` become visible before a lower one commits, so a reader advancing its cursor skips the gap permanently. Allocate a **range per flush** inside the same conditional write that publishes the batch (see 4), and have readers treat a missing `seq` as *not yet committed* rather than *absent*.
2. **Attempt-scoped keys** — `(sessionId, attempt, seq)` — preserving superseded attempts rather than truncating on retry, so replay divergence stays debuggable. But see 3: attempt scoping alone is not an identity scheme.
3. **A *stable logical* event id — `(sessionId, attempt, seq)` is not one.** A freshly allocated `seq` differs on retry, so it cannot deduplicate anything: the retried append gets a new number and lands twice. The id must be derivable from position in the logical event stream (segment + ordinal within segment), not from an allocation counter. Relatedly, "follow the highest attempt" hides the **completed prefix**: replay does not re-emit events from steps that already checkpointed, so a naive latest-attempt filter drops everything before the retried step. Supersession must be **suffix-scoped** — attempt N supersedes only events at or after the retry point, not the whole stream.

4. **Batch the writes.** The append path is token-by-token during a model call, so per-token `PutItem` plus counter contention means hundreds to thousands of sequential DynamoDB round-trips *inside the step*, adding latency to every response and consuming real write capacity. Coalesce: flush every N milliseconds or N kilobytes, allocate `seq` in ranges rather than per event, one item per flush. This is a **user-visible tuning decision** — it sets the client-perceived token cadence, which is the entire reason the NDJSON tail exists — so pick the interval deliberately rather than inheriting it.
5. **Defined client behavior — and this *does* change the stream protocol.** An earlier draft claimed the wire format was unaffected. It is not. `followStreamIterable` (`src/client/open-stream.ts:100`) tracks a single integer `startIndex` advanced per received event; events carry no attempt or generation identifier, and the reducer never sees response metadata. As written, the client cannot detect a rollover, discard superseded events, or reset its cursor. Required: a **generation-aware cursor** — either a composite `(attempt, seq)` cursor on the wire, or an explicit attempt-boundary event the reducer can act on. Duplicates are idempotent on `(attempt, seq)`.

   **"Discard superseded events client-side" is not sufficient on its own**, and this is the
   deepest problem in the design. `runSession` yields each event the moment it arrives
   (`src/client/session.ts:318` — `events.push(event); yield event;`), so by the time
   attempt B exists, attempt A's tokens have already been rendered in a UI, forwarded to
   Slack, or acted on by a consumer. Dropping them from a *future* read changes nothing.
   Steps are at-least-once, and a retried model call streams *different* text, so this is
   visible output that has to be un-said. Pick one and state it:

   - **Retraction semantics on the wire** — an explicit "attempt superseded, discard from `seq` N" event that every consumer must honor. Cheapest to implement, but it pushes the problem onto every client and every channel adapter.
   - **Buffer until checkpoint** — hold model output until the step checkpoints, then release. Correct and simple to reason about, but it forfeits token-level streaming, which is the point of the NDJSON tail.
   - **An independently idempotent outbox** — move model generation behind a boundary that produces the same event sequence on retry. Best semantics, most work.

   This is a **user-visible protocol decision**, not an implementation detail, and it should
   be settled before the client work starts.

**Open sub-problem: where does the attempt id come from?** TypeScript's `StepContext` does
not expose the current retry attempt, so eve cannot simply read it. Options are an
eve-allocated monotonic attempt counter checkpointed at step entry, or deriving generation
from the execution's checkpoint position. Settle this in Phase 1 — the whole scheme rests on it.

### 3. Turn-cancel and session-terminate are different problems
These need separating — an earlier draft of this plan conflated them and wrongly claimed
no stop API exists.

**Session terminate is solved.** Lambda exposes
[`StopDurableExecution`](https://docs.aws.amazon.com/lambda/latest/api/API_StopDurableExecution.html)
(API, CLI, boto3): the execution moves to terminal `STOPPED`, in-progress operations are
terminated, and optional error details can be attached. `terminateSession()` maps to it
directly — no callback machinery, and it is a genuine operational kill switch for wedged
or runaway executions.

**Turn cancel still needs the race**, because the session must *survive* — `Stop` kills the
whole execution, which is exactly what a turn cancel must not do. eve already has the
`{sessionId}:cancel` hook (`src/execution/turn-cancellation-control.ts`); make it a callback
completion the turn races against.

The one remaining unknown is narrower than it looked: whether the SDK supports racing a
callback against an in-flight step, and what happens to the losing branch on replay. If
racing proves unsupported, the fallback — a cancellation flag checked at each step boundary,
bounding worst-case latency to one step — only has to cover **turn** cancel, since terminate
has a real API behind it.

### 4. Lambda is request-scoped — long-lived-process assumptions break
Audit these; each currently assumes a process that outlives a request:

- **Nitro `scheduledTasks`** (`src/internal/nitro/host/schedule-task-routes.ts`) — the in-process cron scheduler cannot run on Lambda. → EventBridge Scheduler (below).
- **`waitUntil`** — see below; this is a public API decision, not an audit item.
- **Per-session MCP connection registry** (`src/runtime/connections/registry.ts`) — already per-session and `dispose()`d, so it is fine, but it now reconnects per step rather than per process. Watch OAuth token cache churn in `scoped-authorization.ts`.
- **Sandbox `shutdown()`** — must become *suspend*, not terminate, and **nothing currently triggers it** (see below).
- **NDJSON response streaming** caps at the Lambda 15-minute limit; client reconnect with `startIndex` covers longer sessions.

### 5. Response streaming mode — a decision, not just a cap
Lambda Function URLs default to **`BUFFERED`**, which returns the response only once the
handler finishes. Under that mode the NDJSON tail delivers *nothing incrementally* and the
streaming UX is silently dead. Incremental delivery requires `RESPONSE_STREAM` invoke mode
**and** Nitro `aws-lambda` support for `streamifyResponse`. Whether the pinned
`nitro@3.0.260610-beta` supports this is **unverified** — put it on the Phase 1 spike list
next to the runner spike; if it does not, the workaround is a separate streaming-only
function entry or client-side polling.

Cost consequence worth pricing before committing: under `RESPONSE_STREAM`, **every
connected tail client pins one HTTP-Lambda concurrent execution for up to 15 minutes**
while long-polling DynamoDB. Idle sessions with an open stream are not free the way they
were on Vercel.

### 6. `waitUntil` — a public API semantic to decide once
`waitUntil` is **not** an internal detail. It sits on the public `ChannelRouteContext`
(`src/channel/routes.ts:40`) and is used by **eight built-in channels** — Slack, Discord,
Telegram, Teams, Twilio, GitHub, Linear, chat-sdk (across nine files; Slack uses it in both
`slackChannel.ts` and `interactions.ts`) — plus `schedule-task.ts` and
`channel-dispatch.ts`. The Slack docs explicitly promise `ctx.waitUntil(...)` for detached
work. Any authored channel may use it.

So "async-invoke the durable function instead" is not a sufficient answer: it covers
starting agent work, but not arbitrary authored post-ack work — posting a Slack
acknowledgment after the 3-second ack deadline is the canonical example, and that is not a
durable invocation.

**Decision: keep the API and reimplement it natively — but the implementation must await,
never assume.** AWS is explicit that Lambda does **not** wait for unresolved promises once
the handler returns or the response stream ends, and the Node.js 24 runtime made that
uniform across buffered and streaming handlers
([runtime announcement](https://aws.amazon.com/blogs/compute/node-js-24-runtime-now-available-in-aws-lambda/),
[streaming docs](https://docs.aws.amazon.com/lambda/latest/dg/config-rs-write-functions.html)).
So *"close the response, then let pending promises drain"* — what an earlier draft of this
plan proposed — is not a semantic eve can lean on. Anything not awaited before the handler
promise resolves is simply dropped when the environment freezes.

`waitUntil` therefore becomes an **explicit registry**, not a reliance on runtime leniency:
`ctx.waitUntil(p)` pushes `p` onto a per-request set; the handler wrapper closes the
response stream (`responseStream.end()`), then `await`s that set, and only then resolves.
Registration must happen before the handler returns — a promise registered from within
already-detached work has nothing left to attach to. Semantics are preserved for every
existing channel and for authored code, because every current caller registers
synchronously during request handling.

Two consequences to document, both real behavioral differences from Vercel:

- **The drain is billed.** Post-ack work now counts against request duration and is bounded
  by the 15-minute handler limit; on Vercel it was neither. A drain that would exceed the
  limit is truncated with no completion signal, so `waitUntil` is for short best-effort work
  — the delayed Slack acknowledgment, a metrics flush.
- **It is not durable.** The registry survives nothing: a crash mid-drain loses the work
  silently. Post-ack work that must survive — anything with an at-least-once requirement —
  belongs in a durable invocation or a queue, not in `waitUntil`. Say so in the channel docs
  alongside the existing `ctx.waitUntil(...)` promise, so authors pick the right tool rather
  than discovering the distinction in production.

### 7. Payload ceilings, and start/delivery idempotency
Async `Invoke` caps at **1 MB** — [raised from 256 KB in October 2025](https://aws.amazon.com/about-aws/whats-new/2025/10/aws-lambda-payload-size-256-kb-1-mb-invocations)
— versus 6 MB sync. An earlier draft said 256 KB; the S3-offload conclusion stands, but any
sizing analysis built on that number was wrong. Durable operations have their own published
table (gap 2), and the two are not interchangeable: the driver→turn hop is a
`CHAINED_INVOKE` at 1 MB, while `STEP`, `CONTEXT`, `WAIT`, and `CALLBACK` get 256 KB —
so a step result is bounded four times tighter than the invoke that carries it.
Note the pricing edge: payloads above
256 KB bill an extra request per 64 KB chunk, so offloading is a cost decision as well as a
limit one. `start()` carries serialized context including the bundle source descriptor and
the initial delivery payload — use the same S3 pointer mechanism Fix A introduces for
snapshots, with a size check at the boundary.

**One offload rule, applied at every DynamoDB write boundary.** DynamoDB items cap at
**400 KB**, and three write paths carry user-controlled content, not just `start()`:
inbox deliveries (arbitrary message content and attachments), event-log appends (tool
results can be large), and snapshots. Specify a single size-check-and-offload helper —
inline below the threshold, S3 pointer above it — and route every write through it.
Handling only `start()` leaves two live failure paths.

**Starts need an idempotency key too — and eve does not currently have one.** Lambda
provides [execution-name idempotency](https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-idempotency.html):
pass the eve session id as the durable execution name and a retried start is a no-op rather
than a second session. But that only works if **every retry presents the same id**, and
today nothing carries one: `RunInput` has no caller-supplied session or idempotency key,
`DeliverInput` has no `deliveryId` (`src/channel/types.ts:342`), and the only request
identifier in the stack is derived from the `x-vercel-id` header
(`channel-dispatch.ts:250`) — which disappears with Vercel.

So Phase 1 needs an **ingress idempotency contract** as new public surface, mapping each
entry point to a stable key:

| Ingress | Natural key |
|---|---|
| Browser / SDK client | client-generated UUID, sent on start and retried unchanged |
| Channel webhooks | provider event id — Slack `event_id`, GitHub `X-GitHub-Delivery`, Stripe-style event ids; every supported provider already sends one, and they are precisely designed for redelivery |
| Schedules | **schedule ARN + scheduled time** (`<aws.scheduler.schedule-arn>` + `<aws.scheduler.scheduled-time>` context attributes). *Not* the execution id: that is unique **per attempt**, so a retried target invocation would present a new key and start a duplicate session — the opposite of what the key is for. |
| Subagent / remote-agent calls | parent-allocated id, derived from the calling step |

Without this, at-least-once webhook redelivery — which every one of these providers does by
design — silently starts duplicate sessions. Deliveries use the same key as `deliveryId` in
the inbox transaction (gap 1).

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
| `cancelRun(id)` — the implementation of `terminateSession()` (`workflow-runtime.ts:184`) | **`StopDurableExecution`** (gap 3). Not a callback completion: completing a callback *wakes* an execution rather than killing it. |
| `resumeHook(sessionCancelHookToken(id))` — turn cancel (`workflow-runtime.ts:252`) | complete the cancel callback |
| `shouldRouteToLatestDeployment()` (`VERCEL_ENV`) | **not** a plain alias swap — see below |

**Preserve per-turn latest-deployment routing deliberately.** Today the long-lived session
driver stays pinned while *each turn* starts against the latest deployment —
`turnWorkflowReference` exists precisely so
`start(turnWorkflowReference, args, { deploymentId: "latest" })` reaches the newest turn
workflow even when the driver is older (`workflow-runtime.ts:102`). Lambda does **not** give
this for free: per the
[invocation docs](https://docs.aws.amazon.com/lambda/latest/dg/durable-invoking.html),
*"When you update an alias, new executions use the new version, while in-progress executions
continue with their original version."* A single long-lived durable execution is pinned at
start, so a days-long session would run stale code forever.

Keep the behavior by dispatching **each turn as its own durable invocation through the
alias** — `context.invoke(name, aliasArn, payload)` chained from the driver, which
checkpoints the result and resumes without re-invoking. New turns then pick up new
deployments while the driver stays pinned, matching today's semantics. `StopDurableExecution`
also becomes per-turn granular, which is a bonus for turn cancel. If this is not done, the
regression must be stated explicitly rather than left implicit.

**But the driver cannot live forever — it needs a rollover strategy.** A single durable
execution is hard-capped at **3,000 operations** and **100 MB of cumulative persisted data**
(`DurableExecutionOperations` and `DurableExecutionStorageWrittenBytes`, both documented with
explicit maxima in the
[monitoring docs](https://docs.aws.amazon.com/lambda/latest/dg/durable-monitoring.html)).
Every turn adds at least one chained `invoke` to the root execution, plus whatever the
driver checkpoints alongside it — so a long-running session walks steadily toward both
ceilings. eve's whole premise is sessions that live for days or months, so this is not a
tail case; it is the expected path.

Design the handoff explicitly in Phase 1:

- **Track all three budgets** as first-class session state: operations (3,000), bytes (100 MB), **and wall-clock age**. The third is easy to forget — a durable execution is capped at **one year**, and a mostly-idle driver reaches that wall without approaching either of the other two. Schedule a rollover wake before the deadline rather than discovering it as a `TIMED_OUT` execution.
- **Hand off to a successor execution** before either ceiling, at a turn boundary where state is quiescent.
- **Do not claim an atomic transfer — fence it instead.** Token-index entries, inbox high-water `seq`, event-log cursor, and the S3 snapshot pointer live in independent stores; no single write moves them together, and an earlier draft of this plan claimed otherwise. What *is* atomic is one conditional update to the DynamoDB session head (Fix A) that bumps `fenceToken` and names the successor — that is the serialization **point**, not the transfer itself. The protocol around it:
  1. **Quiesce.** The predecessor stops at a turn boundary and writes its cursors — inbox high-water `seq`, event-log cursor, snapshot pointer — into the head record as the successor's starting frontier.
  2. **Fence.** One conditional update on the expected `fenceToken` publishes the successor and invalidates the predecessor. From here every operation against *any* store — token index, inbox, event log, snapshot publish — must carry the current fence token and be rejected if stale, so a predecessor that wakes up late (a delayed retry, a slow callback) cannot write behind the successor's back. A fence that only guards the head record fences nothing.
  3. **Reconcile before accepting deliveries.** The successor re-reads the published frontier, replays inbox entries above the recorded high-water mark, re-arms callbacks, and re-checks the token index for entries the predecessor wrote between quiesce and fence — *then* starts serving. A delivery in flight across the boundary is re-processed rather than lost.
  Step 3 is only safe if deliveries are idempotent, which the event log's replay-idempotency scheme (Phase 1) already requires — this is the same invariant, load-bearing in a second place.
- **Keep `sessionId` stable across rollovers.** It is the client-facing identity and the event-log partition key; only the execution behind it changes.

Externalized snapshots make this handoff nearly free: the successor starts with
`{sessionId, snapshotKey, seq}` and reads the frontier snapshot. With inline state, rollover
would mean marshaling the entire session through an invoke payload — the same 1 MB cliff
that already rules inline state out.

Keep the **snapshot format**, `durable-session-migrations/`, and most of
`durable-session-store.ts` — see gap 2: the namespaced-stream read is a legacy fallback that
simply gets deleted, not a read path that must be replaced. Delete
`workflow-callback-url.ts` (Vercel protection-bypass) and
`src/internal/workflow/{validate-world,world-compatibility,local-world-data-directory,development-world-*}.ts`
— the World concept disappears entirely, along with `experimental.workflow.world`
(`src/shared/agent-definition.ts:230`, public experimental surface → `minor` changeset) and
`resolveWorkflowWorldWiring()` (`src/internal/application/compiled-artifacts.ts:264`).

**`@workflow/serde` needs a named owner, or replay will mangle types.** The plan keeps the
snapshot *format*, but the snapshot is encoded by devalue via the Workflow serde layer —
`durable-session-store.ts` documents it explicitly: *"Devalue handles encode/decode so rich
types in the session (URL `FilePart.data`, Buffer, Date, Map, Set) round-trip
structurally."* The AWS SDK brings its own serialization with different fidelity, and a
`Date` or `Map` that silently degrades to a plain object on checkpoint round-trip is
exactly the kind of bug that surfaces only under replay, days later. Decide now: either
vendor the devalue encoding as eve-owned code and pass opaque strings through the AWS SDK,
or supply per-operation `serdes` (the SDK's callback config already accepts one) and prove
the round-trip with a type-fidelity test. `@workflow/errors` and `@workflow/utils` need the
same call — their error shapes feed `harness/workflow-stream-error.ts` and
`semantic-errors/rules/workflow.ts`.

**Decide how AWS service calls are made — this is an unstated workstream.** The design puts
DynamoDB (three tables, `TransactWriteItems`), S3, and Lambda control-plane calls
(`Invoke`, `SendDurableExecutionCallbackSuccess`, `StopDurableExecution`,
`GetDurableExecution`) directly in the request path, but nothing has said how those calls
are issued. The repo's "nitro as the only runtime dependency" invariant makes this a real
fork:

- **Vendor `@aws-sdk/client-*`** — pulls a large smithy dependency tree, working directly against the install-size and cold-start goals that invariant exists to protect.
- **Hand-roll SigV4 over `fetch`** (aws4fetch-style) — a few hundred lines, no dependency tree, and precisely how `@ai-sdk/amazon-bedrock` already talks to Bedrock. **Recommended.**

Either way it implies a credential-resolution story off-Lambda (for `eve dev` against real
AWS) and a check of whether `@aws/durable-execution-sdk-js` drags in `@aws-sdk/client-lambda`
transitively. Settle it in the Phase 1 inventory: discovered late, it either bloats the
bundle or forces a signing-layer rewrite.

Vendor **both** `@aws/durable-execution-sdk-js` and `@aws/durable-execution-sdk-js-testing`
through the existing mechanism (`packages/eve/scripts/vendor-compiled/index.mjs`, one
per-package file each) so the "one runtime dependency" invariant holds. The testing SDK is
not optional here despite its name — the `local` `DurableBackend` is built on
`LocalDurableTestRunner` and ships as part of `eve dev`, so it must be vendored like any
other runtime dependency rather than left a `devDependency`.

### Local development — introduce a `DurableBackend` seam

Local iteration must not regress; today `eve dev` is a single process with a TUI, hot
reload of authored sources, and a dev-world RPC server
(`src/internal/workflow/development-world-{server,client,protocol,codec}.ts`) that keeps
run state alive across Nitro worker reloads.

**No SAM.** The local runner from `@aws/durable-execution-sdk-js-testing` is the local
runtime. Rather than binding `workflow-runtime.ts` directly to either SDK, put thin
eve-owned interfaces behind it.

**One five-method `DurableBackend` is too narrow.** `startExecution` / `completeCallback` /
`getExecution` / `appendEvents` / `tailEvents` omits stop/terminate, token ownership, inbox
operations, callback generations, and snapshot storage — most of what the preceding gaps
actually introduced. It also hides a conflict: resolving a session through *runner
operations* locally cannot satisfy the requirement that token→session ownership survive
long idles when **no callback is armed** (gap 1). Split it into five narrow interfaces with
independent local/AWS implementations:

| Interface | Responsibility | Local | AWS |
|---|---|---|---|
| `ExecutionControl` | start, stop, describe, rollover | runner | `Invoke` / `StopDurableExecution` / `GetDurableExecution` |
| `TokenOwnership` | token → session (session lifetime), token → armed callback (short) | disk/memory | DynamoDB, two lifetimes |
| `DeliveryInbox` | enqueue (transactional dedupe), ordered drain, high-water | in-memory queue | DynamoDB + `TransactWriteItems` |
| `EventLog` | append with attempt/seq, tail from cursor | in-memory ring | DynamoDB + long-poll |
| `SnapshotStore` | put/get with size-based S3 offload | disk | DynamoDB + S3 |

The seams are then independently testable, and the local backend stops being asked to
emulate semantics the runner does not have.

Two implementations of each:

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
`resolveProductionNitroPreset()` (`create-application-nitro.ts:84`) currently returns
`"vercel" | undefined`; change it to return `"aws-lambda"` unconditionally and drop the
`process.env.VERCEL` probe. Build emits two entries: the Nitro handler and the
durable handler. Function URL invoke mode must be `RESPONSE_STREAM` (gap 5). Replace `vercel-build-output-config.ts` / `build-vercel-agent-summary.ts`
with a **deploy manifest** (`.eve/aws-manifest.json`) — the CDK contract, and the natural
successor to the Vercel dashboard summary. It must declare **logical resources and content
hashes**, not concrete deployment-created identifiers: function entries, cron expressions,
required env, logical table *roles* (`hooks`, `events`, `snapshots`), and a sandbox image
content hash. Physical table names and MicroVM image ids are created by CDK at deploy time,
so baking them into a build artifact inverts the dependency and makes builds
environment-specific. Delete `cron-handler-route.ts`,
`vercel-build-prewarm.ts`.

### Models — Bedrock
Vendor `@ai-sdk/amazon-bedrock`. A bare string model id resolves to Bedrock
(`src/runtime/agent/resolve-model.ts`); credentials come from the Lambda execution role,
not an API key. Delete `src/internal/gateway.ts`, `src/internal/runtime-model.ts`'s
`formatLanguageModelGatewayId()`, and the entire gateway setup flow
(`src/setup/{ai-gateway-api-key,validate-gateway-key,gateway-models}.ts`,
`src/setup/boxes/{detect-ai-gateway,apply-ai-gateway-credential}.ts`); `WiringMode`
in `src/setup/state.ts:101` collapses to a single mode.
`DEFAULT_AGENT_MODEL_ID` → a **concrete** Bedrock id, not a wildcard. Get the id shape
right, because an earlier draft of this plan had it backwards. Anthropic dropped the dated
`…-v1:0` suffix starting with Sonnet 4.6, so for Sonnet 5 the
[model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html)
publishes:

- `anthropic.claude-sonnet-5` — the **base foundation-model id**, valid on both `InvokeModel`
  and `Converse`. It is not a Messages-API-only id, and there is no dated Converse variant to
  prefer over it.
- `us.` / `eu.` / `au.` / `global.` `anthropic.claude-sonnet-5` — **geo inference profiles**
  over that same model. Different routing and residency, same API surface.

Pin one exact string, and verify it against the model card **and** the vendored
`@ai-sdk/amazon-bedrock` version at the moment it lands — provider releases lag new model
ids, and a provider that predates Sonnet 5 will reject it regardless of what Bedrock accepts.

Which of the two to pin is a residency decision, not a routing detail, and belongs in the
same commit as the id. A geo profile such as `us.` may route to any region in its geography;
`global.` has no residency constraint at all; the bare model id stays in the calling region
but forfeits cross-region capacity headroom. The choice also changes the IAM shape: with a
profile, the execution role needs `bedrock:InvokeModel` /
`InvokeModelWithResponseStream` on the inference-profile ARN **and** on the underlying
foundation-model ARNs in every region the profile can reach; with the bare model id, only
the single-region foundation-model ARN. **Recommendation: default to `us.anthropic.claude-sonnet-5`**
for capacity, and document the override — deployments under residency constraints point
`agent.ts` at a single-region profile
(`arn:aws:bedrock:<region>:<account>:inference-profile/...`) or the bare model id.
**`src/compiler/model-catalog.ts` needs a decision**: it fetches the Gateway catalog at
build time to bake `contextWindowTokens`/`maxOutputTokens` into the manifest. Bedrock's
`ListFoundationModels` does not reliably expose context windows — bake a static catalog
and let `agent.ts` override.

**Bedrock-as-default partly contradicts the credential-free local story.** The `local`
durable backend needs no AWS credentials, but a Bedrock default means `eve dev` and local
e2e need them for *every model call* — so "no AWS account required" is only true of the
durable layer, not of running an agent. Resolve it explicitly: `@ai-sdk/anthropic` and
`@ai-sdk/openai` are **already vendored**, so keep direct API-key providers as the
first-class local path, and state what bare-string resolution does off-Lambda (recommended:
resolve to Bedrock only when AWS credentials are present, otherwise require an explicit
provider-qualified id and fail with a clear message rather than an opaque credential error).

### Sandbox — Lambda MicroVM
Maps cleanly onto the existing `SandboxBackend` interface
(`packages/eve/src/shared/sandbox-backend.ts`):

| `SandboxBackend` | MicroVM |
|---|---|
| `prewarm({templateKey, bootstrap, seedFiles})` | zip Dockerfile+seed → S3 → create MicroVM image (snapshot); `templateKey` → image id |
| `create({sessionKey, existingMetadata})` | `run-microvm` from snapshot, `resume-microvm` when metadata holds a live suspended id, **or cold-rehydrate when it has expired** |
| `captureState()` | `{ microVmId, endpoint }` |
| `shutdown()` | **`suspend-microvm`**, not terminate — memory+disk preserved, sessions reattach *within the cap* |

**Suspended state is capped at 8 hours, not "days".** MicroVM state preservation and total
MicroVM lifetime both top out around 8 hours, and `idlePolicy.suspendedDurationSeconds`
auto-terminates after a configured suspended duration. eve's durable sessions can idle far
longer than that, so:

- **`create({existingMetadata})` must handle resume-not-possible as a normal path, not an error.** A session idle past the cap comes back to a dead sandbox and needs **cold rehydrate**: recreate from the snapshot image and re-seed files. This is arguably the harder half of the lifecycle work and was missing from this plan entirely. Note eve's sandbox contract already says sessions are keyed per durable session and survive redeploys — that promise now has a time bound, and the rehydrate path is what keeps it honest.
- **Docs must say sandbox state is not durable across long idle periods.** Files written by the agent do not survive an 8-hour gap. Anything that must persist belongs outside the sandbox.
- **The upside:** the cap makes the leaked-MicroVM concern below self-limiting. With `idlePolicy` auto-terminate configured, a sandbox eve forgets about cleans itself up — a strong argument for the lifecycle-policy option over an eve-side sweeper.

**Nothing triggers `shutdown()` on Lambda.** Today suspension rides on
`src/internal/nitro/host/sandbox-shutdown-plugin.ts`, which hooks `SIGINT`/`SIGTERM` and
Nitro close — none of which fire reliably on Lambda. The execution environment simply
freezes and the sandbox dies silently, leaking a running MicroVM. Suspend needs an
explicit trigger; pick one in Phase 3 rather than discovering it in production:
**suspend-at-turn-end** (simplest, from the durable function itself), an EventBridge
sweeper over a DynamoDB lease table, or MicroVM idle lifecycle policies doing the work
natively. The last is the most attractive if the policy granularity fits, since it removes
eve from the loop entirely.

**The other real gap:** `@vercel/sandbox` gave a command SDK (`runCommand`/`readFile`/`writeFile`);
MicroVM gives you a raw HTTPS endpoint with JWE auth. eve must ship a small **in-image
agent server** implementing the `SandboxSession` surface, baked into the image the root
`Dockerfile` produces. Budget real time for this; it is the sandbox long pole.

**Settle the credential and tenant-isolation contract before any endpoint metadata is
persisted.** This is security-critical and easy to get wrong, because `captureState()`
writes reconnect metadata into durable session state that outlives the process, and
`create()` reattaches suspended sessions from it. A sandbox executes model-generated code,
so a token that leaks or over-scopes is a direct cross-tenant code-execution path. Specify:

**The AWS token cannot carry session identity — so this needs two layers.** An earlier draft
said the in-image agent must "reject any request whose token does not bind to its own
session." That is unimplementable. Per the
[networking docs](https://docs.aws.amazon.com/lambda/latest/dg/microvms-networking.html), the
`X-aws-proxy-auth` JWE is scoped to **a MicroVM id, a set of allowed ports, and an
expiry** — not an account, function, or session. And decisively: *"Lambda removes
`X-aws-proxy-*` headers before forwarding the request to your application."* The in-image
server never sees the token and cannot inspect it.

| Layer | Purpose |
|---|---|
| AWS JWE (`X-aws-proxy-auth`) | endpoint **ingress** only — proves the caller may reach this MicroVM on this port. Minted with `create-microvm-auth-token`, short-lived (verify the exact ceiling; the docs' example uses 30 minutes). |
| eve application capability | **session binding** — a separate signed token in an ordinary header, minted by the app runtime and verified *inside* the VM against the session it was provisioned for. |

Both are required: the JWE alone means anyone holding it reaches the VM with no session
check, and an app capability alone cannot get past the endpoint. Remaining rules:

- **Mint both in the app runtime**, never inside the sandbox. Persist a *reference* in session state, not the tokens — the rule `src/runtime/connections/scoped-authorization.ts` already applies to connection tokens.
- **Expiry and rotation.** Both tokens are far shorter-lived than a session, so the reattach path must **re-mint** rather than replay stored tokens; cold rehydrate mints fresh.
- **Revocation** on session termination and on backend switch (`SandboxBackendSessionState.backendName` mismatch). The in-image capability check is what makes revocation meaningful, since AWS tokens cannot be revoked before expiry.
`defaultSandbox()` probe chain (`src/public/sandbox/backends/default.ts`) becomes
MicroVM (when running on Lambda) → Docker → microsandbox → just-bash; the local backends
stay and remain the dev path. Delete `src/execution/sandbox/bindings/vercel*.ts` (8 files)
and `src/public/sandbox/vercel-sandbox.ts`; drop `resolveVercelProjectIdFromEnvironment()`
from template-key derivation (`src/runtime/sandbox/keys.ts`) in favor of an account/function-scoped key.

### Schedules — EventBridge Scheduler
Discovery/compile/dispatch stay as-is. Only the trigger changes: `eve build` emits each
`defineSchedule` cron into the deploy manifest and CDK creates EventBridge Scheduler rules.
Nitro's in-process `scheduledTasks` registration is removed. `eve dev`'s manual dispatch
route (`POST /eve/v1/dev/schedules/:scheduleId`) is unchanged.

**Be concrete about the invoke mechanics.** EventBridge Scheduler invokes Lambda with a
raw event, not an HTTP request, so reaching a Nitro route means synthesizing a
Function-URL-shaped event payload. Going the other way — pointing EventBridge at the
Function URL — does not work: the URL is `AWS_IAM`-authed and EventBridge API destinations
do not sign SigV4. **Decision: direct Lambda invoke with a synthesized event.** One
consequence to carry forward: the unguessable-cron-path secret from `cron-handler-route.ts`
is redundant under IAM auth and should be dropped rather than ported.

**Spell out the IAM contract for that path.** Each schedule gets a CDK-created execution
role with `lambda:InvokeFunction` on exactly the target function's **alias ARN** (not
`$LATEST` — the versioning section pins the driver to an alias), trusted to
`scheduler.amazonaws.com` and scoped with `aws:SourceArn` to the schedule's own ARN so the
role cannot be assumed on behalf of a different schedule. Note what this path does *not*
touch: Scheduler invokes the function directly, so it never crosses either HTTP surface —
see the auth topology below, where EventBridge Scheduler is deliberately absent from both
rows.

**Idempotency needs a stated retry model, not just a stable key.** `(schedule ARN,
scheduled time)` is stable across Scheduler's own retries — a failed target invocation is
retried per the schedule's retry policy (`MaximumRetryAttempts`, optional DLQ) with the same
scheduled time — so duplicates collapse onto one conditional write keyed on that pair. Lambda-side
retries collapse onto the same key too: under async invocation, Lambda's two automatic
retries replay the identical event payload, scheduled time included. Neither layer stamps an
attempt number, so eve cannot distinguish a retry from a first attempt out of the event, and
should not try to. The contract instead:

- dispatch **claims** the run with a conditional write on `(scheduleArn, scheduledTime)`, committed in the same write that records the claim — first attempt wins, every later one fails the condition;
- a handler that finds the key already claimed returns **success**, not error. Returning an error is precisely what drives another retry against a schedule that already ran;
- if a first-versus-retry distinction is ever wanted (alerting, say), it comes from an eve-owned attempt counter on the claim record — never from the event.

Pick the invocation type deliberately rather than inheriting a default: async hands eve
Lambda-level retries it must absorb on top of Scheduler's, while sync (`RequestResponse`)
keeps all retry behavior under the schedule's own policy and surfaces exhausted attempts to
its DLQ. **Decision: sync invoke, Scheduler retry policy, DLQ** — one retry authority is
worth more here than the fire-and-forget latency.

### Auth — needs two surfaces, and two interface changes
An earlier draft proposed a single `AWS_IAM` Function URL with a `sigv4()` framework
default. **That does not work** and the topology must be settled before implementation.

`AWS_IAM` requires *every* request to be SigV4-signed. Slack, GitHub, Twilio, Discord,
Telegram, and Teams webhooks cannot sign; neither can a browser holding a bearer token, nor
an OAuth provider redirecting to a connection callback. A single IAM-authed URL locks out
most of eve's inbound traffic.

**Topology: two surfaces.**

| Surface | Auth | Carries |
|---|---|---|
| Public (Function URL `NONE`, or API Gateway) | eve's own `routeAuth` chain — per-channel HMAC verification, `oidc()`, `jwtEcdsa()`, `httpBasic()` | browser clients, channel webhooks, OAuth callbacks |
| Internal | `AWS_IAM` | agent→agent calls, internal invokes |

EventBridge Scheduler is deliberately in neither row: per the schedules section it invokes
the function directly through its own execution role and never reaches eve over HTTP, so it
is governed by that role's `lambda:InvokeFunction` grant rather than by either URL's auth.

**WebSocket channels have no home in this topology.** `WS` is a public export
(`src/public/definitions/channel.ts:28`), `ChannelRouteMethod` includes `"WEBSOCKET"`, and
`docs/channels/custom.mdx:144` documents the full lifecycle contract (`upgrade`, `open`,
`message`, `close`, `error`) — but **Lambda Function URLs are HTTP-only**; AWS puts
WebSockets behind API Gateway, with connection state and a callback API for server-initiated
sends. That is a fundamentally different execution model from eve's in-process handler
hooks. Decide explicitly: add an API Gateway WebSocket surface with a connection-state
store and an adapter mapping eve's hooks onto `$connect`/`$disconnect`/`$default`, or
**declare `WS` a breaking removal** and say so in the changeset. Silently shipping a
topology where `WS()` routes cannot bind is the one outcome to avoid.

**Two surfaces means route isolation must be enforced, not assumed.** Both URLs front the
same Nitro handler, and AWS performs **no authentication at all** on a `NONE` Function URL —
so without an explicit boundary the public URL reaches every internal route, and the IAM
surface becomes decorative. Enforce it one of two ways: separate handler entries with
disjoint route tables, or a single handler with a **surface-tagged route allowlist** checked
before dispatch in `configure-nitro-routes.ts`. Either way it needs a test asserting that
internal routes 404 on the public surface — this is the kind of boundary that silently
regresses when a route is added.

The existing `oidc()` / `jwtEcdsa()` / `jwtHmac()` / `httpBasic()` strategies are already
generic and cover Cognito, so the public surface is well served today. Each channel's
`verify.ts` already does constant-time signature verification — that is the real webhook
authentication and it is unchanged. Delete `vercelOidc()` from
`src/public/channels/auth.ts` and `src/public/agents/auth.ts`, plus
`src/runtime/governance/auth/vercel-oidc-project.ts` and `src/shared/vercel-project.ts`.

**Two public interfaces must change**, and both are breaking:

1. **`AuthFn<TEvent = Request>`** (`src/public/channels/auth.ts:495`) receives only the request. Lambda validates SigV4 itself and exposes the caller principal at `requestContext.authorizer.iam` — unreachable through a bare `Request`. A `sigv4()` inbound strategy needs the authorizer context threaded into the auth surface.
2. **`OutboundAuthFn = () => Promise<{ headers }>`** (`src/public/agents/auth.ts:11`) receives no method, URL, or body. **SigV4 signs over all three**, so agent→agent SigV4 is impossible without widening this signature. This blocks the internal surface, not just a nicety.

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
**`apps/templates` and `apps/docs` need the same explicit call.** 71 files under `apps/`
mention Vercel, including `apps/templates/web-chat-next` (README and agent channel config)
— and templates scaffold the very deploy story Phase 0 deletes. Left alone, `eve init`
generates projects pointing at a workflow that no longer exists, which is worse than
generating nothing. Decide keep/rewrite/drop per template alongside the framework
integrations; `apps/docs` (including `registry.json`) carries the same Vercel-shaped
assumptions.

Rename: `@vercel/eve-catalog` workspace package, `ghcr.io/vercel/eve` image, the
`vercel-sandbox` OS user in the root `Dockerfile`, and the repo URL in
`packages/eve/package.json`. Drop `@vercel/oidc`, `@vercel/sandbox`, `@vercel/sdk`,
`@vercel/detect-agent` (or vendor the last one's heuristic — it is trivial and useful),
and the `@vercel/connect` scaffolding in `src/setup/scaffold/**`.

### Touchpoints easy to miss
**229 non-test source files** mention Vercel (`grep -rli vercel src --include='*.ts'`, minus
`*.test.ts`). The inventory above covers the load-bearing
majority; these three clusters are easy to overlook and each needs an explicit disposition:

- **The TUI remote-attach flow** — `src/services/dev-client/{vercel-auth-error,request-headers,credential-gate}.ts` plus `src/cli/dev/tui/{remote-auth*,remote-connection*,vercel-status,vercel-trusted-sources*}.ts`. The whole "attach the local TUI to a deployed agent" experience authenticates against a Vercel project. It needs an AWS story (SigV4 from the local credential chain is the obvious one) or explicit removal — do not let it rot half-ported.
- **Error classification** — `src/harness/semantic-errors/rules/{gateway,workflow}.ts`, `src/harness/workflow-stream-error.ts`, `src/harness/model-call-error.ts`. These are keyed to Gateway and Workflow-SDK error *shapes*. Without Bedrock and durable-SDK equivalents, errors silently misclassify and users get misleading hints — worse than no classification.
- **Channel setup** — `src/setup/channel-setup-{slack,deployment}.ts` embed deployment URLs derived from the Vercel project link, which the webhook-registration flows depend on.

### e2e
`.github/workflows/e2e-vercel.yml` and the shared-Vercel-project model in `e2e/README.md`
are replaced by a CDK stack deployed per CI run into a test account, with the 23 fixtures
under `e2e/fixtures/` deployed behind one Function URL each and torn down after.
`e2e/provision/tools-sandbox.ts` is rewritten against MicroVM images.
`e2e-local.yml` needs no Vercel and keeps working throughout.

---

## Phases

**Phase 0 — De-Vercel without changing behavior.** *Mergeable, but local-dev-only.*
Delete `eve deploy`/`eve link`, the `src/setup/` Vercel surface, the framework
`vercel-*` output writers, `vercel-agent-summary`, and the `@vercel/*` deps.
Pin the sandbox to Docker and the workflow world to `@workflow/world-local` so the tree
stays green. Rebrand. This is mostly deletion and lands fast — it makes every later phase
smaller.

**Order the preset switch deliberately.** `aws-lambda` preset + still-Vercel-shaped workflow
output is an untested combination that may not build at all. Either make that build check
the **first task of Phase 0**, gating everything after it, or defer the switch into Phase 1
where the workflow emitter is replaced anyway. Scheduling it mid-phase — as an earlier
draft did — risks discovering the incompatibility with half the deletions already landed
and no clean way back.

Be honest about what it costs: **after Phase 0 there is no production deployment path at
all** until Phases 1 and 4 land — the deploy CLI is gone, Vercel output emission is gone,
and the AWS runtime does not exist yet. That is acceptable for a hard fork, but it is not
"independently shippable" in the usual sense.

**Phase 1 — Durable execution on Lambda. *The long pole.***
Vendor `@aws/durable-execution-sdk-js`.

**One earlier spike is closed by documentation:** `LocalDurableTestRunner` *can* start an
execution without awaiting it — the testing docs show `runner.run()` retained as a promise
while callbacks complete concurrently. The local-dev story stands.

**Four unknowns remain:**

1. Can a callback be **raced against an in-flight step**, and what happens to the losing branch on replay? Gates *turn* cancel only — session terminate has `StopDurableExecution` behind it.
2. **What does the parent observe when a per-turn child execution is stopped mid-flight?** The per-turn `context.invoke` design makes `StopDurableExecution` turn-granular, but the driver's checkpointed `invoke` operation sees *something* when its child is stopped — an error result, presumably. If the SDK retries a failed child invocation it would **resurrect a cancelled turn**, which is worse than not cancelling. Same feature as unknown 1; spike them together.
3. **What does a full session snapshot cost per step, against which budget?** The service limits are documented — 256 KB per `STEP`, 1 MB per `CHAINED_INVOKE`, 100 MB and 3,000 operations per execution (gap 2) — so this no longer gates *whether* to externalize, and it is not a hunt for an unpublished ceiling. What is unmeasured is how much of each budget the durable SDK's own envelope consumes before eve's bytes: that sets the inline-vs-spill threshold for intra-turn step results and calibrates how close a busy session gets to the operation ceiling before rollover.
4. **How is the stream attempt id allocated?** `StepContext` does not expose the retry attempt, so the generation-aware cursor (gap 2) needs an eve-owned scheme. On the critical path for the client protocol change.

**Phase 1 needs a real AWS gate, not just the local backend.** The CDK stack is scheduled in
Phase 4, but the local runner cannot validate IAM, service quotas, callback races, alias
behavior, the 3,000-operation ceiling, or streaming — every one of which this plan now
depends on. Stand up a **minimal spike stack early in Phase 1**: one durable Lambda, a
callback dispatcher, the DynamoDB tables, and a streaming Function URL. It does not need to
be the production stack; it needs to answer the unknowns above before the design hardens
around guesses. Landing the `aws` backend "last in Phase 1" against no deployed environment
is how integration failures get discovered in Phase 4.

**Plus one cheap smoke test, deliberately not struck off.** The Nitro docs say the
`aws-lambda` preset supports `awsLambda: { streaming: true }` over
`awslambda.streamifyResponse`, and that is almost certainly right — but gap 6's `waitUntil`
registry needs the *awaited* case confirmed on a real Function URL: that work explicitly
awaited after `responseStream.end()` still completes, that the client has already received
the full response by then, and that the drain counts against billed duration as expected.
Half a day on a trivial streaming handler is cheap insurance for a public API semantic. Note
what this smoke test is *not* checking any more: whether Lambda implicitly drains unawaited
promises. It does not, that is documented, and gap 6 no longer depends on it.

Then define the `DurableBackend` seam and land the **`local` backend first** — it keeps
`eve dev` and the test tiers working throughout the rewrite, and it is the cheapest place
to prove the design against a fixed `Runtime` interface. Delete the legacy `eve.session`
stream branch from `durable-session-store.ts`; whether snapshots also move to DynamoDB+S3
in this phase is spike 3's call, not a given (gap 2, Fix A).
Build the event log with an explicit replay-idempotency scheme, and the hook-token index
with a park/retry policy for the deliver-before-mapping race. Rewrite `workflow-entry.ts` /
`turn-workflow.ts` / `workflow-steps.ts` against `DurableContext`, keeping
`workflow-runtime.ts`'s `Runtime` interface fixed so channels and the harness are
untouched. Delete `src/internal/workflow-bundle/` and `src/internal/workflow/`. Add the
`aws` backend last.

**Phase 1 is not done until all of these land** — several are assigned to Phase 1 elsewhere
in this document but are easy to lose, and most are public surface, so `AGENTS.md` requires
their docs in the same PRs:

- [ ] Side-effect idempotency audit — classify every externally-visible effect in a step (risk 2)
- [ ] Durable inbox: transactional seq+dedupe, drain protocol, generational callbacks — for **all four** reusable hook families (gap 1)
- [ ] Ingress idempotency contract — `RunInput`/`DeliverInput` changes, client UUID on the wire, provider-event-id plumbing through every channel adapter (gap 7)
- [ ] Session reservation state machine (`RESERVED`→`STARTING`→`ACTIVE`) with sweep/recovery and DLQ reconciliation
- [ ] Driver rollover: operations, bytes, **and age** budgets, plus fenced handoff — quiesce, one conditional `fenceToken` bump on the session head, successor reconciles before serving
- [ ] Serde ownership decision + type-fidelity test
- [ ] AWS service-call layer (SigV4-over-`fetch` vs vendored clients) and off-Lambda credential resolution
- [ ] Event-log batching policy — sets client-perceived token cadence
- [ ] Stream protocol: generation-aware cursor **and** the chosen retraction semantics

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

**Phase 5 — residual docs and e2e consolidation.**
Whatever cross-cutting narrative is left after the per-phase work below: the deployment
guides as a set, `docs/concepts/execution-model-and-durability.md`, and the consolidated CI
e2e workflow.

> **Docs and e2e do not belong in a trailing phase.** `AGENTS.md` requires public-behavior
> changes to update docs *in the same PR*, so deferring all of `docs/**` to Phase 5 violates
> the repo's own rules. Each phase above carries its own docs: Phase 1 owns
> `execution-model-and-durability.md`, Phase 2 the model docs, Phase 3 `sandbox.mdx`
> (including the 8-hour contract), Phase 4 `schedules.mdx` and auth. Likewise **Phase 4 is
> not done until a deployed AWS e2e exercises IAM, callbacks, DynamoDB, streaming, and
> EventBridge together** — the integration failures live in the seams between them, and no
> amount of local testing finds those.

---

## Verification

- **Phase 0:** `pnpm build && pnpm typecheck && pnpm test` all green; `pnpm guard:invariants` passes; `grep -ri vercel packages/eve/src` returns only intentional leftovers.
- **Phase 1:** the tightest signal is `LocalDurableTestRunner`-backed integration tests over `workflow-runtime.ts`'s `Runtime` interface — start a session, deliver a message, tail the event log, cancel a turn, resume after a simulated interruption. Three cases the new design specifically exists for, and which a naive test list misses:
  - **Force a step retry mid-stream** and assert whatever retraction semantics gap 2 settles on — not merely "the client discards superseded events," which the design itself says is insufficient once output has already been yielded. If the choice is wire-level retraction, assert the retraction event is emitted and honored; if buffer-until-checkpoint, assert nothing was yielded before the checkpoint at all.
  - **Two concurrent deliveries**, asserting both are processed in arrival order and neither is lost — the failure the durable inbox exists to prevent.
  - **A type-fidelity round-trip** over the snapshot (`Date`, `Map`, `Set`, `Buffer`, URL `FilePart.data`) to catch serde degradation.

  Then `eve dev` end-to-end against `apps/fixtures/weather-agent`, driving a real multi-turn conversation and confirming events stream and a HITL approval round-trips.
- **Phase 2:** run an agent against a real Bedrock model id; confirm the compiled manifest carries correct context-window limits and that compaction triggers at the right threshold.
- **Phase 3:** `e2e/fixtures/agent-tools-sandbox` locally against Docker, then a deployed MicroVM: create a session, write a file, let it idle into suspend, resume **within the 8-hour cap**, confirm the file survived. Then the case that actually matters — force expiry past the cap and confirm **cold rehydrate** recreates the sandbox and re-seeds files rather than erroring.
- **Phase 4/5:** deploy the CDK stack to a test account and run `eve eval` against the Function URL — the same shape as today's `e2e-vercel.yml`, different target.

## Open risks

1. **Durable-function payload/checkpoint costs.** eve checkpoints a full session snapshot per step (`durable-session-store.ts`), and metering bills per-operation *and* per-payload-byte. Externalizing snapshots to S3 (gap 2, Fix A) removes the ceiling risk and most of the cost; what remains is calibration — spike 3 sizes the inline-vs-spill threshold. The residual risk is **operation count**, not bytes: with rollover in place a session survives indefinitely, but rollover frequency is now a cost driver worth measuring under a realistic turn cadence.
2. **Replayed side effects — the most dangerous item on this list.** "Put nondeterminism inside `context.step()`" is necessary but **not sufficient**: steps are **at-least-once**, so an interrupted step re-runs and repeats what it already did. eve's steps are full of externally-visible effects — adapter delivery to Slack/Discord/etc. (`workflow-steps.ts:225`), event-hook emission, tool execution, and **child agent session starts** (`childRuntime.run()` at `dispatch-runtime-actions-step.ts:145`). A retry can re-post a message, re-run a tool with real-world consequences, or spawn a duplicate subagent session. Every effect needs an explicit disposition: a **stable idempotency key**, **at-most-once** via a pre-committed intent record, or a documented **no-retry** step configuration. Auditing `src/harness/tool-loop.ts` for determinism is the smaller half; enumerating and classifying the side effects is the larger one, and it belongs in Phase 1 — not in production when a user gets two identical Slack messages.
3. **Concurrent callback completion.** Addressed by the durable inbox (gap 1), but the pattern recurs for every reusable hook — see the hook inventory there.
4. **MicroVM API maturity.** It is new; confirm the JS SDK surface, per-account MicroVM quotas, and image build times before committing Phase 3's schedule.
5. **The local runner is a test harness doing a dev-runtime job.** It is the right call — it is the only credential-free, container-free option — but it is not what AWS designed it for. Undocumented limits (max invocations per runner, concurrent executions, payload ceilings) may surface only under a real `eve dev` session. The Phase 1 spike and the `DurableBackend` seam exist to keep that discovery cheap.
6. **Streaming concurrency cost.** Under `RESPONSE_STREAM`, each connected tail client pins one HTTP-Lambda concurrent execution for up to 15 minutes while long-polling DynamoDB. At even modest concurrent-session counts this dominates the compute bill and can hit account concurrency limits — model it before committing to long-poll over a push transport.
7. **Sandbox state has an 8-hour ceiling.** Agent-written files do not survive a longer idle gap, so the cold-rehydrate path is load-bearing, not a fallback. Any agent workflow that assumes a persistent workspace across days needs external storage — a user-visible semantic change from Vercel Sandbox that belongs in the docs, not just the code.
