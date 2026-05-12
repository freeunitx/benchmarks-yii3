# Benchmark Plan (Fair and Reproducible)

## Why this plan exists
Current benchmark conclusions are not fully reliable because:
- During tests, we did not record full host telemetry (`top/atop`) as first-class artifacts.
- `k6` ran on same host as app stack, so load generator competed for CPU/RAM/IO.
- Reference graph context mentions stronger hardware (32 cores, AMD, 96 GB DDR5, NVMe), while local runs were executed on a laptop:
  - 12 vCPU (AMD Ryzen 5 5625U)
  - 8 GB LPDDR4 4266 MT/s
  - NVMe SSD 512 GB (INTEL SSDPEKNU512GZ)

## Goal
Produce a fair FreeUnit vs FrankenPHP Classic vs FrankenPHP Worker comparison with resource context, reproducible scripts, and defensible conclusions.

## Mandatory methodology changes
1. Separate load generator from system under test.
- Run `k6` on a separate host/VM/container.
- If separate host is impossible, isolate with `cpuset` and CPU/memory limits for `k6`.

2. Capture host telemetry for every run.
- CPU: user/system/iowait/idle.
- Memory: used/free, swap used, major page faults.
- Disk: read/write throughput, IO wait, queue depth.
- Per-process: `k6`, `frankenphp`, `php-fpm`, `roadrunner`, `postgres`, `valkey`.
- Operational commands to record during run:
  - `top -bn1 -c | head -30`
  - `free -h`
  - `iostat -x 1 3`
  - `dmesg -T | rg -i 'killed|oom'`

3. Store telemetry with benchmark artifacts.
- Save `top`/`atop` snapshots and parsed CSV in each run directory.
- Keep exact metadata (`BASE_URL`, stages, VU settings, commit SHA, host spec).

4. Use realistic and stress profiles separately.
- Realistic profile: lower peak (e.g. 2k–10k RPS).
- Stress profile: high peak (up to 50k RPS) only for saturation behavior.
- Do not mix conclusions between profiles.

## Execution plan
1. Add `monitor-start`/`monitor-stop` scripts (`top` or `atop`) and integrate into `tools/run-k6-benchmark.sh`.
2. Add host spec snapshot command and persist output into `metadata.env`/`host.txt`.
3. Introduce two benchmark presets in `Makefile`:
- `bench-realistic`
- `bench-stress`
4. Re-run all 3 runtimes under identical presets and environment.
5. Generate unified summary table with:
- throughput (`http_reqs`, req/s)
- reliability (checks pass/fail, error %)
- latency (med/p95/p99)
- system pressure (CPU iowait, swap, OOM/kswapd activity)
- host bottleneck flags:
  - peak `%wa`
  - peak swap used
  - max disk `%util` and `await`
  - OOM yes/no
  - top 3 CPU processes during peak

## Acceptance criteria
- Every run folder contains: `summary.json`, `k6-timeseries.json`, `docker-stats.csv`, `metadata.env`, monitor logs.
- Load generator isolation is documented and reproducible.
- Final comparison report explicitly separates realistic vs stress conclusions.
- No performance claim is made without corresponding resource telemetry.
