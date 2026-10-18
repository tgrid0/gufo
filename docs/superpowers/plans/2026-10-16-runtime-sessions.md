> 2026-10-18 review: deferred by user decision. This historical design must be revised around upstream component-owned sessions, cache backing, and current AR/MTP allocation paths before implementation. See the 2026-10-18 minimal component-cache plan.

# Runtime Sessions Implementation Plan

Status: deferred by the user on 2026-10-17 until disk-only streaming persistence
is implemented and measured. See `2026-10-17-disk-only-streaming-cache.md`.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans for native execution, or superpowers:subagent-driven-development if the user selects delegation. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Resize resident Gufo sessions under llama-swap's memory and queue policy without reloading weights or changing f16 KV.

**Architecture:** Stable slot identities hold optional execution states. The Gufo scheduler owns allocation and retirement, while llama-swap reserves measured memory profiles and waits for acknowledged transitions before loading competing models. The INI converter preserves the runtime policy in generated configuration.

**Tech Stack:** C++20/CMake/Nix/HIP on Linux gfx1151, Go 1.26+, Python/YAML, existing HTTP stacks.

**Spec:** [Approved design](../specs/2026-10-16-runtime-sessions-design.md).

## Global constraints

- Production remains Linux x86-64 Strix Halo gfx1151. Preserve pinned compiler/dependencies for comparisons.
- Preserve f16 KV, configured context length, active requests, model identity, loaded weights and existing disk formats.
- Preserve accepted disk writes: wait for captures/writes to finish rather than silently abandoning them on shrink.
- `--sessions` remains initial capacity; `--sessions-max` opts into resizing; 1 <= initial <= maximum <= 8.
- Initial deployment maximum is 4; measured profiles determine safe coexistence with embeddings.
- Static models/configurations remain compatible. Runtime sessions initially require the memory-pool scheduler and cannot be sharedID members.
- Growth cooldown defaults to 10 seconds. No cache auto-expansion after shrink; existing zero-means-auto semantics stay unchanged.
- Preserve unrelated changes. Use task-owned paths for staging. Gufo uses Conventional Commits; follow sibling repositories' own commit rules.
- Use the user's tgrid/image-build and WSL/scp deployment workflows. No llama.cpp code changes are needed.

## Review focus

1. A completed HTTP generation can still own a pending persistence capture: Task 1 tests blocked retirement and durable completion.
2. A PUT timeout can hide successful allocation: Tasks 3 and 4 test idempotent reconciliation without releasing the larger reservation.
3. A restart can recycle slot counts but invalidate old acknowledgements: Tasks 2 and 4 test process-instance fencing.
4. Busy survivors must continue correctly while another session retires: Tasks 1, 2 and 6 test leases, independent histories and actual decode behavior.
5. SharedID selection and concurrent loads must count the same dynamic memory: Task 4 tests reservations across both planners and fractional rounding.

## Execution preparation

- [ ] Read repository instructions and inspect status in all three repositories. Use using-git-worktrees at execution time; preserve the approved documents in the implementation checkout. Do not reset the current zgfs branches or unrelated changes.
- [ ] Record current revisions and deployed image/binary/config identities. Discover the actual llama-swap unit rather than assuming the system service is named `llama-swap.service`.
- [ ] Establish node-strix and WSL build locations and toolchains. Request sandbox escalation for sibling-repository writes or remote commands where required; do not bypass the sandbox.
- [ ] Retain baseline production artifacts and cache histories before candidate builds. Keep workloads bounded and run only affected timing controls on main.

## Task 1: Resident state lifecycle and checkpoint-safe retirement

**Files:** `src/cli/serve/continuation_cache.{hpp,cpp}`, `text_model_runner.{hpp,cpp}`, `continuation_disk_store.{hpp,cpp}`; Flash Next `engine.{hpp,cpp}`, `kernels/rocm/executor.{hpp,cpp}` and its adapter in `inference_backend.cpp`; existing `tests/cli/{continuation_cache,text_model_runner,continuation_disk_store}_test.cpp`.

**Interfaces:** Introduce `TextSessionLimits { size_t initial; size_t maximum; }` in `text_model_runner.hpp`; keep existing constructors delegating to fixed limits. Add `maximum_capacity()`, `resident_capacity()` and `admission_capacity()` returning `size_t`, plus owner-thread `SetAdmissionCapacity(size_t)`, `GrowOne()` returning the created slot index, and `TryRetire(size_t)` returning `bool` to `TextRunnerPool`. `TryRetire` returns false while references/captures prevent reclamation; errors propagate without pretending memory was freed. Keep `capacity()` meaning resident count for compatibility and audit callers explicitly.

