### Создаем БД, схему и таблицу.

```sql
postgres=# create database testdb;
CREATE DATABASE
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table testnm.t1(c1 integer);
CREATE TABLE
testdb=# insert into t1 values(1);
ERROR:  relation "t1" does not exist
LINE 1: insert into t1 values(1);
                    ^
testdb=# insert into testnm.t1 values(1);
INSERT 0 1
testdb=# 
```

### Создаем роль. Выдаем ей права на подключение к новой бд. И на чтение в схеме testnm
```sql
testdb=# create role rolero login;
CREATE ROLE
testdb=# grant usage on schema testnm to rolero;
GRANT
testdb=# grant select on all tables in schema testnm to rolero;
GRANT
testdb=# alter role rolero password 'rolero';
ALTER ROLE
```

### Создаём пользователя, выдаем ему ранне созданную роль
```sql
testdb=# create user testread password 'test123';
CREATE ROLE
testdb=# grant rolero to testread;
GRANT ROLE
testdb=#
```

### Пробуем подключиться под пользователем testread
```sql
testdb=> \c testdb testread;
You are now connected to database "testdb" as user "testread".
testdb=> select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
                      ^
-- psql не видит эту таблицу, т.к. в запросе не была указана схема, и схемы testdb нету в параметре search_path

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```

### Создание новых таблиц
```sql
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
-- т.к. после 15й версии в PG убрали роль CREATE для схемы public

testdb=> create table testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
                     ^
-- т.к. мы создали пользователя с правами readonly
-- для создание новых таблиц нужно залогиниться под пользователем postgres или создать новую роль с правами на создание

```