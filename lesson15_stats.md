### Создаем таблицы и наполняем данными

```sql
postgres=# create database stats;
CREATE DATABASE
postgres=# \c stats
You are now connected to database "stats" as user "postgres".

stats=#  create table subjects (subject_id int primary key, des varchar(100));
CREATE TABLE
stats=#  create table teachers(teacher_id int primary key, fio varchar(100));
CREATE TABLE
stats=# create table lessons (
lesson_id int,
subject_id int ,
teacher_id int,
lesson_date timestamp(0),
constraint fk_les_sub foreign key (subject_id) references subjects(subject_id),
constraint fk_les_teach foreign key (teacher_id) references teachers(teacher_id)
);
CREATE TABLE
stats=#
stats=# insert into subjects values
(1, 'math'),
(2, 'russian'),
(3, 'english'),
(4, 'phisics');
INSERT 0 4
stats=#
stats=# insert into teachers values
stats-# (1, 'FIO1'),
stats-# (2, 'FIO2'),
stats-# (3, 'FIO3'),
stats-# (4, 'FIO4');
INSERT 0 4
stats=#
stats=# insert into lessons
select id,
random()*3 + 1 as sub,
random()*3 + 1 as teacher,
current_timestamp(0)
from generate_series (1, 10000) id;
INSERT 0 10000
```

### Написание запросов

```sql
--ПРЯМОЕ СОЕДИНЕНИЕ
select * from lessons l
stats-# join teachers t on t.teacher_id = l.teacher_id
stats-# join subjects s on s.subject_id = l.subject_id
stats-# where t.teacher_id = 3
stats-# and s.subject_id = 1; 

--ЛЕВОСТОРОННЕЕ СОЕДИНЕНИЕ
select l.* from lessons l
stats-# left join teachers t on t.teacher_id = l.teacher_id
stats-# where t.fio = 'FIO4'; 

--ПРАВОСТОРОННЕЕ СОЕДИНЕНИЕ 
select l.* from lessons l
stats-# right join teachers t on t.teacher_id = l.teacher_id
stats-# where t.fio = 'FIO4'; 

--КРОСС СОЕДИНЕНИЕ, представим, что каждый учитель будет вести каждый предмет
stats=# select * from teachers t
stats-# cross join subjects s;
 teacher_id | fio  | subject_id |   des
------------+------+------------+---------
          1 | FIO1 |          1 | math
          1 | FIO1 |          2 | russian
          1 | FIO1 |          3 | english
          1 | FIO1 |          4 | phisics
          2 | FIO2 |          1 | math
          2 | FIO2 |          2 | russian
          2 | FIO2 |          3 | english
          2 | FIO2 |          4 | phisics
          3 | FIO3 |          1 | math
          3 | FIO3 |          2 | russian
          3 | FIO3 |          3 | english
          3 | FIO3 |          4 | phisics
          4 | FIO4 |          1 | math
          4 | FIO4 |          2 | russian
          4 | FIO4 |          3 | english
          4 | FIO4 |          4 | phisics
(16 rows)

--ПОЛНОЕ СОЕДИНЕНИЕ
stats=# select * from teachers t
full join subjects s on t.teacher_id = s.subject_id;
 teacher_id | fio  | subject_id |   des
------------+------+------------+---------
          1 | FIO1 |          1 | math
          2 | FIO2 |          2 | russian
          3 | FIO3 |          3 | english
          4 | FIO4 |          4 | phisics
stats=#

--СОСТАВНОЕ СОЕДИНЕНИЕ
stats=# select count(*) from lessons l
join teachers t on t.teacher_id = l.teacher_id
left join subjects s on s.subject_id = l.subject_id
where t.teacher_id = 3
;
 count
-------
  3331
(1 row)
```