# PoC Results — Lottery Search System

Measured 2026-09-04 on Kubernetes (docker-desktop, Apple Silicon) with the resource limits from
TECH_SPEC §10.3 enforced: API pod 500m CPU / 128Mi, Postgres pod 2000m / 1Gi, 10,000,000 tickets,
k6 on the host against the NodePort.

Every number below is backed by a bundle under [`docs/evidence/`](evidence/) holding the complete
k6 log, the summary as JSON and HTML, nine resource charts, the raw samples the charts were drawn
from, and the SQL verification output. See [`docs/evidence/README.md`](evidence/README.md) for how
to read them. Query plans come from `docs/explain-k8s-10M.txt`.

## 0. TL;DR

| Hypothesis | Result |
|---|---|
| **H1** p95 < 100 ms for every wildcard class at 10M rows, 100 VUs | **Pass** with Strategy C: p95 56 ms at 5,346 req/s, 0 errors. B, A and the baseline all fail (§3) |
| **H2** no ticket handed to two users under 200-VU contention | **Pass** — 351,481 allocations across two runs, 0 duplicates, 0 state mismatches (§4) |
| **H3** chosen index strategy ≥ 10× faster than `LIKE` baseline | **Pass** — exact pattern 955 ms → 0.06 ms (≈15,000×), with half the index footprint of Strategy A (§1, §2) |
| Stability: no OOMKilled, memory flat over a 10-min soak | **Pass** — 744k requests, 0 errors, 0 restarts, API peak 27 MiB of 128 MiB (§5) |

Recommended design: **PostgreSQL + Strategy C** (1M-row `numbers` dimension table with fixed leading
digits collapsed into a primary-key range) for search, and **a random concrete-number point lookup
guarded by `FOR UPDATE SKIP LOCKED`** for allocation.

## 1. Seed and storage (10M rows)

| Step (k8s Job) | Time |
|---|---|
| `INSERT … generate_series(1, 10000000)` | 9 s |
| Strategy A: 6 partial expression indexes | 16 s |
| Strategy B/C: `numbers` table + 6 digit indexes + `idx_tickets_number` | 9 s |

| Relation | Size |
|---|---|
| `tickets` heap | 422 MB |
| `tickets_pkey` | 214 MB |
| Strategy A: `idx_d1..idx_d6`, 66 MB each | **397 MB** |
| Strategy B/C: `numbers` heap + pkey + 6 digit indexes | 103 MB |
| Strategy B/C: `idx_tickets_number`, partial on `status = 0` | 89 MB |
| Strategy B/C total | **192 MB** — about half of A, and the `numbers` half never grows with ticket count |

`VACUUM ANALYZE` initially failed on Kubernetes: the pod's default 64 MiB `/dev/shm` is too small
for parallel workers. Fixed with a memory-backed `emptyDir` in `deploy/k8s/03-postgres.yaml`.

## 2. Query plans — `EXPLAIN (ANALYZE, BUFFERS)`, 10M rows, `LIMIT 20`, single client

Execution time in ms:

| Pattern | wildcards | 0 baseline `LIKE` | A expr-indexes | B numbers table | **C = B + prefix range** |
|---|---|---|---|---|---|
| `123456` | 0 | 955 | 163 | 5.5 | **0.06** |
| `123***` | 3 | 2.6 | 19.9 | 3.0 | **0.12** |
| `****23` | 4 | 0.28 | 0.88 | 3.4 | **0.13** |
| `**3***` | 5 | 0.05 | 0.03 | 1.1 | **0.20** |
| `******` | 6 | 0.03 | 0.02 | 0.02 | **0.02** |

- **Baseline** is only slow when matches are rare. With `LIMIT 20` a scan stops as soon as 20 rows
  match, so wide patterns are cheap in every strategy. The hard case is the exact pattern, where the
  scan must read the whole table to find ~10 rows.
- **Strategy A** never gets a full `BitmapAnd`: for `123456` the planner uses *one* digit index
  (10 % selective, 1M rows) and filters the rest. Each digit index is too weak for combining to pay
  off — exactly the risk TECH_SPEC §5.2 anticipated.
- **Strategy B** is stable at 1–5 ms because it only touches the 1M-row `numbers` table plus a few
  `idx_tickets_number` probes.
- **Strategy C** turns fixed leading digits into `n BETWEEN lo AND hi` on `numbers_pkey`
  (index-only) and an exact pattern into `number = X` straight on tickets.

### 2.1 `ORDER BY id` was the real bottleneck and was removed

