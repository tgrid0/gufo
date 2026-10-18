# Flash Next measured memory and full-context slot cost

The startup-precalculation plan is cancelled in favor of measured configuration
profiles. On the minimal-cache branch, each additional 262144-token MTP slot
added approximately **7 GiB** to the observed peak over the tested 1-4 slot range.

Four slots completed the text workload with some process paging. Five slots
were rejected before readiness by the existing request-state capacity check.
This is empirical capacity evidence, not a proof of an allocation bound.

## Configuration and results

Measured 2026-10-18 on Strix Halo gfx1151, production candidate
`c236a86c1609`, binary SHA256 `a7d5fd41bbbba85d6`.
The five changed production-file hashes and LF-normalized flake lock match
the retained production build. Full identities and phase measurements are in
[the machine-readable results](2026-10-18-flash-next-memory-slots.json).

The baseline uses the live config's Flash Next command on this candidate:
UD-Q4_K_XL, 262144 context, two sessions, shared-Q8 MTP with four drafts,
BF16 projector, thinking on/medium, temperature 1, top-p 0.95, top-k 20,
4 GiB RAM cache and 128 GiB disk quota. Test servers use private loopback ports
and fresh task-owned disk directories. The slot sweep changes only sessions
and RAM retention to off. The old 8 GiB disk-staging argument is preserved
from the config, but this branch logs it as ignored and uses 1 MiB staging.

All GiB values use 2^30 bytes. HIP is a device-wide counter on shared physical
memory; it is not process RSS. The empty-system HIP reading is 2.964 GiB.
The peak-above-idle column subtracts that observed baseline, without claiming
complete process attribution. **Do not add HIP usage and RSS.**

| Configuration | Ready HIP GiB | Peak HIP GiB | Peak above idle GiB | Minimum host available GiB | Peak process swap MiB |
| --- | ---: | ---: | ---: | ---: | ---: |
| Current cache, 2 slots | 101.73 | 104.19 | 101.22 | 20.02 | 0.0 |
| RAM off, 1 slot | 95.03 | 96.55 | 93.59 | 27.71 | 0.0 |
| RAM off, 2 slots | 101.92 | 103.59 | 100.63 | 20.62 | 0.0 |
| RAM off, 3 slots | 108.83 | 110.63 | 107.66 | 13.52 | 0.0 |
| RAM off, 4 slots | 115.65 | 117.53 | 114.56 | 6.67 | 158.9 |

Successive RAM-off peak increases were **7.045, 7.033, 6.899 GiB** (1->2, 2->3, 3->4).
Use about 7 GiB per additional slot as an observed planning increment within
this range; keep operational headroom separate and remeasure after changes to
the model, context, drafts, cache policy, libraries or workload.

Host memory pressure provides a separate cross-check: the decrease from
prelaunch MemAvailable to the minimum during each run was:

| Configuration | Additional host pressure GiB |
| --- | ---: |
| current-2 | 102.13 |
| off-1 | 94.59 |
| off-2 | 101.64 |
| off-3 | 108.59 |
| off-4 | 115.67 |

This is a host-wide availability change, not an exact process allocation total;
reclaimable file cache and kernel accounting differ from HIP accounting.
Four slots had up to 158.9 MiB process swap, beginning during the short warmup.
The other configurations had zero sampled process swap. Host swappiness was 60.
Four slots therefore are not qualified as swap-free capacity. Embeddings and
other models were not loaded; these numbers do not establish coexistence.

## Workload and coverage

Each point starts a fresh server, warms all slots concurrently with 2040 input
tokens and 128 output tokens, then sends one independent 260990-token prompt
per slot concurrently, requesting another 128 output tokens. These are near-full
requests in full 262144-token-capacity slots, not exactly 262144 occupied tokens.
All generation requests returned 128 tokens and reported actual MTP drafts.
Full requests reported zero reused prompt tokens.

The monitor samples hipMemGetInfo, /proc/meminfo and process RSS/high-water/swap
through startup, concurrent prefill/decode, capture settlement, restart, replay
and shutdown. Median cadence was 100 ms; maximum observed interval across the
completed runs was 139 ms. Brief unsampled peaks can be missed. The guard would
terminate the test below 1.5 GiB MemAvailable or above 256 MiB net additional
host swap; it did not trigger.

After capture settlement each server restarts and replays the last prompt.
RAM-off C1 and C2 restored 260985 tokens from disk. The baseline and RAM-off
C3/C4 restored only 2021 tokens and recomputed the remainder. The logs contain
skipped large snapshots, so successful generation must not be interpreted as
durable persistence of every full context. All replay requests completed.

The BF16 projector's 903984064 resident bytes are included, but no image
encoding requests were exercised. Prompts use a deterministic repeated text
archive with distinct leading slot identities. There is one observation per
configuration, not an exhaustive set of MoE routing, image, grammar, sampling
or simultaneous-restore shapes. No new numerical-equivalence or throughput
claim is made; model/runtime code was not changed.

## Capacity boundary and disposition

The five-slot startup-only check exited 1 with
`text runner request-state claim exceeds state capacity` before readiness.
It did not hit the monitor's pressure guard, execute requests or demonstrate
an actual out-of-memory failure. This establishes the current admission limit
for this configuration, not a theoretical hardware limit.

The original llama-swap service was restored and verified active. Live YAML,
production image, model files and existing production checkpoints were not
changed. Tests used the existing production candidate in a separate container.
At measurement completion, the original conflicted checkout and unfinished
startup-memory implementation were preserved; their later disposition is recorded below.

Raw harnesses, configuration snapshot, logs, requests and 100-ms samples are
retained in the editing workspace's ignored `build/memory-slots-20261018/`
and `build/memory-slots-limit5-20261018/` directories, with matching remote
evidence directories under the existing minimal-cache qualification root.
The JSON result records raw report/sample hashes. No configuration secrets
or generated response text are included in the versioned summary.

The retained direction is measured profiles for this model/configuration,
rather than complete startup reservation/warmup coverage for every library.
The old admission plan and spec are now cancellation records; their former
contents remain in Git history.


## Final user decision: cancel precalculation and retain FP16 KV

On 2026-10-18 the user accepted the measured results, chose to continue with the
existing implementation while waiting for Q8 KV support to merge upstream,
and explicitly cancelled the startup-memory precalculation work.

A fresh upstream check at commit `bff0eec6f1ea` found Flash Next target and
MTP KV still stored as FP16. [Q8 KV PR #561](https://github.com/gufo-org/gufo/pull/561)
was open and unmerged: its proposed option is `--kv-quant q8` and its Q8_0
rows occupy 544 rather than 1024 bytes. That reduction applies to KV storage,
not the entire measured per-slot cost. Temporary FP16 prefill staging and
quality validation still require consideration. The author's
[latest validation](https://github.com/gufo-org/gufo/pull/561#issuecomment-6207053280)
reported no recurrence of earlier severe behavior but 6-7/8 agent tasks passed.
No Q8 code was integrated or deployed.

The cancelled work was removed from the original checkout: 53 tracked files
were restored to that branch's HEAD and 36 untracked implementation/test files
were removed. All were checked against the startup-memory stash and an
existing byte-identical recovery backup. Its stash was removed from the active
stash list after preserving an archive reference. Unrelated disk-cache and
runtime-session notes were preserved. These backups are recovery history,
not a deferred implementation queue.

Documentation links and JSON syntax are checked before committing these
findings. No runtime changes are included in this findings commit.
