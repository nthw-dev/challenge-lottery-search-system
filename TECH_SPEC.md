# Tech Spec — PoC: Lottery Search System

| | |
|---|---|
| **Status** | Draft for PoC |
| **Scope** | Proof of Concept only — not a production implementation |
| **Owner** | Senior Backend Developer |
| **Stack** | Go 1.24.7 (darwin/arm64) / Gin / PostgreSQL 17 / Docker Compose / Kubernetes (docker-desktop) / k6 |

---

## 1. Background

The original problem is a **design exercise** that explicitly states no code is required. This document, however, defines the scope of a PoC to be built in order to **produce real numbers that back the design document** — because an architectural proposal supported by actual benchmarks is far more credible than one resting on theory alone.

### 1.1 Key facts about the problem

- A 6-digit lottery number → only **10⁶ = 1,000,000 possible values**
- 10,000,000 tickets → an average of **~10 tickets per number**
- The real search space is **1M, not 10M** — this insight is the foundation of Strategy B in section 5
- Wildcards can appear at any position → a plain B-tree index only helps in the prefix case

### 1.2 Result size varies exponentially

Number of matching values = `10^(count of *)`

| Pattern | `*` | Matching numbers | Matching tickets (approx.) |
|---|---|---|---|
| `123456` | 0 | 1 | ~10 |
| `123***` | 3 | 1,000 | ~10,000 |
| `****23` | 4 | 10,000 | ~100,000 |
| `**3***` | 5 | 100,000 | ~1,000,000 |
| `******` | 6 | 1,000,000 | 10,000,000 |

That is a spread of up to 6 orders of magnitude, so the system must stay fast even for wide patterns.

---

## 2. Objectives / Non-Objectives

### 2.1 Objectives

Prove three hypotheses with measurable numbers:

| # | Hypothesis | How it is proven |
|---|---|---|
| **H1** | Wildcard search at any position achieves p95 < 100 ms over 10M rows under an API pod limited to 500m CPU / 128Mi RAM | k6 + `EXPLAIN (ANALYZE, BUFFERS)` |
| **H2** | Many users searching the same pattern concurrently never receive the same ticket | k6 contention scenario + SQL verification |
| **H3** | The chosen index strategy is significantly faster than a baseline `LIKE`, at an acceptable storage cost | Compare 3 strategies on latency + `pg_relation_size` |

### 2.2 Non-Objectives

The following are **deliberately out of scope** for the PoC — listed here to keep scope from creeping:

- Authentication / Authorization
- Payment flow, prize-draw system, draw-period management
- Horizontal scaling, replication, read replicas
- Caching layer (Redis) — intentionally omitted, to prove a single PostgreSQL instance suffices
- Production observability stack (Prometheus / Grafana / tracing)
- CI/CD pipeline

---

## 3. Architecture Overview

```
┌──────────┐        ┌─────────────────────┐        ┌──────────────────┐
│    k6    │──HTTP──▶│  API Pod (Gin)      │──SQL──▶│  Postgres Pod    │
│ (host)   │        │  500m CPU / 128Mi   │        │  2000m / 1Gi     │
└──────────┘        │  replicas: 1        │        │  StatefulSet+PVC │
                    └─────────────────────┘        └──────────────────┘
                              │                              ▲
                              │                              │
                       kubectl top pod              Job: seed 10M rows
```

**Design principle:** no component beyond what is necessary. Duplicate-result prevention happens at the database layer via `SELECT ... FOR UPDATE SKIP LOCKED`, which is atomic on its own — so no distributed lock and no external cache are needed.

---

## 4. Data Model

```sql
CREATE TABLE tickets (
  id          bigint    PRIMARY KEY,
  number      int       NOT NULL,                 -- 0..999999
  status      smallint  NOT NULL DEFAULT 0,       -- 0=available, 1=reserved, 2=sold
  reserved_by int,
  reserved_at timestamptz
);

-- append-only ledger, specifically for proving H2
CREATE TABLE allocations (
  ticket_id bigint      NOT NULL,
  user_id   int         NOT NULL,
  at        timestamptz NOT NULL DEFAULT now()
);
```

