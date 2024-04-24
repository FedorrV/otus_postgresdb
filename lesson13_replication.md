### Создаем необходимую инфаструкту на 1, 2 и 3 ВМ

```sql
--ВМ1
postgres=# create user rep with password 'rep';
CREATE ROLE
postgres=# create database rep;
CREATE DATABASE
postgres=# alter user rep with replication;
ALTER ROLE
rep=# alter user rep with superuser ;
ALTER ROLE
rep=# alter schema public owner to rep;
ALTER SCHEMA
rep=# grant create on schema public to rep;
GRANT
rep=> create table users (id int, name text);
CREATE TABLE
rep=> create table students (id int, name text);
CREATE TABLE
rep=> insert
into
users
select id, id || '_name' from
generate_series(1,50) as id;
INSERT 0 50
rep=> create publication test_rep for table users;
WARNING: wal_level is insufficient to publish logical changes
HINT: The wal_level network to "logical" before creating subscriptions.
CREATE PUBLICATION
rep=> alter system set wal_level THEN logical;
ALTER SYSTEM
rep=# create subscription test_sub
connection 'host=192.168.1.12 port=5432 user=rep1 password=rep1 dbname=rep1'
publication test_rep1 with (copy_data = true);
NOTICE: created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```

```sql
--ВМ2
postgres=# create user rep1;
CREATE ROLE
postgres=# alter user rep1 with replication ;
ALTER ROLE
postgres=# alter user rep1 with superuser;
ALTER ROLE
postgres=# create database rep1;
CREATE DATABASE
postgres=# \c rep1;
You are now connected to database
"rep1" as user "postgres".
rep1=# alter database rep1 owner to rep1;
ALTER DATABASE
rep1=# alter schema public owner to rep1;
ALTER SCHEMA
rep1=> alter system set wal_level THEN logical;
ALTER SYSTEM
rep1=# create table users(id int, name text);
CREATE TABLE
rep1=> create table students (id int, name text);
CREATE TABLE
rep1=> insert into students
select id, id || '_name' from
generate_series(1,50) as id;
INSERT 0 50
rep1=# lter user rep1 with password 'rep1';
ALTER ROLE
rep1=# create subscription test_sub1
connection 'host=192.168.1.11 port=5432 user=rep password=rep dbname=rep'
publication test_rep with (copy_data = true);
NOTICE: created replication slot "test_sub1" on publisher
CREATE SUBSCRIPTION
rep1=> create publication test_rep1 for table students;
CREATE PUBLICATION
```

```sql
--ВМ3
postgres=# create user rep2;
CREATE ROLE
postgres=# alter user rep2 with replication ;
ALTER ROLE
postgres=# alter user rep2 with superuser;
ALTER ROLE
postgres=# create database rep2;
CREATE DATABASE
rep2=# alter database rep2 owner to rep2;
ALTER DATABASE
rep2=# alter schema public owner to rep2;
ALTER SCHEMA
rep2=> alter system set wal_level THEN logical;
ALTER SYSTEM
rep2=# create table users(id int, name text);
CREATE TABLE
rep2=> create table students (id int, name text);
CREATE TABLE
rep2=# create subscription test_sub21
connection 'host=192.168.1.11 port=5432 user=rep password=rep dbname=rep'
publication test_rep with (copy_data = true);
NOTICE: created replication slot "test_sub21" on publisher
CREATE SUBSCRIPTION
rep2=# create subscription test_sub22
connection 'host=192.168.1.12 port=5432 user=rep1 password=rep1 dbname=rep1'
publication test_rep1 with (copy_data = true);
NOTICE: created replication slot "test_sub22" on publisher
CREATE SUBSCRIPTION
```

#### Пример вывода информации о работе репликации

![alt text](image-2.png)

#### P.S. делал работу на ВМ в VirtualBox. Буфер обмена упорно не работал. Поэтому делал скриншоты и с помощью программы переводил их в текст, возможны опечатки)