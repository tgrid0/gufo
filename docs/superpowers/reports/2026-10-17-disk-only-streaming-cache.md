> Historical evidence only: these results qualify the recorded pre-component-cache revision, not main 0.17.1 or the replacement patch. Preserve all failures and deployment blockers.

# Flash Next disk-only persistence qualification

Implementation: `936fbfafeb5ec4153ae97d0814169bc5d3918094`, branch
`feat/disk-only-streaming`. Runtime slot resizing remains deferred.

The implementation is verified, but the requested four 262144-token f16 MTP
slots plus embeddings failed the capacity/pressure test. It is not deployed.
Direct persistence also introduces a measured frozen-slot latency tradeoff.

## Artifacts and scope

All remote evidence is retained under
`/home/walker/llm/gufo-disk-only-20261017` on `walker@node-strix`.

| Artifact | Identity |
| --- | --- |
| Baseline source | `275e7425fbc0090e1556796dc5816e3f018f21e2` |
| Candidate source | `936fbfafeb5ec4153ae97d0814169bc5d3918094` |
| Baseline binary SHA256 | `93bedc319edcd9fd3a7372a97a14dfbf0be5d5841432ea6e42354e1d4e413b00` |
| Final candidate binary SHA256 | `bad2b2e7fd23e6fdeda827fa5c5dbb1bf3effec3a0af7f55278c9cbb09a398fb` |
| Pinned flake.lock SHA256 | `faabad820ec09caab7351bf48910cec895a38cf1766e098e3065aea86f405ed6` |
| Candidate production store | `/nix/store/2mrk9mjgl7zr9hbmqn01cn0k162r9bhq-gufo-0-unstable-dirty` |

Both source archives were compared against their declared commits for production
source and build inputs; no differences. Both use the same pinned Nix compiler
and dependencies. The archive build's version suffix is dirty; it is not a
claim that the binary embeds the final commit identifier.

The user explicitly limited model qualification to Flash Next. The target is
UD-Q4_K_XL with shared Q8_0 MTP, BF16 projector for image tests, and Qwen3
Embedding 8B Q8_0 for coexistence. Other model profiles were not qualified.
Complete model file hashes are retained in `model-sha256.txt`.

Initial HTTP v3 runs used candidate SHA256
`cb6f7f203bf5d0e764a980e887e64a70474f83957f2c6860417a93af91338fa7`.
The final binary adds the narrowly scoped ready-future immediate-EOS routing
fix. Final CPU stress, targeted AR controls, MTP replay, and memory tests use
the final binary. Persistence bytes and numerical kernels did not change.

## Verification

- Pinned C++ formatting: 569 files, clean.
- Bounded hosted PR check: 56 tests plus format, dependency inventory,
  documentation and CLI construction checks passed.
- Focused ASAN/UBSAN cache, runner, disk store and scheduler: 4 tests passed.
- Flash Next AR/MTP streamed bytes equal the materialized snapshot oracle,
  including peer graph overlap. Text/image/image-append restored logits agree
  exactly. Independent MTP numerical audit passed.
- HTTP state-edges, long-context and cache correctness passed for AR and MTP;
  execution counters verify actual draft use in MTP and zero drafts in AR.
- RAM-off lifecycle restart, oversized checkpoint and crash-at-fsync passed
  in both modes. A peer responds while publication is held; SIGKILL exposes
  no published partial checkpoint.
- Converter literal `off`, `0`, positive byte limit: 3 tests passed. No parser
  change is necessary. Its broader suite has one pre-existing stale canonical
  model-name assertion; this is not reported as a complete converter-suite pass.
- Fresh review's three important findings were fixed and tested: terminal EOS
  persistence, cancellation/deadline cleanup after an accepted boundary write,
  and missing queue/transfer/write/durability timings. No deferred minor findings.

## Requested capacity fails before persistence can help

Embedding-first startup rejects the state-capacity claim on both versions.
Gufo-first candidate startup succeeds with four 262144-token f16 MTP slots,
using 115036 MiB of GPU-reported memory before loading embeddings.
After embeddings load, idle usage is 123.846 GiB. The first prefill reaches
123.898 GiB and aborts with HIP out-of-memory in the existing MMQ scratch arena,
before a checkpoint is published. A later attempted stop found the test had
already exited. This is a failed capacity test, not a successful deployment.

