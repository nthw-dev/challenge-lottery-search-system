# PoC Results — Lottery Search System

Measured 2026-09-05 on Kubernetes (docker-desktop, Apple Silicon) with the resource limits from
TECH_SPEC Section 10.3 enforced: API pod 500m CPU / 128Mi, Postgres pod 2000m / 1Gi, **PostgreSQL 17.11**,
10,000,000 tickets, k6 on the host against the NodePort.

Every number below is backed by a bundle under [`docs/evidence/`](evidence/) holding the complete
k6 log, the summary as JSON and HTML, nine resource charts, the raw samples the charts were drawn
from, and the SQL verification output. See [`docs/evidence/README.md`](evidence/README.md) for how
to read them. Query plans come from `docs/explain-k8s-10M.txt`. The capture scripts that produced
the bundles live in the proof-of-concept repository.

## 0. TL;DR

| Hypothesis | Result |
|---|---|
| **H1** p95 < 100 ms for every wildcard class at 10M rows, 100 VUs | **Pass** with Strategy C: p95 46.7 ms at 6,068 req/s, 0 errors, all k6 thresholds met. B, A and the baseline all fail (Section 3) |
| **H2** no ticket handed to two users under 200-VU contention | **Pass** — 342,412 allocations across two runs, 0 duplicates, 0 state mismatches (Section 4) |
| **H3** chosen index strategy ≥ 10× faster than `LIKE` baseline | **Pass** — exact pattern 829 ms → 0.087 ms (≈9,500×, four orders of magnitude), with half the index footprint of Strategy A (Sections 1 and 2) |
| Stability: no OOMKilled, memory flat over a 10-min soak | **Pass** — 942,015 requests, 0.0 % errors, 0 restarts, API peak 26 MiB of 128 MiB (Section 5) |
| Sizing for an illustrative 10,000 req/s target under a 60 % budget (both values chosen by the author to demonstrate the method) | API 4 × 500m (52 %), Postgres 6000m / 3Gi (36 % CPU, 44 % memory), from a measured linear cost model (Section 6) |

Recommended design: **PostgreSQL 17 + Strategy C** (1M-row `numbers` dimension table with fixed
leading digits collapsed into a primary-key range) for search, and **a random concrete-number point
lookup guarded by `FOR UPDATE SKIP LOCKED`** for allocation.

## 1. Seed and storage (10M rows)

| Step (k8s Job) | Time |
|---|---|
| `INSERT … generate_series(1, 10000000)` | 7 s |
| Strategy A: 6 partial expression indexes | 12 s |
| Strategy B/C: `numbers` table + 6 digit indexes + `idx_tickets_number` | 7 s |

| Relation | Size |
|---|---|
| `tickets` heap | 438 MB |
| `tickets_pkey` | 214 MB |
| Strategy A: `idx_d1..idx_d6`, 68 MB each | **405 MB** |
| Strategy B/C: `numbers` heap + pkey + 6 digit indexes | 104 MB |
| Strategy B/C: `idx_tickets_number`, partial on `status = 0` | 97 MB |
| Strategy B/C total | **201 MB** — about half of A, and the `numbers` half never grows with ticket count |

`idx_tickets_number` is a partial index on `status = 0`; it had grown from 89 MB to 97 MB by the
time these sizes were taken, because thousands of allocate/release cycles in the soak leave dead
entries until vacuum reclaims them. Routine autovacuum handles this in production.

`VACUUM ANALYZE` initially failed on Kubernetes: the pod's default 64 MiB `/dev/shm` is too small
for parallel workers. Fixed with a memory-backed `emptyDir` in `deploy/k8s/03-postgres.yaml`.

## 2. Query plans — `EXPLAIN (ANALYZE, BUFFERS)`, 10M rows, `LIMIT 20`, single client, warm cache

Execution time in ms. Single runs vary by tens of percent between passes; the ordering between
strategies does not.

| Pattern | wildcards | 0 baseline `LIKE` | A expr-indexes | B numbers table | **C = B + prefix range** | D array (Section 8) |
|---|---|---|---|---|---|---|
| `123456` | 0 | 829 | 130 | 5.1 | **0.087** | 0.061 |
| `123***` | 3 | 2.5 | 21.6 | 1.7 | **0.148** | 0.117 |
| `12*4**` | 3, scattered | 1.5 | 21.6 | 1.3 | **0.158** | 0.103 |
| `****23` | 4 | 0.17 | 0.11 | 0.22 | **0.132** | 0.130 |
| `**3***` | 5 | 0.05 | 0.05 | 0.33 | **0.206** | 0.217 |
| `******` | 6 | 0.04 | 0.02 | 0.02 | **0.024** | 0.029 |

