> 2026-10-18 review: superseded for Flash Next by the minimal component-cache plan. Historical requirements and results remain below; do not execute the legacy implementation tasks against main 0.17.1.

# Disk-only Streaming Cache Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans for native execution or superpowers:subagent-driven-development if the user selects delegation. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support explicit RAM-snapshot disablement and bounded-memory Flash Next disk persistence without changing f16 KV, live slot capacity or prefix reuse.

**Architecture:** The disk worker streams a frozen leased state through fixed buffers instead of owning a full serialized snapshot. The scheduler preserves that state until durable completion and continues other sessions. Legacy adapters retain their existing materialized persistence budgets.

**Tech Stack:** C++20, CMake/Nix, HIP gfx1151, existing HTTP/functional Python harness and INI converter.

**Spec:** [Disk-only cache design](../specs/2026-10-17-disk-only-streaming-cache-design.md).

Status: planning only; user review precedes implementation. Runtime resizing is deferred.

## Global constraints

- Linux x86-64 Strix Halo gfx1151; pinned toolchain for all matched comparisons.
- Preserve f16 KV, four full-context sessions, MTP correctness and disk payload version/identity.
- `--cache-ram-bytes off|N`: off disables retention, 0 remains auto, positive N remains bytes.
- Transfer chunks <= 1 MiB, existing direct-I/O buffer 1 MiB; no full KV snapshot for direct persistence.
- One direct job per frozen session, one persistence worker per store; peer sessions stay runnable.
- Accepted writes drain on graceful shutdown and survive client cancellation; failures remain visible.
- Legacy staging budgets remain enforced for copied-snapshot providers; streamed read eligibility ignores them.
- No resizing API, llama-swap changes or arbitrary recurrent checkpoint rewind in this increment.

## Review focus

1. Cancellation after write acceptance cannot destroy the producer state: Tasks 2 and 4.
2. A file larger than staging must index/restore without a full-file allocation: Tasks 2 and 5.
3. GPU transfers must not interfere with another session's graph capture or shared scratch: Tasks 3 and 5.
4. RAM-off must cover stable/shared/cancellation checkpoint branches, not just the main prompt path: Tasks 1 and 4.
5. A slow writer must not stall all peers or be misclassified as a lost GPU: Tasks 4 and 5.

## Task 1: Explicit RAM retention policy

**Files:** `src/cli/serve/text_model_runner.{hpp,cpp}`, `continuation_cache.{hpp,cpp}`, `serve.cpp`, `tests/cli/text_model_runner_test.cpp`, `tests/cli/serve_test.py`.

**Interfaces:** Add `bool enabled{true}` to TextRunnerRamCacheOptions. Parse the existing CLI value as either exact `off` or an unsigned decimal byte count. Disabled retention supplies zero actual RAM snapshot capacity without changing model prefix-reuse/snapshot capabilities or disk eligibility.

- [ ] Add tests asserting omitted/0/positive values preserve behavior, off reports disabled retention and invalid/negative/overflow values reject. Disabled RAM plus disk must not early-return merely because RAM reservation failed.
- [ ] Add fake-provider assertions for no RAM publication under off, correct live-prefix reuse, and disk-only save/restore. Keep internal execution rollback separate.
- [ ] Run tests to confirm missing behavior fails, implement parsing/policy, and rerun focused CLI/cache/runner tests. Use existing test harness conventions.
- [ ] Format and review; commit only task-owned code/tests once checks pass.

## Task 2: Size-independent direct publication and streamed read eligibility

**Files:** `src/cli/serve/continuation_disk_store.{hpp,cpp}`, `tests/cli/continuation_disk_store_test.cpp`; existing DirectFileWriter/SnapshotSource only if a shared bounded primitive is missing.

**Interfaces:** Add `PersistentStateProducer` containing exact `payload_bytes`, a `write(const TextModelRunner::SnapshotSink&)` callable and owned lifetime token. Add asynchronous direct-job submission returning a pollable completion with pending/stored/skipped/failed, byte count and error. The job must own its producer until completion. Factor PublishImage around an exact-size producer; legacy Save supplies its existing snapshot serializer. Completion publishes durable success only after existing fsync/rename steps.

