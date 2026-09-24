---
name: benchmark
description: Measure pesu-auth latency, throughput and CSRF token lifetime with the scripts in scripts/benchmark against a running instance, compare a change with a baseline, and report the numbers without overloading PESU Academy. Use for performance work, concurrency changes, or before and after changes to the request path.
---

# Benchmarking

The scripts live in `scripts/benchmark/` and import `util.py` from the same folder, so run them from
there. Results (CSV and plots) go to `benchmark/results/` at the repo root, named
`{script}_{date}_{time}[_tag].{ext}`, and are gitignored.

**Every `/authenticate` request logs in to PESU Academy for real with the `TEST_*` account.** Keep
request counts small (tens, not thousands), keep parallelism modest, and do not run benchmarks while
tests or another benchmark use the account. Routes `health` and `readme` do not touch PESU.

## Baseline and comparison

```bash
# terminal 1: the version under test
uv run python -m app.app

# terminal 2
cd scripts/benchmark
uv run python benchmark_requests.py --num-requests 20 --parallel --max-workers 5 --tag baseline
uv run python benchmark_requests.py --num-requests 20 --no-profile --tag baseline-noprofile
uv run python analyze_benchmark.py -f ../../benchmark/results/benchmark_requests_*_baseline.csv
```

Then switch to your branch, restart the server, and repeat with `--tag change`. Compare success rate,
mean and median latency, and the tail (max). Also read `/metrics?fmt=json` after each run:
`upstream.*.latency.averageSeconds` separates PESU's latency from ours, `csrfCache` shows how often
callers waited for a token, and `httpClients` created minus closed must be back to 1 at rest.

Useful flags (`--help` on each script for all): `--host` (default `http://localhost:5000`),
`--route authenticate|health|readme`, `--timeout`, `--output-dir`, `--output`, `--verbose`.

## CSRF token lifetime

`unauthenticated_csrf_token_expiry.py` tests whether a prefetched token still works after waiting;
it informs `CSRF_TOKEN_REFRESH_INTERVAL_SECONDS` (45 min). It runs for a long time by design; run it
only when changing that interval, and read its `--help` first.

## Reporting

Numbers vary a lot with PESU's own latency and the network (production runs in Singapore). Report
both runs side by side with request counts, parallelism, time of day, and the upstream latency from
`/metrics`, and do not claim an improvement smaller than the run-to-run noise. The PR template has a
"Benchmarks still meet expected performance" box: tick it only if you ran them.
