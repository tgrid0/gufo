> 2026-10-18 review: deferred by user decision. This historical design must be revised around upstream component-owned sessions, cache backing, and current AR/MTP allocation paths before implementation. See the 2026-10-18 minimal component-cache plan.

# Flash Next Startup Memory Admission Implementation Plan

> **For agentic workers:** Use superpowers:executing-plans for native execution, or superpowers:subagent-driven-development if explicitly selected. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reject insufficient Flash Next memory configurations before readiness by calculating and allocating a bounded working set.

**Architecture:** A model-local memory plan is shared by sizing and allocation. Serving reserves execution arenas, warms bounded library/graph paths, resets state, and only then publishes readiness; external demand is an explicit allowance, not a guarantee against other processes.

**Tech Stack:** C++20, HIP/ROCm gfx1151, CMake/Nix, existing C++ and Python HTTP tests.

**Spec:** [Startup memory admission design](../specs/2026-10-17-startup-memory-admission-design.md).

Status: planning only. The user requested writing and committing this plan;
runtime code, services and deployment configuration are unchanged. Recommended
execution is native because sizing, allocation and readiness contracts are coupled.

## Global constraints

- Base on disk-only streaming implementation `936fbfaf` and qualification `6994dd6`.
- Linux x86-64 AMD Strix Halo gfx1151; keep pinned toolchain/dependencies for comparisons.
- Preserve f16 KV, slot/context settings, numerical behavior, MTP and disk format.
- Flash Next qualification only; other backend defaults remain unchanged.
- Account remaining allocations against current free memory without charging resident allocations twice; shared physical GPU/RSS observations are not additive.
- `--memory-reserve-bytes N` defaults to `0`; named operational headroom defaults to 1073741824 bytes. External allowance is additional free space, not allocated memory.
- MMQ bounds share the allocator's alignment and capacity rounding, currently a 268435456-byte minimum and doubling.
- No automatic resizing, new llama-swap scheduler, kernel optimization, or persistence-latency redesign.
- No readiness until allocation, bounded warmup, reset and final headroom checks succeed.
- No claim of protection from later external memory consumption or unbounded input paths.

## Review focus

1. Arithmetic/rounding overflow, zero dimensions and platform size limits: Task 1 CPU tests must reject before allocation.
2. Worst routed-expert concentration and coexisting scratch lifetimes: Tasks 1-2 must prove bounds beyond a representative warmup.
3. Allocation failure after a successful free-memory check, or failed reload: Tasks 2-3 must clean up and never publish partial state.
4. Warmup changing RNG, draft state, cache contents or counters: Tasks 3-4 must compare first real requests to clean-state controls.
5. Optional vision, RAM retention, graph/sampling variants and other processes: Tasks 1, 3 and 4 must enforce declared bounds and keep external limits visible.

## File map and interface contracts

New model-local `src/models/qwen38_flash_next/memory_plan.{hpp,cpp}` owns pure
checked accounting; it must compile in CPU builds without initializing HIP.
`engine.{hpp,cpp}` bridges the plan to the loaded model; `kernels/rocm/executor.{hpp,cpp}`
owns reservation/warmup. MMQ sizing stays beside its existing allocation code in
`kernels/rocm/mmq/qfn_mmq.{h,hip.cpp}` and `qfn_ggml_stubs.{h,hip.cpp}`.
Serving integration belongs in `src/cli/serve/{serve.cpp,inference_backend.hpp,inference_backend.cpp}`;
preserve generic runner validation in `text_model_runner.{hpp,cpp}`.

Proposed model namespace contracts, shared across tasks:

- `StartupMemoryLimits`: validated context tokens, session count, physical decode widths, maximum prefill rows, maximum drafts/logit rows, sampling and vision bounds, resolved RAM-cache budget, disk-persistence mode, and external-reserve bytes. Use existing model types where available; no duplicate enum for AR/MTP.
- `StartupMemoryPlan`: component rows identifying host/device/shared backing, resident/remaining classification, peak bytes and overlap group, plus own peak, remaining demands, headroom and external allowance.
- `BuildStartupMemoryPlan(const Config&, const StartupMemoryLimits&) -> StartupMemoryPlan`: pure checked planning; invalid/unbounded limits throw descriptive validation errors.
- `Model::PrepareServingMemory(const StartupMemoryLimits&, StartupMemoryPlan*, std::string* error) -> bool`: reserve/warm/reset and return the final plan; failure leaves no published backend.
- `Executor::PrepareServingMemory(const StartupMemoryLimits&, StartupMemoryPlan*, std::string* error) -> bool`: device implementation used by the model bridge.
- `TextRunnerStartupMemoryOptions { std::size_t external_reserve_bytes = 0; }`: serving plumbing; generic backends reject nonzero unsupported policy rather than silently ignore it.