- [ ] Add a generated multi-chunk producer test with payload larger than staging; assert durable save, restart indexing and restore succeed with the same small staging allowance and no payload-sized buffer. Assert checksum corruption and producer overflow/truncation reject.
- [ ] Test producer lifetime while blocked, mixed direct/legacy jobs, duplicate/spacing skips before producer invocation, accepted-write failure reporting, and Flush/shutdown drain.
- [ ] Keep legacy tests that block/reject oversized materialized captures. Update read-only staging tests to the new behavior; do not remove old format/safety coverage.
- [ ] Implement bounded direct-job admission and producer publication. Remove staging-size gates from ReadImage/startup indexing. Preserve file validation, disk quota and metadata checks; remove obsolete startup treatment that leaves oversized-but-valid files unindexed.
- [ ] Verify `nix develop -c cmake --preset cpu-test`, build `continuation_disk_store_test`, then `nix develop -c ctest --preset cpu-test -R '^continuation_disk_store_test$' --output-on-failure`.
- [ ] Format, review and commit Task 2 paths.

## Task 3: Flash Next direct live-state serializer

**Files:** `src/cli/serve/text_model_runner.{hpp,cpp}`, `inference_backend.cpp`, `src/models/qwen38_flash_next/engine.{hpp,cpp}`, `kernels/rocm/executor.{hpp,cpp}`, `tests/models/qwen38_flash_next/snapshot_test.cpp`.

**Interfaces:** Add runner capability `direct_persistence`; add `PersistentStatePayloadBytes(const TextRunnerState&) -> size_t` and `StreamPersistentState(const TextRunnerState&, const SnapshotSink&) -> void`, with unsupported defaults. Flash Next implements model framing plus an executor region stream. Contract: caller freezes and owns state; serializer reads only that state using a private transfer stream and fixed buffer, never calls SnapshotForPersistence or materializes a borrowed snapshot.

- [ ] Add tests comparing streamed output byte-for-byte against the current materialized serializer for AR/MTP, multiple token depths and image state. Reject a pending speculative batch just as the old snapshot path does.
- [ ] Assert emitted size matches declared size, maximum emitted transfer chunk <= 1 MiB and no host/device allocation proportional to payload. Collect bytes only in test oracle code; production sinks must remain streaming.
- [ ] Implement traversal using existing header/WalkSnapshot order, with deterministic padding and bounded device-to-host copies. Preserve recurrent, indexer, MTP, logits, token, vision and draft-policy fields. Do not touch numerical kernels.
- [ ] Build `qwen38_flash_next_snapshot_test` under gpu-test and run it with the repository's documented model arguments on node-strix. Verify both existing and streamed restoration against numerical checks, including peer execution/graph capture overlap.
- [ ] Format, review and commit Task 3 paths. Do not claim direct persistence for other model adapters.

## Task 4: Frozen-session integration and peer progress

**Files:** `src/cli/serve/text_model_runner.{hpp,cpp}`, `text_generation_scheduler.{hpp,cpp}`, existing `tests/cli/text_model_runner_test.cpp` and `text_generation_scheduler_test.cpp`.

**Interfaces:** Extend request persistence preparation with a pollable direct completion owned by the request. Keep a lease until completion and expose a persistence-pending state to scheduler work selection. The disk job holds an explicit shared ownership token; no detached raw state reference. Existing synchronous pool entry points may wait, but scheduler paths only submit/poll.