- **Baseline** is only slow when matches are rare. With `LIMIT 20` a scan stops as soon as 20 rows
  match, so wide patterns are cheap in every strategy. The hard case is the exact pattern, where the
  scan must read the whole table to find ~10 rows.
- **Strategy A** never gets a full `BitmapAnd`: for `123456` the planner uses *one* digit index
  (10 % selective, 1M rows) and filters the rest. Each digit index is too weak for combining to pay
  off — exactly the risk TECH_SPEC Section 5.2 anticipated.
- **Strategy B** is stable at 1–5 ms because it only touches the 1M-row `numbers` table plus a few
  `idx_tickets_number` probes.
- **Strategy C** turns fixed leading digits into `n BETWEEN lo AND hi` on `numbers_pkey`
  (index-only) and an exact pattern into `number = X` straight on tickets.

### 2.1 `ORDER BY id` was the real bottleneck and was removed

The spec's query had `ORDER BY id LIMIT 20`. At 10M rows that forces a walk of `tickets_pkey`,
probing `numbers` per row via Memoize. Re-measured on 17 (`docs/explain-k8s-10M-orderby.txt`):

| Pattern | C without `ORDER BY` | C with `ORDER BY id` | slowdown |
|---|---|---|---|
| `123***` | 0.148 ms | 13.4 ms | 90× |
| `12*4**` | 0.158 ms | 14.6 ms | 92× |
| `****23` | 0.132 ms | 7.1 ms | 54× |
| `**3***` | 0.206 ms | 2.1 ms | 10× |

Under 100 VUs the ordered form saturated the 2-core Postgres and every pattern class queued equally.
Search results have no meaningful order, so `ORDER BY` was dropped.

## 3. S1 — Latency (H1): 10 → 100 VUs, `/v1/search`

Same dataset and pods for all four runs; only the `STRATEGY` env var changes. Latency in ms.
Evidence: [`s1-C`](evidence/s1-C/), [`s1-B`](evidence/s1-B/), [`s1-A`](evidence/s1-A/), [`s1-0`](evidence/s1-0/).

| Strategy | p95 w0 `dddddd` | p95 w3 `ddd***` | p95 w4 `****dd` | p95 w5 `**d***` | **p95 all** | max | req/s | failed | k6 exit | H1 |
|---|---|---|---|---|---|---|---|---|---|---|
| **C** numbers + prefix range | 46.8 | 46.9 | 46.5 | 46.7 | **46.7** | 71 | **6,068** | 0 % | 0 | **pass** |
| B numbers table | 176.0 | 114.1 | 103.3 | 103.6 | 108.3 | 196 | 975 | 0 % | 99 | fail |
| A per-digit expression indexes | 5,663 | 4,503 | 4,503 | 4,392 | 4,705 | 6,514 | 25 | 0 % | 99 | fail |
| 0 baseline `LIKE` | 1,320 | 1,049 | 1,018 | 1,053 | 1,051 | 22,197 | 128 | **94.3 %** | 99 | fail |

Peak resource use during each run, from the samples in each bundle:

| Strategy | API CPU | API memory | Postgres CPU | Postgres working set | Postgres free |
|---|---|---|---|---|---|
| C | **100 % (at limit)** | 17 MiB | 85 % | 976 MiB | 48 MiB |
| B | 24 % | 16 MiB | **102 % (at limit)** | 1,007 MiB | 17 MiB |
| A | 2 % | 12 MiB | **102 % (at limit)** | 999 MiB | 25 MiB |
| 0 | 1 % | 9 MiB | **101 % (at limit)** | 789 MiB | 235 MiB |

Reading the table:

- **C is the only strategy that meets H1**, and the only one where the bottleneck moved out of
  Postgres and into the 500m API pod. Per-class p95 is flat to within 0.4 ms, because every class
  costs the same handful of index probes. Postgres has CPU headroom, so an extra API replica would
  raise throughput.
- **B** does 6.2× less throughput than C purely because of the per-digit `BitmapAnd` on the
  `numbers` table. The prefix-range trick removes that work for any pattern with fixed leading digits.
- **A** saturates the 2-core Postgres at 25 req/s. `26-db_cpu_throttle.png` in its bundle shows the
  cgroup being denied CPU well beyond its quota every second, which is the direct mechanism behind
  the 4.7 s p95.