- [ ] Extend existing fake-state tests before implementation. Assert initial=2/max=4 creates exactly two states; grow creates one; retirement destroys exactly its chosen state; live leases and snapshot indices survive 4->2->4. Test invalid limits and allocation failure with unchanged usable states.
- [ ] Add capture-lifetime tests: a held capture blocks retirement; releasing it permits retirement; an already accepted disk write remains durable. Snapshot eviction during retirement must not call a preservation path that allocates a new full snapshot. A leased snapshot must remain valid.
- [ ] Run the existing CPU targets and confirm the new lifecycle assertions fail for the missing behavior, not build environment errors.
- [ ] Store maximum slot metadata separately from resident/admission counts. Keep snapshot records indexed after the fixed maximum boundary. Retain the factory for growth, publish a slot only after successful allocation, and restrict Acquire to eligible resident entries.
- [ ] Track snapshot/capture ownership sufficiently to evict unleased histories borrowing a retiring state and await references still used by active work. Stop new capture publication for the retiring state. Use existing persistence completion mechanisms and never drain disk work while holding metadata locks.
- [ ] Add model-specific retirement cleanup only where existing ownership is insufficient: release safe allocator spares and verify the destructor cannot preserve discarded snapshots. Keep other models' active state untouched; fail unsupported retirement safely.
- [ ] Verify with `nix develop -c cmake --preset cpu-test`, then `nix develop -c cmake --build --preset cpu-test --target continuation_cache_test text_model_runner_test continuation_disk_store_test` and `nix develop -c ctest --preset cpu-test -R '^(continuation_cache_test|text_model_runner_test|continuation_disk_store_test)$' --output-on-failure`. All must pass.
- [ ] Run the required format check and commit only Task 1 paths after review.

## Task 2: Scheduler-owned asynchronous resizing

**Files:** new `src/cli/serve/session_capacity.hpp`; existing `text_generation_scheduler.{hpp,cpp}`, `text_generation_backend.hpp`, `inference_backend.{hpp,cpp}`, `tests/cli/text_generation_scheduler_test.cpp`.

**Interfaces:** Define `SessionCapacityStatus` with `instance_id`, `operation_id`, `target`, `resident`, `busy`, `retiring`, `maximum`, `phase`, optional `error`, and separate `session_bytes`, `snapshot_bytes`, `staging_bytes`. Define `SessionCapacityCommand { size_t target; string operation_id; }` and `SessionCapacityResult` with accepted/invalid/conflict/unsupported/unavailable outcomes. Add `SessionCapacity() const -> optional<SessionCapacityStatus>` and `RequestSessionCapacity(const SessionCapacityCommand&) -> SessionCapacityResult` to the scheduler and backend. Published status is an immutable synchronized snapshot; the HTTP caller never accesses owner state directly.

- [ ] Add tests for idle shrink, busy shrink, two survivors decoding while peers retire, no new admission above target, and growth only between work units. Assert every generation terminates once and keeps its own tokens/RNG.
- [ ] Test repeated operation ID/target, ID conflict, overlapping operations, partial growth failure, cancellation, shutdown and device loss. A new scheduler instance must have a distinct instance ID. Completion status must lag actual destruction, never precede it.
- [ ] Implement a command queue consumed by the scheduler owner. Select idle retirees first; mark busy retirees and complete reclamation as leases/captures finish. Admission ceiling is target; resident count remains actual. Keep `/slots` IDs stable and distinguish disabled/retiring states.
- [ ] Use Task 1 lifecycle methods. Hold a maximum-sized session table and audit every capacity-dependent batch plan, metric, wait predicate and request descriptor. Publish status after each lifecycle change, including partial failure.
- [ ] Build `text_generation_scheduler_test`; run `nix develop -c ctest --preset cpu-test -R '^text_generation_scheduler_test$' --output-on-failure`. Run the affected CPU sanitizer test once ownership tests pass. Require no sanitizer findings.
- [ ] Format, review and commit Task 2 paths.

## Task 3: CLI and runtime HTTP contract

**Files:** `src/cli/serve/serve.cpp`, `http_server.cpp`, `tests/cli/http_server_test.cpp`, `tests/cli/serve_test.py`, serving documentation and `docs/KV-CACHE.md`.

