> 2026-10-18 review: superseded for Flash Next by the minimal component-cache plan. Historical requirements and results remain below; do not execute the legacy implementation tasks against main 0.17.1.

# Disk-only cache with bounded-memory persistence

Status: proposed design. The user requested planning first on 2026-10-17;
runtime session resizing is deferred. No implementation or deployment changes.

## Intended result

Allow Qwen3.8 Flash Next to retain conversation checkpoints only on disk, find
them by token prefix, and save/restore large histories without allocating another
complete KV snapshot. Keep four full-context sessions, f16 KV, MTP and the current
disk checkpoint format. This reduces persistence-related peaks, not live KV.

The inspected llama.cpp fork streams live sequence state directly to disk.
Gufo currently streams file I/O but first materializes a complete Flash Next
snapshot in host memory through `SnapshotForPersistence(kMaterialized)`.
Removing the staging checks alone would retain that copy and remove its budget.

## Configuration

Extend the existing canonical option to `--cache-ram-bytes off|N`. `off` disables
retained RAM snapshots; numeric 0 continues to mean automatic sizing and positive
values remain byte budgets. Do not introduce a second switch or change existing
numeric configuration meaning. Internally represent enabled separately from size.

Keep `--cache-disk DIR` and `--cache-disk-bytes N`. A suitable eventual deployment
uses `--cache-ram-bytes off`, the existing directory and 128 GiB disk budget.
Do not reduce context or slots as part of this work. Preserve model execution
rollback, mandatory live state and lightweight token/index metadata; these are
not retained prompt snapshots and are not disabled by this option.

For Flash Next use direct persistence by default whenever disk caching is enabled,
including when RAM snapshots are enabled. Other model adapters retain their
existing persistence behavior in this first increment; report the selected mode.
RAM-off is a general retention control, not a promise that every model has direct
persistence. No new mode switch hides the improved Flash Next path.

`--cache-disk-staging-bytes` continues to bound legacy materialized captures and
queues. It does not limit direct-streamed checkpoint size or streamed restores.
Log that distinction explicitly. Do not remove safety accounting for providers
that still materialize snapshots. Setting it to zero retains legacy auto sizing.

## Prefix matching and compatibility

Preserve longest eligible exact saved-prefix lookup, compatibility/input identity
checks, stable chat boundaries, learned shared-prefix checkpoints and checkpoint
spacing. Disk lookup must operate without any RAM snapshot. Retain existing live
state reuse where valid. This is not a rewrite to llama.cpp's overlap/similarity
selection and does not add arbitrary rewind inside a recurrent checkpoint.

Examples: saved A-B-C can restore A-B-C-D-E and prefill D-E; saved A-B-C-X alone
cannot supply recurrent state at A-B-C unless that boundary was also saved.
Test both positive reuse and this intentional miss. Do not compare all tiers to
choose a longer disk hit over an existing RAM hit in this work.

Keep payload version, field order, checksum, compatibility identity and published
filename/metadata format unchanged. Old files remain readable; new files must be
readable by the existing restore implementation.

## Direct writer and session lifetime

Add a model capability for computing serialized size and streaming from a frozen
live state. Flash Next reuses its snapshot region walker, emitting model framing,
tokens, logits, recurrent state, attention KV, indexer and MTP state in existing
wire order. Device regions pass through a reusable 1 MiB host transfer buffer;
file I/O uses the existing 1 MiB aligned buffer. Neither path allocates memory
proportional to the KV payload. Token/identity metadata can scale with token count
and must be measured separately. Padding and reserved bytes are deterministic.

Use the existing single persistence worker per store. Queue jobs reference a
leased, frozen state rather than a copied snapshot. At most one outstanding
direct job per session is allowed, bounding jobs by resident session count.
Admission performs duplicate/spacing/disk-capacity checks before freezing and
queuing. A temporary queue wait must not block the scheduler owner thread.

