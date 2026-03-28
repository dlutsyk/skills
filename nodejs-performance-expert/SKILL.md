---
name: nodejs-performance-expert
description: >
  Expert-level Node.js performance skill covering V8 internals (heap, stack, GC, JIT) and libuv
  architecture (event loop phases, thread pool, async I/O). Use this skill whenever the user asks
  to write, review, optimize, or debug Node.js code — including HTTP servers, async pipelines,
  file/crypto operations, memory-heavy services, or any code where latency, throughput, or memory
  usage matters. Also trigger for questions about event loop behavior, GC pauses, thread pool
  saturation, memory leaks, or "why is my Node.js slow". Activate even if the user doesn't
  explicitly mention performance — any non-trivial Node.js code task benefits from this skill.
---

# Node.js performance expert

You are a senior Node.js engineer with deep knowledge of V8 internals and libuv architecture.
When writing or reviewing Node.js code, apply this knowledge automatically — do not explain it
unless asked. Surface only actionable findings: what is wrong, why it matters at the runtime
level, and what to do instead.

---

## How to use this skill

| Task                   | What to do                                                                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Write new code**     | Apply all rules below proactively. Never produce anti-patterns.                                                                            |
| **Code review**        | Scan against all rule categories. Report only real violations, not style opinions. Group by severity: Critical → Warning → Suggestion.     |
| **Diagnose a problem** | Map symptom to root cause using the diagnostics section. Recommend the minimal reproduction and measurement step before prescribing a fix. |

---

## 1. V8 memory model

### Authoritative rules

**Stack vs Heap boundary**

- Primitives (number, boolean, string, null, undefined, Symbol, BigInt) live on the stack.
- All objects, arrays, functions, closures, and Buffers live on the heap.
- Every closure captures a reference to its enclosing scope's variable object — not a value copy. A closure that captures a large object keeps that object alive for the closure's lifetime.

**Heap regions — know what lives where**

- New Space (Young generation, ~1–8 MB): short-lived objects. Scavenged frequently and cheaply.
- Old Space: objects that survived two Minor GCs. Collected infrequently via Mark-Compact. Old Space pressure = visible GC pauses.
- Large Object Space: objects > ~512 KB. Never moved by GC. Fragmentation accumulates here.
- Code Space: JIT-compiled machine code. Size grows with number of optimized functions.

**Hidden classes (Shapes / Maps)**

- V8 assigns a hidden class to every object. Objects with identical property insertion order share a hidden class → V8 can optimize property access via inline caches.
- Changing an object's structure after creation (adding/deleting properties, changing property order) forces a hidden class transition → deoptimization of all inline caches on that shape.
- Rule: define all properties in the constructor. Do not conditionally add properties later.

**Inline caches (ICs)**

- Monomorphic IC (one hidden class seen) = fastest path. Polymorphic (2–4 classes) = slower. Megamorphic (5+) = no caching.
- Functions that receive objects of varying shapes become megamorphic and cannot be optimized.

### GC behavior rules

**Generational hypothesis**

- Most objects die young. Design data flows so intermediary objects (parsed payloads, mapped arrays, temporary aggregates) are created and discarded within a single request scope — they stay in New Space and are collected cheaply.

**Minor GC (Scavenger)**

- Runs on Young generation only. Uses semi-space copy: live objects evacuated from From-Space to To-Space.
- Objects surviving two scavenges are promoted to Old Space.
- Scavenging is parallel but still causes a short pause. High allocation rate in hot paths = frequent scavenges.

**Major GC (Mark-Compact / Orinoco)**

- Three phases: Marking (concurrent, background threads), Sweeping (parallel), Compaction (selective).
- Concurrent marking means JS continues running during most of the marking phase — but there is still a short stop-the-world finalization pause.
- Anything that keeps Old Space objects alive longer than needed delays Major GC triggering and increases pause duration when it finally runs.

**GC pressure sources — always check for these in review**

- Global or module-level caches that grow without eviction (Maps, Sets, plain objects used as registries).
- EventEmitter listeners added without `removeListener` / `once` in long-lived services.
- Closures in request handlers that capture large request/response objects and are stored somewhere (timers, queues, caches).
- `setInterval` callbacks that reference large data structures.
- Accumulation in arrays that are only appended to (logs, metrics buffers without flush).

**Heap size flags**

- `--max-old-space-size=N` (MB): hard cap on Old Space. Default varies by platform (~1.5 GB on 64-bit). Set explicitly in production containers to slightly below available RAM.
- `--max-semi-space-size=N` (MB): size of each New Space semi-space. Increasing reduces scavenge frequency at the cost of higher peak memory.

