# Performance plan

Goal: raise this submission's score, i.e. the most k6 users that pass a 5-minute hold with p95 < 500 ms,
p99 < 1 s and < 1% errors on the $12 droplet (1 vCPU, 2 GB RAM, no swap), within the rules (especially 3, 5, 6 and 7).

Written 2026-10-14 against commit `8b77719`. Tasks are checkboxes; tick them off as they land.

## Decisions

| Question | Decision |
|---|---|
| Scope | Measurement tools and `Bun.serve` tuning (Phases 0–1), a `Bun.listen` server (Phase 2), runs on a real droplet (Phase 3). A native rewrite is deferred. |
| Where the tooling lives | In this folder (`docker/`, `droplet/`), so reviewers can reproduce the numbers |
| `locking_mode=EXCLUSIVE` vs. a checkpoint Worker | Build both, measure, keep the winner (they can't be combined) |
| If `Bun.listen` wins | Both servers stay in the PR; `start.sh` runs the winner |
| Load generator for droplet runs | Later: k6 in Docker on a separate server you own. Until then, the local Docker k6 |

## Starting point

Average time per call on an Apple Silicon Mac (Bun 1.4.2, seed DB opened read-only). On macOS bun:sqlite uses
the system SQLite (3.51.0), on Linux its bundled one, so compare these with each other, not with the droplet.

| Hot path | Time per call |
|---|---|
| Feed SQL as written (`json_group_array(json(j))`) | 19.9 µs |
| Feed SQL with `group_concat(j, ',')` (byte-identical output) | 14.6 µs |
| Feed SQL with `ORDER BY` inside `group_concat` | 22.4 µs |
| Feed rows without any JSON building (lower limit) | 12.2 µs |
| `GET /posts/:id` SQL | ~1 µs |
| Full JWT verification | 1.1 µs (HMAC alone 0.35–0.44 µs) |
| `new Response(...)`: headers as a plain object / as one shared `Headers` | 169 ns / 114 ns |
| mmap on vs. off | feed 14.0 vs. 14.5 µs, post 1.3 vs. 2.0 µs: keep it on |

Traffic, from `bench/load.js`: one user loop is 1 feed + 1 post + 0.15 likes + 0.02 creates = 2.17 requests per
~20.5 s of think time, so ~0.106 req/s per user (20k users ≈ 2.1k req/s, 60k ≈ 6.4k req/s). The mix is 46% feed,
46% post, 7% like, 1% create.

Ceilings outside the code:

- `LimitNOFILE=65535` and one connection per user: direct mode tops out around 65k users.
- One k6 machine with Linux's default local port range (32768–60999) opens only ~28k connections to one IP and
  port, unless the organizer widened the range.

## Ground rules

- One commit per change. Each must pass `docker/test.sh` (42/42) before it's measured.
- Experiment switches are `AB_*` environment variables whose defaults are the production settings, so `start.sh`
  still runs with only the SPEC variables.
- Main metric: server CPU µs per request at a fixed load below saturation (e.g. ~60% of the local knee), plus
  p95, p99, error rate and memory. Run variants alternately (A B A B A B, 3 runs each). Start with an A/A run
  (the baseline against itself) to see how much results vary on their own.

## Phase 0: measurement tools

- [ ] **Clean shutdown** in `server.js`: on SIGTERM/SIGINT, `server.stop(true)`, flush queued writes, close the
  DB, `process.exit(0)`. Bun is PID 1 in the container, and PID 1 ignores signals it has no handler for;
  `--cpu-prof` only writes its file on exit.
- [ ] **`start.sh`**: `exec bun ${AB_BUN_FLAGS:-} "${AB_IMPL:-server.js}"` (`:-` because of `set -u`).
- [ ] **`docker/docker-compose.yml`**:
  - pass through `AB_*` and `BUN_JSC_*`;
  - replace `cpus: 1` with `cpu_period: 10000` and `cpu_quota: ${SERVER_CPU_QUOTA:-10000}`; the 10 ms period
    keeps throttling pauses to a few ms instead of up to 100 ms when emulating a slower CPU;
  - `cpuset: "${SERVER_CPUSET:-0}"` for the server and other cores for k6 (e.g. `1-3`), so the server's threads
    share one core as on the droplet (check that `nproc` prints 1 inside). Local only: nothing gets pinned on the
    real box;
  - a `tools` service under compose profile `tools`: a small image with `strace` and `sysstat`
    (`docker/tools.Dockerfile`), `pid: "service:server"`, `cap_add: [SYS_PTRACE]`. The server image stays exactly
    what `install.sh` and `build.sh` produce. If compose rejects `pid: "service:server"`, use
    `docker run --pid container:<id>`.
- [ ] **`docker/calibrate.js`**: a fixed CPU workload on the seed DB opened read-only (feed SQL, post SQL, JWT,
  JSON) that prints µs per operation. Run it in the container and on the droplet; the ratio sets
  `SERVER_CPU_QUOTA`.
- [ ] **`docker/ab.sh`**: for each variant, alternating:
  1. fresh start, 30 s warm-up;
  2. read `usage_usec` from `/sys/fs/cgroup/cpu.stat` in the server container (cgroup v2);
  3. k6 at a fixed user count for 2 minutes with `--summary-export` (supported by `grafana/k6:1.8.2`);
  4. read `usage_usec` again.

  It writes CPU µs per request (change in `usage_usec` ÷ `http_reqs`), p95, p99, failure rate, `memory.peak`, and
  `anon` and `sock` from `memory.stat` to `results/ab-<timestamp>.tsv`. The DB copy's page cache also counts
  toward `memory.peak`, so on its own it overstates the risk of running out of memory.
- [ ] **`docker/profile.sh`**: run with `AB_BUN_FLAGS="--cpu-prof-md --cpu-prof-dir=/tmp/prof"` under load, then
  `docker compose stop` and `docker compose cp` the profile into `profiles/`. Optionally run
  `strace -c -f -p 1` from the `tools` service at low load to count system calls per request. Use `--cpu-prof`
  instead of `--cpu-prof-md` for a `.cpuprofile` (DevTools, speedscope); passing both flags in one run fails on
  Bun 1.4.2.
- [ ] **`docker/knee.sh`**: the organizer's procedure against any `BASE_URL`: 1,000 users for 2 minutes
  (warm-up), then from 2,500 users, doubling until a run fails, narrowing down to within 250, confirmed by a
  5-minute hold.
- [ ] **`.gitignore`**: `results/`, `profiles/`.
- [ ] **Checkpoint**: the baseline's noise, CPU µs per request and a profile are recorded before Phase 1.

## Phase 1: `Bun.serve` improvements

Each step is measured against the state before it.

- [ ] **Feed SQL** (`feedStmt` in `db.js`): `'{"posts":[' || coalesce(group_concat(j, ','), '') || ']}'` instead
  of `json_group_array(json(j))`. The `coalesce` keeps `{"posts":[]}` for an empty table (`group_concat` of no
  rows is NULL). The golden files check the bytes. Like the current query, it relies on the subquery's row
  order; don't put `ORDER BY` inside the aggregate (slower, see above).
- [ ] **Likes in one statement** (`like()` in `db.js`):
  `INSERT INTO likes (user_id, post_id) SELECT $uid, $pid WHERE EXISTS (SELECT 1 FROM posts WHERE id = $pid) ON CONFLICT (user_id, post_id) DO NOTHING`.
  Only when `changes === 0`, check whether the post exists (200 vs. 404).
- [ ] **Commit batching** (`enqueue()`/`flush()` in `db.js`): `AB_FLUSH_MS` = 0 (today's `setImmediate`), 2 or
  5; `AB_WAL_AUTOCHECKPOINT` = 1000, 4000 or 16000.
- [ ] **Locking** (`AB_LOCKING`):
  - `normal`: today's behaviour, the baseline.
  - `exclusive`: `PRAGMA locking_mode = EXCLUSIVE` before `PRAGMA journal_mode = WAL`. Check that no `-shm` file
    appears next to the DB, which confirms the WAL index lives in process memory.
  - `worker`: the main connection sets `wal_autocheckpoint = 0`; a new `checkpoint-worker.js` opens its own
    connection and runs `PRAGMA wal_checkpoint(PASSIVE)` every ~500 ms. If the worker dies, the main connection
    turns automatic checkpoints back on.
  - Expected: `exclusive` wins on CPU per request; `worker` wins on worst-case and p99 latency, because the
    checkpoint fsyncs leave the event loop.
- [ ] **`Bun.serve` cleanup** (`route()` and `Bun.serve` in `server.js`): `routes` for every endpoint with
  `fetch` as the 404 fallback, one shared `Headers` object, `development: false`.
- [ ] **JavaScriptCore settings**: print the Linux x64 defaults in the container with
  `BUN_JSC_dumpOptions=2 bun -e 1` (on macOS `useConcurrentGC=true` and `useParallelMarkingConstraintSolver=true`).
  Compare `BUN_JSC_useConcurrentGC=false`, `BUN_JSC_useParallelMarkingConstraintSolver=false` and larger
  `BUN_JSC_smallHeapGrowthFactor` / `BUN_JSC_mediumHeapGrowthFactor`. Write the winners into `start.sh`.

## Phase 2: `Bun.listen` server

- [ ] **`handlers.js`**: move feed, getPost, createPost, like and health into functions that return
  `[status, body]` (or a Promise of it), shared by both servers so the comparison is fair. `server.js` keeps its
  behaviour and passes 42/42.
- [ ] **`server-raw.js`** on `Bun.listen`:
  - parse the request line and the headers that matter (`content-length`, `authorization`, `connection`,
    `transfer-encoding`, `expect`) from the raw bytes, in any case; wait for the full body;
  - pipelined requests: answer strictly in order, and pause parsing a connection while one of its writes waits
    for its commit;
  - one `socket.write()` per response: status line, `Content-Type`, a `Date` refreshed once per second,
    `Content-Length` in bytes, body. If the socket takes only part of it, finish on `drain`;
  - `TCP_NODELAY`: check Bun's default or call `setNoDelay`, otherwise delayed ACKs can add 40 ms stalls;
  - keep-alive by default for HTTP/1.1; honour `Connection: close` and HTTP/1.0 rules; close idle connections
    after at least 65 s;
  - `Expect: 100-continue`: send `HTTP/1.1 100 Continue` (curl sends it for larger bodies and waits up to 1 s
    for the reply);
  - headers over 16 KB: 431 and close; chunked request bodies: 400 and close; exceptions: 500.
- [ ] **`docker/http-edge.js`**: sends the same bytes to both servers and requires the same status and body:
  - two pipelined requests in one packet;
  - a request split one byte at a time;
  - a POST body split across packets;
  - lowercase and mixed-case header names;
  - `Connection: close`;
  - HTTP/1.0;
  - `Expect: 100-continue`;
  - 1,000 pipelined `GET /feed` without reading the responses (write backpressure);
  - unknown paths.
- [ ] **Decide**: `AB_IMPL=server.js` vs. `AB_IMPL=server-raw.js`, both with the Phase 1 winners, and a profile
  of each. Both stay in the PR; the winner becomes the default in `start.sh`.

## Phase 3: real droplet

- [ ] **`droplet/setup.sh`**: copy the repo to a $12 Ubuntu 24.04 droplet created in the web UI (git clone or
  rsync), then run it as root. It:
  - installs `sysstat`, `strace`, `sqlite3` and `nodejs` (Ubuntu's 18.19 is enough for `seed/make-seed.sh`);
  - runs `install.sh` as root, then `build.sh` as a user named `bench`;
  - runs `seed/make-seed.sh`;
  - installs a systemd unit like the organizer's: non-root, `LimitNOFILE=65535`,
    `AmbientCapabilities=CAP_NET_BIND_SERVICE`, the SPEC environment variables, and a fresh DB copy in
    `ExecStartPre`.
- [ ] **`droplet/run.sh start|stop`**: `start` copies a fresh DB, restarts the unit, waits for `/health`, and
  records `mpstat`, `vmstat` (CPU steal), `pidstat` and `ss -s` into `results/<timestamp>/`. `stop` collects the
  unit's `cpu.stat`, `memory.peak` and `memory.stat`.
- [ ] **Calibrate**: run `calibrate.js` on the droplet and set the local `SERVER_CPU_QUOTA` so Docker matches.
- [ ] **Later: `droplet/k6-remote.sh`** for your own load server: `grafana/k6:1.8.2` in Docker against
  `BASE_URL=http://<droplet-ip>`, reusing `knee.sh` and `ab.sh`.
  - Use `--network host` so 60k connections don't go through Docker's NAT. Docker can't set `net.*` sysctls
    with host networking, so widen `net.ipv4.ip_local_port_range` to `1024 65535` on the load server itself
    (rule 11 only covers the benchmark box).
  - Raise `nofile` to 1M.
  - Check first: round-trip time to the droplet, bandwidth (~150 Mbit/s out of the droplet at 60k users), k6
    memory per user (measure at a few thousand users and extrapolate), and the conntrack table size.
- [ ] **Final runs**: the top 2–3 variants on the droplet, then the maximum user count and a 5-minute hold with
  the default configuration.

## Phase 4: wrap-up

- [ ] Final defaults in `start.sh`.
- [ ] README: the optimizations and their measured effect, how to reproduce them, that the `AB_*` switches are
  for development only, and, if `exclusive` wins, that no other process can open the DB while the server runs.
- [ ] Run `test/run.sh` on Linux x64, as CI does.

## Out of bounds

- Rule 5 (no caching across requests): no merging of simultaneous `/feed` requests into one query (a result
  cache in disguise), no remembered JWT verifications, no like counts kept in memory.
- Rules 3, 6 and 7: no new indexes, no `synchronous=OFF`, no feed sorted by `id` instead of `created_at` (it only
  gives the same order because of how the data was generated).
- Not worth it: faster JWT code (measured: 1.1 µs on ~8% of requests), compression (CPU runs out before
  bandwidth), turning mmap off (measured: slower).
- Deferred: a native rewrite (C, Zig or Rust; epoll; SQLite compiled on the box). Revisit if the profiles after
  Phases 1–2 show Bun's HTTP layer dominating.

## Risks

- Local CPU numbers come out low: in Docker, part of the network processing is charged to the k6 container. Use
  local runs to rank variants; trust only the droplet for absolute numbers.
- With `EXCLUSIVE` locking, no other process can open the DB while the server runs.
- Already checked (2026-10-14): `grafana/k6:1.8.2` supports `--summary-export`, so no `handleSummary` wrapper is
  needed; a SIGTERM handler that calls `process.exit(0)` does write the CPU profile (macOS, Bun 1.4.2; the pinned
  version is 1.4.3).

## Effort

Phase 0 about half a day, Phase 1 a few hours plus runs, Phase 2 about a day, Phase 3 a few hours plus runs.
