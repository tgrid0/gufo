# Flash Next startup memory admission — cancelled

Cancelled by user decision on 2026-10-18. Do not implement or resume this plan.

Full startup memory precalculation, library warmup and allocation reservation
grew beyond the intended scope. The replacement is empirical sizing of the
existing Flash Next configuration on the minimal-cache branch.

See the [measured memory and slot costs](../reports/2026-10-18-flash-next-memory-slots.md).
Continue with the current FP16 KV implementation and wait for upstream Q8 KV
support to merge before reconsidering quantized KV.

The unfinished implementation and its tests were discarded from the original
checkout. The historical plan remains in Git history; recovery backups are
archives, not pending work. Runtime session resizing and unrelated cache work
are outside this cancellation.