---

## 2. V8 JIT compilation

**Compilation pipeline**
Ignition (bytecode interpreter) → Sparkplug (fast baseline compiler) → Maglev (mid-tier optimizer) → TurboFan (full optimizing compiler).

Hot functions are promoted through tiers based on invocation count and type stability. A function deoptimizes back to Ignition when a runtime type assumption is violated.

**JIT optimization rules**

- **Type stability**: a function called with consistent argument types is compiled to monomorphic machine code. Mixing types forces deoptimization.
- **Avoid arguments object**: use rest parameters (`...args`) instead. `arguments` prevents several TurboFan optimizations.
- **Avoid `delete` on object properties**: it forces a hidden class transition to a "slow" dictionary mode object.
- **Avoid `eval`, `with`, and dynamic `import()` in hot paths**: they prevent static analysis and disable many optimizations.
- **Try/catch in hot functions**: TurboFan historically could not optimize across try/catch boundaries. Maglev improved this, but isolating try/catch to wrapper functions is still safer in extreme hot paths.
- **Array type stability**: V8 tracks array element kinds (SMI_ELEMENTS, DOUBLE_ELEMENTS, OBJECT_ELEMENTS). Pushing mixed types into an array degrades to OBJECT_ELEMENTS and disables numeric optimizations.

---

## 3. libuv event loop

### Phase order (per iteration)

```
timers → pending callbacks → idle/prepare → poll → check → close callbacks
```

Between every phase transition, Node.js drains:

1. `process.nextTick()` queue (full drain)
2. Promise microtask queue (full drain)

This means a recursive `process.nextTick()` starves all I/O. A recursive Promise chain does the same. Never recurse in microtasks without a bailout condition.

### Phase rules

**timers**: `setTimeout` and `setInterval` callbacks. Timer resolution is ~1 ms on Linux, coarser on macOS/Windows. `setTimeout(fn, 0)` is not guaranteed to fire before `setImmediate` outside an I/O callback.

**poll**: the blocking phase. libuv calls `epoll_wait` (Linux), `kqueue` (macOS), or IOCP (Windows). Blocks until an I/O event arrives or the nearest timer deadline. All network I/O callbacks fire here.

**check**: `setImmediate` callbacks. Inside an I/O callback, `setImmediate` always fires before `setTimeout(fn, 0)` because check phase precedes the next timers phase.

**close callbacks**: `socket.on('close')`, `stream.on('close')`, etc.

### What blocks the event loop

The event loop is single-threaded. Any synchronous work on the main thread blocks all I/O, timers, and incoming connections for its duration.

Blocking operations to detect in review:

- `fs.*Sync` calls in request handlers or middleware
- `crypto.*Sync` (pbkdf2Sync, scryptSync, randomFillSync)
- Large `JSON.parse` / `JSON.stringify` on the main thread
- Regular expressions with catastrophic backtracking on user-supplied input (ReDoS)
- Synchronous DNS: `dns.lookup` is NOT async — it goes to thread pool, but `require('dns').lookupSync` does not exist; however native `getaddrinfo` via C++ bindings can block
- Heavy computation in route handlers without chunking (sorting large datasets, matrix operations, image processing in JS)
- `child_process.execSync` / `spawnSync`

**Rule**: any synchronous operation taking more than ~1–2 ms in a hot path is a candidate for offloading.

---

## 4. libuv thread pool

### What uses the thread pool

| Uses thread pool                                       | Uses OS async (no thread pool)            |
| ------------------------------------------------------ | ----------------------------------------- |
| `fs.*` (all async file operations)                     | TCP / UDP networking                      |
| `crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes` | `http` / `https` / `net`                  |
| `dns.lookup`                                           | `dns.resolve*` (uses c-ares, truly async) |
| `zlib` compression                                     | Timers                                    |
| `uv_queue_work` (native addons)                        | Child processes (separate OS process)     |

### Thread pool rules

- Default size: 4 threads. Maximum: 1024.
- Set via `UV_THREADPOOL_SIZE` environment variable **before process start**, not via `process.env` at runtime (too late after first async call initializes the pool).
- Thread pool is shared across all `require`d native modules and the Node.js core.
- **Saturation**: if concurrent thread pool operations > pool size, excess operations queue. Under high crypto or fs load, this queuing adds latency invisible in application code.
- **Sizing heuristic**: `UV_THREADPOOL_SIZE` = number of CPU cores for CPU-bound work (crypto). For fs-heavy I/O workloads, experiment with 2–4× core count, but note each thread consumes memory (~8 MB stack since libuv 1.45).
- `dns.resolve*` (c-ares) does not use thread pool — prefer it over `dns.lookup` for high-concurrency DNS resolution.

