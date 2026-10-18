> 2026-10-18 review: superseded for Flash Next by the minimal component-cache plan. Historical requirements and results remain below; do not execute the legacy implementation tasks against main 0.17.1.

# Disk cache bounded I/O and persistence admission

The user accepts checkpoint spacing and no slot-wide exit flush. They requested the same bounded direct-I/O treatment as llama.cpp, and selected durable accepted checkpoints with waiting/retry for temporary queue pressure. Permanent disk or memory-capacity failures must remain visible.

1. Add a model-neutral checked snapshot byte source. Keep existing span-based model APIs, and add bounded restore overloads for production Qwen, Flash-Next, and DeepSeek adapters. Model data goes into existing live storage; only required tokens/logits/provider state remains allocated. Preserve payload formats and validation.
2. Use aligned direct I/O for bulk cache files. Stream checksum verification and startup indexing without full-file buffers. Keep atomic file publication, fsync of payload and parent directory, safe file checks, and existing cache compatibility.
3. Apply spacing before accepting queued work. Wait for transient queue/capture pressure without exceeding staging limits or blocking cache metadata locks. Notify waiters when reservations disappear. Accepted jobs bypass later spacing rejection. Record failures and make explicit draining report them. Keep terminal capacity errors distinct from pressure.
4. Add focused direct-I/O, source bounds, backpressure/cancellation, durable drain/failure, and existing format regression tests. Run CPU disk-store and text-runner checks, repository formatting, and review. This editing host has no gfx1151; model/GPU functional validation must be reported as unrun unless a remote workflow is supplied.

Separate model adapter work may run in parallel. Keep shared inference-backend edits as integration patches to avoid concurrent rewrites. Do not modify llama.cpp or unrelated cache-redesign work.
