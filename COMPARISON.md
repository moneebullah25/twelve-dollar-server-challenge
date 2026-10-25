# Same-machine comparison: c-iouring-moneebullah25 vs. the two closest C entries

Date: 2026-10-24. Git-untracked working note for the PR #156 discussion.

Compared: our entry `submissions/c-iouring-moneebullah25/` (branch
`c-iouring-moneebullah25`, commit `1b60132`) against
`submissions/c-iouring-ssmurfgg04-gif/` (C + io_uring, PR #112) and
`submissions/c-epoll-TheFirstIstari/` (C + epoll, PR #64). The Rust entry
(`rust-epoll-scooter1337`, PR #82) was not built (no cargo here); only the
organizer's published numbers are quoted for it.

Every number in section 3 is **measured here** by the method in 3.1. Every
number in section 2 is **the organizer's** (`benchmarks/batch-2.md`). The prior
entries were built with their own `build.sh` and were not modified.

## 1. Technique diff

Only what was verified in the code is claimed. Where a README disagrees with
the code, the code is cited and the disagreement is noted. Paths are relative
to `submissions/<entry>/`.

| Axis | ours (c-iouring-moneebullah25) | c-iouring-ssmurfgg04-gif | c-epoll-TheFirstIstari |
|---|---|---|---|
| Threads / rings | 1 loop thread + 1 nice-10 checkpoint thread (`src/db.c:220`). `ceil(max_conns/65535)` rings, 7 here (`src/net.c:1092-1099`; log: "7 ring(s) x 65535 slots"). Ring 0 `SINGLE_ISSUER\|DEFER_TASKRUN\|COOP_TASKRUN\|SUBMIT_ALL`, secondary rings without `DEFER_TASKRUN`, watched from ring 0 (`src/net.c:1020-1028`) | 8 worker threads (`src/server.c:43`, `988-1000`), each with its own ring (`SINGLE_ISSUER\|DEFER_TASKRUN\|COOP_TASKRUN\|SUBMIT_ALL\|CQSIZE`, `src/server.c:517-518`) and its own 65,535-slot file table (`534`, `969`); plus a checkpointer thread (`src/db.c:156-158`). 10 threads at run time | 1 thread, no others (`src/server.c:1-3`, `497-561`) |
| Accept / receive | Multishot accept into direct descriptors (`src/net.c:278`), one-shot accepts near a full table; multishot `RECV` with `IOSQE_BUFFER_SELECT` from 2,048 x 2 KiB classic provided buffers per ring (`src/net.c:269`, `1060`; config `src/config.c:17-18`). ENOBUFS re-arms deferred to end of iteration (`src/net.c:1188-1190`) | Multishot accept direct (`src/server.c:305-311`), multishot recv with buffer select (`322-330`); tries a buffer ring, falls back to classic provided buffers (`549-571`), and did fall back on this kernel (log: "buffer ring unavailable (Invalid argument), classic provided buffers"). ENOBUFS re-arms immediately (`447`) | `accept4` loop up to 8,192 per wake-up, level-triggered listener (`src/server.c:466-492`, `687-691`); edge-triggered `recv` into an 8 KiB stack buffer copied into a per-connection `realloc`'d accumulator (`388-409`) |
| Batching policy | After an iteration with >= 4 CQEs wait for 32 CQEs or 250 us, else 1 CQE / 50 ms (`src/net.c:1268-1276`, `src/config.c:21-22`); at most 256 requests handled per iteration (`work_budget`, `src/config.c:23`, `src/net.c:1168`) | After an iteration with >= 4 CQEs wait for **128 CQEs or 4 ms**, else 1 CQE / 2 s (`src/server.c:45-50`, `591-599`). No per-iteration request cap | `epoll_wait(…, 512, 100 ms)` with no minimum-batch wait (`src/server.c:505`); the batch is whatever is ready, up to 8 pipelined requests per connection (`412`) |
| Connection record | 48 B (`struct conn`, `src/net.c:73-90`; measured `sizeof` 48), in a prefaulted table; an idle connection owns no buffer | 72 B by `sizeof` (`conn_t`, `src/server.c:72-83`: three 16-byte `buf_t` + 4 u32 + 3 u8); README says 88 B. `calloc`'d table per worker (`506`) | 104 B by `sizeof` (`src/conn.h:23-40`), one `calloc` per connection plus `malloc`'d `in`/`out` buffers (`src/server.c:134-147`); README says "64 B struct" |
| Send buffering | Bounded pool of 1,024 x 72 KiB prefaulted chunks (`src/config.c:19-20`, `src/net.c:378`); response written in place, head placed before the body, one `send` SQE, chunk returned on completion; connection parks when the pool is empty | Per-connection growable `buf_t` (`src/util.c:63-75`: chunk pool then `malloc`/doubling); `out` swapped into `fly` and sent with one `send` SQE after the CQ drain (`src/server.c:484-502`, `342-351`); 64 KiB unsent cap pauses reading (`41`, `234-237`) | `snprintf` header + `malloc` per response (`src/server.c:357-372`), `send()` in a send phase after the batch (`553-560`), `EPOLLOUT` on `EAGAIN` and reading paused while a response drains (`446-463`) |
| SQLite: VFS/locking, mmap, cache | `unix-excl` VFS, `SQLITE_OPEN_NOMUTEX` (`src/db.c:128-131`); `mmap_size=0` (compiled out, `Makefile:40`); `cache_size=-32768` (32 MiB, `src/config.c:29`), checkpoint connection 2 MiB (`src/db.c:142`); `temp_store=MEMORY`, `wal_autocheckpoint=0`, `journal_size_limit=64 MiB` (`144-149`); `busy_timeout` 50 ms (`140`); `THREADSAFE=2` (`Makefile:36`) | One writer connection (`SQLITE_OPEN_NOMUTEX`) with WAL, `synchronous=NORMAL`, `mmap_size=0`, `cache_size=-65536` (64 MiB), `temp_store=MEMORY`, `journal_size_limit=64 MiB`, `wal_autocheckpoint=0` (`src/db.c:118-135`); one read-only connection per worker with `cache_size=-16384` (`93-99`); `busy_timeout` 5 s (`98`, `122`); compiled `THREADSAFE=2`, `DEFAULT_CACHE_SIZE=-131072` (`build.sh:22-26`). README's "`locking_mode=EXCLUSIVE`, one connection, 128 MiB cache, `THREADSAFE=0`" does not match the code | One connection: `locking_mode=EXCLUSIVE`, WAL, `synchronous=NORMAL`, **`mmap_size=1 GiB`**, `cache_size=-8192` (8 MiB), `temp_store=MEMORY`, `foreign_keys=ON`, `journal_size_limit=64 MiB`, `wal_autocheckpoint=0` (`src/db.c:50-59`); `busy_timeout` 5 s (`49`); `THREADSAFE=0` (`build.sh:25-37`) |
| Checkpoint strategy | PASSIVE checkpoints on the nice-10 thread whenever >= 4,000 frames are not yet backfilled (`src/db.c:245-249`, `src/config.c:30`); when the WAL reaches 8,192 frames the loop holds writes (reads continue) until the WAL is fully backfilled, 500 ms cap (`src/db.c:310-319`, `295-305`) | **TRUNCATE** checkpoint every 2 s on the checkpointer thread while holding the writer mutex (`src/db.c:161-175`); request handlers that need the writer wait on that mutex (`222-225`) | PASSIVE checkpoint **on the loop thread**, called from `db_commit` when `wal_frames >= 4000` and >= 1 s since the last one (`src/db.c:7`, `101-106`, `116-126`) |
| Group commit and send ordering | Writes of an iteration share one `BEGIN IMMEDIATE … COMMIT` in `app_end_batch` (`src/app.c:426-`, `src/db.c:371`); reads of the iteration share one deferred `BEGIN … COMMIT` ended before the write txn (`src/app.c:223-224`, `src/db.c:365`); all sends submitted after `COMMIT`; a failed commit or statement error turns every write response of the batch into a 500 | **No group commit**: each write request runs its own `BEGIN IMMEDIATE … COMMIT` under the mutex, with up to 20 retries (`src/db.c:222-251`; header comment `1-5`). Responses are queued and flushed after the CQ drain (`src/server.c:634`). README's "group commit" paragraph does not match the code | Group commit: first write of an `epoll_wait` batch opens `BEGIN IMMEDIATE` (`src/api.c:195`, `249`; `src/db.c:94-99`), `COMMIT` after the batch, then the send phase (`src/server.c:527-560`); a failed commit or mid-batch rollback converts exactly the batch's responses to 500s (`531-551`, `src/db.c:109-114`). Reads run in autocommit, one statement each |
| Feed query | Three range scans when the 20 newest ids span <= 256 (`src/db.c:19-21`, `50-52`), reference query otherwise or on anomaly; both strategies compared in our black-box suite | Same idea: ids from the index, `likes` range scan, `posts JOIN users` range scan, `FEED_SPAN` 256, reference query as fallback or when a concurrent write changes the shape (`src/db.c:22`, `105-109`, `490-556`); `create` uses `RETURNING` (`138`) | Reference query with the correlated `count(*)` subquery (`src/db.c:63-67`); single post via `LEFT JOIN … GROUP BY` (`71-74`); `create` is `INSERT` + `SELECT created_at` (`75-76`) |
| JWT | HMAC pads precomputed once (`src/crypto.c:107`, `126`), constant-time compare, `HS256` header check, `exp` required and numeric (`src/jwt.c:49-93`) | Pads precomputed once (`src/jwt.c:1`, `108-141`); signature compared constant-time on the base64url form, then `alg`, `exp`, optional `nbf`, `sub`, `username` (`216-273`) | Pads rebuilt on every call (`src/jwt.c:71-80`), constant-time compare (`200`, `233`), `alg` check (`225`), `exp` (`242-`) |
| Memory guard | Every 250 ms (`src/net.c:29`) `MemAvailable` and cgroup `memory.max` headroom (`826-844`): below 192 MB stop accepting, below 96 MB shed idle keep-alives (`src/config.c:27-28`) | Four times a second, `MemAvailable - SReclaimable - CmaFree` (`src/server.c:261-291`): < 192 MB stop accepting, < 96 MB shed 2-5 % of open connections per sweep (`52-55`, `650-676`) | Once a second, cgroup `memory.max` or `MemAvailable - SReclaimable` (`src/server.c:239-268`); plus **fd-pressure eviction**: at 60k/63k open sockets idle keep-alives get `Connection: close`, token-bucket paced at 10k/s, release at 55k (`src/conn.h:13-20`, `src/server.c:271-342`, `385`). 75 s keep-alive timeout (`src/conn.h:8`) |
| Build | `-O3 -march=native -fno-plt -flto=auto` (`Makefile:43-44`, `64-65`); PGO implemented (`Makefile:72-88`) but not shipped (measured no gain, `docs/RESULTS.md` s.2) | `-O3 -march=native -fomit-frame-pointer -flto`, **PGO of both SQLite and the server** trained on a synthetic DB and a 16 s synthetic load (`build.sh:22-27`, `85-129`); falls back to plain on failure. The PGO build succeeded here | `-O3 -fno-plt -g0`, **no `-march=native`** (portability), **PGO + LTO** trained in-process on a synthetic DB (`build.sh:4`, `19`, `53-56`, `src/server.c:579-642`). Built fine here |

## 2. The organizer's batch-2 row for each entry (theirs, `benchmarks/batch-2.md`)

| # | Submission | PR | Language | Users held 5 min | req/s | p95 ms | p99 ms | err % | CPU % | us/req | RAM MB |
|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | rust-epoll-scooter1337 | #82 | Rust (io_uring) | 264,000 (ran out of load generators) | 25,147 | 259 | 499 | 0.00 | 84 | 30 | 630 |
| 5 | c-iouring-ssmurfgg04-gif | #112 | C (io_uring) | 150,000 | 14,611 | 385 | 524 | 0.00 | 78 | 50 | 413 |
| 8 | c-epoll-TheFirstIstari | #64 | C (epoll) | 107,000 | 10,493 | 212 | 409 | 0.94 | 78 | 69 | 169 |
| – | c-iouring-moneebullah25 | #156 | C (io_uring) | not yet benchmarked | | | | | | | |

The organizer's "us/req" is CPU% x 1 vCPU / req/s on their box; "RAM MB" is the
service's memory at the top user count, which includes kernel socket memory
for 100k+ connections. Neither is directly comparable to the per-process
numbers below.

## 3. Same-machine results (measured)

### 3.1 Method

- Machine: AMD Athlon Silver 3050U (Zen1, 2 cores, no SMT), 14 GB RAM, Ubuntu
  kernel 6.8.0-147-generic, gcc 13.3. Loopback; the load generator shares the
  two cores with the server and the user's desktop was running. `ulimit -n`
  soft and hard are 1,048,576 here (the challenge box has 65,535), so no entry
  hit a descriptor limit; ours sized 7 rings x 65,535 slots, ssmurfgg04 8
  workers x 65,535 slots, TheFirstIstari saw `fds=1048576`.
- Each server started fresh on a fresh copy of `seed/feed.db` with
  `SQLITE_PATH=<copy> JWT_SECRET=twelve-dollar-challenge HOST=127.0.0.1
  PORT=<random>`; ours additionally `--stats-interval-s N` (logging only).
  Correctness first: `bash test/test.sh http://127.0.0.1:PORT` on a fresh seed
  copy: **42/42 for all three**.
- Generator: our `tools/mload` (closed loop, the `bench/load.js` user model;
  `make tools`). Runs, each twice, servers restarted between runs:
  - A: `mload --vus 64 --duration 20 --ramp 2 --think-scale 0 --src-ips 1`
  - B: `mload --vus 10000 --duration 60 --ramp 10 --think-scale 1 --src-ips 2`
  - C: `mload --vus 50000 --duration 60 --ramp 20 --think-scale 2 --src-ips 4`
  (`MemAvailable` was ~10 GB before every run.)
- Server CPU: `utime+stime` from `/proc/<pid>/stat` (all threads) sampled
  just before mload started and just after it finished, x 10,000 us per tick
  (`CLK_TCK=100`), divided by mload's completed-request count. Startup work
  (our prefault and cache warm-up, their PGO-free init) is therefore excluded
  for all three. Peak RSS is `VmHWM` from `/proc/<pid>/status`. The harness is
  `scratchpad/compare/run_one.sh`; raw logs in `scratchpad/compare/run{A,B,C}.log`.
- Latency percentiles are mload's, over the whole run including the ramp,
  and include the generator's own queueing on a shared core.

### 3.2 Results (two runs each, shown as run 1 / run 2)

**A: 64 VUs, no think time, 20 s (tight loop; CPU per request under batching)**

| Server | CPU us/req | req/s (avg incl. ramp) | p50 ms | p95 ms | p99 ms | errors | VmHWM MB |
|---|---|---|---|---|---|---|---|
| ours | **28.2 / 26.2** | 14.0k / 33.6k | 2.0 / 1.6 | 13.4 / 2.5 | 22.6 / 3.5 | 0 / 0 | 134 / 137 |
| ssmurfgg04 (io_uring, PGO) | 39.3 / 37.8 | 12.8k / 16.8k | 3.2 / 2.7 | 7.7 / 3.7 | 17.4 / 6.9 | 0 / 0 | 12 / 12 |
| TheFirstIstari (epoll, PGO) | 35.1 / 35.8 | 23.1k / 22.3k | 2.3 / 2.3 | 3.7 / 3.7 | 5.5 / 5.4 | 0 / 0 | 9 / 9 |

Our stats line, run 2: `cpu_us/req=26.2 (total 26.3) rss=137 MB iters=22926
cqes=1428793 batches=17078 writes=52143 cfail=0 ckpts=30 ckpt_avg_ms=102.1
ckpt_max_ms=733.6 holds=4 hold_max_ms=532.5 hold_to=1 iter_max_ms=3.0
batch_max=64 2xx=671249 4xx=0 5xx=0`. Run 1 had `iters_over_20ms=8`,
`iter_max_ms=11.3` (desktop contention during that run, see throughput).

**B: 10,000 users, think scale 1, 60 s (~1,060 req/s; one request per wake-up)**

| Server | CPU us/req | req/s | p50 ms | p95 ms | p99 ms | errors | VmHWM MB |
|---|---|---|---|---|---|---|---|
| ours | **61.2 / 60.8** | 1,061 / 1,064 | 0.1 / 0.1 | 0.5 / 0.4 | 1.3 / 1.2 | 0 / 0 | 141 / 141 |
| ssmurfgg04 | 67.5 / 67.7 | 1,062 / 1,062 | 3.0 / 3.1 | 8.8 / 29.4 | 396 / 1,202 | 0 / 0 | 23 / 23 |
| TheFirstIstari | 69.0 / 69.3 | 1,055 / 1,062 | 0.1 / 0.1 | 3.7 / 1.9 | 2,345 / 81.5 | 0 / 0 | 25 / 25 |

Our stats line, run 2: `cpu_us/req … (total 63.9) rss=141 MB iters=101276
batches=4236 writes=4363 cfail=0 ckpts=6 ckpt_avg_ms=359.1 ckpt_max_ms=1784.7
holds=2 hold_max_ms=502.4 hold_to=1 iter_max_ms=2.1 batch_max=5 5xx=0`
(the "total" includes startup; the per-run figure in the table excludes it).
ssmurfgg04's own `dbstats` reported `req … max=704.5ms` / `max=1462.5ms`
(longest request handler), i.e. the stall is inside the server, not only in
the generator.

**C: 50,000 users, think scale 2, 60 s (~2,590 req/s; connection holding)**

| Server | CPU us/req | req/s | p50 ms | p95 ms | p99 ms | errors | VmHWM MB |
|---|---|---|---|---|---|---|---|
| ours | **57.2 / 56.9** | 2,595 / 2,592 | 0.2 / 0.2 | 0.8 / 0.7 | 2.7 / 1.3 | 0 / 0 | 152 / 152 |
| ssmurfgg04 | 56.2 / 57.3 | 2,574 / 2,596 | 5.0 / 4.7 | 533 / 12.2 | 712 / 59.6 | 0 / 0 | 36 / 36 |
| TheFirstIstari | 60.6 / 63.6 | 2,475 / 2,589 | 0.3 / 0.2 | 2,016 / 123 | 2,823 / 1,045 | 0 / 0 | 38 / 38 |

Our stats line, run 2: `open=50000 max=50000 … cpu_us/req=49.8 (total 56.8)
rss=152 MB iters=175331 cqes=452641 enobufs=0 chunks_free=1023 ckpts=10
ckpt_avg_ms=141.4 ckpt_max_ms=726.5 holds=2 hold_max_ms=506.0 hold_to=1
batches=7950 writes=8603 cfail=0 iter_max_ms=5.0 batch_max=153 5xx=0`.
ssmurfgg04 `dbstats`: `req max=472.0ms` / `max=105.6ms`.

**Run-to-run spread.** CPU per request moved by <= 2 us for every server in
every pair of runs (A: ours 2.0, ssm 1.5, ist 0.7; B: <= 0.4; C: <= 3.0), so
differences of 5 us or more are above the noise; differences of 1-2 us are
not. Throughput in A varied 2.4x between our two runs (14.0k vs 33.6k, with
the desktop busy in run 1) and 1.3x for ssmurfgg04, so A's req/s column is not
a ranking. B and C are closed-loop at ~1.06k and ~2.59k req/s by construction,
so there req/s only says all three kept up. Tail latencies for the two prior
servers varied 3-30x between runs (B p99 396 vs 1,202 ms; C p95 533 vs 12 ms),
ours stayed within 1.2-2.7 ms across all six B/C runs.

## 4. Reading

What the numbers show, on this box:

- **CPU per request, batched (A):** ours 26-28 us vs 35-36 (TheFirstIstari)
  vs 38-39 (ssmurfgg04). That is 20-30 % less CPU than both PGO builds for the
  same request mix. Tied to the diff: both we and ssmurfgg04 use the
  three-range-scan feed, so the feed query is not what separates us from
  them; the differences on the hot path are our per-iteration group commit
  (one `BEGIN IMMEDIATE … COMMIT` per 256-request batch vs. one transaction
  per write request under a mutex in ssmurfgg04, `src/db.c:222-251`), the
  batched read transaction, no allocation per response (fixed chunks vs.
  growable `buf_t` / `malloc` per response), and 1 thread vs. 8 workers that
  serialize on the DB. Against TheFirstIstari the feed strategy (reference
  query with 20 correlated `count(*)` subqueries) and its `malloc` +
  `snprintf` per response are the visible differences; it already group-commits.
  The ~2.7 us we measured for the feed strategy alone (`docs/RESULTS.md`)
  does not explain the whole 8-9 us gap, so the rest is the loop and send
  path; we did not profile their binaries to apportion it further.
- **CPU per request, one request per wake-up (B):** ours 61 vs 68 vs 69 us.
  Here every server pays one kernel round trip per request and the per-request
  logic matters less, so the gap shrinks to ~10 %. This is the regime the
  real benchmark lives in at 10k users; at 100k+ users (10k+ req/s) batches
  of 2-4 form again and the A-side advantages return (our own 100k runs in
  `docs/RESULTS.md` s.3 and s.5: 34-39 us).
- **Connection holding (C):** CPU per request is equal within noise for ours
  and ssmurfgg04 (56-57 us) and 4-7 us higher for TheFirstIstari. All three
  held 50,000 keep-alive connections with zero errors. Nothing here predicts a
  user count on the real box; 50k is a third of what ssmurfgg04 held there.
- **Tail latency (B, C):** this is the clearest separation, but read it with
  the caveat that mload's percentiles include the generator on a shared core.
  Ours stayed at p99 1.2-2.7 ms in all four B/C runs. ssmurfgg04 had p99 of
  60-1,200 ms and its own `dbstats` logged single request handlers of
  105-1,462 ms; its code runs a `TRUNCATE` checkpoint every 2 s while holding
  the writer mutex that every write request takes (`src/db.c:161-175`,
  `222-225`), and its 4 ms / 128-CQE batch wait (`src/server.c:45-50`,
  `591-594`) is why its p50 is a steady 3-5 ms. TheFirstIstari had p99 of
  81-2,823 ms; its PASSIVE checkpoint runs on the loop thread from `db_commit`
  (`src/db.c:116-126`), so an fsync-bound checkpoint on this laptop disk stalls
  every connection. Our checkpoints ran on the nice-10 thread (max 0.7-1.8 s
  per checkpoint in these runs, the loop's longest iteration 2-19 ms). The
  attribution to checkpoints is consistent with the code and the logs, but
  it was not confirmed by profiling their processes, and the organizer's box
  has a faster disk than this laptop, where the effect may be smaller.
- **Memory:** ours is the heaviest process by far (134-152 MB `VmHWM` vs
  9-38 MB), by design: a prefaulted 72 MiB send pool, 7 x 2,048 x 2 KiB
  receive pools, a 458k-slot connection table and a 32 MiB SQLite cache are
  touched at startup so the first requests take no page faults. Per
  additional connection the growth is similar (ours +18 MB from 0 to 50k,
  ssmurfgg04 +24 MB, TheFirstIstari +29 MB across the same range). On a 2 GB
  box the ~130 MB fixed cost is real and is the price of the pools; the
  prior entries' 169 MB / 413 MB organizer numbers at 107k / 150k users
  are whole-service figures including kernel socket memory and are not
  comparable to any `VmHWM` here.

What the numbers do **not** show:

- Any user count, p95 at 100k+ users over a real network, or behaviour at the
  2 GB memory wall. Nothing here predicts a batch-2 row for #156.
- Absolute CPU on the organizer's box: loopback includes the kernel's delivery
  into the client's socket in the server's `stime` (estimated 30-40 %
  overstatement in our exploration; a virtio NIC costs more per packet), and
  the generator competed for the same two cores. Rankings and the direction
  of each difference are what transfer, with the usual risk that a Zen1
  laptop favours different code than a Xeon vCPU.
- The Rust entry: not built here; its organizer row (30 us/req at 25k req/s)
  is the one to beat and no same-machine number exists for it.
- Nothing about PGO on the organizer's box: both prior entries ship PGO and
  we ship plain LTO (PGO measured no gain here, `docs/RESULTS.md` s.2); on a
  different microarchitecture that decision may go the other way.

Bottom line for the PR comment: on the same machine and generator, our entry
uses 20-30 % less server CPU per request than both prior C entries in the
batched regime and ~10 % less at one request per wake-up, equal CPU to
ssmurfgg04 at 50k held connections, with flat tails where both prior entries
show checkpoint-shaped stalls, at the cost of a ~130 MB larger resident
footprint from prefaulted pools. Everything above is loopback on a shared
2-core laptop; the real box will decide.

## 5. Files

- Harness: `/tmp/claude-1000/-home-moneebullah25-Desktop-Muneeb/adb7ac99-b938-4eb6-bf0d-c91bc44e03ad/scratchpad/compare/run_one.sh`
- Raw run logs: `.../scratchpad/compare/runA.log`, `runB.log`, `runC.log`, per-run server and mload logs under `.../scratchpad/compare/runs/`
- Test logs: `.../scratchpad/compare/tests/<entry>/test.log`
- Prior-entry build log: `.../scratchpad/compare/build.log` (ssmurfgg04: "built bin/server (PGO)", smoke ok; TheFirstIstari: "built bin/server")
