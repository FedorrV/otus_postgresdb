### Создаем БД 

```sql
postgres=# create database dbwal;
CREATE DATABASE
postgres=# create schema schwal;
CREATE SCHEMA
postgres=# set search_path = schwal, "$user", public;
SET
```

### Меняем значение параметра  checkpoint_timeout

```sql
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 5min
(1 row)

postgres@ubuntu:~$ pg_ctlcluster 16 main reload

postgres=# show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 30s
(1 row)
```

### запускаем тест pgbench

```sql
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/3514A14C
(1 row)

postgres@ubuntu:~$ pgbench -P 60 -T 600 dbwal
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 1028.3 tps, lat 0.972 ms stddev 0.428, 0 failed
progress: 120.0 s, 1056.4 tps, lat 0.946 ms stddev 0.381, 0 failed
progress: 180.0 s, 1042.9 tps, lat 0.958 ms stddev 0.355, 0 failed
progress: 240.0 s, 1038.5 tps, lat 0.962 ms stddev 0.355, 0 failed
progress: 300.0 s, 1042.3 tps, lat 0.959 ms stddev 0.383, 0 failed
progress: 360.0 s, 1033.0 tps, lat 0.968 ms stddev 0.453, 0 failed
progress: 420.0 s, 1032.7 tps, lat 0.968 ms stddev 0.435, 0 failed
progress: 480.0 s, 1038.3 tps, lat 0.963 ms stddev 0.388, 0 failed
progress: 540.0 s, 1045.3 tps, lat 0.956 ms stddev 0.350, 0 failed
progress: 600.0 s, 1044.7 tps, lat 0.957 ms stddev 0.391, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 624154
number of failed transactions: 0 (0.000%)
latency average = 0.961 ms
latency stddev = 0.393 ms
initial connection time = 8.314 ms
tps = 1040.269755 (without initial connection time)
postgres@ubuntu:~$ 

postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/0/447F2D80
(1 row)

postgres=# select '0/447F2D80'::pg_lsn - '0/2413B9F0'::pg_lsn as b; --размер журнальных файлов WAL, сгенерированных Pgbench в байтах, на один checkpoint в среднем 25,94 Mб
     b     
-----------
 543912848
(1 row)

--посмотрим, какого вида чекпоинты были сделаны
postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 32
checkpoints_req       | 0
checkpoint_write_time | 566510
checkpoint_sync_time  | 176
buffers_checkpoint    | 41760
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3991
buffers_backend_fsync | 0
buffers_alloc         | 3985
stats_reset           | 2024-03-24 12:22:08.052878+03
--можно увидеть, что все чекпоинты были инициированы по расписанию.

postgres@ubuntu:~$ less /var/log/postgresql/postgresql-16-main.log
2024-03-24 12:22:27.116 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:22:54.020 MSK [12380] LOG:  checkpoint complete: wrote 1822 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.890 s, sync=0.008 s, total=26.905 s; sync files=16, longest>
2024-03-24 12:22:57.023 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:23:24.021 MSK [12380] LOG:  checkpoint complete: wrote 1921 buffers (11.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.974 s, sync=0.008 s, total=26.998 s; sync files=14, longest>
2024-03-24 12:23:27.023 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:23:54.011 MSK [12380] LOG:  checkpoint complete: wrote 1908 buffers (11.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.977 s, sync=0.004 s, total=26.988 s; sync files=6, longest=>
2024-03-24 12:23:57.011 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:24:24.105 MSK [12380] LOG:  checkpoint complete: wrote 2036 buffers (12.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=27.073 s, sync=0.006 s, total=27.095 s; sync files=14, longest>
2024-03-24 12:24:27.108 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:24:54.099 MSK [12380] LOG:  checkpoint complete: wrote 1912 buffers (11.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.980 s, sync=0.005 s, total=26.991 s; sync files=6, longest=>
2024-03-24 12:24:57.102 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:25:24.098 MSK [12380] LOG:  checkpoint complete: wrote 2037 buffers (12.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.983 s, sync=0.007 s, total=26.997 s; sync files=13, longest>
2024-03-24 12:25:27.100 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:25:54.078 MSK [12380] LOG:  checkpoint complete: wrote 1910 buffers (11.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.969 s, sync=0.004 s, total=26.979 s; sync files=6, longest=>
2024-03-24 12:25:57.080 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:26:24.068 MSK [12380] LOG:  checkpoint complete: wrote 2028 buffers (12.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.973 s, sync=0.008 s, total=26.988 s; sync files=11, longest>
2024-03-24 12:26:27.068 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:26:54.067 MSK [12380] LOG:  checkpoint complete: wrote 1905 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.974 s, sync=0.007 s, total=26.999 s; sync files=6, longest=>
2024-03-24 12:26:57.068 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:27:24.078 MSK [12380] LOG:  checkpoint complete: wrote 2023 buffers (12.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.992 s, sync=0.011 s, total=27.010 s; sync files=11, longest>
2024-03-24 12:27:27.081 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:27:54.075 MSK [12380] LOG:  checkpoint complete: wrote 1904 buffers (11.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.982 s, sync=0.006 s, total=26.994 s; sync files=6, longest=>
2024-03-24 12:27:57.078 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:28:24.091 MSK [12380] LOG:  checkpoint complete: wrote 2042 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.987 s, sync=0.008 s, total=27.014 s; sync files=14, longest>
2024-03-24 12:28:27.094 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:28:54.079 MSK [12380] LOG:  checkpoint complete: wrote 1901 buffers (11.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.971 s, sync=0.006 s, total=26.986 s; sync files=6, longest=>
2024-03-24 12:28:57.080 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:29:24.073 MSK [12380] LOG:  checkpoint complete: wrote 2043 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.980 s, sync=0.008 s, total=26.993 s; sync files=16, longest>
2024-03-24 12:29:27.075 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:29:54.079 MSK [12380] LOG:  checkpoint complete: wrote 1902 buffers (11.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.983 s, sync=0.006 s, total=27.005 s; sync files=6, longest=>
2024-03-24 12:29:57.081 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:30:24.072 MSK [12380] LOG:  checkpoint complete: wrote 2441 buffers (14.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.980 s, sync=0.007 s, total=26.992 s; sync files=11, longest>
2024-03-24 12:30:27.075 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:30:54.056 MSK [12380] LOG:  checkpoint complete: wrote 1906 buffers (11.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.968 s, sync=0.005 s, total=26.981 s; sync files=6, longest=>
2024-03-24 12:30:57.059 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:31:24.043 MSK [12380] LOG:  checkpoint complete: wrote 2049 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.969 s, sync=0.009 s, total=26.984 s; sync files=15, longest>
2024-03-24 12:31:27.045 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:31:54.028 MSK [12380] LOG:  checkpoint complete: wrote 1909 buffers (11.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.005 s, total=26.984 s; sync files=6, longest=>
2024-03-24 12:31:57.030 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:32:24.139 MSK [12380] LOG:  checkpoint complete: wrote 2438 buffers (14.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=27.065 s, sync=0.034 s, total=27.110 s; sync files=13, longest>
2024-03-24 12:32:57.169 MSK [12380] LOG:  checkpoint starting: time
2024-03-24 12:33:24.058 MSK [12380] LOG:  checkpoint complete: wrote 1818 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.868 s, sync=0.014 s, total=26.890 s; sync files=11, longest>
```

