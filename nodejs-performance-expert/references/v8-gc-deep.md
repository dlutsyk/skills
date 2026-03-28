# V8 GC deep dive

Read this file when the user asks about GC internals, pause tuning, or Orinoco specifics.

## Orinoco — parallel / incremental / concurrent GC

V8's GC project (codename Orinoco) transformed the original stop-the-world collector into a
mostly concurrent system. Three orthogonal techniques are combined:

**Parallel**: main thread + helper threads all pause JS and split GC work. Reduces pause
duration proportionally to thread count. Used in Minor GC (Scavenger).

**Incremental**: GC work split into small slices interleaved with JS execution. Increases total
GC time slightly but eliminates long individual pauses. Used for Major GC marking phase.

**Concurrent**: GC work runs on helper threads while JS continues on main thread. Requires
write barriers to track heap mutations during concurrent marking. Used for Major GC marking
and sweeping. Only a short stop-the-world finalization step remains.

## Write barriers

During concurrent marking, when JS code writes a pointer to a heap object, a write barrier
fires to notify the GC. This ensures the marker does not miss newly created references.
Write barriers have a small but non-zero cost — visible in extremely allocation-heavy code.

## Scavenger internals

Semi-space design: New Space is split into From-Space (allocation target) and To-Space (empty).

Scavenge steps:

1. Scan roots (stack, globals, old-to-new references tracked via remembered set)
2. Copy live objects from From-Space to To-Space (evacuation)
3. Update all pointers to evacuated objects (forwarding pointers)
4. Swap From/To spaces

Objects surviving two scavenges are promoted to Old Space. The threshold is tracked per-object.

Parallel scavenging: multiple helper threads evacuate objects concurrently using thread-local
allocation buffers (TLABs) to avoid synchronization on every allocation.

## Mark-Compact internals

**Marking**: starts from root set (execution stack, globals, compiled code). Follows all
pointers recursively. Uses tri-color marking (white = unvisited, grey = queued, black = done).
Concurrent marking runs on helper threads; main thread handles finalization.

**Sweeping**: iterates heap pages, adds dead object memory to per-size free-lists. Done in
parallel on helper threads after marking.

**Compaction**: selected highly-fragmented pages are compacted by copying all live objects
to a fresh page. Not all pages are compacted — only those where fragmentation waste exceeds
a heuristic threshold. Reduces fragmentation in Old Space.

## Large Object Space behavior

Objects allocated in Large Object Space are never moved. This means:

- No compaction pressure on large objects.
- Fragmentation accumulates permanently if large objects are repeatedly allocated and freed.
- Pointer updates are simpler (no forwarding needed).
- Large objects do not participate in generational GC — they are only collected in Major GC.

Threshold for Large Object Space: approximately 512 KB (may vary by V8 version).

## GC notification API

`v8.getHeapStatistics()` — returns detailed breakdown: total heap size, used heap size,
heap size limit, malloced memory, external memory, etc. Useful for monitoring.

`v8.getHeapSpaceStatistics()` — per-space breakdown (new space, old space, code space,
large object space, etc.).

`--trace-gc` flag — logs every GC event to stderr with type (Scavenge / Mark-Compact),
before/after heap size, and pause duration. Essential for GC pause diagnosis.
