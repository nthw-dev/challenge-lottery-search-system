# Architecture decision records

One record per significant decision in the Lottery Search System design. Each states the forces at
play, what was decided, what was rejected and why, the consequences accepted, and the evidence bundle
under `docs/evidence/` that supports it. To change a decision, add a new ADR that supersedes the old
one; do not edit the design document's prose silently.

| ADR | Decision | Design doc |
|---|---|---|
| [ADR-001](ADR-001.md) | Use a single PostgreSQL instance; no cache, no lock service | §6.1 |
| [ADR-002](ADR-002.md) | Resolve patterns on a 1M-row `numbers` table, then join (Strategy C) | §5.3, §6.2 |
| [ADR-003](ADR-003.md) | Guarantee duplicate-free allocation with `FOR UPDATE SKIP LOCKED` in one statement | §5.4 |
| [ADR-004](ADR-004.md) | Allocate by random concrete-number probe before falling back to a pattern scan | §5.4 |
| [ADR-005](ADR-005.md) | No `ORDER BY` on search | §5.3 |
| [ADR-006](ADR-006.md) | Liveness must not touch the database; readiness must | §7.3 |
| [ADR-007](ADR-007.md) | Deploy on PostgreSQL 17 | §6.4 |
| [ADR-008](ADR-008.md) | Do not adopt Strategy D | §6.3 |
