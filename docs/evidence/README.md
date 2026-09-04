# Evidence bundles

One folder per load-test run, produced by `make capture-s1|capture-s2|capture-s3` (or
`make capture-all`). The file numbering matches the reference bundle in `k6-test-result-1/`
so the two are directly comparable. Regenerate the charts from a captured run without
re-running the test: `make charts DIR=docs/evidence/<run>`.

| File | What it is |
|---|---|
| `00-pod.txt` | API replica count |
| `01-resource.txt` | API container limits/requests |
| `02-service_cpu.png` `03-service_memory.png` | API utilization, % of limit, green line = request |
| `10-k6.log` | complete k6 console output, including the trailing `running (…)` line |
| `11-k6-summary.json` `12-k6-report.html` | k6 summary, machine- and human-readable |
| `18-db_pod.txt` `19-db_resource.txt` | Postgres replica count and limits/requests |
| `20-db_cpu.png` `25-db_memory.png` | Postgres utilization, % of limit |
| `21-db_mem_free.png` `22-db_disk_free.png` | Postgres headroom, absolute units |
| `23-db_io_read.png` `24-db_io_write.png` | Postgres block IO, MB/s |
| `26-db_cpu_throttle.png` | CFS throttling, % of limit — how much CPU the cgroup was denied |
| `30-verify.txt` | H2 proof: ledger counts and the duplicate query from TECH_SPEC §9.2 |
| `40-samples.csv` | raw counters the charts are drawn from |
| `41-run.txt` | scenario, strategy, k6 version, exit code, restart counts, DB size, caveats |

## How to read the charts

**The two axes have different cadences on purpose.** API series come from the Kubernetes
metrics API, which the kubelet refreshes about every 15 s; repeated server timestamps are
collapsed so each point sits where the measurement actually happened. Postgres series are
read straight from its cgroup files at the full sampling interval. The API pod is never
exec'd into: at 500m CPU with `GOMAXPROCS=1` it is the bottleneck in most scenarios, and
probing it from the inside would change what is being measured.

**Memory free is the working set** (`memory.current` minus reclaimable page cache), not raw
usage. Postgres fills its cgroup with page cache, so raw usage sits near 100 % of the limit
at all times and would read as an imminent OOM kill when the pod has ample headroom.

**Disk free is the docker-desktop VM volume**, not a dedicated device, because the PVC is
host-path backed. It is a sanity check, not a measurement. `41-run.txt` records the actual
database size instead.

**A k6 exit code of 99 means a threshold was crossed**, not that the run failed. It is the
expected outcome for strategies A, B and 0, and for the contention and soak scenarios.
`41-run.txt` records the code.