### 4.1 Rationale for the decisions

| Decision | Reason |
|---|---|
| `number` as `int`, not `char(6)` | Saves ~60 MB at 10M rows, compares faster, and digits can be extracted arithmetically without a type conversion |
| No `GENERATED ... STORED` for d1–d6 | It would add ~120 MB to the heap; an **expression index** gives the same effect without growing the table |
| `status` as `smallint`, not `enum`/`text` | Smaller and faster, and sufficient for a PoC |
| Separate `allocations` table | A ledger that can be checked for duplicates directly with a single SQL query — the evidence for H2 |

### 4.2 Seed Strategy

**No Go seeder needed** — a single SQL statement does it:

```sql
INSERT INTO tickets (id, number)
SELECT g, (random() * 999999)::int
FROM generate_series(1, 10000000) g;
```

> **Note:** always create indexes **after** seeding — it is many times faster than inserting into a table that already has indexes. And don't forget `VACUUM ANALYZE tickets;` before measuring, otherwise the planner works from stale statistics and picks the wrong plan.

---

## 5. Index Strategy — three variants must be compared

This section is the core of the *Performance Analysis* chapter in the submitted document. Picking a single approach and claiming it is best without any point of comparison will not satisfy the Performance criterion.

### 5.1 Strategy 0 — Baseline (the comparison point)

```sql
SELECT id, number FROM tickets
WHERE status = 0 AND lpad(number::text, 6, '0') LIKE '____23'
LIMIT 20;
```

- No index is usable → **full Seq Scan**
- Expectation: seconds (2–10 s)
- **Its purpose is to measure how many times over we improved**, not to be a real option

### 5.2 Strategy A — Partial Expression Indexes + BitmapAnd

```sql
CREATE INDEX idx_d6 ON tickets (((number)        % 10)) WHERE status = 0;
CREATE INDEX idx_d5 ON tickets (((number / 10)   % 10)) WHERE status = 0;
CREATE INDEX idx_d4 ON tickets (((number / 100)  % 10)) WHERE status = 0;
CREATE INDEX idx_d3 ON tickets (((number / 1000) % 10)) WHERE status = 0;
CREATE INDEX idx_d2 ON tickets (((number / 10000)  % 10)) WHERE status = 0;
CREATE INDEX idx_d1 ON tickets (((number / 100000) % 10)) WHERE status = 0;
```

The query generates predicates only for the non-wildcard positions and lets the planner perform the BitmapAnd itself.

| Pros | Cons |
|---|---|
| Straightforward query, no join | The indexes together may be **larger than the table itself** (~6 × 160 MB) |
| The partial index (`WHERE status = 0`) shrinks as tickets get reserved over time | A single digit has only 1/10 selectivity — the planner may still choose a Seq Scan |

> **Issue we expect to hit:** for patterns with many wildcards (e.g. `**3***`) the planner will likely reject BitmapAnd because it estimates too many rows will be returned. **If this happens, do not treat it as a failure — it is an excellent finding for the tradeoff discussion.**

### 5.3 Strategy B — Dimension Table (expected winner)

Uses the insight from 1.1: the real search space holds only 1M values.

```sql
CREATE TABLE numbers (
  n  int PRIMARY KEY,          -- 0..999999, only 1M rows
  d1 smallint, d2 smallint, d3 smallint,
  d4 smallint, d5 smallint, d6 smallint
);
-- a per-digit index on this table is only ~20 MB
```

**Flow:** find candidate numbers from the 1M-row table first → join into `tickets(number)`, which has a plain B-tree.

| Pros | Cons |
|---|---|
| Indexes roughly **8× smaller** than Strategy A | Adds one join layer |
| The `numbers` table barely changes → very high cache hit rate, it fits entirely in `shared_buffers` | An extra table must be seeded (but it is a one-line SQL statement) |
| Growing the dataset to 50M/100M does not grow the index size | |

### 5.4 Numbers to collect from this section

