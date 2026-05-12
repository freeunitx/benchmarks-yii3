# FrankenPHP worker benchmark report

## Контекст
Тестовая команда:

```bash
BASE_URL=http://localhost:9992 make bench BENCH_NAME="FrankenPHP worker"
```

Профиль: `bench-ramp.js`, `CAPTURE_METRICS=1`, `PEAK_RATE=50000`, auto VU (`PREALLOCATED_VUS=10000`, `MAX_VUS=100000`).

## Обработанные результаты

| Run (UTC) | Папка | http_reqs | req/s (avg) | fail % | dropped_iterations | latency avg (ms) | p95 (ms) | p99 (ms) | vus_max |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 2026-05-12T20:14:11Z | `runtime/benchmarks/20260512T201411Z-FrankenPHP-worker-home-bench-ramp` | 367,508 | 1,510.92 | 0.0000 | 2,018,814 | 2,812.21 | 13,926.44 | 18,545.09 | 12,878 |
| 2026-05-12T20:26:14Z | `runtime/benchmarks/20260512T202614Z-FrankenPHP-worker-home-bench-ramp` | 343,062 | 614.10 | 1.3540 | 905,681 | 6,925.85 | 3,979.32 | 415,792.16 | 12,597 |

Незавершенный/частичный ран:
- `runtime/benchmarks/20260512T200617Z-FrankenPHP-worker-home-bench-ramp`
- есть `k6-metrics.raw.json` и `k6-timeseries.json`, нет `summary.json`.

## Где смотреть артефакты
В каждом завершенном ране:
- `summary.json` — агрегаты k6.
- `k6-timeseries.json` — компактный time-series.
- `docker-stats.csv` — контейнерные CPU/Memory/IO.
- `metadata.env` — параметры запуска.

## Вывод
1. Профиль с пиком 50k RPS перегружает dev-хост (high I/O wait, swap pressure).
2. Поведение нестабильно между ранами: второй ран деградирует по `req/s`, `fail%`, и имеет экстремальный `p99`.
3. Для воспроизводимого сравнения нужен более мягкий профиль (ниже peak rate / ниже max VUs).

## Рекомендуемый безопасный запуск

```bash
BASE_URL=http://localhost:9992 make bench \
  BENCH_NAME="FrankenPHP worker safe" \
  STAGES='[{"target":1000,"duration":"30s"},{"target":2000,"duration":"30s"},{"target":3000,"duration":"30s"},{"target":5000,"duration":"30s"}]' \
  AUTO_MAX_VUS_LIMIT=20000
```