The spec's query had `ORDER BY id LIMIT 20`. At 10M rows that forces a walk of `tickets_pkey` with
parallel workers, probing `numbers` per row via Memoize: 52k probes and 16–424 ms for `123***`
(`docs/explain-k8s-10M-orderby.txt`). Under 100 VUs it saturated the 2-core Postgres and every
pattern class queued equally. Search results have no meaningful order, so `ORDER BY` was dropped.

## 3. S1 — Latency (H1): 10 → 100 VUs, `/v1/search`

Same dataset and pods for all four runs; only the `STRATEGY` env var changes. Latency in ms.
Evidence: [`s1-C`](evidence/s1-C/), [`s1-B`](evidence/s1-B/), [`s1-A`](evidence/s1-A/), [`s1-0`](evidence/s1-0/).

| Strategy | p95 w0 `dddddd` | p95 w3 `ddd***` | p95 w4 `****dd` | p95 w5 `**d***` | **p95 all** | max | req/s | failed | H1 |
|---|---|---|---|---|---|---|---|---|---|
| **C** numbers + prefix range | 56.3 | 55.9 | 55.5 | 55.6 | **55.8** | 142 | **5,346** | 0 % | **pass** |
| B numbers table | 147.1 | 144.6 | 108.4 | 108.7 | 142.3 | 199 | 926 | 0 % | fail |
| A per-digit expression indexes | 6,718 | 5,162 | 4,859 | 4,860 | 5,972 | 8,151 | 22 | 0 % | fail |
| 0 baseline `LIKE` | 1,053 | 1,021 | 1,023 | 1,021 | 1,022 | 22,550 | 145 | **95.7 %** | fail |

Peak resource use during each run, from the charts in each bundle:

| Strategy | API CPU | API memory | Postgres CPU | Postgres working set | Postgres free |
|---|---|---|---|---|---|
| C | **100 % (at limit)** | 19 MiB | 88 % | 823 MiB | 203 MiB |
| B | 24 % | 17 MiB | **101 % (at limit)** | 745 MiB | 279 MiB |
| A | 1 % | 12 MiB | **101 % (at limit)** | 1,016 MiB | **8 MiB** |
| 0 | 1 % | 9 MiB | **102 % (at limit)** | 456 MiB | 568 MiB |

Reading the table:

- **C is the only strategy that meets H1**, and the only one where the bottleneck moved out of
  Postgres and into the 500m API pod. Per-class p95 is now flat, because every class costs the same
  handful of index probes. Postgres has headroom, so an extra API replica would raise throughput.
- **B** does 5.8× less throughput than C purely because of the per-digit `BitmapAnd` on the
  `numbers` table. The prefix-range trick removes that work for any pattern with fixed leading digits.
- **A** saturates the 2-core Postgres at 22 req/s and comes within **8 MiB** of the 1 GiB memory
  limit, confirming the TECH_SPEC §16 risk that Strategy A's 397 MB of indexes do not fit
  comfortably. `26-db_cpu_throttle.png` shows the cgroup being denied up to **182 % of its limit**
  in CPU time per second, which is the direct mechanism behind the 5,972 ms p95.
- **Baseline** did not merely get slow, it went *down*: the 10-connection pool filled with ~1 s full
  scans, `/readyz` timed out, Kubernetes pulled the pod from the Service, and 19,134 of 19,990
  requests failed — 19,127 of them with a bare `EOF`, the connection dropped mid-request
  (95.7 % failed, `max=22.5 s`). That is correct behaviour for an overloaded
  backend, and it is why liveness must not depend on the database while readiness should.

## 4. S2 — Contention (H2): 200 VUs, one pattern, `/v1/allocate limit=1`, 60 s

Strategy C, ledger truncated and all tickets released before each run.

| | `****23` ≈ 100k tickets ([`s2`](evidence/s2/)) | `**3***` ≈ 1M tickets ([`s2b`](evidence/s2b/)) |
|---|---|---|
| Allocations in 60 s | 95,804 (1,536/s) | **255,677 (4,258/s)** |
| HTTP errors | 0 | 0 |
| Ledger rows = reserved = distinct ids | 95,804 | 255,677 |
| **Duplicate ticket ids (TECH_SPEC §9.2)** | **0** | **0** |
| Ledger vs ticket-state mismatches | 0 | 0 |
| Latency p95 / max | 548 ms / 2.5 s | **74 ms** / 467 ms |
| Peak API / Postgres CPU | 98 % / 101 % | 98 % / 86 % |

