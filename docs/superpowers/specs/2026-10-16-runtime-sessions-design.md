> 2026-10-18 review: deferred by user decision. This historical design must be revised around upstream component-owned sessions, cache backing, and current AR/MTP allocation paths before implementation. See the 2026-10-18 minimal component-cache plan.

# Runtime session capacity without model reloads

Status: approved by the user on 2026-10-16. Implementation has not started.

Deferred on 2026-10-17: first implement and measure disk-only streaming persistence;
revisit slot resizing afterward using the resulting memory evidence.

## Outcome and constraints

Keep Qwen3.8 Flash Next weights, MTP weights, tokenizer, vision model and shared
execution resources loaded while changing the number of resident execution
sessions. Preserve f16 KV and the configured 262144-token context per session.
Support up to four sessions in the user's deployment, releasing sessions so
Qwen3 Embedding 8B can run alongside it. Do not cancel active generation to
make room. When insufficient memory can be released, queue the competing work.

Production remains Linux x86-64 Strix Halo gfx1151. Use the pinned Nix/CMake
toolchain. Preserve unrelated work and the existing remote deployment workflow:
Gufo through the tgrid remote and the multi-backend image build on node-strix;
llama-swap built under WSL and copied with scp.

This is runtime capacity management, not KV quantization, context truncation,
paged attention, arbitrary command-line mutation or a new model variant system.

## Existing behavior and affected boundaries

- `src/cli/serve/continuation_cache.cpp` creates all execution states eagerly.
  Its execution entries and snapshot entries share a fixed indexed container.
- `text_model_runner.cpp` owns the shared runner and continuation/disk caches.
  Model state creation already uses a factory independent of model loading.
- `text_generation_scheduler.cpp` owns inference work on one scheduler thread;
  capacity and session reporting currently assume a fixed pool.
- Flash Next `kernels/rocm/executor.cpp` allocates f16 context storage per
  session. Destruction calls `PreserveSnapshots`, which can allocate copies of
  borrowed rows. Creation uses device synchronization. Neither operation should
  be initiated concurrently from an HTTP handler.
- llama-swap's FIFO scheduler owns queue/load reservations. Its memory pool and
  sharedID resolver currently reason about static per-model `memoryGB` values.
- `convert_models_ini.py` generates the deployed YAML; configuration support
  must survive regeneration rather than existing only as a hand edit.

## Gufo interface

Keep `--sessions N` as the initial resident session count. Add the canonical
option `--sessions-max N`. When omitted, preserve current fixed-capacity behavior
and do not enable mutation. Validate 1 <= initial <= maximum <= 8. The intended
deployment starts with two and permits four; actual sizing is decided from
measured memory profiles before deployment.

Expose `GET /admin/sessions` and `PUT /admin/sessions` for enabled backends.
PUT accepts an integer `target` and an opaque client operation ID. GET reports
process-instance ID, operation ID, target, resident, busy, retiring, maximum,
phase and an optional error. Phases are stable, draining, resizing and failed.
Also report measured session allocation bytes and cache/staging usage separately;
these are not advertised as a complete process-memory measurement.

PUT acknowledges acceptance with 202; it does not claim memory was released.
Repeating an operation ID with the same target is idempotent; conflicting reuse
or a different operation during a transition returns 409. Invalid targets return
400; unavailable/unsupported backends reject without changing state. GET is the
source of truth after timeout or uncertain delivery. Backend restart changes the
process-instance ID, invalidating previous acknowledgements.

Management routes must not be forwarded through llama-swap's public inference
proxy. Deploy the Gufo listener on loopback for this configuration; llama-swap
uses its private backend address. No inference request field can mutate capacity.

## Pool and scheduler lifecycle

Allocate stable slot metadata up to the maximum, but create GPU state only for
resident sessions. Separate immutable checkpoint indexing from the resident
count; changing the latter must not reinterpret existing checkpoint indices or
invalidate leases. Report disabled slots explicitly rather than treating them as
free execution capacity. Admission and batching use available resident sessions.

The scheduler thread owns resizing between work units. A shrink immediately
lowers the admission ceiling. Choose idle sessions first, then mark enough busy
sessions for retirement when their requests finish. Retiring sessions receive no
new requests. Other sessions continue serving within the reduced admission
ceiling; no active token history is moved or truncated. A request completing its
generation may still own a persistence capture, so terminal output alone does
not prove the session can be freed.

Before freeing a retiring state, stop new checkpoint publication from it and
release unleased RAM checkpoints that borrow its storage. Drain existing captures
and restore users before releasing their backing allocations. Cached histories
may be evicted; already durable disk records remain reusable. Do not create a new
full snapshot merely to retire a session. Preserve references needed by active
requests and wait for them rather than invalidating them. Release allocator-held
spares that otherwise defeat reclamation. A transition reports stable only after
its allocations and dependent work have actually been released.

Grow one session at a time on the scheduler owner thread. Allocate and validate a
candidate before admitting work to it. Allocation failure retains usable sessions
and reports the actual resident count; partially completed growth is reported
explicitly, not disguised as an atomic rollback. Keep model/executor ownership
unchanged. Measure synchronization stalls before considering allocator changes.