Artifacts: `memory-candidate`, `memory-baseline`, `memory-candidate-v2`.

## Diagnostic memory comparison

To isolate persistence overhead, both versions were measured with four
196608-token f16 MTP slots and embeddings, Gufo loaded first. This is a diagnostic
setting only; no reduction of the deployment context was made. Baseline used
4 GiB RAM retention and 8 GiB legacy staging; candidate used RAM off and 1 MiB
legacy staging. Both used a fresh task-owned cache on the dedicated cache NVMe.

| Phase | Baseline peak GPU GiB | Candidate peak GPU GiB | Baseline minimum available MiB | Candidate minimum available MiB |
| --- | ---: | ---: | ---: | ---: |
| Idle with embeddings | 117.143 | 117.143 | 7656.1 | 7740.8 |
| 32030-token request | 119.149 | 117.578 | 4854.5 | 7155.8 |
| 128030-token request | 123.999 | 117.701 | 40.7 | 7045.7 |
| Four concurrent 32032-token requests plus embeddings | 123.999 | 118.340 | 79.7 | 6300.5 |

All six generation requests returned ALPHA and the embedding request returned
four vectors. Candidate Gufo RSS was about 486 MiB after the small save and
491 MiB after the large save, despite checkpoint sizes of 998732148 and
3635276148 bytes. Direct staging usage stayed zero. Combined process RSS and
GPU memory must not be added: Strix Halo shares physical memory.

The large request is not a matched latency comparison: baseline reused a
30720-token intermediate RAM checkpoint, whereas candidate prefilled the full
128030 tokens. The changed final instruction diverged before the saved disk
boundary. The recurrent model cannot rewind that later saved boundary to an
arbitrary common prefix. Disabling RAM retention therefore also changes which
intermediate checkpoints are available.

Both diagnostic runs failed the subsequent Gufo reload while embeddings
remained loaded, at the existing startup capacity guard. Their reports remain
marked failed. Focused Gufo-first restores passed on both versions with embeddings loaded
after Gufo. Both reused 128023 tokens and prefilled seven. Candidate request
time was 13600.2 ms versus baseline 14305.9 ms; candidate peak GPU memory was
117.568 GiB versus 120.840 GiB. Candidate Gufo RSS at completion was 517 MiB
versus 4087 MiB. These separate checks establish large-file restoration without
changing the failed reload reports.

Artifacts: `memory-{baseline,candidate}-192k` and
`memory-{baseline,candidate}-192k-restore`.

## Latency and speed

The new path keeps its source slot frozen through durable publication. Peers
remain runnable, but a request needing that state pays the save time.

| Candidate checkpoint | Transfer/serialization ms | File write/copy/checksum ms | Durability ms | Total frozen save ms |
| --- | ---: | ---: | ---: | ---: |
| 998732148 bytes | 111.336 | 1860.50 | 1.891 | 1974.74 |
| 3635276148 bytes | 373.622 | 6648.75 | 3.936 | 7028.57 |

Short-request alternating AR controls reproduce roughly 100 ms of additional
accepted-save/first-token delay. Small saves spend roughly 12-15 ms in transfer,
85-90 ms in writing and 2-3 ms in durability. This exceeds the unchanged 5% and
3 ms criteria. First-image prefill and some inter-token outliers overlap variable
unchanged-baseline controls and remain inconclusive, not passed.

Standard no-disk pp2048/tg128 speed, depth zero, one observation per cell:

| Workload | Baseline prefill tok/s | Candidate prefill tok/s | Baseline decode tok/s | Candidate decode tok/s |
| --- | ---: | ---: | ---: | ---: |
| AR | 1700.57 | 1703.62 | 28.45 | 28.48 |
| MTP | 1638.06 | 1636.94 | 34.00 | 34.03 |
| MTP repetition | 1646.55 | 1640.36 | 65.12 | 65.23 |

Accepted tokens per MTP step match (0.66 and 3.74). This is evidence that the
no-disk path's standard speed is unchanged, not a blanket cache-latency pass.
All request/phase comparisons and unchanged-main controls are retained without
averaging or widening margins.

Artifacts: `http-*-v3`, `controls-*`, `ar-controls-qualification.json`,
`mtp-controls-qualification.json`, `speed-*-v2`.