- **Baseline** did not merely get slow, it went *down*: the 10-connection pool filled with ~1 s full
  scans, `/readyz` timed out, Kubernetes pulled the pod from the Service, and 15,592 of 16,528
  requests failed — 15,588 of them with a bare `EOF`, the connection dropped mid-request
  (94.3 % failed, `max=22.2 s`). That is correct behaviour for an overloaded backend, and it is why
  liveness must not depend on the database while readiness should.
- **Postgres memory runs close to its 1 GiB limit under every strategy** (working set 976–1,007 MiB
  for A, B and C): the hot set of a 10M-row table plus its indexes is roughly the size of the pod.
  Nothing was OOM-killed, but the 1 GiB baseline profile is tight, which is one reason the capacity
  plan (Section 6) allocates 3Gi.

## 4. S2 — Contention (H2): 200 VUs, one pattern, `/v1/allocate limit=1`, 60 s

Strategy C, ledger truncated and all tickets released before each run.

| | `****23` ≈ 100k tickets ([`s2`](evidence/s2/)) | `**3***` ≈ 1M tickets ([`s2b`](evidence/s2b/)) |
|---|---|---|
| Allocations in 60 s | 95,630 (1,540/s) | **246,782 (4,110/s)** |
| HTTP errors | 0 | 0 |
| Ledger rows = reserved = distinct ids | 95,630 | 246,782 |
| **Duplicate ticket ids (TECH_SPEC Section 9.2)** | **0** | **0** |
| Ledger vs ticket-state mismatches | 0 | 0 |
| Latency p95 / max | 510 ms / 2.4 s | **75 ms** / 760 ms |
| Peak API / Postgres CPU | 80 % / 100 % | 93 % / 88 % |
| k6 exit | 99 (p95 threshold) | 0 |

**H2 passes in both regimes**: 342,412 allocations under 200-way contention on a single pattern, and
not one ticket reached two users. The latency difference between the two columns is the whole story:
`****23` was **96 % drained** in 60 seconds, so late in the run most random probes hit already-taken
numbers and fall back to a pattern scan over an index full of dead entries. `**3***` cannot be
exhausted in 60 s and meets every threshold. This is a near-exhaustion effect, not a concurrency
defect, and in a real product it is where a "this pattern is almost gone" cut-off belongs.

### 4.1 Why allocation does not use `ORDER BY id … LIMIT` either

The spec's allocate statement has a "hot head": every reserved row stays at the front of the id
order, so each new allocation walks past all previous ones. Measured during development after only
713 reservations: 68,453 rows walked, 145 ms per allocate, growing with every allocation.

`repository.Allocate` instead picks a **random concrete number inside the pattern** (`****23` →
e.g. `481723`) and does a point lookup on `idx_tickets_number`, retrying up to 3 times before
falling back to the pattern scan. Concurrent users spread over 10^wildcards numbers, so `SKIP
LOCKED` rarely has anything to skip and the cost is O(1) regardless of how much has already been
allocated. `SKIP LOCKED` remains the correctness guarantee on both paths; the random probe is purely
a performance measure.

## 5. S3 — Soak: 50 VUs × 10 min, search + allocate + release ([`s3`](evidence/s3/))

| Metric | Value |
|---|---|
| Requests | **942,015** in 10 min (1,568 req/s, 314,005 iterations) |
| HTTP errors | **0.00 %** |
| Latency | p95 104 ms, max 6.8 s |
| API pod memory | start 26 MiB → end 18 MiB, peak **26 MiB** of a 128 MiB limit (`GOMEMLIMIT` 100 MiB) |
| API pod CPU | peak 58 % of 500m |
| Postgres | peak 79 % CPU, 1,011 MiB working set |
| API restarts / OOMKilled | **0 / none** |
| Ledger rows | 1,556,674 over 1,336,481 distinct tickets |
| Re-allocated after release | 220,193 |
| Inconsistent ticket rows / orphan ledger rows | **0 / 0** |
| k6 exit | 99 |

**Stability passes**: no OOM kill, no restart, and memory flat at roughly a fifth of the limit.

Note on the duplicate check: S3 is the only scenario that calls `/v1/release`, so a freed ticket
being handed out again is correct behaviour and the append-only ledger legitimately holds two rows
for it. The 220,193 repeats are re-allocations, not H2 violations, and `scripts/capture.sh`
runs a different verification for this scenario — state consistency and orphan checks, both zero. H2
itself is proven by Section 4, where nothing is ever released.

