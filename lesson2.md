# SQL и реляционные СУБД. Введение в PostgreSQL 

### Создаем таблицу в 1й сессии, наполняем её данными

```sql
create table persons(id serial, first_name text, second_name text); 
commit;
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```

### Проверяем наполнение таблицы во второй сессии

```sql
--по умолчанию стоит уровень изоляции транзакций "read committed", поэтому 0 строк
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
(0 rows)
```

### Такой же пример с уровнем изоляции repeatable read

```sql
--Создаём две новые сессии с уровнем изоляции repeatable read
--1я сессия
set transaction isolation level repeatable read;
--2я сессия
set transaction isolation level repeatable read;

--1я сессия
insert into persons(first_name, second_name) values('sveta', 'svetova');
--2я сессия - набор строк не меняется, т.к. PG не поддерживает грязные чтения
postgres=*# select * from persons
;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)


--1я сессия
commit; --завершаем транзакцию
--2я сессия
postgres=*# select * from persons
;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows) --набор строк снова не поменялся, т.к. PG в режиме repeatable read не допускает неповторяющиеся чтения


--2я сессия фиксирует изменения
postgres=*# commit;
COMMIT
postgres=# select * from persons
;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
  --т.к. транзакция завершилась, включился режим read committed, и уже зафиксированные изменения отразились в запросе
```