MTP alternating controls also have zero quality/coverage differences, but the
overall timing qualification is failed. Save-related delays reproduce. Most
decode/restore outliers overlap variable controls and remain inconclusive.
One RAM-enabled legacy-tool one-token decode phase remains outside margin:
baseline range 48.90-49.56 ms, candidate 53.20-74.81 ms. In the final matched
pair, total request time nevertheless improved from 313.87 to 292.78 ms because
restore/capture were faster. This does not override the strict per-phase flag.
The new RAM-enabled path retains borrowed snapshots instead of materializing
them for disk; a work shift is plausible but not established by a kernel profile.
Queue and cancelled-state restore flags also remain visible in the report.

A separate focused legacy-tool control disabled retention in both versions
(baseline's 1-byte limit, candidate's explicit off). Two matched observations
and an unchanged-baseline control show the first one-token decode flag did not
reproduce consistently: baseline 54.00-55.02 ms, candidate 53.32-58.85 ms. It
remains inconclusive. Repeated warm requests were within margin. This diagnostic
does not erase the RAM-enabled flag or qualify all latency metrics.
See `legacy-no-ram-qualification.json` and `mtp-stall.txt`.

## Operational disposition

The original `llama-swap.service` was restarted and verified active after all
GPU tests. Live INI/YAML and existing checkpoints are unchanged. The previous image is
tagged `localhost/llama-multi-backend:before-disk-only-20261017`; service config
is backed up as `config.before.yaml`. Proposed four-slot/RAM-off/1-MiB-staging
configuration is reviewable under the editing workspace's
`build/disk-only-streaming/config-change.diff`, but failed capacity and measured
latency prevent treating it as qualified for deployment.

A local evidence archive containing 611 logs/measurement files is available at
`W:/projects/llm/gufo/build/disk-only-streaming/qualification-evidence.tar.gz`.
Initial invalid environment/harness runs are retained alongside valid results;
they are not counted as passing tests.

The approved plan requires returning an unavoidable frozen-slot tradeoff for
user review rather than silently shipping slower behavior. Slot resizing and
other changes to live-state capacity are separate work.

## Recorded decisions

- Ruling: Use WSL GCC for focused CPU iteration and pinned Nix on Strix for production qualification — host has no Nix; GCC evidence alone is not production qualification.
- Ruling: Direct queue job count defaults to 8 for standalone stores and is set to pool session count in serving — one job per owner and frozen session bounds accepted storage; busy admissions skip without waiting — a too-small standalone cap skips optional checkpoints.
- Ruling: Defer per-task commits until pinned formatting and integrated checks — shared runner interfaces overlap Tasks 1/3/4; no unverified C++ commit — larger review range.
- Ruling: GPU tests use the production package development environment with GUFO_BUILD_TOOLS=OFF plus pinned Python — avoids unrelated profiling and Torch dependencies, retains compiler/dependencies and assertions — profiling tools are not qualified by this run.
- Ruling: Do not select experimental tiered lifecycle case — it requires cross-tier best-prefix lookup explicitly excluded by the approved spec — that pre-existing behavior remains unsupported.
- Ruling: User explicitly narrowed model qualification to Flash Next only; AR/MTP retained, other model profiles omitted by authorization. Shared hosted CPU and sanitizer checks remain required.
- Ruling: Image byte oracle extends the existing shared Qwen vision fixture rather than duplicating it in snapshot_test; both Flash Next modes and text/image/image append paths are exercised.
- Ruling: Review declined other adapters/profiles, cross-tier lookup, arbitrary rewind, resizing, pending qualification and deployment; agree with all exclusions because they match explicit deferred/user-narrowed scope or remain pending execution, not silently passed.
- Ruling: Proposed deployment restores sessions from the experimental 2 to the explicitly requested 4, preserving 262144 context and f16 KV; RAM off and 1 MiB legacy staging are the only other command changes. No live configuration changed yet. The spec's four-session target and explicit user request take precedence over the plan's wording about unchanged slots; risk is the larger mandatory live-state peak, measured before deployment.
- Ruling: Diagnose bounded persistence at 196608 context with four slots and embeddings, keeping 262144 failure visible; this is a measurement setting only, not an authorized deployment reduction — separating unavoidable live allocation from snapshot overhead — cost if wrong: diagnostic cannot qualify the requested capacity.