- [ ] Add fake-worker tests for blocked write/durability: source cannot decode, rewind, invalidate or be reassigned; a peer continues advancing. Completion releases the freeze exactly once and resumes the correct request.
- [ ] Test cancellation before/after acceptance, queue pressure, worker failure, shutdown, device loss and delayed publication. Accepted writes remain durable unless an explicit failure occurs; failures are reported without dangling ownership.
- [ ] Exercise ordinary prompt, stable framing, learned shared-prefix, in-pass and cancellation capture paths. Under RAM-off no full snapshot is created solely for persistence. Under RAM-on normal retention may remain, but direct writes do not materialize it.
- [ ] Integrate direct jobs by capability, preserving legacy fallback for other models. Keep accepted-work spacing and exact replacement semantics. Add separate logical queued bytes, actual transfer-buffer bytes, persistence mode and latency/progress observations.
- [ ] Ensure watchdog progress covers real transfer/write completion without heartbeat masking of a hung GPU. Avoid busy polling and blocking the owner on metadata locks, I/O or queue space.
- [ ] Build/run `continuation_cache_test`, `text_model_runner_test`, `continuation_disk_store_test`, `text_generation_scheduler_test`; run affected ownership tests under CPU sanitizers. Verify peer progress with bounded deterministic synchronization rather than fragile timing alone.
- [ ] Format, review and commit Task 4 paths.

## Task 5: Functional qualification, docs and deployment configuration

**Files:** `tests/functional/cache_lifecycle.py`, `cache_disk_spacing.py`, `cache_disk_faults_test.py`, `tests/functional/README.md`, `docs/KV-CACHE.md`; converter `tests/test_gufo_engine.py` and parser only if needed to preserve literal off; `models.ini`/generated `config.yaml` after qualification.

- [ ] Extend disk lifecycle coverage to RAM-off, larger-than-staging checkpoints, restart reuse, shared-prefix branches, failed writes and concurrent peer work. Replace the direct-provider oversized-staging expected-skip case with successful persistence/restore; retain legacy oversized-capture tests.
- [ ] Adapt the crash-at-fsync case: direct persistence holds the source request, so wait for the publication gate rather than requiring that request to complete before killing the child. Assert a peer remains responsive and restart sees only durable files; keep the legacy asynchronous case where applicable.
- [ ] Establish matched production baseline/candidate artifacts on node-strix. Record exact revisions, pinned toolchain, model hashes, context/slots, MTP settings and cache history. Preserve existing checkpoint files and the actual systemd service configuration.
- [ ] Measure long-context saves/restores and four-slot workload peaks with embeddings: process RSS, host available memory, GPU allocations, actual buffer allocations, queue depth, frozen duration, TTFT and decode timings. Prove memory grows with bounded buffers/metadata rather than checkpoint payload. Do not infer this solely from chunk-size counters.
- [ ] Run independent numerical quality and standard speed checks. Run affected shared functional suites across the four documented profiles/modes, including cache/long-context and affected cancellation/image/state edges. Verify actual drafts. Missing models remain incomplete, never a pass.
- [ ] Compare affected timings per request/phase against matched main at 5% and 3 ms; investigate flags with alternating affected histories and retain every result. Explicitly report unavoidable frozen-slot latency; resolve confirmed regressions or return the tradeoff for user review rather than silently shipping slower behavior.
- [ ] Verify converter round-trip of `--cache-ram-bytes off`; test any necessary parser change in `tests/test_gufo_engine.py` and affected converter suites. No llama-swap scheduler implementation is needed.
- [ ] Document off versus numeric zero, direct versus legacy staging semantics, exact-prefix limitations, and new counters. Mark the resize plan deferred pending these memory results.
- [ ] Run required format check and bounded hosted check; stage only task-owned files for Nix. Review the integrated change before deployment.
- [ ] When implementation/deployment is authorized and qualification passes, use tgrid plus the node-strix multi-backend image build. Update generated config to RAM-off without changing slots/context, preserve previous image/config for rollback, restart the discovered service and verify real disk hits plus embedding coexistence.

## Execution recommendation

Native execution is recommended: serializer, state lifetime and disk publication
have tightly coupled contracts. The plan does not authorize implementation by
itself. Review the design and plan, then choose native or delegated execution
when ready. Reassess runtime resizing after the measured memory improvement.
