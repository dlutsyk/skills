# Thread pool operations

Read this file when precise classification of which APIs use the thread pool is needed,
or when diagnosing thread pool saturation.

## Complete classification

### Uses libuv thread pool

| Module        | Functions                                                                                                                                                                                                               |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fs`          | All async variants: readFile, writeFile, open, close, stat, fstat, lstat, rename, unlink, rmdir, mkdir, readdir, access, chmod, chown, link, symlink, readlink, realpath, truncate, copyFile, watch (on some platforms) |
| `crypto`      | pbkdf2, scrypt, randomBytes, randomFill, generateKeyPair, createHash (when used with stream), sign, verify (for some key types)                                                                                         |
| `zlib`        | deflate, inflate, gzip, gunzip, deflateRaw, inflateRaw, brotliCompress, brotliDecompress                                                                                                                                |
| `dns`         | `dns.lookup` only (uses getaddrinfo via thread pool)                                                                                                                                                                    |
| Native addons | Any addon using `uv_queue_work`                                                                                                                                                                                         |

### Uses OS async mechanisms (epoll/kqueue/IOCP) — no thread pool

| Module           | Functions                                                                         |
| ---------------- | --------------------------------------------------------------------------------- |
| `net`            | All TCP connections, createServer, createConnection                               |
| `http` / `https` | All HTTP/HTTPS operations                                                         |
| `dgram`          | All UDP operations                                                                |
| `tls`            | TLS handshake I/O (crypto computation may use thread pool for key operations)     |
| `dns`            | `dns.resolve*`, `dns.reverse` — uses c-ares, truly async                          |
| `timers`         | setTimeout, setInterval, setImmediate                                             |
| `child_process`  | spawn, fork (process creation is OS-level; communication via pipes uses OS async) |

### Special cases

**`crypto.randomBytes`**: Uses thread pool for blocking `/dev/urandom` reads. For
non-blocking usage in hot paths, consider pre-generating a pool of random bytes.

**`dns.lookup` vs `dns.resolve`**: Critical distinction. `dns.lookup` calls `getaddrinfo`
which is a blocking POSIX call — libuv runs it in the thread pool. `dns.resolve` uses
c-ares (asynchronous DNS library) and does not use the thread pool. For high-concurrency
services making many DNS lookups, `dns.lookup` can saturate the thread pool.

**`tls`**: The I/O layer (reading/writing encrypted bytes from the socket) is OS async.
Key operations during handshake (RSA decrypt, DH/ECDH computation) use OpenSSL which
libuv may run in a thread. In practice, TLS handshake is the main thread pool consumer
in HTTPS-heavy services.

**`fs.watch` / `fs.watchFile`**: Platform-dependent. On Linux (inotify) and macOS (FSEvents),
file watching is OS async. On some platforms libuv falls back to thread pool polling.

## Thread pool saturation diagnosis

Symptoms:

- `fs` or `crypto` operations have high latency under concurrent load
- CPU is not saturated but operations are slow
- Operations appear to serialize rather than parallelize

Measurement approach:

- Time N concurrent operations of the same type (e.g., N concurrent `crypto.pbkdf2`)
- If N=4 and UV_THREADPOOL_SIZE=4: total time ≈ time of one operation (parallel)
- If N=8 and UV_THREADPOOL_SIZE=4: total time ≈ 2× time of one operation (serialized in batches)
- Visible serialization confirms saturation

Fixes in priority order:

1. Increase `UV_THREADPOOL_SIZE` (set via env var before process start)
2. Offload to `worker_threads` for CPU-bound work (each worker has its own thread pool)
3. Use OS-async alternatives where available (dns.resolve vs dns.lookup)
4. Rate-limit concurrent thread pool operations at application level

## UV_THREADPOOL_SIZE sizing guide

| Workload type                         | Recommended size                                   |
| ------------------------------------- | -------------------------------------------------- |
| Default / mixed                       | 4 (default)                                        |
| Heavy crypto (bcrypt, pbkdf2, scrypt) | = number of CPU cores                              |
| Heavy fs I/O                          | 2–4× CPU cores (I/O waits, threads can overlap)    |
| Mixed crypto + fs                     | CPU cores + 2–4                                    |
| Maximum useful                        | 128 (beyond this, thread contention reduces gains) |
| Hard maximum                          | 1024 (libuv 1.30+)                                 |

Note: each thread allocates an 8 MB stack (libuv 1.45+). 64 threads = 512 MB stack memory.
Account for this in container memory limits.
