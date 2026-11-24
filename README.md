# Intel vs AMD cloud servers: same price, same code

The question: on a cloud server, does picking Intel or AMD at the same price change how fast everyday code runs?

## Setup

- Host: Vultr, Seattle region. Debian 13, kernel 6.12.117.
- Plans, all 2 vCPU and 4 GB RAM:
  - High Performance Intel (`vhp-2c-4gb-intel`), $24 a month: 4 servers
  - High Performance AMD (`vhp-2c-4gb-amd`), $24 a month: 8 servers
  - High Frequency (`vhf-2c-4gb`), $24 a month: 1 server
  - Regular Performance (`vc2-2c-4gb`), $20 a month: 1 server
- Date: 2026-11-24, every job started between 06:16 and 06:28 UTC.
- Versions: Python 3.13.5, pandas 3.1.1, numpy 2.5.5, gcc 14.2.0, zstd 1.5.7. Everything at its defaults.
- The CPU each server got is what `lscpu` printed on it (`results/per_server/*.hardware.txt`).

## The jobs

Each server ran the same 5 jobs on the same input files (made with a fixed seed, so every server reads the same bytes):

| Job | What it does | Cores |
| --- | --- | --- |
| `orderbook` | replays 1,000,000 orders through a pure Python order book, 3 times | 1 |
| `pandas_csv` | reads a 3,000,000 row trades CSV, adds a column, groups by symbol, sorts, writes | 1 |
| `json` | loads 500,000 orders from JSON, adds a field, dumps them back | 1 |
| `compile_sqlite` | `gcc -O2 -c sqlite3.c` (SQLite 3.54.1 amalgamation) | 1 |
| `zstd_2_cores` | `zstd -T2 -12` on the trades CSV | 2 |

3 rounds, the job order rotated each round. The score is seconds, the median of the 3 rounds; the total is the sum of the 5 medians. Lower is faster.

Steal time (CPU time the host gave to other customers while a job ran, read from `/proc/stat`) was logged for every job. It never went above 0.19%, so the differences below come from the chip, not from busy neighbours.

## Results

Total seconds for all 5 jobs, median of 3 rounds:

| Server | Plan | CPU | Total (s) |
| --- | --- | --- | --- |
| amd-c | High Performance AMD | AMD EPYC Genoa | 25.65 |
| amd-a | High Performance AMD | AMD EPYC Genoa | 25.69 |
| hf | High Frequency | Intel Skylake | 31.06 |
| intel-c | High Performance Intel | Intel Xeon Cascade Lake | 39.38 |
| intel-d | High Performance Intel | Intel Xeon Cascade Lake | 39.51 |
| intel-b | High Performance Intel | Intel Xeon Cascade Lake | 39.59 |
| amd-g | High Performance AMD | AMD EPYC Rome | 39.80 |
| amd-e | High Performance AMD | AMD EPYC Rome | 40.51 |
| intel-a | High Performance Intel | Intel Xeon Cascade Lake | 40.97 |
| amd-h | High Performance AMD | AMD EPYC Rome | 41.83 |
| amd-f | High Performance AMD | AMD EPYC Rome | 41.92 |
| amd-b | High Performance AMD | AMD EPYC Rome | 42.02 |
| amd-d | High Performance AMD | AMD EPYC Rome | 43.13 |
| regular | Regular Performance | Intel Xeon Skylake | 66.98 |

Per job medians are in `results/summary_median_of_3.csv`, every single run in `results/all_runs.csv`.

What it shows:

- The 8 AMD servers were the same plan at the same price, but they did not get the same chip. 2 got EPYC Genoa and 6 got EPYC Rome.
- The Genoa servers finished in 25.65 s and 25.69 s. The Rome servers took 39.80 s to 43.13 s. The median Rome server (41.88 s) was 1.63 times slower than the median Genoa server (25.67 s).
- All 4 Intel servers got Xeon Cascade Lake and took 39.38 s to 40.97 s (median 39.55 s), about the same as the Rome servers.
- So on this host, AMD was only faster when the server landed on Genoa. 2 of 8 is what this sample got; it is not a promise of the odds.
- The High Frequency and Regular servers were run in the same window for a later comparison of plan lines.

## Reruns and checks

The first run used 2 Intel and 2 AMD servers. The two AMD servers got different chips, so 6 more AMD and 2 more Intel servers ran the same rig to see how often each chip came up. All 14 are in the tables above; nothing was dropped.

On `amd-d`, the zstd job took 7.26 s and 8.11 s in two rounds and 4.86 s in the third, the widest spread in the data. It is kept as measured.

## Run it yourself

On a fresh Debian 13 server, as root:

```
mkdir -p /root/exp && cp rig/* /root/exp/ && cd /root/exp
sh run.sh
cat STATUS results.csv hardware.txt
```

`run.sh` runs `setup.sh` (installs Python, pandas, gcc, zstd, downloads SQLite and checks its sha3-256, builds the input files), then `bench.sh` (the 5 jobs, 3 rounds). Run `lscpu` first to see which chip you got.
