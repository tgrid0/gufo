# Minimal component-cache qualification

Status: implementation and correctness checks complete; performance qualification
inconclusive. Not a deployment or zgfs replacement approval.

## Implementation and preserved work

The replacement starts at upstream `a8ed7af2160bdc574514e465bd19d56d804ccd75`
(0.17.1), on `fix/zgfs-minimal-cache` in `build/reconcile-main`.
The five production files add explicit `--cache-ram-bytes off` parsing and
forward a retention policy to the current cache. Numeric zero remains automatic;
positive budgets preserve upstream behavior. Off refuses retained history and
uses the existing durable capture path when disk caching is enabled.

The original conflicted zgfs checkout remains untouched. Its files and index
were backed up under `build/main-review-backup-20261018`; the backup branch
`backup/zgfs-before-minimal-cache-20261018` preserves `1eb8d857` and the original
startup-memory stash remains available. Startup admission and runtime resizing
are deferred. Old serializers and scheduler changes are not ported.

Off still permits transient private state, tails, metadata, free backing and
transfer staging. It preserves upstream borrowed-source lifetime semantics:
queued writes can retire if their live source is overwritten. It does not
promise that every accepted queue entry becomes durable.

## Build provenance

Remote evidence root:
`walker@node-strix:/home/walker/llm/gufo-disk-only-20261017/minimal-cache-20261018`.

Upstream and candidate were exported from the same base and built with the
pinned Nix production toolchain. Both exports received identical UTF-8 CRLF to
LF normalization after shell install checks failed on archive line endings.
The failed build logs and normalization path manifests are retained. This
normalization does not change the checked-out source or dependency versions.

The source archives lack release metadata, so both binaries identify themselves
as `development (dirty)`. Revision, patch manifest, binary hashes and Nix store
paths are recorded separately; these are not claimed to be release binaries.
Production binaries copied to `runtime/{upstream,candidate}/result/bin/gufo`
were verified against their Nix outputs.

## Completed checks

- Fresh WSL CPU baseline: component cache and runner, 2/2 passed.
- Regression tests failed before implementation: CLI rejected off; component
  and legacy paths retained snapshots despite the disabled policy.
- Candidate focused CPU: CLI, component cache and runner, 3/3 passed.
- Debug ASan/UBSan rerun: component cache and runner, 2/2 passed (1412.03 s).
- Pinned format check: all 584 C++ files passed.
- Both pinned production package builds passed, including install checks.
- Candidate hosted PR check passed formatting, dependency inventory,
  documentation, CPU contracts and the mkGufoServe CLI string check.
- Independent review found no critical, important or minor issues.
- Initial candidate HTTP runs passed state-edges, long-context and cache
  correctness in AR and MTP modes, with no quality or coverage changes against
  the matched upstream runs. Their timing result is **inconclusive**, not passed.
- Retention-off AR and MTP disk lifecycle: restart, oversized transfer, crash
  and side traffic passed.
- GPU snapshot numerical test passed (151.18 s), including exact snapshot replay.
- Independent MTP oracle passed (24.03 s): full-width norm, byte-exact split,
  attention, recursive carry, full-Q8 head, image IDs and independent batches.
- Revised retention-off startup/restart check passed in AR and MTP, including
  actual draft execution in MTP mode.

Standard depth-zero speed observations, same pinned builds and model paths:

| Workload | Upstream prefill tok/s | Candidate prefill tok/s | Upstream generation tok/s | Candidate generation tok/s |
| --- | ---: | ---: | ---: | ---: |
| Single AR | 1793.10 | 1801.12 | 28.46 | 28.43 |
| Single MTP | 1723.27 | 1736.73 | 35.90 | 35.86 |
| Single MTP repetition | 1732.30 | 1745.96 | 116.36 | 116.56 |

These single observations do not establish a speedup or replace per-request
functional timing gates. Accepted tokens per step matched at 0.66 and 6.53 for
the two MTP workloads. Raw artifacts are under `speed-upstream` and
`speed-candidate` in the remote evidence root.

## Investigation and pending gates

The initial startup script assumed that retrying an earlier prompt always uses
a RAM checkpoint. With retention off and no disk, the live state is already
past that prompt and recomputation is expected. Both initial startup runs failed
that assertion while reporting zero captures and zero snapshot bytes. The test
now checks this policy explicitly, waits for durable publication before the
disk retry, and verifies disk restoration. The failed artifacts remain retained;
the revised test is run separately without altering the original timing harness.

The Windows host reboot interrupted the first local sanitizer run and cleared
its temporary log. That run is unqualified. A fresh run writes to
`build/sanitizer-reboot.log` and passed. An optimized ASan/UBSan build with
assertions was also started while investigating the long Debug runtime, then
stopped after the required Debug run passed. Its incomplete result is retained
in `build/sanitizer-opt-test.log` and is not counted as a pass.

Per-request timings retain the 5% and 3 ms gates. Matched follow-up runs are
retained for the flagged histories, including preceding cache requests required
by disk-restore measurements. Before controls, the AR pairs reproduced one failing
phase: `legacy-tool-cancel-disk` snapshot time was 19.58/20.43 ms upstream and
29.19/26.48 ms in the candidate. MTP showed the same phase flag.
For that request, decode shifted from 69.32/67.24 ms to 62.41/62.21 ms and total
server duration stayed 268.4/268.5 versus 271.0/268.9 ms. This suggests phase
attribution of asynchronous work, but does not waive the phase-level failure.
An unchanged-upstream control then reproduced the slower snapshot measurement:
28.69 ms in AR and 28.52 ms in MTP. Including these controls yields:

| Mode | Passed phase measurements | Confirmed regressions | Inconclusive |
| --- | ---: | ---: | ---: |
| AR | 1011 | 0 | 18 |
| MTP | 1016 | 0 | 13 |

The upstream ranges are variable and overlap the candidate ranges within the
existing margins. This does not qualify performance: timing remains explicitly
inconclusive. No timing margin has been widened and no failed or inconclusive
observation has been discarded. The initial failed paired analyses are retained
as `timing-qualified-{off,mtp}.json`; the combined results and evidence hashes
are `timing-with-control-{off,mtp}.json`. Further full matrix repetitions are not
warranted by these variable observations; no speculative production fix was made.

The AR lifecycle exercised prompts up to 24,314 tokens. Its 13,498-token restart
request restored 13,491 tokens, and a later continuation reused 20,711 live
tokens. These requests reported zero retained snapshot bytes; transient cache
RAM reached roughly 3.14 GiB on the inspected continuation requests.

One-second host samples during that lifecycle recorded at least 34,174 MiB
MemAvailable, a peak 18.8 MiB Dirty, and zero sampled Writeback. Cached grew from
9,038 to 10,638 MiB; SwapFree varied by 0.2 MiB. Sampled process RSS peaked at
5,872 MiB (this excludes GPU allocations and is not total model memory).
These are host-wide observations, not isolated accounting of cache files; they
do not qualify constrained-memory workloads. They show no pressure warranting
a direct-I/O port in this bounded run, so buffered upstream I/O is retained.

Keep the branch separate while performance is unqualified. The existing service
has not been restarted or deployed over.
