### 1. Сделал тест производительности на настройках posgtgres по умолчанию

```sql
postgres@ubuntu:~$ pgbench -c 100 -j 2 -P 20 -T 120 -p 5432
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 873.5 tps, lat 112.783 ms stddev 122.994, 0 failed
progress: 40.0 s, 876.7 tps, lat 114.105 ms stddev 141.075, 0 failed
progress: 60.0 s, 889.5 tps, lat 111.969 ms stddev 128.766, 0 failed
progress: 80.0 s, 861.4 tps, lat 115.545 ms stddev 133.433, 0 failed
progress: 100.0 s, 804.4 tps, lat 125.099 ms stddev 150.435, 0 failed
progress: 120.0 s, 824.1 tps, lat 121.357 ms stddev 144.947, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 102691
number of failed transactions: 0 (0.000%)
latency average = 116.798 ms
latency stddev = 137.212 ms
initial connection time = 173.935 ms
tps = 854.711554 (without initial connection time)
```

### 2. Выставил конфигурацию, сгенерированную с помощью https://pgconfigurator.cybertec.at/

```sql
postgres@ubuntu:~$ pgbench -c 100 -j 2 -P 20 -T 120 -p 5433
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 1300.1 tps, lat 75.591 ms stddev 99.735, 0 failed
progress: 40.0 s, 1344.6 tps, lat 74.444 ms stddev 82.317, 0 failed
progress: 60.0 s, 1375.9 tps, lat 72.469 ms stddev 77.052, 0 failed
progress: 80.0 s, 1280.1 tps, lat 78.107 ms stddev 87.649, 0 failed
progress: 100.0 s, 1291.5 tps, lat 77.177 ms stddev 90.437, 0 failed
progress: 120.0 s, 1326.4 tps, lat 75.283 ms stddev 89.333, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 158471
number of failed transactions: 0 (0.000%)
latency average = 75.569 ms
latency stddev = 88.128 ms
initial connection time = 215.804 ms
tps = 1319.900336 (without initial connection time)
postgres@ubuntu:~$ 
```

### 3. Пробую увеличить производительность, изменив параметры

```sql
-- Увеличим количество процессов, в которых вычисления могут идти параллельно

-- max_worker_processes = 8
-- max_parallel_workers = 8

tps = 854.711554 (without initial connection time)
postgres@ubuntu:~$ pgbench -c 100 -j 2 -P 20 -T 120 -p 5433
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 1312.4 tps, lat 75.084 ms stddev 92.944, 0 failed
progress: 40.0 s, 1289.9 tps, lat 77.526 ms stddev 90.056, 0 failed
progress: 60.0 s, 1154.3 tps, lat 86.627 ms stddev 110.052, 0 failed
progress: 80.0 s, 1239.9 tps, lat 80.298 ms stddev 84.827, 0 failed
progress: 100.0 s, 1157.9 tps, lat 86.261 ms stddev 99.179, 0 failed
progress: 120.0 s, 1201.4 tps, lat 83.197 ms stddev 93.858, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 147224
number of failed transactions: 0 (0.000%)
latency average = 81.416 ms
latency stddev = 95.498 ms
initial connection time = 153.088 ms
tps = 1225.015953 (without initial connection time)

-- Параметр huge_pages так же дал прирост производительности

--huge_pages = try


postgres@ubuntu:~$ pgbench -c 100 -j 2 -P 20 -T 120 -p 5433
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 1146.1 tps, lat 85.927 ms stddev 96.946, 0 failed
progress: 40.0 s, 1123.6 tps, lat 88.957 ms stddev 97.437, 0 failed
progress: 60.0 s, 1145.7 tps, lat 87.248 ms stddev 94.863, 0 failed
progress: 80.0 s, 1093.5 tps, lat 91.208 ms stddev 109.593, 0 failed
progress: 100.0 s, 1004.3 tps, lat 99.309 ms stddev 141.387, 0 failed
progress: 120.0 s, 1089.5 tps, lat 91.838 ms stddev 117.141, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 132154
number of failed transactions: 0 (0.000%)
latency average = 90.668 ms
latency stddev = 110.214 ms
initial connection time = 164.312 ms
tps = 1100.029300 (without initial connection time)

```