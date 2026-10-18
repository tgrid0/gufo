# Flash Next startup memory admission — cancelled design

Cancelled by user decision on 2026-10-18. This is not an active design or a
request for future implementation.

Use the [measured configuration profile](../reports/2026-10-18-flash-next-memory-slots.md)
instead of the proposed complete allocation bounds, startup warmup, sealed
arenas, readiness admission contract and external-reserve option.

No new startup-precalculation guarantee or reserve option is being introduced.
The existing runtime and its existing capacity checks remain in use.
The former design is retained in Git history only.