The p95 misses the 100 ms target because the soak is write-heavy: two of every three requests mutate
rows, so Postgres is doing hundreds of write transactions per second with WAL fsync. The API pod has
ample headroom; the resource that would need to grow first is Postgres.

## 6. C1 — Capacity model: stepped 1,000 → 4,000 req/s on the scaled profile ([`c1-capacity`](evidence/c1-capacity/))

The 10,000 req/s target and the 60 % budget are illustrative values chosen to demonstrate the
method; the challenge specifies neither. The target is derived, not driven: k6 on the same host as the cluster would compete
with the pods it is measuring. Instead a `ramping-arrival-rate` test held 1,000 / 2,000 / 3,000 /
4,000 requests per second for 60 s each on the proposed profile (4 API pods at 500m / 128Mi,
Postgres at 6000m / 2560Mi), sampling only the steady 50 s of each step. Traffic mix: 90 % search,
10 % allocate-and-release, so the pool never drains and the cost is steady-state.

k6 delivered 579,999 of 580,000 requested calls (100.00 %), 0 dropped
iterations, 0.0 % failed, p95 1.12 ms across the whole run.

| Driven rate | API CPU (4 pods) | API % of limit | Postgres CPU | Postgres % of limit | Postgres working set |
|---|---|---|---|---|---|
| 1,000 | 100m | 5.0 % | 320m | 5.3 % | 876 MiB |
| 2,000 | 192m | 9.6 % | 484m | 8.1 % | 1,109 MiB |
| 3,000 | 293m | 14.6 % | 706m | 11.8 % | 1,283 MiB |
| 4,000 | 415m | 20.7 % | 942m | 15.7 % | 1,323 MiB |

The step increments are near-constant, which licenses a linear model. Fitting the first and last
step separates fixed background work from marginal per-request cost:

```
CPU used  =  background  +  cost per request × rate

API       =    -4.6 m-core  +  0.1049 m-core per req/s   →  1,044m at 10,000 req/s
Postgres  =   112.1 m-core  +  0.2075 m-core per req/s   →  2,187m at 10,000 req/s
```

The slightly negative API intercept is an artefact of fitting through two points; its real meaning is
that the API has essentially no fixed background cost, and almost all of its CPU scales with load.

Holding both components under **60 % of their limits** at 10,000 req/s:

| Component | Replicas | CPU limit | Memory limit | Projected use at 10,000 req/s |
|---|---|---|---|---|
| API | **4** | 500m per pod (2,000m total) | 128Mi per pod | 1,044m = **52 % CPU**, 17 MiB = 13 % memory |
| PostgreSQL | **1** | 6,000m | 3Gi | 2,187m = **36 % CPU**, ~1,363 MiB = 44 % memory |

Three points a purely CPU-based model would miss:

- **Database memory does not grow linearly with load.** It grows with the hot data pulled into cache
  and then flattens: per-step increments were 232 → 174 → 41 MiB, converging near
  1,363 MiB. Extrapolating linearly would have called for roughly
  2,217 MiB.
  At the 2,560Mi tested limit the plateau is 53 %, inside the budget but with no
  margin; 3Gi (44 %) is the recommendation.
- **Connection pooling is not a constraint.** Required concurrency is rate × service time =
  10,000 × 0.69 ms ≈ **7 connections**, against 40 provisioned (4 pods × pgxpool 10).
- **Scale the API out, not up.** The Go runtime is pinned to one processor per pod, so enlarging a
  pod does not help; adding pods does, and it improves fault tolerance at the same time.

Beyond roughly 20,000 req/s the single database becomes the limit. The next step there is read
replicas for `/v1/search` only, with `/v1/allocate` continuing to use the primary, because
`SKIP LOCKED` requires the authoritative copy. The full plan is in [`docs/CAPACITY.html`](CAPACITY.html).

## 7. Concurrency tests (`go test`, real PostgreSQL)

`internal/repository/repository_test.go`, run against all five strategies including the rejected
ones:

- 50 goroutines × `Allocate("****23", limit 5)` against 120 matching tickets, so demand of 250
  exceeds supply: every ticket handed out exactly once, ledger duplicate query returns 0 rows.
- An exhausted pool returns an empty result, not an error (TECH_SPEC Section 8.1.2).
- A stale reservation is freed by the TTL sweep; explicit release only works for the owner.

