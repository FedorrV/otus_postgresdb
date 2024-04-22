### 1. Создаём тестовую бд и таблицу

```sql
postgres=# create database pgbk;
CREATE DATABASE
postgres=# \c pgbk
You are now connected to database "pgbk" as user "postgres".
pgbk=# create table users as
select id, id::text || '_name'
from generate_series(1, 100) as id;
SELECT 100
```

### 2. Делаем бэкап с помощью COPY

```sql
-- создаем директорию для бэкап-файлов
postgres@ubuntu:~/16/main$ mkdir main_backup 

-- делаем бэкап
pgbk=# \copy users to './16/main/main_backup/users_backup.sql' delimiter ';'
COPY 100
pgbk=# drop table users;
DROP TABLE
pgbk=# \dt    
Did not find any relations.
pgbk=# create table new_users (id integer, descip text);
CREATE TABLE
pgbk=# \copy new_users from './16/main/main_backup/users_backup.sql' delimiter ';'
COPY 100
pgbk=# select count(*) from new_users ;
 count 
-------
   100
(1 row)
```

### 3. Бэкап с помощью pg_dump

```sql
--создаём вторую БД и вторую таблицу

pgbk=# create table old_users as select * from new_users ;
SELECT 100
pgbk=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner   
--------+-----------+-------+----------
 public | new_users | table | postgres
 public | old_users | table | postgres
(2 rows)
pgbk=# create database pgbk2;
CREATE DATABASE
```

#### Делаем бэкап в кастомном формате
```bash
postgres@ubuntu:~/16/main/main_backup$ pg_dump -d pgbk -Fc > tables.dump
#восстанавливаем одну из таблиц в новую БД
postgres@ubuntu:~/16/main/main_backup$ pg_restore -d pgbk2 -t new_users -p 5432 tables.dump 
```

```sql
pgbk2=# \conninfo 
You are connected to database "pgbk2" as user "postgres" via socket in "/var/run/postgresql" at port "5432".
pgbk2=# \dt
Did not find any relations.
pgbk2=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner   
--------+-----------+-------+----------
 public | new_users | table | postgres
(1 row)
```