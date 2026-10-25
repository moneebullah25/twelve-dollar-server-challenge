## What this entry does differently from the earlier C entries, with same-machine numbers

The two closest prior submissions are `c-iouring-ssmurfgg04-gif` (#112, C + io_uring, batch 2: 150,000 users, 50 µs/req) and `c-epoll-TheFirstIstari` (#64, C + epoll, batch 2: 107,000 users, 69 µs/req). This entry started from the same ideas (direct descriptors, provided buffers, the three-range-scan feed, group commit, a memory guard) and changes the parts below. Everything is cited to the code as it is in the repo today.

### Design differences

| Axis | this entry (`c-iouring-moneebullah25`) | `c-iouring-ssmurfgg04-gif` | `c-epoll-TheFirstIstari` |
|---|---|---|---|
| Threads | 1 loop thread + 1 nice-10 checkpoint thread (`src/db.c:220`); 7 rings × 65,535 direct-descriptor slots, ring 0 with `DEFER_TASKRUN` (`src/net.c:1020-1099`) | 8 worker threads, each with its own ring and file table, serialized on one DB mutex (`src/server.c:43, 517-534, 988-1000`), plus a checkpointer thread | 1 thread, `accept4` loop + edge-triggered `recv` (`src/server.c:388-492`) |
| Batching | after a busy iteration wait for 32 CQEs or 250 µs; ≤ 256 requests per iteration (`src/net.c:1168, 1268-1276`) | wait for 128 CQEs or 4 ms (`src/server.c:45-50, 591-599`) | whatever `epoll_wait` returns, 100 ms timeout (`src/server.c:505`) |
| Write path | all writes of an iteration share one `BEGIN IMMEDIATE … COMMIT`; reads share one deferred txn; every send is submitted after `COMMIT` (`src/app.c:223-224, 426`; `src/db.c:365-371`) | one `BEGIN IMMEDIATE … COMMIT` per write request under the mutex, retried up to 20× (`src/db.c:222-251`) | group commit per `epoll_wait` batch, sends after `COMMIT` (`src/api.c:195, 249`; `src/server.c:527-560`) |
| Checkpoints | PASSIVE on the nice-10 thread at ≥ 4,000 un-backfilled frames; at 8,192 frames the loop holds writes only (reads continue) until the WAL restarts, 500 ms cap (`src/db.c:245-249, 295-319`) | `TRUNCATE` every 2 s on the checkpointer thread while holding the writer mutex (`src/db.c:161-175`) | PASSIVE on the loop thread from `db_commit` (`src/db.c:116-126`) |
| SQLite | `unix-excl` VFS, `mmap_size=0` (compiled out), 32 MiB cache, `busy_timeout` 50 ms (`src/db.c:128-149`, `Makefile:40`) | `mmap_size=0`, 64 MiB cache on the writer + 16 MiB per reader, `busy_timeout` 5 s (`src/db.c:93-135`) | `locking_mode=EXCLUSIVE`, `mmap_size=1 GiB`, 8 MiB cache, `busy_timeout` 5 s (`src/db.c:49-59`) |
| Feed query | 3 range scans when the 20 newest ids span ≤ 256, reference SQL otherwise (`src/db.c:19-52`) | same strategy (`src/db.c:490-556`) | reference query with 20 correlated `count(*)` (`src/db.c:63-67`) |
| Responses | written in place into fixed 72 KiB chunks from a prefaulted pool of 1,024, one `send` SQE each, no allocation on the request path (`src/net.c:378`) | growable per-connection buffer, chunk pool then `malloc` (`src/util.c:63-75`) | `malloc` + `snprintf` per response (`src/server.c:357-372`) |
| Connection record | 48 B, idle connections own no buffer (`src/net.c:73-90`) | 72 B by `sizeof` (`src/server.c:72-83`) | 104 B + per-connection `in`/`out` buffers (`src/conn.h:23-40`) |
| JWT | HMAC pads precomputed once, constant-time compare, HS256 only, `exp` required (`src/crypto.c:107-126`, `src/jwt.c:49-93`) | pads precomputed once (`src/jwt.c:108-141`) | pads rebuilt per call (`src/jwt.c:71-80`) |
| Build | `-O3 -march=native -flto`; PGO implemented but not shipped (no gain measured) | PGO of SQLite and server | PGO + LTO, no `-march=native` |

Memory guard, half-close handling, slow-client timeouts and TCP keep-alive are present in all three in some form; the differences are in thresholds, not approach.

### Same-machine, same-generator measurements

Hardware: 2-core AMD Athlon Silver 3050U (Zen1), 14 GB, Ubuntu kernel 6.8.0-147, loopback, generator on the same two cores. Not the benchmark box; see caveats. All three servers passed `test/test.sh` 42/42 first and had zero errors in every run. Generator: this entry's `tools/mload` (closed loop, the `bench/load.js` user model). Server CPU = `utime+stime` from `/proc/<pid>/stat` over the run ÷ completed requests, measured the same way for all three; RSS = `VmHWM`. Two runs each, shown as run 1 / run 2. 2026-10-24.

**A. 64 connections, no think time, 20 s (batched regime)**

| Server | CPU µs/req | p50 ms | p99 ms | VmHWM MB |
|---|---|---|---|---|
| this entry | **28.2 / 26.2** | 2.0 / 1.6 | 22.6 / 3.5 | 134 / 137 |
| c-iouring-ssmurfgg04-gif (PGO) | 39.3 / 37.8 | 3.2 / 2.7 | 17.4 / 6.9 | 12 / 12 |
| c-epoll-TheFirstIstari (PGO) | 35.1 / 35.8 | 2.3 / 2.3 | 5.5 / 5.4 | 9 / 9 |

**B. 10,000 users, real think times, 60 s (~1,060 req/s; one request per wake-up)**

| Server | CPU µs/req | p50 ms | p95 ms | p99 ms | VmHWM MB |
|---|---|---|---|---|---|
| this entry | **61.2 / 60.8** | 0.1 / 0.1 | 0.5 / 0.4 | 1.3 / 1.2 | 141 / 141 |
| c-iouring-ssmurfgg04-gif | 67.5 / 67.7 | 3.0 / 3.1 | 8.8 / 29.4 | 396 / 1,202 | 23 / 23 |
| c-epoll-TheFirstIstari | 69.0 / 69.3 | 0.1 / 0.1 | 3.7 / 1.9 | 2,345 / 81.5 | 25 / 25 |

**C. 50,000 users, think ×2, 60 s (~2,590 req/s; connection holding)**

| Server | CPU µs/req | p50 ms | p95 ms | p99 ms | VmHWM MB |
|---|---|---|---|---|---|
| this entry | 57.2 / 56.9 | 0.2 / 0.2 | 0.8 / 0.7 | 2.7 / 1.3 | 152 / 152 |
| c-iouring-ssmurfgg04-gif | **56.2** / 57.3 | 5.0 / 4.7 | 533 / 12.2 | 712 / 59.6 | 36 / 36 |
| c-epoll-TheFirstIstari | 60.6 / 63.6 | 0.3 / 0.2 | 2,016 / 123 | 2,823 / 1,045 | 38 / 38 |

CPU per request moved by ≤ 2-3 µs between the two runs of any server, so differences of 5 µs or more are above the noise and 1-2 µs are not.

### Reading

- **Batched regime (A):** 20-30 % less server CPU per request than both PGO builds. Both this entry and ssmurfgg04 use the range-scan feed, so the feed is not what separates them; the visible differences are one transaction per batch instead of per write, the batched read transaction, fixed send chunks instead of per-response allocation, and one thread instead of eight serialized on a mutex. Against TheFirstIstari the feed strategy and `malloc`/`snprintf` per response are the visible differences.
- **One request per wake-up (B):** the gap shrinks to ~10 %, since every server pays a kernel round trip per request.
- **50k held connections (C):** CPU equal to ssmurfgg04 within noise, 4-7 µs under TheFirstIstari.
- **Tails (B, C):** the clearest separation. This entry stayed at p99 1.2-2.7 ms in all four runs; both prior entries showed 60 ms-2.8 s p99s that vary 3-30× between runs. The code points to checkpoints: a `TRUNCATE` checkpoint every 2 s under the writer mutex in one, a PASSIVE checkpoint on the loop thread in the other (ssmurfgg04's own `dbstats` logged single handlers of 105-1,462 ms). This is consistent with the code and logs but was not confirmed by profiling their processes, and this laptop's disk is slower than the benchmark box's, so the effect may be smaller there.
- **Memory:** this entry is the heaviest process (134-152 MB vs 9-38 MB) by design: the 72 MiB send pool, receive pools, connection table and SQLite cache are prefaulted at startup. Growth per connection is similar across the three.

### What this does not show

No user count, no p95 at 100k+ users over a real network, nothing about the 2 GB wall. Loopback puts part of the client's delivery cost into the server's `stime`, and the generator shared the cores, so absolute µs/req will differ on the benchmark box; the direction of the differences is what should transfer. The Rust entry (#82, 30 µs/req, 264k) was not built here (no cargo) and remains the number to beat. The full method, commands and raw logs are reproducible from `tools/bench_local.sh` and `docs/RESULTS.md` in the entry.