## 8. Strategy D — sending candidate numbers as an array (measured, not adopted)

PostgreSQL 17 can satisfy `= ANY(array)` with a single B-tree descent, which makes a query shape
viable that a subquery cannot match: have the API enumerate the matching numbers and send them as an
`int[]`. This is **Strategy D**, applied only to patterns with at most three wildcards (1,000 values,
about 4 KB); anything wider falls back to Strategy C. Same ramp as S1; pattern classes are `w0`
(exact), `w3lead` (`ddd***`) and `w3scatter` (`dd*d**`) — the last is where Strategy C has no
leading run to turn into a key range, so it is the only class where D can differ materially.
Evidence: [`d1-C`](evidence/d1-C/), [`d1-D`](evidence/d1-D/).

| Strategy | p95 `w0` | p95 `w3lead` | p95 `w3scatter` | **p95 all** | max | req/s | API CPU | DB CPU | k6 exit |
|---|---|---|---|---|---|---|---|---|---|
| C | 48.4 | 48.4 | 48.6 | **48.4** | 153 | 4,856 | 100 % | 71 % | 0 |
| **D** | 55.5 | 55.7 | 55.7 | **55.7** | 77 | 6,110 | 100 % | 35 % | 0 |

Both runs: 0 HTTP errors, 0 duplicate tickets, 0 ledger mismatches.

Reading it: D moves work from the database to the API. Peak database CPU goes from 71 % to
35 %, p95 changes by +15 % and throughput by +26 %, while the API is the saturated
resource in both runs. The single-query plans in Section 2 show the same shape: D is a little faster than C
on the three-wildcard patterns and indistinguishable elsewhere.

**Recommendation: leave Strategy D on the shelf.** The capacity plan (Section 6) puts the database at well
under half its allocation at 10,000 requests/sec, so buying database headroom solves a problem this
system does not have, while D adds a second predicate path, a per-request allocation of up to 1,000
integers, and roughly 4 KB of extra traffic on every narrow query. Strategy D becomes the right
answer only if the database later turns into the bottleneck. The implementation stays in the code
behind `STRATEGY=D` for that day.

## 9. Answers to the challenge

| Challenge question | Answer |
|---|---|
| Database / storage | A single PostgreSQL 17. Row locking with `SKIP LOCKED` makes allocation atomic without Redis or a distributed lock, 10M rows plus indexes fit in about 1 GB, and there are no extra components to operate. |
| Algorithm / indexing for wildcards | The search space is the 10⁶ possible numbers, not the 10⁷ tickets. Resolve the pattern against a 1M-row `numbers(n, d1..d6)` table — a primary-key range for a fixed prefix, per-digit B-trees for the rest — then join into `tickets(number)`. Per-digit indexes on the ticket table itself are twice as large and the planner cannot combine them profitably (Section 3). |
| Preventing duplicate results | One statement: `SELECT … FOR UPDATE SKIP LOCKED` → `UPDATE status = 1` → `INSERT INTO allocations`. Proven by 342,412 allocations under 200-VU contention with zero duplicate ticket ids (Section 4). Reservations expire through a TTL sweep. |
| Performance at 10M+ | Search cost is independent of ticket count, bounded by the 1M `numbers` table plus a couple of index probes; allocation is a point lookup. Growing to 50M or 100M tickets only grows `tickets_pkey` and `idx_tickets_number`. |

## 10. Caveats

- All measurements come from one docker-desktop node, so the API pod, Postgres and k6 share the same
  physical CPU. Absolute throughput would differ on separate machines; the comparison between
  strategies, which is what the argument rests on, is unaffected.
- `22-db_disk_free.png` tracks the whole docker-desktop VM volume, not a dedicated device, because
  the PVC is host-path backed. Actual database size is recorded in each bundle's `41-run.txt`.
- API CPU and memory series come from metrics-server, which refreshes about every 15 s. Postgres
  series are read from cgroup files at the 5 s sampling interval, so the two have different
  cadences by design.
- A k6 exit code of 99 means a threshold was crossed, not that the run failed. It is the expected
  outcome for strategies A, B and 0 and for the exhaustion-regime S2; each bundle records the code.
- Single-query `EXPLAIN` times vary between passes (a cold-cache pass earlier the same day gave
  0.55 ms for Strategy C on `123***` against 0.148 ms warm). The tables in Section 2 are from a warm pass;
  the load tests in Sections 3–8 are what the conclusions rest on.
