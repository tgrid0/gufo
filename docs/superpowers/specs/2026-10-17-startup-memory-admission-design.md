> 2026-10-18 review: deferred by user decision. This historical design must be revised around upstream component-owned sessions, cache backing, and current AR/MTP allocation paths before implementation. See the 2026-10-18 minimal component-cache plan.

# Flash Next startup memory admission design

Status: proposed design accompanying the requested committed plan; implementation
is not authorized by this document. Builds on disk-only streaming commit
`936fbfaf` and its [qualification report](../reports/2026-10-17-disk-only-streaming-cache.md).

## Outcome and limits

A Flash Next server must establish a bounded execution working set before it
reports ready. Configurations that cannot allocate that set must fail loading
with a useful memory breakdown, rather than abort on the first prefill or first
concurrent decode. Preserve f16 KV, configured slot/context capacity, MTP behavior,
outputs, and checkpoint compatibility. No automatic reduction of settings.

The motivating failure is four 262144-token slots with MTP plus Qwen3 Embedding
8B: Gufo loaded first, combined idle usage reached 123.846 GiB, and the first
prefill aborted in the lazy MMQ arena before saving a checkpoint. Existing
`DeferredScratchBytes()` accounts for some buffers but omits that arena.

This is a conditional guarantee about bounded Gufo-owned execution allocations,
not protection against another process consuming memory, driver failure, host
OOM, or arbitrary inputs with no enforced bound. RAM-off disk persistence stays
bounded; optional RAM retention and enabled image paths must be explicitly
accounted for too. Do not label partial path coverage as complete admission.

Scope: Linux x86-64 gfx1151, Flash Next AR/MTP, existing serving/cache modes.
Other models retain their behavior. Runtime slot resizing, llama-swap scheduling,
embedding implementation, kernel optimization and persistence-latency changes
are separate work. Initial numerical/model qualification remains Flash Next only.

## Approach

Use analytical bounds plus real allocation and bounded warmup. An estimate alone
misses allocation failure/fragmentation; warmup alone does not prove the maximum
over routing distributions, batch widths or input shapes. A fixed extra allowance
alone hides missing components and is not an adequate solution.

Create one model-local memory plan consumed by both admission and allocation.
Derive bounds from validated model dimensions, context, slot count, scheduler
prefill limit, physically reachable decode widths, draft limit, logits/sampling
limits, enabled vision limits and cache policy. Use checked byte arithmetic.
MMQ allocation sizing and the estimator must share alignment and capacity-rounding
helpers, including the current 256 MiB minimum/power-of-two policy unless that
policy is deliberately changed and tested together.

Account separately for resident weights, live session state, existing scratch,
remaining execution arenas, rollback/checkpoint storage, graph/library overhead,
cache backing, bounded persistence transfers, and host metadata. Distinguish
logical checkpoint/disk quota bytes from physical backing. Audit allocation
lifetime and sharing: sum concurrently live regions, take maxima only where
execution/ownership proves exclusion. RAM-off must not reserve a full disk image.

Already allocated memory is descriptive, not charged again against current free
memory. Compare remaining GPU demand against HIP free memory and remaining
shared physical demand against host available memory separately. Never add
GPU usage and RSS as independent totals on Strix Halo.

## Admission and readiness

1. Validate limits and build the memory plan before creating the serving state.
2. Check remaining requirements against current capacity. Include a named
   1 GiB default operational headroom allowance, not a substitute for bounds.
3. Allocate sessions and all bounded execution arenas exactly once, using the
   plan. Handle real allocation failures even after a successful precheck.
4. Warm supported execution families/physical widths that create library or
   graph allocations, within the planned envelope. Bound graph variants and
   workspace caches; do not warm an unbounded Cartesian product of prompts.
5. Reset warmup state, RNG/draft policy and counters, drain work, recheck headroom,
   then publish the scheduler/model state and readiness.

Loading need not bind HTTP early. If health is already served, readiness remains
false during planning, allocation and warmup. Failure returns the existing load
error path/nonzero CLI exit, releases partial allocations, and never publishes
the new backend. A failed reload must not replace a usable old backend.

After admission, Gufo-owned execution arenas have sealed capacities: bounded
requests reuse them. A bound violation is a controlled internal/load-contract
error, not an attempted arena growth followed by HIP_CHECK/abort. Input validation
rejects requests outside the declared envelope before expensive allocations.
Mandatory library allocations that cannot be bounded are a release blocker;
a successful representative warmup is not evidence that no future growth exists.

Vision preprocessing dimensions/token counts and concurrent image work must have
enforced finite bounds if a projector is enabled. Use existing limits where
available; if a required bound is absent, document and implement the smallest
explicit limit as part of the envelope, preserving established supported inputs.
Do not ship a nominally ready projector path that bypasses accounting.

## External memory allowance

Add one canonical serve option, `--memory-reserve-bytes N`, default `0`, unsigned
bytes. It specifies additional capacity to leave available beyond Gufo's planned
working set and the operational headroom. It is not an allocation or a lock on
memory, and is not included in Gufo's own reported working-set total.

The option can budget a model that will load later. Already resident processes
are reflected in observed free memory; specifying their full budget again is
intentionally conservative, not automatic detection. Reject invalid/negative/
overflowing values. Nonzero values on unsupported backends must fail explicitly.
No new aliases, automatic slot/context changes, or environment-only overrides.

llama-swap continues to enforce the combined pool budget externally. Document
how measured Gufo peak plus future embedding demand informs its existing
`memoryGB` accounting, but do not modify llama-swap/configuration in this feature.
The reserve option cannot prevent future external allocation from violating it.

## Observability and acceptance

Emit structured startup stages and byte breakdowns: planned own working set,
already committed, remaining host/device demand, operational headroom, external
allowance, observed HIP free/total and host available, and coverage limits.
Report planned and observed post-warmup values separately. Failure identifies
the component/stage, required and available bytes, and selected context/slots/MTP.
Tests must observe allocation growth directly, not infer it from RSS alone.

Accept when CPU accounting/failure tests, GPU allocation-envelope checks and
Flash Next functional/numerical checks pass; a controlled low-memory scenario
fails before readiness without SIGABRT; and normal in-envelope first/subsequent
requests introduce no unplanned execution-arena growth. Include maximum prefill,
skewed MoE routing, physical decode widths, MTP, sampling, image, cancellation,
restore and cache-boundary paths. Measure startup/warmup time separately from
request latency. Apply existing per-request/phase 5% and 3 ms criteria without
averaging away regressions. Preserve old evidence and do not manufacture a
whole-machine OOM merely to reproduce the earlier failure.