**Interfaces:** `--sessions-max` is the only new option. GET/PUT `/admin/sessions` use snake_case JSON matching Task 2 fields. PUT body is `{ "target": 2, "operation_id": "opaque-client-id" }`; require a nonempty ID of at most 128 ASCII bytes, an integer target and no unknown fields. GET exposes `supported:false` for unsupported backends; PUT maps unsupported to 501, unavailable to 503, invalid to 400, conflict to 409 and accepted to 202. Successful GET returns 200. A 202 response contains status, never an assertion of completed reclamation.

- [ ] Add fake-backend HTTP tests covering all outcomes, missing/malformed fields, target 0/9, duplicate IDs and readback after client timeout. Existing completion/streaming APIs and health checks remain unchanged.
- [ ] Add CLI tests for omitted maximum retaining fixed behavior, 2/4 accepting, and 4/2 or 0/4 rejecting. Pass limits from launch through inference-backend pool construction.
- [ ] Implement routes and serialization using the existing server conventions. Expose separate byte counters without calling their sum total process memory. Keep the management route outside public llama-swap routing in Task 4.
- [ ] Build/run `http_server_test` and the existing serve CLI test using its documented invocation. Run the bounded hosted checks with `nix build .#checks.x86_64-linux.pr` once the integrated Gufo change is ready.
- [ ] Document cache-eviction effects and operation reconciliation; format, review and commit Task 3 paths.

## Task 4: llama-swap control client, reservations and queue policy

**Files (llama-swap):** `internal/config/model_config.go`, `matrix.go`, `config-schema.json`, `config.example.yaml`; new `internal/process/session_capacity.go` and its test; `internal/router/{base,memory_swapper}.go`, `internal/router/scheduler/{scheduler,fifo}.go`, `internal/variant/resolver.go`, their existing tests, and `docs/configuration.md`.

**Interfaces:** `RuntimeSessionsConfig` has YAML fields `min`, `initial`, `max`, `memoryGB` (map[int]float64 of total process profiles), and optional `growCooldownSecs` (default 10). Values must be finite, profiles positive/nondecreasing and complete for min..max, with 1 <= min <= initial <= max <= 8. Add optional pointer `RuntimeSessions` to ModelConfig. Legacy `memoryGB` remains the initial-profile value and must match when supplied; normalize it from the profile if omitted. Unsupported schedulers/sharedID membership are validation errors.

Add `SessionCapacityClient.Get(ctx context.Context) (SessionCapacityStatus, error)` and `Set(ctx context.Context, target int, operationID string) (SessionCapacityStatus, error)` using the resolved private process URL. Add scheduler `ResizeDone` events containing model ID, expected process/operation identity, latest status and error. Worker goroutines deliver events; only the router owner mutates reservations. Use a single immutable reservation snapshot for variant resolution and memory-pool planning rather than mutating shared config maps.

- [ ] Write configuration and fake-HTTP client tests for all validation cases and Task 3 JSON/status contracts, including unsupported capability and timeout after backend acceptance.
- [ ] Write FIFO/planner tests with fake effects: embeddings arriving at four sessions cause shrink before load; no stop/start is issued for Gufo; busy retirees drain; survivors can finish; repeated arrivals share the operation. Assert reservation uses max(old,new) until reconciled and committed loads cannot race for the same capacity.
- [ ] Cover cancelled last waiter, manual unload, process restart, partial growth, stale completion, control failure and reconnection. Unknown backend state keeps the conservative reservation and blocks dependent loads; it does not claim a successful shrink. Use bounded HTTP calls and scheduled reconciliation through the owner event loop, not blocking polling there.
- [ ] Test demand-based growth up to max with a fake clock: no idle growth, no growth inside the 10-second cooldown, and pending competing loads prevent conflicting growth. Existing minimum/initial capacity is immediately usable. Maintain FIFO admission while keeping extra generation requests in llama-swap.
- [ ] Extend memory planning to consider legal per-model capacities before eviction, preserving reserveWhileLoaded and priority costs. Round memory costs up and capacity down consistently. Update sharedID forced-memory calculations to consume current/committed runtime reservations. Test a sharedID peer load alongside Gufo and noninteger profile boundaries.
- [ ] Implement the backend client and async effects, runtime reservation snapshot, planner and FIFO transitions. Discover capacity after load/reconnect before granting requests; fence all completions by instance and operation identity. Deny proxy forwarding of management routes. Keep static model paths unchanged.
- [ ] Run `go test ./internal/config ./internal/process ./internal/router/... ./internal/variant`, then `go test -race ./internal/router/... ./internal/process ./internal/variant`. After changes run `make test-dev`; run required `make test-all` before committing. All checks must pass; use gofmt before commit.
- [ ] Document the new config, review and commit only Task 4 paths under sibling repo instructions.

