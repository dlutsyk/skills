# Event loop phases — deep dive

Read this file when the user asks about event loop ordering edge cases, nextTick vs
setImmediate vs Promise behavior, or libuv phase internals.

## Full phase sequence with microtask drain points

```
[start of iteration]
  → nextTick queue drain
  → Promise microtask queue drain

[timers phase]
  → run all expired setTimeout / setInterval callbacks
  → after each callback: nextTick drain → Promise drain

[pending callbacks phase]
  → run I/O callbacks deferred from previous iteration
  → after each callback: nextTick drain → Promise drain

[idle, prepare phase]
  → internal libuv use only

[poll phase]
  → if check queue non-empty OR no timers: do not block
  → else: block via epoll_wait/kqueue/IOCP until next timer or I/O event
  → drain all ready I/O callbacks
  → after each callback: nextTick drain → Promise drain

[check phase]
  → run all setImmediate callbacks
  → after each callback: nextTick drain → Promise drain

[close callbacks phase]
  → run close event callbacks (socket close, stream close)
  → after each callback: nextTick drain → Promise drain
```

## Key ordering guarantees

**nextTick vs Promise**: `process.nextTick` drains before Promise microtasks. Both drain
fully before the event loop advances to the next phase.

**setImmediate vs setTimeout(fn, 0) — outside I/O callback**: order is non-deterministic.
Depends on OS timer resolution and event loop startup cost. Do not rely on a specific order.

**setImmediate vs setTimeout(fn, 0) — inside I/O callback**: `setImmediate` always fires
first. The current I/O callback is in poll phase; after poll comes check (setImmediate),
then the next iteration's timers phase.

**nextTick starvation**: `process.nextTick` queue is drained completely before continuing.
Recursive `process.nextTick` calls (a nextTick callback schedules another nextTick)
will spin forever and starve all I/O, timers, and the rest of the event loop.
Same applies to recursive resolved Promises.

## ESM vs CJS event loop start

In CommonJS: the module is executed synchronously, then the event loop starts.
In ESM: the module is loaded in an async context. The module's top-level `await` and
Promise resolution happen before user microtasks in a subtly different order.
Specifically: in ESM, Promise microtasks drain before `process.nextTick` at the very
start. This is the one exception to the "nextTick before Promise" rule.

## poll phase blocking behavior

libuv calculates the poll timeout before blocking:

- 0 if `UV_RUN_NOWAIT` flag
- 0 if `uv_stop()` called
- 0 if no active handles or requests
- 0 if idle handles active
- 0 if handles pending close
- 0 if setImmediate queue non-empty
- Otherwise: min(nearest timer deadline, ∞)

This means if there are no timers and no setImmediate, the process blocks indefinitely
in poll waiting for I/O. This is correct behavior for a server.

## libuv handles vs requests

**Handle**: long-lived object representing an ongoing operation (TCP server, timer, UDP
socket, file watch). Active handles keep the event loop alive.

**Request**: short-lived, represents a single operation on a handle (write request, DNS
query, thread pool work item). Outstanding requests also keep the loop alive.

The event loop exits when there are no active ref'd handles and no pending requests.
`handle.unref()` marks a handle as non-keeping — the loop can exit even if this handle
is still active (useful for keepalive timers that should not prevent process exit).

## Async context tracking

`AsyncLocalStorage` (Node.js 12.17+) propagates context across async boundaries using
V8's async hooks internally. Each async operation (Promise, callback, timer) gets an
async ID. Context flows through the async call graph.

Cost: async hooks add overhead to every Promise creation and resolution. In
extremely high-throughput services (>100k req/s), measure the impact before enabling.
