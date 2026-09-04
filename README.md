# lottery-search-system

A design for searching 10M six-digit lottery tickets by a 6-character wildcard
pattern (`****23`), and handing each match to exactly one user at a time.

This is **Part 2** of [backend-challenge](https://github.com/7-solutions/backend-challenge)
— the Lottery Search System section, a design exercise with no code required.
Author: **Natthawat Narin**.

## Start here

**[DESIGN_DOCUMENT.md](DESIGN_DOCUMENT.md)** is the deliverable. It answers the four questions the
challenge asks — storage choice, indexing strategy for wildcard matching, performance analysis, and
the concurrency strategy that prevents duplicate distribution — and every number in it was measured,
not estimated. The same document is also available as a self-contained HTML page in
[English](docs/DESIGN_DOCUMENT.html) and [Thai](docs/DESIGN_DOCUMENT.th.html).

| Goal from the challenge | Result |
|---|---|
| Wildcard search in any position at 10M rows | p95 **55.8 ms** at 5,346 req/s on one 500m-CPU pod, flat across all wildcard classes |
| Same pattern, many concurrent users, no duplicate tickets | **351,481** allocations under 200-way contention, **0** duplicates |
| Chosen index strategy vs a plain `LIKE` baseline | four orders of magnitude on the hardest pattern, with half the index size of the intuitive alternative |
| Stability under sustained mixed load | 743,994 requests over 10 minutes, 0 errors, 0 restarts, API memory peak 27 MiB of 128 MiB |

## What is in this repository

| Path | Contents |
|---|---|
| [`DESIGN_DOCUMENT.md`](DESIGN_DOCUMENT.md) | The design, in the Engineering Design Doc structure: context, goals, overview, detailed design, alternatives considered, cross-cutting concerns, validation, risks, appendix |
| [`adr/`](adr/README.md) | Eight architecture decision records — one per significant decision, each with the alternatives rejected and the evidence bundle that supports it |
| [`docs/RESULTS.md`](docs/RESULTS.md) | Full analysis of every load-test run, including the PostgreSQL 16 vs 17 comparison and the rejected Strategy D |
| [`docs/REPORT.html`](docs/REPORT.html), [`docs/CAPACITY.html`](docs/CAPACITY.html) | The test report and the 10,000 req/s capacity plan, written for a wider audience (Thai) |
| [`docs/evidence/`](docs/evidence/README.md) | One bundle per run: the complete k6 log, summary JSON and HTML, nine resource charts, the raw samples the charts were drawn from, and the SQL verification output |
| [`docs/explain-k8s-10M.txt`](docs/explain-k8s-10M.txt) | `EXPLAIN (ANALYZE, BUFFERS)` for every strategy and pattern class at 10M rows; `-orderby.txt` is the rejected design |
| [`TECH_SPEC.md`](TECH_SPEC.md) | The proof-of-concept specification the measurements were built to (Thai) |
| [`challenge-lottery-search-system.md`](challenge-lottery-search-system.md) | The challenge as received |

Every bundle under `docs/evidence/` records in `41-run.txt` the scenario, strategy, PostgreSQL version,
k6 version and exit code that produced it, so any figure in the design document can be traced to a
log line.

## What is not here

The proof-of-concept source is in a separate repository, in keeping with the challenge's "no code
required": **[github.com/nthw-dev/poc-lottery-search-system](https://github.com/nthw-dev/poc-lottery-search-system)** — the solution architecture,
data structures, algorithms and load tests that produced every measurement cited here. This
repository holds the design and the evidence for it.
