# Minimal component-cache implementation plan

> **For agentic workers:** Use superpowers:executing-plans for native execution.

**Goal:** Replace obsolete zgfs cache patches with explicit retention-off support on main 0.17.1.

**Architecture:** Keep the upstream component cache, stream transfers, disk format and scheduler. Separate the RAM retention policy from the existing transient resource budget.

**Tech Stack:** C++20, CMake, Linux gfx1151, pinned Nix production toolchain, focused CPU and HTTP functional tests.

**Spec:** [Design and review findings](../specs/2026-10-18-minimal-component-cache-design.md).

Status: implemented and correctness-checked on 2026-10-18. Timing remains inconclusive after matched follow-ups and unchanged-upstream controls; keep the branch separate. See the qualification report.

## Global constraints

- Base a8ed7af; preserve original zgfs checkout, stash, index and untracked files.
- `--cache-ram-bytes off|N`; zero remains automatic; positive budgets retain upstream behavior.
- Preserve f16 KV, slots/context, live reuse, disk formats and upstream queue/admission semantics.
- No startup admission, resizing, other-model serializer changes or deployment.
- Off disables retention, not bounded transient metadata/private-state/backing allocations.

## Review focus

1. Off without disk must still serve and reuse live state without capturing retained snapshots.
2. Off with disk must publish and restore across restart, including small checkpoints that would otherwise fit RAM.
3. Cancellation/peer mutation must preserve upstream transfer ownership and coherent private state.
4. Resource pressure must refuse optional capture cleanly without claiming durable success.
5. Default and positive budgets must preserve upstream retention and incremental chunk reuse.

## Task 1: Retention policy on the current cache

**Files:** src/cli/serve/{serve.cpp,text_model_runner.hpp,text_model_runner.cpp,component_text_cache.hpp,component_text_cache.cpp}; tests/cli/{serve_test.py,component_text_cache_test.cpp,text_model_runner_test.cpp}.

**Interfaces:** TextRunnerRamCacheOptions::enabled defaults true. Add a final default-true retention-enabled parameter to ComponentTextCache and forward it from TextRunnerPool. The component Impl stores the policy; existing ram bytes remain the resource ceiling.

- [x] Configure a fresh CPU baseline and run component_text_cache_test and text_model_runner_test before edits.
- [x] Add CLI tests accepting off, zero and positive values and rejecting invalid/negative/overflow values; observe failure on upstream.
- [x] Add component tests proving off/no-disk refuses retained captures while live-prefix reuse works, and off/disk publishes small and large checkpoints, leaves no RAM checkpoint available, and restores after restart. Observe failure before implementation.
- [x] Parse off through the canonical option; preserve numeric behavior. Apply enabled to the legacy retention policy as well as component routing.
- [x] Make TryReserveSnapshot reject history-only retention when off and optional snapshots without disk. Route Capture directly to CaptureDurable when off; prevent publication of retained RAM checkpoints. Preserve current durability and ownership APIs.
- [x] Add tests for cancellation, peer progress, insufficient transient budget and shutdown drain using existing fixtures. Keep default/positive-policy and shared-chunk tests.
- [x] Run cmake --preset cpu-test; cmake --build --preset cpu-test --target component_text_cache_test text_model_runner_test; ctest --preset cpu-test -R '^(component_text_cache_test|text_model_runner_test)$' --output-on-failure. Run CLI coverage through its existing CMake target and affected cache queue/streaming tests and sanitizers.

## Task 2: Current documentation and qualification

**Files:** docs/{CLI.md,KV-CACHE.md}, historical plan/spec status notices; tests/functional only where an existing assertion needs explicit policy coverage.

- [x] Document off versus zero, transient resource memory, existing 1 MiB disk staging, current disk format and policy metrics. Keep historical failed measurements visible and mark old implementation plans superseded; startup admission and resizing stay deferred.
- [x] On the remote GPU, build candidate and matched upstream with the pinned toolchain. Run affected Flash Next AR/MTP cache/long-context correctness, numerical quality and standard speed checks. Inspect actual draft activity and retained RAM versus transient allocations.
- [x] Compare affected per-request/phase timings at 5% and 3 ms. Investigate flags with alternating affected histories, retaining all evidence. Do not reuse old-architecture timings.
- [x] Measure buffered disk I/O/page-cache pressure before adding O_DIRECT. If a gap is demonstrated, specify a focused compatible patch and regression test before implementation; otherwise record why no direct-I/O patch was retained.
- [x] Run the repository format check and bounded hosted contract check. Review the final diff against a8ed7af; it must exclude legacy serializers and startup-memory kernel/JSON rewrites.
- [x] Report exact passing checks and any incomplete GPU qualification. Commit only verified task-owned paths with Conventional Commits. Do not deploy or replace zgfs until validation supports doing so.
