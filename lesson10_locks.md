### 1. Сделаем так, чтобы в журнале появлялась информация о блокировках

```sql
postgres=# show log_lock_waits ;
 log_lock_waits 
----------------
 off
(1 row)


postgres=# show deadlock_timeout ;
 deadlock_timeout 
------------------
 1s
(1 row)


postgres=# alter system set deadlock_timeout to '0.2s';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show deadlock_timeout ;
 deadlock_timeout 
------------------
 200ms
(1 row)

postgres=# alter system set log_lock_waits to on;
ALTER SYSTEM
postgres=# show log_lock_waits ;
 log_lock_waits 
----------------
 off
(1 row)

postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show log_lock_waits ;
 log_lock_waits 
----------------
 on
(1 row)
```

```sql
-- СЕССИЯ 1
postgres=# create table test (n integer);
CREATE TABLE
postgres=# insert into test values(2);
INSERT 0 1
postgres=# begin;
BEGIN
postgres=*# update test set n = 33 where n = 2;
UPDATE 1
postgres=*# 
```

```sql
-- СЕССИЯ 2
postgres=# select * from test; 
 n 
---
 2
(1 row)
postgres=# update test set n = 55 where n = 2;
^CCancel request sent
ERROR:  canceling statement due to user request
CONTEXT:  while updating tuple (0,1) in relation "test"
postgres=# select pg_backend_pid();
 pg_backend_pid 
----------------
          36887
(1 row)
```

```sql
-- СМОТРИМ В ЖУРНАЛ
2024-04-09 16:28:11.336 MSK [36887] postgres@postgres LOG:  process 36887 still waiting for ShareLock on transaction 1902451 after 419.728 ms
2024-04-09 16:28:11.336 MSK [36887] postgres@postgres DETAIL:  Process holding the lock: 36881. Wait queue: 36887.
2024-04-09 16:28:11.336 MSK [36887] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2024-04-09 16:28:11.336 MSK [36887] postgres@postgres STATEMENT:  update test set n = 55 where n = 2;
```


### 2. Обновление одной и той же строки тремя сессиями через update

```sql
-- сессия 1
postgres=# begin;
BEGIN
postgres=*# update test
set n = 4
where n = 3;
UPDATE 1
postgres=*# 
```

```sql
-- сессия 2
postgres=# update test
postgres-# set n = 5 where n = 3; 
```

```sql
-- сессия 3
postgres=# update test
set n = 55 where n = 3;
```

```sql
-- блокировки 2й сессии
[ RECORD 1 ]------+------------------------------
locktype           | relation
database           | 5
relation           | 16422
page               | 
tuple              | 
virtualxid         | 
transactionid      | 
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 3/30
pid                | 3034
mode               | RowExclusiveLock
granted            | t
fastpath           | t
waitstart          | 
-[ RECORD 2 ]------+------------------------------
locktype           | virtualxid
database           | 
relation           | 
page               | 
tuple              | 
virtualxid         | 3/30
transactionid      | 
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 3/30
pid                | 3034
mode               | ExclusiveLock
granted            | t
fastpath           | t
waitstart          | 
-[ RECORD 3 ]------+------------------------------
locktype           | transactionid
database           | 
relation           | 
page               | 
tuple              | 
virtualxid         | 
transactionid      | 1902455
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 3/30
pid                | 3034
mode               | ExclusiveLock
granted            | t
fastpath           | f
waitstart          | 
-[ RECORD 4 ]------+------------------------------
locktype           | turple
database           | 
relation           | 
page               | 
tuple              | 
virtualxid         | 
transactionid      | 1902455
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 3/30
pid                | 3034
mode               | ExclusiveLock
granted            | t
fastpath           | f
waitstart          | 
```

```sql
-- блокировки 3й сессии
-[ RECORD 1 ]------+------------------------------
locktype           | relation
database           | 5
relation           | 16422
page               | 
tuple              | 
virtualxid         | 
transactionid      | 
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 5/34
pid                | 3188
mode               | RowExclusiveLock
granted            | t
fastpath           | t
waitstart          | 
-[ RECORD 2 ]------+------------------------------
locktype           | virtualxid
database           | 
relation           | 
page               | 
tuple              | 
virtualxid         | 5/34
transactionid      | 
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 5/34
pid                | 3188
mode               | ExclusiveLock
granted            | t
fastpath           | t
waitstart          | 
-[ RECORD 3 ]------+------------------------------
locktype           | transactionid
database           | 
relation           | 
page               | 
tuple              | 
virtualxid         | 
transactionid      | 1902457
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 5/34
pid                | 3188
mode               | ExclusiveLock
granted            | t
fastpath           | f
waitstart          | 
-[ RECORD 4 ]------+------------------------------
locktype           | turple
database           | 
relation           | 
page               | 
tuple              | 
virtualxid         | 
transactionid      | 1902457
classid            | 
objid              | 
objsubid           | 
virtualtransaction | 5/34
pid                | 3188
mode               | ExclusiveLock
granted            | f
fastpath           | f
waitstart          | 

-- 2я и 3я транзакции имеют блокировки кортежа Turple
```

### 3. Взаимоблокировка 3мя транзакции

```sql
postgres=# insert into test values (1),( 2);
INSERT 0 2
postgres=# 
```

```sql
-- сессия 1
postgres=# begin;
BEGIN
postgres=*# update test
postgres-*# set n = 11 where n = 1;
UPDATE 1
postgres=*# 
```

```sql
-- сессия 2
postgres=# update test
postgres-# set n = 111 where n = 1;
...
```

```sql
postgres=# begin;
BEGIN
postgres=*# update test
postgres-*# set n = 22
postgres-*# where n  = 2;
UPDATE 1
postgres=*# update test
postgres-*# set n = 1111 where n = 1;
...
```

```sql
-- сессия 2
postgres=*# update test
postgres-*# set n = 222 where n = 2;
ERROR:  deadlock detected
DETAIL:  Process 3041 waits for ShareLock on transaction 1902462; blocked by process 3188.
Process 3188 waits for ExclusiveLock on tuple (0,5) of relation 16422 of database 5; blocked by process 3034.
Process 3034 waits for ShareLock on transaction 1902460; blocked by process 3041.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,6) in relation "test"

--session 1 rollback;
--2 and 3 commit;
```
### 4. Взаимоблокировка 2мя транзакции

```sql
-- Предполагаю, что две транзакции могут одновременно заблокировать разные строки и будут ждать, пока каждая не освободит свою соответствующую строку. В итоге СУБД придётся убить одну из сессий.
```