**H2 passes in both regimes**: 351,481 allocations under 200-way contention on a single pattern, and
not one ticket reached two users. The latency difference between the two columns is the whole story:
`****23` was **96 % drained** in 60 seconds, so late in the run most random probes hit already-taken
numbers and fall back to a pattern scan over an index full of dead entries. `**3***` cannot be
exhausted in 60 s and comfortably meets the p95 target. This is a near-exhaustion effect, not a
concurrency defect, and in a real product it is where a "this pattern is almost gone" cut-off belongs.

### 4.1 Why allocation does not use `ORDER BY id … LIMIT` either

The spec's allocate statement has a "hot head": every reserved row stays at the front of the id
order, so each new allocation walks past all previous ones. Measured after only 713 reservations:
68,453 rows walked, 145 ms per allocate, growing with every allocation.

`repository.Allocate` instead picks a **random concrete number inside the pattern** (`****23` →
e.g. `481723`) and does a point lookup on `idx_tickets_number`, retrying up to 3 times before
falling back to the pattern scan. Concurrent users spread over 10^wildcards numbers, so `SKIP
LOCKED` rarely has anything to skip and the cost is O(1) regardless of how much has already been
allocated. `SKIP LOCKED` remains the correctness guarantee on both paths; the random probe is purely
a performance measure.

## 5. S3 — Soak: 50 VUs × 10 min, search + allocate + release ([`s3`](evidence/s3/))

| Metric | Value |
|---|---|
| Requests | **743,994** in 10 min (1,240 req/s) |
| HTTP errors | **0** |
| Latency | p95 148 ms, max 6.8 s |
| API pod memory | peak **27 MiB** of a 128 MiB limit (`GOMEMLIMIT` 100 MiB) |
| API pod CPU | peak 52 % of 500m |
| Postgres | peak 80 % CPU, 1,012 MiB working set |
| API restarts / OOMKilled | **0 / none** |
| Ledger rows | 1,229,667 over 1,089,126 distinct tickets |
| Re-allocated after release | 140,541 |
| Inconsistent ticket rows / orphan ledger rows | **0 / 0** |

**Stability passes**: no OOM kill, no restart, and memory flat at roughly a fifth of the limit.

Note on the duplicate check: S3 is the only scenario that calls `/v1/release`, so a freed ticket
being handed out again is correct behaviour and the append-only ledger legitimately holds two rows
for it. The 140,541 repeats are re-allocations, not H2 violations, and `scripts/capture.sh` runs a
different verification for this scenario — state consistency and orphan checks, both zero. H2 itself
is proven by §4, where nothing is ever released.

The p95 misses the 100 ms target because the soak is write-heavy: two of every three requests mutate
rows, so Postgres is doing roughly 800 write transactions per second with WAL fsync. The API pod has
ample headroom; the resource that would need to grow first is Postgres.

## 6. Concurrency tests (`go test`, real PostgreSQL)

`internal/repository/repository_test.go`, run against all four strategies:

- 50 goroutines × `Allocate("****23", limit 5)` against 120 matching tickets, so demand of 250
  exceeds supply: every ticket handed out exactly once, ledger duplicate query returns 0 rows.
- An exhausted pool returns an empty result, not an error (TECH_SPEC §8.1.2).
- A stale reservation is freed by the TTL sweep; explicit release only works for the owner.

## 7. Answers to the challenge

| Challenge question | Answer |
|---|---|
| Database / storage | A single PostgreSQL 16. Row locking with `SKIP LOCKED` makes allocation atomic without Redis or a distributed lock, 10M rows plus indexes fit in about 1 GB, and there are no extra components to operate. |
| Algorithm / indexing for wildcards | The search space is the 10⁶ possible numbers, not the 10⁷ tickets. Resolve the pattern against a 1M-row `numbers(n, d1..d6)` table — a primary-key range for a fixed prefix, per-digit B-trees for the rest — then join into `tickets(number)`. Per-digit indexes on the ticket table itself are 4× larger and the planner cannot combine them profitably (§3). |
| Preventing duplicate results | One statement: `SELECT … FOR UPDATE SKIP LOCKED` → `UPDATE status = 1` → `INSERT INTO allocations`. Proven by 351,481 allocations under 200-VU contention with zero duplicate ticket ids (§4). Reservations expire through a TTL sweep. |
| Performance at 10M+ | Search cost is independent of ticket count, bounded by the 1M `numbers` table plus a couple of index probes; allocation is a point lookup. Growing to 50M or 100M tickets only grows `tickets_pkey` and `idx_tickets_number`. |

## 8. PostgreSQL 17 comparison

Re-measured on PostgreSQL 17.11 against the same 10M-row dataset and the same baseline pod profile
(1 API pod at 500m, database at 2000m/1Gi). Evidence: `docs/evidence/pg16/`, `docs/evidence/pg17/`
and the `pg17-*` bundles.