Internal layout helpers must be reused by actual allocation sites. The pure plan
receives resolved cache/vision limits; it must not independently choose a second
RAM-auto budget. Detailed component formulas are established and tested in Task 1,
then consumed unchanged by later tasks.

## Task 1: Build a complete checked allocation plan

**Files:** create `memory_plan.{hpp,cpp}` and `tests/models/qwen38_flash_next/memory_plan_test.cpp`; modify model/test CMakeLists, `engine.{hpp,cpp}`, `executor.{hpp,cpp}`, MMQ sizing headers/source as needed.

**Consumes:** model `Config`, existing `SessionBytes`, `DeferredScratchBytes`, scheduler limits and cache policies.
**Produces:** `StartupMemoryLimits`, `StartupMemoryPlan`, `BuildStartupMemoryPlan`, shared arena layout/rounding helpers.

- [ ] Audit all allocation families reachable from prefill, AR/MTP decode, sampling, image handling, snapshot protection and restore. Record owner, size expression, lifetime, sharing, and existing input limit in a model-local allocation table. Include library/graph caches and the currently omitted MMQ stack arena; identify any missing finite bound explicitly.
- [ ] Add CPU tests: `checked_capacity_rounding` (1 byte -> 268435456, 268435457 -> 536870912, overflow rejects); `sum_live_take_exclusive_max`; `resident_not_charged_twice`; `ram_off_disk_bytes_not_staging_allocation`; `external_allowance_not_own_peak`; and `invalid_or_unbounded_limits_rejected`. Assert monotonicity across context/slots/batch/draft bounds and skewed expert routing.
- [ ] Configure `cmake --preset cpu-test`, build `qwen38_flash_next_memory_plan_test`, and run `ctest --preset cpu-test -R '^qwen38_flash_next.memory_plan$' --output-on-failure`. First observe missing implementation/incorrect bounds fail.
- [ ] Implement pure planning using checked add/multiply/align and shared actual-allocation layouts. Separate remaining GPU and shared-physical checks; document proven exclusion before using a max. Resolve optional RAM-cache budget once, including physical backing/copy-on-write worst cases. Do not reserve a full checkpoint merely for direct disk persistence.
- [ ] Make the CPU tests pass and review the allocation table against every reachable allocation site. A missing bound blocks the next readiness contract; do not substitute an arbitrary safety percentage.
- [ ] Commit task-owned paths: `feat(qwen): model bounded startup memory requirements`.

## Task 2: Reserve and seal execution arenas

**Files:** model `engine.{hpp,cpp}`, `kernels/rocm/executor.{hpp,cpp}`, MMQ files above; create `tests/models/qwen38_flash_next/startup_memory_test.cpp` and `startup_allocation_fault.cpp`; extend test CMakeLists and reuse `device_headroom_fault.cpp`.

**Consumes:** Task 1 plan/layouts. **Produces:** model/executor `PrepareServingMemory`, checked MMQ reservation/sealing, allocation-observation test hooks.

- [ ] Add tests `mmq_reserved_before_first_prefill`, `sealed_arena_rejects_growth`, `allocation_failure_releases_partial_state`, and `concentrated_routes_stay_in_plan`. Use allocation interception to count live/high-water/new allocations by phase; inject failure at each reservation boundary, including a real allocation failure despite a successful capacity query.
- [ ] Build only `qwen38_flash_next_startup_memory_test` under `gpu-test`; run against Flash Next on Strix. Observe lazy first-use allocation and fatal allocation handling fail the new expectations.
- [ ] Implement reservation and sealed capacity using the plan. Replace MMQ's startup allocation-abort path with errors propagated to the model load boundary. Keep LIFO and pointer-lifetime invariants; no arena relocation with live pointers. Reserve batched logits, sampling, rollback/checkpoint and other bounded lazy scratch once. Share already allocated buffers instead of double-reserving them.
- [ ] Add bounded library/graph warmup coverage for reachable AR/MTP widths/shapes. Prove a finite cache bound or reuse/evict bounded variants; reject envelope violations before resource growth. Do not claim success from warming only the default width or average MoE routing distribution.
- [ ] Verify injected failures exit cleanly without SIGABRT, RAII releases partial work, and in-envelope operations cannot grow sealed Gufo-owned execution arenas. Library growth outside the declared bound is a failure, not ignored measurement noise.
- [ ] Commit: `feat(qwen): reserve execution memory before serving`.

## Task 3: Gate readiness and expose admission diagnostics

**Files:** serving files in the map, `tests/cli/{serve_test.py,text_model_runner_test.cpp}`, `tests/models/qwen38_flash_next/{session_test.cpp,startup_memory_test.cpp}`, `docs/CLI.md`, `docs/SERVER.md`.