The scheduler does not prefill, decode, restore, rewind, invalidate or reassign
that session until its accepted job finishes. Keep its lease/model alive through
publication or failure. Other sessions remain runnable; the worker must use a
private nonblocking transfer stream and must not mutate shared execution scratch
or invalidate peer graph capture. A host-thread callback holding a raw pointer
without an owning lifetime is insufficient.

Hold the session through completion of the durable publication initially. Releasing
it immediately after the last device read is a later latency optimization only
if measurements justify the extra state transition. Expose queue wait, transfer,
write and durability timing so any latency cost is attributable.

Retain durable accepted-write semantics: client cancellation after acceptance does
not abandon the file or release its state prematurely. Graceful shutdown drains;
I/O/device errors release ownership once and emit visible failure. Failed cache
optimizations do not fabricate inference success/failure or corrupt subsequent
requests. Forced termination cannot guarantee durability, but leaves no published
partial file. Watchdogs distinguish legitimate persistence progress from device
loss and retain existing lost-device shutdown behavior.

RAM-enabled runs may separately retain their normal borrowed snapshots within
their budget. Disk serialization must not materialize those snapshots. RAM-off
runs must not create them merely to persist a checkpoint. Ordinary, stable,
learned-shared and cancellation checkpoint paths must obey the same rule.

## Disk store and reads

Refactor durable publication to accept an exact-size producer without requiring
a TextRunnerSnapshot. Preserve O_DIRECT, bounded checksum processing, overflow
and truncation checks, temporary files, fsync, rename, directory fsync, permissions
and corruption handling. Never evict a valid older checkpoint until replacement
publication succeeds, subject to the existing retention contract.

Remove staging-capacity rejection from startup indexing and streamed reads. Do
not replace it with an unchecked full-file allocation: validate file/header sizes,
overflow, identity and checksum with bounded I/O before restoration. Disk quota,
valid model context and format limits still apply. A checkpoint larger than the
legacy staging allowance is eligible; unlimited checkpoint sizes are not promised.

Legacy snapshot jobs retain existing reservations and waiting semantics. Direct
jobs are charged as fixed transfer memory plus metadata, separately from logical
queued file bytes; both kinds share the writer without deadlock or starvation.

## Verification and success criteria

- CPU tests prove RAM-off performs zero retained-snapshot captures, prefix lookup
  and restart reuse still work, large direct jobs bypass the legacy staging-size
  ceiling, and legacy jobs remain bounded. Include mixed job types and shutdown.
- Scheduler tests prove frozen lifetime, peer progress, cancellation, writer failure,
  delayed durability and exactly-once release. Prevent asynchronous GPU errors from
  becoming use-after-free or deadlocks.
- Flash Next GPU tests compare streamed bytes with the existing materialized oracle
  and independently validate restored logits/numerical behavior. Include MTP on/off,
  long context, image continuation, recurrent state and pending-speculation rejection.
- Instrument transfer allocation, largest chunk, RAM snapshot bytes and host/GPU
  memory. Larger payloads must not produce a checkpoint-sized host or GPU copy.
  Synthetic logical-byte tests supplement rather than replace actual memory evidence.
- On node-strix compare existing and candidate builds under identical weights,
  context, slots, cache history and pinned toolchain. Retain standard speed and
  numerical checks plus affected functional suites including long-context/cache.
  Shared serving changes require the four documented functional profiles/modes;
  missing-model checks remain incomplete. Apply per-request/phase 5% and 3 ms
  timing criteria and investigate regressions rather than averaging them away.
- Report persistence latency and frozen-session occupancy explicitly. A lower-memory
  path that serializes all sessions behind disk I/O does not satisfy this design.
- Only after qualification, update the generated deployment configuration, preserve
  existing disk files and use the authorized tgrid/image-build workflow. No
  llama-swap runtime-control feature is required for this stage.

## Deferred work

Runtime slot resizing, pool memory profiles, arbitrary checkpoint rewinding,
cross-tier best-match selection and direct-writer adapters for other models are
separate work. Revisit the resize design after measuring the new four-slot peak;
the need and safe policy may change materially.
