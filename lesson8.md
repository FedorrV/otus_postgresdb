### Создаем БД запускаем тест

```sql
postgres=# create database vac;
CREATE DATABASE
postgres=# \q
postgres@otuspg:/home/fedor$ pgbench -i vac
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.52 s (drop tables 0.01 s, create tables 0.01 s, client-side generate 0.15 s, vacuum 0.04 s, primary keys 0.31 s).
postgres@otuspg:/home/fedor$
```

### Выполняем pgbench -c8 -P 6 -T 60 -U postgres postgres
```sql
postgres@otuspg:/home/fedor$ pgbench -c8 -P 6 -T 60 -U postgres vac
pgbench (16.0 (Ubuntu 16.0-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 559.7 tps, lat 14.239 ms stddev 16.058, 0 failed
progress: 12.0 s, 426.7 tps, lat 18.753 ms stddev 13.275, 0 failed
progress: 18.0 s, 360.0 tps, lat 22.185 ms stddev 16.650, 0 failed
progress: 24.0 s, 494.2 tps, lat 16.211 ms stddev 11.477, 0 failed
progress: 30.0 s, 397.8 tps, lat 20.117 ms stddev 18.432, 0 failed
progress: 36.0 s, 356.2 tps, lat 22.386 ms stddev 20.271, 0 failed
progress: 42.0 s, 186.5 tps, lat 42.738 ms stddev 19.165, 0 failed
progress: 48.0 s, 376.0 tps, lat 21.408 ms stddev 17.790, 0 failed
progress: 54.0 s, 504.7 tps, lat 15.831 ms stddev 10.047, 0 failed
progress: 60.0 s, 383.5 tps, lat 20.898 ms stddev 15.327, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 24279
number of failed transactions: 0 (0.000%)
latency average = 19.767 ms
latency stddev = 16.716 ms
initial connection time = 16.592 ms
tps = 404.646952 (without initial connection time)
postgres@otuspg:/home/fedor$
```

### Меняем конфигурацию кластера
```sql
postgres=# \c vac
You are now connected to database "vac" as user "postgres".
vac=# alter system set max_connections to 40;
ALTER SYSTEM
vac=# alter system set shared_buffers to '1GB';
ALTER SYSTEM
vac=# alter system set effective_cache_size to '3GB';
alter system set maintenance_work_mem to '512MB';
alter system set checkpoint_completion_target to 0.9;
alter system set wal_buffers to '16MB';
alter system set default_statistics_target to 500;
alter system set random_page_cost to 4;
alter system set effective_io_concurrency to 2;
alter system set work_mem to '6553kB';
alter system set min_wal_size to '4GB';
alter system set max_wal_size to '16GB';
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
vac=# pg_reload_conf();
ERROR:  syntax error at or near "pg_reload_conf"
LINE 1: pg_reload_conf();
        ^
vac=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

vac=#
```

### Запуск теста после перезапуска
```sql
postgres@otuspg:/home/fedor$ pgbench -c8 -P 6 -T 60 -U postgres vac
pgbench (16.0 (Ubuntu 16.0-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 325.5 tps, lat 24.478 ms stddev 20.822, 0 failed
progress: 12.0 s, 497.7 tps, lat 16.071 ms stddev 11.165, 0 failed
progress: 18.0 s, 434.0 tps, lat 18.421 ms stddev 13.393, 0 failed
progress: 24.0 s, 384.7 tps, lat 20.820 ms stddev 13.685, 0 failed
progress: 30.0 s, 433.8 tps, lat 18.396 ms stddev 12.069, 0 failed
progress: 36.0 s, 384.2 tps, lat 20.868 ms stddev 18.133, 0 failed
progress: 42.0 s, 323.0 tps, lat 24.761 ms stddev 15.742, 0 failed
progress: 48.0 s, 522.3 tps, lat 15.314 ms stddev 10.242, 0 failed
progress: 54.0 s, 528.7 tps, lat 15.140 ms stddev 9.946, 0 failed
progress: 60.0 s, 547.5 tps, lat 14.607 ms stddev 9.223, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26296
number of failed transactions: 0 (0.000%)
latency average = 18.250 ms
latency stddev = 13.737 ms
initial connection time = 16.765 ms
tps = 438.271590 (without initial connection time)
postgres@otuspg:/home/fedor$

--Существенных зменений не обнаружил, показатели чуть улучшились
```
### Создаем новую таблицу и наполняем ее данными
```sql
vac=# create table tab(val text);
CREATE TABLE
vac=# insert into tab select array_to_string(array(select string_agg(substring('qwertyuiopasdfghjklzxcvvmn', round(rando
m() * 25)::integer, 1), '') from generate_series(1, 9)), '') from generate_series(1,1000000);
INSERT 0 1000000
vac=# select pg_size_pretty( pg_total_relation_size( 'tab' ));
 pg_size_pretty
----------------
 42 MB
(1 row)
--Существенных зменений не обнаружил, показатели чуть улучшились
vac=#
```





### Обновляем данные
```sql
vac=# update tab
set val = val || '-';
UPDATE 1000000
vac=# update tab
set val = val || '+';
UPDATE 1000000
vac=# update tab
set val = val || '4';
UPDATE 1000000
vac=# update tab
set val = val || '6';
UPDATE 1000000
vac=# update tab
set val = val || '5';
UPDATE 1000000
vac=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'tab';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 tab     |    1000000 |    1999846 |    199 | 2023-11-12 15:35:21.177251+00
(1 row)

vac=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'tab';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 tab     |    1000000 |          0 |      0 | 2023-11-12 15:38:20.690655+00
(1 row)

vac=# select pg_size_pretty( pg_total_relation_size( 'tab' ));
 pg_size_pretty
----------------
 253 MB
(1 row)

--автовакум работал сразу и пока вставлял команду, часть данных уже отчистились, поэтому всего 200к мертвых

vac=# update tab
vac-# set val = val || '&';
UPDATE 1000000
vac=# update tab
set val = val || '#';
UPDATE 1000000
vac=# update tab
set val = val || '@';
UPDATE 1000000
vac=# update tab
set val = val || '!';
UPDATE 1000000
vac=# update tab
set val = val || '$';
UPDATE 1000000
vac=# select pg_size_pretty( pg_total_relation_size( 'tab' ));
 pg_size_pretty
----------------
 299 MB
(1 row)

--размер не увеличился в разы, как при первом update, т.к. использовалось место, освобожденное от мертвых строк
```

### Обновляем таблицу 10 раз и смотрим результаты
```sql
vac=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'tab';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 tab     |    1000000 |   10000000 |    999 | 2023-11-12 15:45:21.185487+00
(1 row)

vac=# select pg_size_pretty( pg_total_relation_size( 'tab' ));
 pg_size_pretty
----------------
 609 MB
(1 row)

vac=#
--размер увеличился примерно в два раза, что соответствует логиге 5 апдейтов + 5 апдейтов = 10 апдейтов

```