- `pg_relation_size` and `pg_indexes_size` for each strategy
- p50 / p95 / p99 **broken down by wildcard count** (0, 3, 4, 5)
- The plan the planner actually chose in each case (Seq Scan / Bitmap / Index Scan)
- Index build time

---

## 6. Concurrency Design

### 6.1 Core mechanism: `FOR UPDATE SKIP LOCKED`

```sql
UPDATE tickets t
SET status = 1, reserved_by = $1, reserved_at = now()
WHERE t.id IN (
  SELECT id FROM tickets
  WHERE status = 0
    AND <pattern predicate>
  ORDER BY id
  LIMIT $2
  FOR UPDATE SKIP LOCKED          -- ← the key
)
RETURNING id, number;
```

`SKIP LOCKED` makes a later transaction **skip** rows that are already locked and take the next ones instead of queueing behind them.

**Results:**
- No lock contention → latency does not spike when users compete
- Atomic by itself → no distributed lock, no Redis
- No deadlocks, because nothing waits

> This is the direct answer to item 3 of the problem statement (Result Distribution), and the main reason PostgreSQL was chosen.

### 6.2 Alternatives considered and rejected

| Alternative | Why it was rejected |
|---|---|
| `SELECT FOR UPDATE` (without SKIP LOCKED) | Every transaction queues up → latency jumps to seconds the moment there is contention |
| Redis distributed lock | Adds a component, adds a failure point, requires managing lock expiry manually, and still requires the DB write anyway |
| Optimistic locking (version column) | Under high contention the retry rate explodes and turns into livelock |
| Application-level in-memory lock | Breaks as soon as you scale to multiple replicas |

### 6.3 Reservation TTL

A background job is required to release stale reservations (users who close the browser and walk away):

```sql
UPDATE tickets SET status = 0, reserved_by = NULL, reserved_at = NULL
WHERE status = 1 AND reserved_at < now() - interval '5 minutes';
```

For the PoC a goroutine plus `time.Ticker` inside the API pod is enough (production should use a separate CronJob).

---

## 7. API Surface

Three endpoints only.

| Method | Path | Purpose | Notes |
|---|---|---|---|
| `POST` | `/v1/search` | Preview only, no reservation | `limit` capped at 100 |
| `POST` | `/v1/allocate` | Actual reservation, using the statement in 6.1 + writing to `allocations` | `limit` capped at 20 |
| `POST` | `/v1/release` | Release a reservation | For TTL testing |
| `GET` | `/healthz` | Readiness/Liveness probe | Required for k8s to behave correctly |

### 7.1 Validation

- The pattern must match `^[0-9*]{6}$` exactly — **reject at the handler, before touching the DB**
- `******` (all wildcards) is allowed, but still capped by `limit` like everything else
- The response must carry an estimated `matched_count`, not a real `COUNT(*)` (a true count over 1M rows is a full scan)

### 7.2 Key principle

> **Do not materialize the whole result set and then trim it.** `**3***` matches a million tickets while the user wants only 20 — the `LIMIT` must be pushed down to the index-scan level so that time-to-first-page stays constant and does not vary with the size of the match set.

---

## 8. Testing Strategy

| Layer | Tool | Scope |
|---|---|---|
| **Unit** | Testify + Mockery | Pattern parser/validator, predicate builder — mock only the single `TicketRepository` interface |
| **Behavior / Integration** | Ginkgo + Gomega | Run against a real PostgreSQL from docker-compose; cover the happy path, edge cases (`******`, `000000`), TTL release |
| **Concurrency** | Ginkgo (parallel specs) | Simulate N goroutines reserving the same pattern; verify no ticket is handed out twice |
| **Load** | k6 | See section 9 |

### 8.1 Required concurrency test cases

1. Two transactions reserve the same pattern simultaneously → each gets a different ticket
2. Reserve until stock runs out → remaining transactions get an empty result, not an error
3. Reserve, then let the TTL expire → the ticket returns to available

---

## 9. Load Test Plan (k6)