Cache budgets remain explicit and do not automatically expand when sessions
shrink. Scope any new cache APIs to retirement and accurate usage reporting;
do not change existing zero-means-auto CLI semantics in this feature.

## llama-swap policy and accounting

Add an optional typed `runtimeSessions` model configuration for the Gufo backend.
It contains minimum, initial and maximum counts, a conservative total `memoryGB`
profile for each allowed count, and a growth cooldown. Validate complete,
nondecreasing positive profiles and consistent limits. Defaults for the cooldown
are 10 seconds. Static models retain existing behavior. Reject runtime sessions
on sharedID members in this first version; sharedID remains for genuinely
different model processes, while a resizable model has one process identity.

Enable this configuration only with the memory-pool scheduler in this version;
reject unsupported scheduler combinations at config load. Existing sharedID
models in the same pool must see runtime models' current/committed reservations,
not stale startup estimates. Keep existing priority and reserveWhileLoaded
policies when deciding which models to retain or evict.

Use measured total-memory profiles rather than assuming a linear per-slot cost.
Profiles include fixed weights, shared scratch, permitted cache/staging peaks and
per-session state. Use integer conservative GiB reservations consistently with
the current pool planner; round costs up and capacity down. Keep an explicit
machine headroom reserve outside the usable pool. Host available memory is a
secondary growth guard, never a replacement for reservations.

On incoming work, first consider a plan that retains loaded weights and shrinks
resizable models enough to fit the target. Prefer this over eviction when legal
under existing priorities. Mark retirement and queue the target if busy sessions
must drain. Block new admissions that would indefinitely prevent the shrink.
Only after confirmed completion may the target load consume released capacity.
If even minimum profiles cannot coexist, retain ordinary safe eviction/queueing.

Grow only when outstanding generation demand exceeds resident capacity and the
larger profile fits all loaded/committed models. Reserve growth before sending
the command. Free memory alone does not trigger growth. Rate-limit opportunistic
growth with the cooldown; a pending competing model load takes precedence over
growth that would consume its reservation. Shrink on actual competing demand,
not after every idle interval. Keep excess requests in llama-swap while resizing
so its queue reflects demand and Gufo cannot build an unbounded hidden backlog.

Run backend HTTP operations outside FIFO's owner goroutine and return completion
events to it. A timeout is unknown state: retain the larger reservation and
reconcile through GET. Never release capacity on an accepted PUT alone. Check
process-instance and operation IDs on completion. Reconcile process restart,
manual unload, cancelled waiters, partial growth and failed control requests.
Serialise operations per process and include resize reservations in collision
checks for parallel loads. An abandoned waiter does not erase a resize already
accepted by the backend. Publish status and logs for decisions and outcomes.

## Converter and rollout

Add round-tripped relations/config support for runtimeSessions, including limits,
profiles and cooldown. Emit the matching Gufo initial/maximum launch options and
private listener binding. Reject conflicting manual session arguments rather
than silently emitting two values. Add generated YAML fixtures and schema/docs.
Keep embeddings as an ordinary model with its existing memory reservation.

Measure profiles before replacing the current deployed config. Include warmed
MTP, vision when configured, long-context cache retention and outstanding disk
captures. Do not reuse the current static 105 GB estimate as a verified profile.
Back up deployed binaries/config, verify image and process revisions, and retain
rollback instructions. Preserve disk checkpoint files through deployment.

## Verification and acceptance

1. CPU tests with instrumented fake states prove resident creation/destruction,
   stable lease identities, idle/busy retirement, no new admission to retirees,
   partial allocation failure, cancellation and cache/capture ownership.
2. HTTP tests cover validation, unsupported backends, idempotency, status,
   conflicting operations, device loss and unchanged ordinary slot reporting.
3. llama-swap tests prove reserve-before-grow and acknowledge-before-load,
   queue fairness, concurrent embedding arrivals, priority/sharedID coexistence,
   fractional-cost rounding, timeout reconciliation, restart and no reload.
   Use a controllable fake backend; include race tests for owner/event boundaries.
4. Converter tests prove configuration round trips and reject incomplete profiles,
   incompatible scheduler settings and contradictory launch options.
5. On node-strix repeat 4->2->4 and 4->1->4 with idle and busy sessions. Confirm
   unchanged process identity and model load count, real host/GPU memory release,
   embedding admission only after release, bounded resize latency and no OOM.
6. Run the required formatting and focused repository checks. Because shared
   continuation/scheduling behavior changes, run the four functional profiles
   with affected modes including long-context and cache. Include cancellation,
   disk restore and image continuation where changed. Retain numerical quality
   checks and the standard speed benchmark. Compare affected timings against
   matched main controls at 5% and 3 ms per request/phase, retaining all results.
   Fix confirmed regressions; label noisy or missing-model evidence incomplete.

No full model sweep or unrelated video generation is required. Completion means
both projects and generated configuration work together on the production host,
with retained evidence; a passing mock-only resize is not sufficient.
