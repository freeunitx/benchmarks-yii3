# SUMMARY: FreeUnit vs FrankenPHP classic vs FrankenPHP worker

## Источники
- `README.Franken-classic.ru.md`
- `frankenphp-worker:README.Franken.worker.md`
- `runtime/benchmarks/*/summary.json`

## Профиль нагрузки (одинаковый)
- `bench-ramp.js`
- Peak rate: `50000`
- Stages: `5000 -> 10000 -> 15000 -> 20000 -> 25000 -> 30000 -> 40000 -> 50000` (по `30s`)
- Auto VU: `PREALLOCATED_VUS=10000`, `MAX_VUS=100000`

## Тестовый хост
- Ноутбук: `12 vCPU`, `AMD Ryzen 5 5625U`
- RAM: `8 ГБ LPDDR4` (`4266 MT/s`)
- Диск: `M.2 NVMe 512 ГБ` (`INTEL SSDPEKNU512GZ`)

## Сводная таблица (3 теста)

| Runtime | Benchmark dir | http_reqs | req/s | checks OK | checks fail | latency med | latency p95 | latency p99 | dropped_iterations | vus_max |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| FreeUnit | `runtime/benchmarks/20260511T212150Z-FreeUnit-home-bench-ramp` | 229,627 | 915.50 | 27.99% | 72.01% | 4.88s | 27.63s | 60.06s | 1,860,596 | 21,455 |
| FrankenPHP classic | `runtime/benchmarks/20260512T111803Z-Franken-classic-home-bench-ramp` | 285,231 | 1,099.35 | 23.87% | 76.13% | 5.15s | 33.49s | 64.24s | 2,039,269 | 20,726 |
| FrankenPHP worker | `runtime/benchmarks/20260512T201411Z-FrankenPHP-worker-home-bench-ramp` | 367,508 | 1,510.92 | 100.00% | 0.00% | 1.25s | 13.93s | 18.55s | 2,018,814 | 12,878 |

## Краткий вывод
1. По пропускной способности лидер — **FrankenPHP worker** (`1510.92 req/s`), затем classic, затем FreeUnit.
2. По latency (`med/p95/p99`) лучший результат также у **FrankenPHP worker**.
3. `dropped_iterations` высокие у всех трех запусков; у worker меньше, чем у classic, но выше, чем у FreeUnit.
4. Для classic/freeunit доля failed checks высокая (72–76%), у выбранного worker-run — 0%.

## Важно
- Worker результаты нестабильны между прогонами (см. второй worker run `20260512T202614Z...` в `README.Franken.worker.md`).
- Для корректного финального сравнения нужен повтор всех 3 runtime на "safe" профиле (меньший peak/VU), иначе хост уходит в swap pressure.

## Нагрузка на сервер в режиме worker (наблюдение из `top`)
Снимок во время worker benchmark:
- `load average`: `20.79, 17.24, 14.58`
- CPU: `us 8.4%`, `sy 6.0%`, `id 10.8%`, `wa 74.1%`
- RAM: `7331.1 MiB total`, `6805.4 MiB used`, `348.5 MiB free`
- Swap: `15258.0 MiB total`, `13085.6 MiB used`, `2172.3 MiB free`

Топ процессов:
- `k6`: `77.4% CPU`, `1.8 GiB RES`
- `frankenphp`: `34.2% CPU`, `319 MiB RES`
- `docker-proxy`: `21.3% CPU`, `181 MiB RES`
- `kswapd0`: активен (`16.3% CPU`, state `D`)

Вывод по нагрузке:
1. Worker-профиль в текущем ramp (`peak 50k`) утилизирует практически все доступные ресурсы хоста.
2. Узкое место в этом прогоне — I/O и swap pressure (очень высокий `wa`, активный `kswapd0`), а не чисто CPU.
3. Результаты worker на таком профиле зависят от состояния хоста и могут деградировать между прогонами.