**Consumes:** `PrepareServingMemory`, resolved startup limits. **Produces:** CLI policy, transactional startup ordering and structured breakdown/error events.

- [ ] Add CLI tests for reserve `0`, positive value, negative, malformed and overflowing values; unsupported backend with nonzero reserve must reject. Add tests `not_ready_during_warmup`, `failed_warmup_never_published`, `failed_reload_preserves_old_backend`, and `warmup_does_not_change_first_request`.
- [ ] Run affected CPU tests and Flash Next session/startup tests, observe the new assertions fail before implementation.
- [ ] Thread `TextRunnerStartupMemoryOptions` through serving. Construct validated limits from actual scheduler/model/cache settings, check current HIP/host capacity, prepare memory, reset all warmup session/token/RNG/draft/cache/counter state, recheck headroom, then publish the new backend. Preserve existing cold load/error lifecycle and generic validation without charging reserved memory twice.
- [ ] Enforce finite image/preprocessing/sampling limits at admission. If the inventory exposed an absent bound, implement and document the minimal required limit and regression-test existing supported inputs; do not silently disable projector support or claim it is covered by text warmup.
- [ ] Emit stages and named component byte totals, remaining demand, external reserve, headroom, observed capacity and coverage limits. Low-capacity and allocation errors identify the failing component and selected context/slots/MTP; CLI exits nonzero, no readiness success or fatal signal.
- [ ] Run CLI/runner tests and bounded GPU startup tests to green. Document the additional-free-memory semantics and that neither the allowance nor startup allocation prevents external consumers from exhausting memory later.
- [ ] Commit: `feat(serve): gate Flash Next readiness on memory admission`.

## Task 4: Qualify startup bounds and first-request behavior on Strix

**Files:** create `tests/functional/startup_memory.py`; extend `tests/functional/README.md`, `docs/TESTING.md` and the model allocation table. Keep model HTTP qualification outside hosted CI.

**Consumes:** final candidate, baseline `6994dd6` implementation lineage, allocation counters and existing functional harness.
**Produces:** retained per-phase memory/startup/request evidence and a qualification report.

- [ ] Add a deterministic low-capacity/fault scenario asserting nonzero load exit, named memory error, no ready state and no SIGABRT. Use test-only capacity/allocation injection; do not deliberately drive the whole machine to OOM.
- [ ] On node-strix record exact source/binary/model/toolchain hashes. Preserve service configuration, current images and existing disk files; stop the idle service only for authorized testing and restore it afterward. Stage only task-owned paths for Nix builds.
- [ ] Test both embedding-first and Gufo-first order. For the latter, include an explicit additional embedding allowance (15 GiB is the initial test budget, not a claim about measured demand). Four 262144-token f16 MTP slots must either reserve/warm successfully with the promised headroom or reject before readiness. Separately measure the smaller viable setting; never relabel it as a 256K success.
- [ ] Exercise cold/max-prefill, skewed expert routing, all reachable physical decode widths up to configured slots, AR/MTP draft limits, sampling variants, supported image bounds, cancellation, cache save/restore and RAM-off/on. Record owned allocation high-water/growth, host available memory, HIP usage and process RSS separately. Input beyond the envelope must reject before GPU allocation.
- [ ] Compare warmup-reset output/logits to independent reference fixtures and run Flash Next numerical/snapshot checks. Run affected HTTP state-edges, long-context/cache, concurrency and image paths with actual draft counters. Retain standard speed checks. No other-model sweep unless shared changes expose additional risk.
- [ ] Compare matched per-request/phase timings at 5% and 3 ms; report load/warmup time separately. Investigate flagged histories with alternating controls, retain all results, and fix confirmed regressions before publishing. No average-only pass or missing-model pass.
- [ ] Run pinned format check (`nix shell --inputs-from . nixpkgs#clang-tools -c python3 tools/ci/check-format.py`), bounded hosted check (`nix build .#checks.x86_64-linux.pr`), and focused sanitizer checks for changed ownership/error paths. Run documentation checks.
- [ ] Commit the harness/report/docs: `test(qwen): qualify startup memory admission`. Report conditional guarantees and any remaining blockers accurately. Do not deploy, change pool budgets, or implement runtime resizing as an implicit final step.

## Completion and handoff

Review must establish that every enabled execution allocation family has a bound,
reservation/ownership path and test; representative successful requests alone are
insufficient. Hand the measured own peak and explicit reserve semantics to the
later llama-swap/resizing work. This plan does not assert that four full-context
slots plus embeddings will fit: early, diagnostic rejection is a valid outcome.

Self-review: each design section maps to Tasks 1-4; the five review risks have
explicit tests; interface names are consistent. The external allowance/default
headroom and image-envelope requirements are proposed design decisions for review.