| Scenario | Shape | Proves |
|---|---|---|
| **S1 — Latency** | Ramping 10 → 100 VUs, random patterns from 4 classes (0/3/4/5 wildcards), hitting `/v1/search` | H1 |
| **S2 — Contention** | 200 VUs hitting `/v1/allocate` with **the same pattern** `****23` concurrently for 60 seconds | **H2 (most important)** |
| **S3 — Soak** | 50 VUs steady for 10 minutes, watching RSS | Stability / no OOMKill at 128Mi |

### 9.1 Thresholds

```js
thresholds: {
  http_req_duration: ['p(95)<100', 'p(99)<250'],
  http_req_failed:   ['rate<0.01'],
}
```

### 9.2 Verifying H2 after running S2

A single query that **must return 0 rows**:

```sql
SELECT ticket_id, count(*) FROM allocations
GROUP BY ticket_id HAVING count(*) > 1;
```

> Every performance number must be measured **under contention**, not from a single-user run — the real bottleneck usually moves from search to allocation.

---

## 10. Local Environment

### 10.1 Docker Compose (dev loop)

Used while developing milestones 1–4 — iteration is far faster.

- `postgres:17-alpine` + volume
- Seed 1M rows (not 10M) to keep iteration quick
- API runs directly on the host with `go run`

### 10.2 Kubernetes (the real measurement environment)

**Numbers that go into the document must come from k8s only**, because it is the only environment with enforced resource limits.

Required manifests:

| File | Contents |
|---|---|
| `00-namespace.yaml` | namespace `lottery` |
| `01-configmap-pg.yaml` | PostgreSQL tuning |
| `02-secret.yaml` | DB credentials |
| `03-postgres.yaml` | StatefulSet + PVC + Service |
| `04-seed-job.yaml` | Job: seed 10M rows + create indexes + `VACUUM ANALYZE` |
| `05-api.yaml` | Deployment + Service |

### 10.3 Resource Configuration

**API Pod (as specified):**
```yaml
resources:
  limits:   { cpu: 500m,  memory: 128Mi }
  requests: { cpu: 250m,  memory: 64Mi }
```

**Postgres Pod (as specified):**
```yaml
resources:
  limits:   { cpu: 2000m, memory: 1Gi }
  requests: { cpu: 1000m, memory: 512Mi }
```

---

## 11. Resource Budget & Tuning

### 11.1 API Pod — 128Mi is where OOMKills usually happen

| Item | Estimate |
|---|---|
| Go runtime + Gin (idle) | 15–25 MiB |
| `pgxpool` (`max_conns = 10`) | 10–15 MiB |
| Response buffer (cap 100 rows) | < 1 MiB |
| **Headroom left** | **~60 MiB** |

**These two env vars must be set**, otherwise the Go runtime will not know a cgroup limit exists and the pod will be OOMKilled:

```yaml
env:
  - name: GOMEMLIMIT
    value: "100MiB"     # below the limit, so GC runs before the kill
  - name: GOMAXPROCS
    value: "1"          # because the CPU limit is only 500m
```

> If `GOMAXPROCS` is unset, Go reads the **host's** core count (e.g. 10) and creates far too many Ps, causing context switching and heavy CPU throttling.

### 11.2 PostgreSQL Pod — 1Gi

Set via ConfigMap:

```
shared_buffers = 256MB
work_mem = 8MB
effective_cache_size = 768MB
max_connections = 50
random_page_cost = 1.1        # PVC on SSD
shared_preload_libraries = 'pg_stat_statements'
```

### 11.3 Disk

10M rows plus indexes may consume **1.5–2.5 GB** depending on the chosen strategy.
→ Check free space on the docker-desktop VM before starting (`docker system df`).

---

## 12. Observability

| What to measure | Tool |
|---|---|
| Pod CPU / Memory | `kubectl top pod` |
| Query plan and real timing | `EXPLAIN (ANALYZE, BUFFERS)` |
| Slowest queries | `pg_stat_statements` |
| Lock contention | `pg_locks`, `pg_stat_activity` |
| Index size | `pg_relation_size`, `pg_indexes_size` |
| HTTP latency distribution | k6 summary output |

