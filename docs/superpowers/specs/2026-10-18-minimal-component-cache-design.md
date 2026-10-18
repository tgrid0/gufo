# Minimal component-cache adaptation design

Status: design and implementation plan approved by the user on 2026-10-18.

## Outcome

Keep the custom fork small on upstream a8ed7af (0.17.1). Preserve explicit
`--cache-ram-bytes off` as a retention policy, numeric zero as automatic sizing,
and positive values as the existing byte budget. Keep live-prefix reuse and
restart-safe disk reuse. Defer startup-memory admission and runtime resizing.

## Review findings

- zgfs 1eb8d85 already contains upstream a8ed7af plus four custom commits.
  Restoring startup-memory work left ten unmerged files and 93 changed/untracked
  paths. Original files, index, stash and commits are preserved separately.
- Flash Next now uses ComponentTextCache. The rebased runner constructor ignores
  TextRunnerRamCacheOptions::enabled in that branch and returns before creating
  the legacy disk store. The old direct serializer is not the active serving path.
- Upstream already streams through a 1 MiB pinned staging buffer, shares completed
  disk chunks, and fsyncs payloads/directories/manifests. Retain that implementation.
- Component resource accounting combines metadata, private-state/tail captures,
  and backing under ram_bytes. Zeroing this ledger cannot implement retention off.
- Upstream disk payload I/O is buffered; old O_DIRECT changes do not reach it.
  This is a distinct remaining difference, not proof of a performance regression.
- Startup-memory work assumes the former session owner and references JSON memory
  APIs absent from the current tree. Conflict resolution alone cannot repair it.
  Upstream also changed MTP vocabulary/ngram paths, attention kernels, and startup
  preparation. Old memory estimates and benchmark results require revalidation.

## Implementation boundary

Add a retention-enabled policy to the existing runner and component-cache plumbing.
When disabled, refuse history-only RAM checkpoints and route persistence captures
through the existing CaptureDurable path, even when they fit the RAM budget.
Do not publish retained RAM checkpoints. Preserve live slots, disk catalog entries,
private captures and transfer ownership through upstream mechanisms.

Keep the existing bounded automatic resource ceiling for transient cache work;
report retention disabled separately from that ceiling. Off does not mean zero
host memory: private recurrent state, tails, metadata, staging and committed free
backing remain possible. Do not claim the old two-buffer-only memory contract.
No new memory-budget switch or allocator redesign belongs in this patch.

Use current upstream disk formats, incremental publication, admission and queue
semantics. Do not restore the legacy whole-state serializers, frozen-slot scheduler,
other-model restore changes, or accepted-write policy merely to retain old commits.
Do not migrate/delete historical disk cache files as part of this task.

Assess buffered I/O with matched affected measurements. Add a narrow direct-I/O
change only if evidence demonstrates a remaining user-relevant memory/I/O problem;
otherwise document it as a deliberate difference. Preserve format and filesystem
compatibility if such a change becomes necessary.

## Validation

CPU tests must prove off/no-disk, off/disk, restart reuse, no retained RAM capture,
live-prefix reuse, cancellation, peer progress, pressure and shutdown behavior.
Default/zero/positive policies must retain upstream behavior. Actual Flash Next
AR/MTP checks must cover cache and long-context continuation, numerical quality,
standard speed and affected per-request timings under the repository rules.
Old performance evidence is historical, never a pass for this branch. Deployment
and branch replacement wait for qualification; the original checkout stays intact.