## Task 5: Converter configuration that survives regeneration

**Files (convert_models_ini):** `convert_models_ini.py`, existing tests under `tests/test_convert_models_ini.py` and `tests/test_gufo_engine.py`, converter documentation; deployment `models.ini` and generated `config.yaml` only after Task 6 profiling.

**Interfaces:** Add relation directive `runtimeSessions <section> = <JSON object>` with the exact Task 4 keys. Parse with the existing line-numbered error conventions and emit a YAML mapping, never a JSON string. Limit to Gufo engine sections. Generate one `--sessions initial` and one `--sessions-max max`; existing equal values may be deduplicated, contradictory values reject. Gufo's listener must resolve to loopback for this deployment; preserve the resolved port. Do not alter other engines' host binding.

- [ ] Add golden tests for a profile covering 1..4, default cooldown, emitted options and round-trip YAML typing. Assert malformed JSON, missing profiles, invalid ranges, non-Gufo engines and contradictory launch values reject with a source line.
- [ ] Implement parsing, group propagation, validation and emission following existing relations handling. Carry the configuration through both single-model and grouped section generation without silently losing it.
- [ ] Run `python -m pytest tests/test_convert_models_ini.py tests/test_gufo_engine.py`, then the repository's required broader check if shared parsing changes affect other engine tests. Require unchanged unrelated fixtures.
- [ ] Review and commit code/tests/docs only. Leave production memory values for measured deployment.

## Task 6: Production measurements, regression qualification and deployment

**Files:** new `tests/functional/session_capacity.py`, `tests/functional/README.md`; measured deployment configuration in converter repo. Retain raw output under ignored `build/runtime-sessions/` or the remote equivalent; never commit model files or large captures.

**Interfaces:** The standalone functional runner accepts `--base-url`, `--output`, `--max-sessions` and exercises the Task 3 contract against an already running candidate. Record full commands/revisions, process-instance IDs, transitions, request IDs, responses, phase timings, host/GPU memory and cache counters. It is opt-in, outside hosted model CI.

- [ ] Add the runner and test its protocol/error handling with a fake HTTP backend. Implement idle/busy 4->2->4 and 4->1->4 cycles, repeated ID reconciliation and an uninterrupted survivor stream. Validate token history and cache behavior, not only HTTP success.
- [ ] Build candidate Gufo with the pinned production toolchain and llama-swap under WSL. Preserve baseline artifacts. Stop the discovered service for isolated measurement as authorized; do not leave competing test servers running.
- [ ] Measure each 1..4-session footprint under warmed MTP, representative context/cache histories, configured vision and accepted disk writes. Record maximum observed memory and uncertainty; choose conservative integer GiB profiles including configured cache allowances and external headroom. Use actual available memory and memory pressure alongside allocation counters.
- [ ] Run numerical checks and standard speed benchmark. For shared scheduling/cache changes run affected functional suites across all four documented profiles/modes, including `long-context`, `cache`, batching and state edges; include cancellation and image continuation where applicable. Check loaded modes and actual drafts. A missing model remains incomplete evidence.
- [ ] Compare affected request/phase timings against matched main controls using the 5% and 3 ms requirements. Alternate only affected histories to investigate flags, retain every result and fix confirmed regressions. Record growth synchronization pauses and peak memory during retirement separately from steady decode speed.
- [ ] Exercise llama-swap plus embeddings: confirmed release precedes embedding launch, Gufo PID/instance remains stable, total memory retains headroom, concurrent loads cannot oversubscribe, and growth happens only with demand. Test cache restoration after retirement and restart without deleting checkpoint files.
- [ ] Write measured profiles to models.ini, regenerate config.yaml and verify emitted launch commands. Use `gh auth status` before any GitHub operation. Push task-owned Gufo work to the user's tgrid workflow, rebuild the multi-backend image in `/home/walker/llm/tools/amd-strix-halo-toolboxes/toolboxes`, and copy the WSL-built llama-swap binary using scp. Retain previous image/binary/config for rollback.
- [ ] Restart the actual llama-swap unit, verify loaded versions, health, capacity status and one embedding/generation coexistence cycle. Finish with repository status and explicit evidence/limitations. Do not claim completion if required correctness or memory qualification failed.

## Review and execution choice

Proposed execution: native, in this session. These changes share lifecycle and
reservation invariants, so one implementer can carry the interfaces through all
three repositories without repeated handoffs. Review the final integrated branch
and retained test evidence before deployment. If the user selects delegated
execution instead, follow the subagent-driven-development workflow.

Implementation begins after the user reviews this plan and selects execution.