> ⚠️ **metrics-server does not ship with docker-desktop.** Without installing it, `kubectl top pod` will not work and there will be no memory numbers for the document. Install it with the `--kubelet-insecure-tls` flag.

---

## 13. Repository Structure

```
challenge-lottery-search-system/
├── cmd/api/main.go
├── internal/
│   ├── handler/          # Gin handlers (3 endpoints)
│   ├── repository/       # interface + pgx implementation
│   ├── pattern/          # validator + predicate builder
│   └── config/
├── migrations/
│   ├── 001_schema.sql
│   ├── 002_seed.sql
│   ├── 003_index_strategy_a.sql
│   └── 004_index_strategy_b.sql
├── deploy/
│   ├── compose/docker-compose.yml
│   └── k8s/              # 00..05 manifests
├── loadtest/
│   ├── s1_latency.js
│   ├── s2_contention.js
│   └── s3_soak.js
├── docs/
│   ├── TECH_SPEC.md
│   └── DESIGN_DOCUMENT.md    # ← the actual deliverable of the exercise
└── Makefile
```

**Estimated total code: ~350–450 LOC** (excluding tests)

---

## 14. Milestones

| # | Task | Environment |
|---|---|---|
| 1 | Schema + seed **1M rows** | compose |
| 2 | `/v1/search` + Strategy A → measure with `EXPLAIN ANALYZE` | compose |
| 3 | Add Strategy B → compare the numbers | compose |
| 4 | `/v1/allocate` + `SKIP LOCKED` + concurrency tests | compose |
| 5 | Scale up to **10M rows** → deploy to k8s | k8s |
| 6 | Run all 3 k6 scenarios → collect the numbers | k8s |
| 7 | Write `DESIGN_DOCUMENT.md` from the real numbers | — |

> **Finish milestones 1–4 on docker-compose first.** Moving to k8s too early wastes time debugging in the wrong place (you can't tell a query problem from a manifest problem).

---

## 15. Acceptance Criteria

| Criterion | Passes when |
|---|---|
| **H1** | p95 < 100 ms for every pattern class at 10M rows |
| **H2** | The duplicate-check query returns **0 rows** and the k6 error rate is < 1% |
| **H3** | At least **10×** faster than the baseline, with index-size figures to back it up |
| **Stability** | No `OOMKilled` throughout S3 and RSS peak < 110 Mi |
| **Deliverable** | `DESIGN_DOCUMENT.md` covers all 4 sections required by the exercise |

---

## 16. Risks & Open Questions

| Risk | Impact | Mitigation |
|---|---|---|
| The planner rejects BitmapAnd in Strategy A | Strategy A is slower than expected | **Not a failure** — record it as a selectivity finding in the tradeoff section |
| API pod OOMKilled at 128Mi | S3 fails | Set `GOMEMLIMIT`, lower `pgxpool.max_conns`, cap response size |
| CPU throttling at 500m | p99 spikes | Set `GOMAXPROCS=1`, check `container_cpu_cfs_throttled_seconds` |
| Postgres 1Gi is not enough for Strategy A's indexes | High cache miss rate, I/O spikes | Direct supporting evidence for Strategy B |
| Disk fills up during seeding | The Job fails | Check free space beforehand, seed in batches |

### Open Questions

1. Should the `numbers` table in Strategy B be joined via `IN (subquery)` or a `JOIN` — let the planner decide based on real numbers
2. Is a 5-minute TTL appropriate — depends on UX that has not been defined yet
3. Should we test an uneven `status` distribution (80% already reserved) — it would affect the selectivity of the partial indexes

---

## Appendix A — Mapping to the exercise's grading criteria

| Criterion | Sections that answer it |
|---|---|
| **Feasibility** | §3 Architecture, §7 API Surface |
| **Performance** | §5 Index Strategy, §9 Load Test, §11 Resource Budget |
| **Correctness** | §6 Concurrency, §8.1 Concurrency Tests, §9.2 Verification |
| **Real-world practicality** | §6.2 Rejected alternatives, §10–§12 Environment & Observability |
| **Creativity** | §1.1 The 1M search-space insight, §5.3 Strategy B |