---

## 5. Code review checklist

When reviewing Node.js code, check every category. Skip categories not applicable to the code shown.

### Event loop safety

- [ ] No `*Sync` fs/crypto calls outside of startup/CLI scripts
- [ ] No unbounded synchronous loops over user-controlled data
- [ ] No `process.nextTick` recursion without termination condition
- [ ] No ReDoS-vulnerable regex on untrusted input
- [ ] CPU-intensive work offloaded to `worker_threads` or child process

### Memory / GC

- [ ] All EventEmitter listeners removed when the emitter's lifecycle ends
- [ ] No module-level caches (Map, plain object) that grow without eviction
- [ ] Closures in long-lived contexts do not capture large transient objects
- [ ] `setInterval` handlers do not hold references to large data
- [ ] Streams are destroyed / closed on error paths, not just on success
- [ ] No `Buffer.allocUnsafe` without immediate fill when data is user-visible

### Thread pool awareness

- [ ] Concurrent `crypto` or `fs` operations bounded to ≤ UV_THREADPOOL_SIZE
- [ ] `dns.resolve*` preferred over `dns.lookup` in hot paths
- [ ] `UV_THREADPOOL_SIZE` set appropriately for workload type

### V8 JIT friendliness

- [ ] Object shapes consistent throughout their lifetime (no post-construction property addition/deletion)
- [ ] Hot functions receive consistent argument types
- [ ] Array element types consistent (no mixing numbers and objects in the same array)
- [ ] No `delete` on hot-path objects

### Async correctness

- [ ] All Promise rejections handled (`.catch` or `try/catch` in async function)
- [ ] No floating promises (unawaited async calls where errors are silently lost)
- [ ] `async` functions not called without `await` in loops where sequential execution is required
- [ ] `Promise.all` used for independent parallel operations; not sequential `await` chains

---

## 6. Diagnostics — symptom → cause → action

### High memory / OOM

1. `process.memoryUsage()` — check `heapUsed` trend over time. Growing `heapUsed` = leak.
2. `--expose-gc` + `global.gc()` before measurement to isolate retained objects from GC lag.
3. Heap snapshot via `node --inspect` → Chrome DevTools Memory tab → "Take heap snapshot". Compare two snapshots (before/after load) — objects growing in count are the leak source.
4. Common causes: unbounded cache, EventEmitter accumulation, closure over request scope stored in module-level structure.

### Event loop lag / high latency

1. Measure with `perf_hooks.monitorEventLoopDelay()` — baseline < 5 ms, problem > 20 ms under load.
2. CPU profile: `node --prof` → `node --prof-process` or `clinic flame`. Identify top synchronous frames.
3. Common causes: `*Sync` in hot path, large JSON, regex, thread pool saturation showing as I/O latency.

### Thread pool saturation

1. Symptom: `fs` or `crypto` operations slow under concurrent load even with spare CPU.
2. Verify: time multiple concurrent `crypto.pbkdf2` calls and check if they serialize.
3. Fix: increase `UV_THREADPOOL_SIZE` or switch to `worker_threads` for CPU-bound crypto.

### GC pauses causing latency spikes

1. Symptom: p99 latency spikes periodically (every few seconds) with no CPU spike.
2. Confirm: `node --trace-gc` — look for Major GC (Mark-Compact) pause duration.
3. Fix: reduce Old Space promotion rate by eliminating long-lived object accumulation. Tune `--max-semi-space-size` to reduce scavenge frequency.

---

## 7. Production configuration baseline

Flags and env vars worth knowing for production Node.js services:

- `UV_THREADPOOL_SIZE=N` — set based on workload (crypto → cores, fs-heavy → 2–4× cores)
- `--max-old-space-size=N` — set to ~75–80% of container memory limit
- `--max-semi-space-size=N` — default 8 MB; increase for allocation-heavy services
- `NODE_ENV=production` — enables production optimizations in most frameworks (Express, Fastify, etc.)
- `--enable-source-maps` — for readable stack traces from compiled/transpiled code
- `--trace-warnings` — surfaces unhandled promise rejections and deprecation warnings with stack traces

---

## References

For extended detail on specific topics, see:

- `references/v8-gc-deep.md` — GC algorithm internals, Orinoco phases, write barriers
- `references/event-loop-phases.md` — Per-phase libuv source behavior and edge cases
- `references/thread-pool-ops.md` — Full table of which Node.js APIs use thread pool vs OS async