| Measurement | PG 16.13 | PG 17.11 | Change |
|---|---|---|---|
| Strategy C search, p95 | 55.8 ms | **41.9 ms** | −25 % |
| Strategy C search, throughput | 5,346 req/s | **5,952 req/s** | +11 % |
| Allocation under 200-VU contention, p95 | 73.8 ms | 68.0 ms | −8 % |
| Allocation throughput | 4,258/s | 4,419/s | +4 % |
| Duplicate tickets under contention | 0 | **0** | unchanged |
| Baseline `LIKE` under load | 96 % failed | 96 % failed | unchanged |

**The upgrade is worth taking and changes no code.** Strategy C gains 25 % on latency and 11 % on
throughput for free. Correctness is unaffected: the duplicate-free guarantee rests on
`SKIP LOCKED`, which behaves identically.

**It does not change the architecture.** The strategy ranking is the same, and the baseline still
collapses at 96 % failures under 100 concurrent users. PostgreSQL 17's faster sequential scans made
individual baseline queries up to 83 % quicker, but that does not rescue an index strategy that
cannot serve concurrent load — a useful reminder that single-query benchmarks and load behaviour are
different questions.

One claim in §2 must be qualified: the exact-pattern advantage over the baseline is
17,000× on PG16 and 11,700× on PG17, because 17 speeds the baseline up more than it speeds up
Strategy C. Both figures come from single `EXPLAIN` runs and vary between runs; the honest statement
is "four orders of magnitude on either version".

### 8.1 Strategy D — sending candidate numbers as an array

PostgreSQL 17 can satisfy `= ANY(array)` with a single B-tree descent, which makes a query shape
viable that was not before: have the API enumerate the matching numbers and send them as an
`int[]` rather than resolving them with a subquery. This is **Strategy D**, applied only to patterns
with at most three wildcards (1,000 values, about 4 KB); anything wider falls back to Strategy C.

Measured with a dedicated script whose pattern classes include `dd*d**`, where the fixed digits are
*not* a leading run and Strategy C therefore has no key range to exploit:

| Version | Strategy | p95 | Throughput | API CPU | DB CPU |
|---|---|---|---|---|---|
| PG16 | C | 58.3 ms | 5,023 req/s | 100 % | 72 % |
| PG16 | **D** | 39.1 ms | **6,051 req/s** | 100 % | **37 %** |
| PG17 | C | 38.8 ms | 4,982 req/s | 100 % | 73 % |
| PG17 | **D** | **33.7 ms** | **6,047 req/s** | 100 % | **37 %** |

Three things stand out, and one of them contradicts the prediction made before running it.

- **Strategy D works on PostgreSQL 16 too.** The array form was expected to need 17's optimization to
  pay off. It does not: D halves database CPU on both versions. The benefit comes mostly from
  avoiding the join against `numbers`, not from the new index scan.
- **Upgrading to 17 buys the same latency as writing Strategy D.** Strategy C on PG17 (38.8 ms) is
  indistinguishable from Strategy D on PG16 (39.1 ms). For latency alone, the upgrade and the code
  change are interchangeable — and the upgrade carries no code.
- **D's real product is database headroom, not speed.** Its throughput is identical on both versions
  (6,051 and 6,047 req/s) because it is API-bound at 37 % database CPU, and a faster database cannot
  help a workload that is not waiting on the database. That is also why 17 barely moves D.

**Recommendation: take the PostgreSQL 17 upgrade, and leave Strategy D on the shelf.** The capacity
plan puts the database at 39 % of its allocation at 10,000 requests/sec, so buying more database headroom solves a
problem this system does not have, while D adds a second predicate path, a per-request allocation of
up to 1,000 integers, and 4 KB of extra traffic on every narrow query. Strategy D becomes the right
answer only if the database later turns into the bottleneck — at which point this measurement says it
would roughly halve database CPU.

## 9. Caveats

- All measurements come from one docker-desktop node, so the API pod, Postgres and k6 share the same
  physical CPU. Absolute throughput would differ on separate machines; the comparison between
  strategies, which is what the argument rests on, is unaffected.
- `22-db_disk_free.png` tracks the whole docker-desktop VM volume, not a dedicated device, because
  the PVC is host-path backed. Actual database size is recorded in each bundle's `41-run.txt`.
- API CPU and memory series come from metrics-server, which refreshes about every 15 s. Postgres
  series are read from cgroup files at the 5 s sampling interval, so the two have different
  cadences by design.
- A k6 exit code of 99 means a threshold was crossed, not that the run failed. It is the expected
  outcome for strategies A, B and 0 and for S2 and S3; each bundle records the code.
