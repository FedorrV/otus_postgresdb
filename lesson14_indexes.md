### Создаем таблицу и наполняем данными

```sql
postgres=# create table orders(
id integer,
code varchar, des text,
is_ready bool)
;
CREATE TABLE
postgres=# insert into orders
select
id,
'random_code_' || substr(md5(random()::text), 1, 30),
'random_text_' || substr(md5(random()::text), 1, 30),
postgres-# case when mod(id, 2) = 0 then true else false end
postgres-# from generate_series(1, 15000) id;
INSERT 0 15000
postgres=#
```
### Тестируем запросы с индексом и без
```sql
postgres=# alter table orders add primary key (id);
ALTER TABLE
postgres=# analyze orders;
ANALYZE
postgres=# explain select * from orders where id = 4444;
                                QUERY PLAN
---------------------------------------------------------------------------
 Index Scan using orders_pkey on orders  (cost=0.29..8.30 rows=1 width=91)
   Index Cond: (id = 4444)
(2 rows)

postgres=# explain analyze select * from orders where id = 4444;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Index Scan using orders_pkey on orders  (cost=0.29..8.30 rows=1 width=91) (actual time=0.035..0.036 rows=1 loops=1)
   Index Cond: (id = 4444)
 Planning Time: 0.260 ms
 Execution Time: 0.053 ms
(4 rows)
--используется сканирование индекса
postgres=#
postgres=# alter table orders drop constraint orders_pkey;
ALTER TABLE
postgres=# \di
Did not find any relations.
postgres=# explain select * from orders where id = 4444;
                       QUERY PLAN
---------------------------------------------------------
 Seq Scan on orders  (cost=0.00..418.50 rows=1 width=91)
   Filter: (id = 4444)
(2 rows)

postgres=# explain analyze select * from orders where id = 4444;
                                            QUERY PLAN
---------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..418.50 rows=1 width=91) (actual time=0.393..1.231 rows=1 loops=1)
   Filter: (id = 4444)
   Rows Removed by Filter: 14999
 Planning Time: 0.056 ms
 Execution Time: 1.249 ms
(5 rows)
--удалил индекс, время увеличилось, т.к. используется сканирование всей таблицы
```

### Функциональный индекс
```sql
postgres=# create index on orders(upper(code));
CREATE INDEX
postgres=# explain analyze select * from orders where upper(code) like upper('%ccf9%')
;
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..456.00 rows=120 width=91) (actual time=2.429..11.796 rows=6 loops=1)
   Filter: (upper((code)::text) ~~ '%CCF9%'::text)
   Rows Removed by Filter: 14994
 Planning Time: 0.066 ms
 Execution Time: 11.810 ms
(5 rows)
--Индекс создал, но не работает, т.к. оператор Like не подхватывает.
--Не подскажите, может ли в каком то случае подхватиться индекс при like?
postgres=#
```

### Полнотекстовый индекс
```sql
postgres=# ALTER TABLE orders ADD COLUMN content_tsvector TSVECTOR GENERATED ALWAYS
AS (to_tsvector('english', des)) STORED;
ALTER TABLE
postgres=# select * from orders;
CREATE INDEX idx_orders_des_tsvector ON orders USING gin
(content_tsvector);
CREATE INDEX
postgres=#

postgres=# EXPLAIN SELECT * from orders WHERE content_tsvector @@
to_tsquery('english', 'ffr')
;
                                       QUERY PLAN
----------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=12.58..195.47 rows=75 width=123)
   Recheck Cond: (content_tsvector @@ '''ffr'''::tsquery)
   ->  Bitmap Index Scan on idx_orders_des_tsvector  (cost=0.00..12.56 rows=75 width=0)
         Index Cond: (content_tsvector @@ '''ffr'''::tsquery)
(4 rows)

postgres=#

```