### Включаем асинхронный режим записи журнала

```sql
postgres=# set synchronous_commit = off;
SET
postgres@ubuntu:~$ pgbench -P 60 -T 600 dbwal
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 1120.8 tps, lat 0.892 ms stddev 0.382, 0 failed
progress: 120.0 s, 1053.8 tps, lat 0.948 ms stddev 0.422, 0 failed
progress: 180.0 s, 1025.9 tps, lat 0.974 ms stddev 0.497, 0 failed
progress: 240.0 s, 1127.3 tps, lat 0.887 ms stddev 0.357, 0 failed
progress: 300.0 s, 1053.1 tps, lat 0.949 ms stddev 0.399, 0 failed
progress: 360.0 s, 1076.7 tps, lat 0.928 ms stddev 0.372, 0 failed
progress: 420.0 s, 1094.6 tps, lat 0.913 ms stddev 0.410, 0 failed
progress: 480.0 s, 1062.1 tps, lat 0.941 ms stddev 0.418, 0 failed
progress: 540.0 s, 1107.6 tps, lat 0.902 ms stddev 0.303, 0 failed
progress: 600.0 s, 1067.4 tps, lat 0.936 ms stddev 0.441, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 647358
number of failed transactions: 0 (0.000%)
latency average = 0.926 ms
latency stddev = 0.403 ms
initial connection time = 3.460 ms
tps = 1078.935792 (without initial connection time)
postgres@ubuntu:~$ 

-- показатель tps выше, т.к. сессия в данном случае не ждёт, когда журналы будут сброшены на диск
```

### создаем копию кластера с выключенной контрольной суммой страниц

```sql
fedor@ubuntu:~$ echo "initdb_options = '--data-checksums'" | sudo tee /etc/postgresql-common/createcluster.d/initdb-options.conf
initdb_options = '--data-checksums'
fedor@ubuntu:~$ sudo pg_createcluster 16 main2
Creating new PostgreSQL cluster 16/main2 ...
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/main2 --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with this locale configuration:
  provider:    libc
  LC_COLLATE:  en_US.UTF-8
  LC_CTYPE:    en_US.UTF-8
  LC_MESSAGES: en_US.UTF-8
  LC_MONETARY: ru_RU.UTF-8
  LC_NUMERIC:  ru_RU.UTF-8
  LC_TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/16/main2 ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory               Log file
16  main2   5433 down   postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
fedor@ubuntu:~$ sudo pg_ctlcluster 16 main2 start
fedor@ubuntu:~$ 

fedor@ubuntu:~$ sudo -u postgres psql -p 5433
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table checksumtest(t text);
CREATE TABLE
postgres=# insert into checksumtest select x from 
postgres-# generate_seri
postgres-# generate_series(1, 100) x;
INSERT 0 100
postgres=# select pg_relation_filepath('checksumtest') \gx
-[ RECORD 1 ]--------+-------------
pg_relation_filepath | base/5/16388

postgres=# 


postgres@ubuntu:/home/fedor$ pg_ctlcluster 16 main2 stop;
postgres@ubuntu:/home/fedor$ nano /var/lib/postgresql/16/main2/base/5/16388
postgres@ubuntu:/home/fedor$ pg_ctlcluster 16 main2 start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-main2
postgres@ubuntu:/home/fedor$ 



postgres=# select * from checksumtest ;
 t 
---
(0 rows) -- видимо удалил данных в файле, больше, чем нужно

postgres=# show ignore_checksum_failure ;
 ignore_checksum_failure 
-------------------------
 off
(1 row)

postgres=# 
```