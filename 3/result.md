# Задание 1. COLLATION: сортировка

```sql
CREATE TABLE customer_c AS SELECT * FROM customer WITH NO DATA;
CREATE TABLE customer_unicode AS SELECT * FROM customer WITH NO DATA;
INSERT INTO customer_c
 SELECT * FROM customer;
INSERT INTO customer_unicode
 SELECT * FROM customer;
ALTER TABLE customer_c
 ALTER COLUMN c_name SET DATA TYPE varchar(25) COLLATE "C",
 ALTER COLUMN c_address SET DATA TYPE varchar(40) COLLATE "C",
 ALTER COLUMN c_phone SET DATA TYPE char(15) COLLATE "C";
ALTER TABLE customer_unicode
 ALTER COLUMN c_name SET DATA TYPE varchar(25) COLLATE "en_US.utf8",
 ALTER COLUMN c_address SET DATA TYPE varchar(40) COLLATE "en_US.utf8",
 ALTER COLUMN c_phone SET DATA TYPE char(15) COLLATE "en_US.utf8";

CREATE TABLE customer_icu AS SELECT * FROM customer WITH NO DATA;
INSERT INTO customer_icu SELECT * FROM customer;
ALTER TABLE customer_icu
 ALTER COLUMN c_name SET DATA TYPE varchar(25) COLLATE "en-x-icu",
 ALTER COLUMN c_address SET DATA TYPE varchar(40) COLLATE "en-x-icu",
 ALTER COLUMN c_phone SET DATA TYPE char(15) COLLATE "en-x-icu";
```

Измерить время выполнения запросов без использования индексов:

```sql
SELECT * FROM customer_c ORDER BY c_name DESC
SELECT * FROM customer_unicode ORDER BY c_name DESC
SELECT * FROM customer_icu ORDER BY c_name DESC
```

```sql
EXPLAIN ANALYZE SELECT * FROM customer_c ORDER BY c_name DESC;
```

```
"Gather Merge  (cost=242617.87..417641.99 rows=1500102 width=158) (actual time=571.337..1146.380 rows=1800000 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Sort  (cost=241617.84..243492.97 rows=750051 width=158) (actual time=550.626..666.364 rows=600000 loops=3)"
"        Sort Key: c_name COLLATE ""C"" DESC"
"        Sort Method: external merge  Disk: 106592kB"
"        Worker 0:  Sort Method: external merge  Disk: 109776kB"
"        Worker 1:  Sort Method: external merge  Disk: 86672kB"
"        ->  Parallel Seq Scan on customer_c  (cost=0.00..50496.51 rows=750051 width=158) (actual time=0.097..84.754 rows=600000 loops=3)"
"Planning Time: 0.079 ms"
"Execution Time: 1238.560 ms"
```

```sql
EXPLAIN ANALYZE SELECT * FROM customer_unicode ORDER BY c_name DESC;
```

```
"Gather Merge  (cost=242614.89..417632.94 rows=1500050 width=158) (actual time=4087.902..5973.889 rows=1800000 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Sort  (cost=241614.87..243489.93 rows=750025 width=158) (actual time=4001.113..4287.246 rows=600000 loops=3)"
"        Sort Key: c_name COLLATE ""en_US.utf8"" DESC"
"        Sort Method: external merge  Disk: 99840kB"
"        Worker 0:  Sort Method: external merge  Disk: 96728kB"
"        Worker 1:  Sort Method: external merge  Disk: 106480kB"
"        ->  Parallel Seq Scan on customer_unicode  (cost=0.00..50496.25 rows=750025 width=158) (actual time=0.062..84.381 rows=600000 loops=3)"
"Planning Time: 0.087 ms"
"Execution Time: 6072.528 ms"
```

```sql
EXPLAIN ANALYZE SELECT * FROM customer_icu ORDER BY c_name DESC;
```

```
"Gather Merge  (cost=192449.06..250975.48 rows=501620 width=552) (actual time=1368.446..2257.883 rows=1800000 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Sort  (cost=191449.04..192076.06 rows=250810 width=552) (actual time=1351.690..1513.952 rows=600000 loops=3)"
"        Sort Key: c_name COLLATE ""en-x-icu"" DESC"
"        Sort Method: external merge  Disk: 103352kB"
"        Worker 0:  Sort Method: external merge  Disk: 99848kB"
"        Worker 1:  Sort Method: external merge  Disk: 99840kB"
"        ->  Parallel Seq Scan on customer_icu  (cost=0.00..45504.10 rows=250810 width=552) (actual time=0.205..88.205 rows=600000 loops=3)"
"Planning Time: 0.094 ms"
"Execution Time: 2356.389 ms"
```

# Задание 2. Составные типы

Сравните производительность денормализованной таблицы с обычным JOIN по нормализованным таблицам.

```sql
CREATE TYPE simple_lineitem AS (
 product_id integer,
 quantity decimal(15,2),
 price decimal(15,2)
);
CREATE TYPE simple_order_details AS (
 order_date date,
 total_amount decimal(15,2),
 items simple_lineitem[]
);
CREATE TABLE simple_denormalized_orders (
 order_id integer PRIMARY KEY,
 customer_id integer,
 order_info simple_order_details
);
INSERT INTO simple_denormalized_orders
 SELECT
 o.o_orderkey as order_id,
 o.o_custkey as customer_id,
 (
    o.o_orderdate,
    o.o_totalprice,
    ARRAY(
        SELECT (l.l_partkey, l.l_quantity, l.l_extendedprice)::simple_lineitem
        FROM lineitem l
        WHERE l.l_orderkey = o.o_orderkey
    )::simple_lineitem[]
 )::simple_order_details
 FROM orders o;
```

Избавился от LIMIT, предварительно создав индекс:

```sql
CREATE INDEX ON lineitem(l_orderkey);
```

ВОПРОС Можно ли было еще ускориться с помощью индексов?

```sql
Explain ANALYZE SELECT order_id, item.product_id, item.quantity
FROM simple_denormalized_orders,
     unnest((order_info).items) AS item
WHERE item.quantity > 30;
```

```
"Nested Loop  (cost=0.00..3603630.51 rows=53990682 width=26) (actual time=68.768..29632.646 rows=28796266 loops=1)"
"  ->  Seq Scan on simple_denormalized_orders  (cost=0.00..814111.94 rows=17996894 width=252) (actual time=0.063..5112.781 rows=18000000 loops=1)"
"  ->  Function Scan on unnest item  (cost=0.00..0.13 rows=3 width=22) (actual time=0.001..0.001 rows=2 loops=18000000)"
"        Filter: (quantity > '30'::numeric)"
"        Rows Removed by Filter: 2"
"Planning Time: 0.185 ms"
"JIT:"
"  Functions: 6"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.836 ms (Deform 0.241 ms), Inlining 14.007 ms, Optimization 32.303 ms, Emission 22.333 ms, Total 69.479 ms"
"Execution Time: 30622.790 ms"
```

```sql
EXPLAIN ANALYZE SELECT o.o_orderkey, l.l_partkey, l.l_quantity
FROM orders o
JOIN lineitem l ON l.l_orderkey = o.o_orderkey
WHERE l.l_quantity > 30;
```

```
"Hash Join  (cost=788809.00..4410654.59 rows=29074774 width=13) (actual time=12904.872..74460.832 rows=28796266 loops=1)"
"  Hash Cond: (l.l_orderkey = o.o_orderkey)"
"  ->  Seq Scan on lineitem l  (cost=0.00..2249981.50 rows=29074774 width=13) (actual time=87.769..38167.071 rows=28796266 loops=1)"
"        Filter: (l_quantity > '30'::numeric)"
"        Rows Removed by Filter: 43188811"
"  ->  Hash  (cost=493496.00..493496.00 rows=18000000 width=4) (actual time=12811.452..12811.468 rows=18000000 loops=1)"
"        Buckets: 262144  Batches: 128  Memory Usage: 6981kB"
"        ->  Seq Scan on orders o  (cost=0.00..493496.00 rows=18000000 width=4) (actual time=0.599..8742.063 rows=18000000 loops=1)"
"Planning Time: 0.838 ms"
"JIT:"
"  Functions: 12"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 4.630 ms (Deform 1.711 ms), Inlining 21.177 ms, Optimization 36.258 ms, Emission 30.242 ms, Total 92.307 ms"
"Execution Time: 75529.817 ms"
```

# Задание 3. JSON

Расширил исходный LIMIT до 10000000:

```sql
CREATE TABLE simple_json_orders (
 order_id integer PRIMARY KEY,
 customer_id integer,
 order_info jsonb
 );
 INSERT INTO simple_json_orders
 SELECT
 o.o_orderkey as order_id,
 o.o_custkey as customer_id,
 jsonb_build_object(
    'order_date', o.o_orderdate,
    'total_amount', o.o_totalprice,
    'items', (
        SELECT jsonb_agg(jsonb_build_object(
            'product_id', l.l_partkey,
            'quantity', l.l_quantity,
            'price', l.l_extendedprice
        ))
        FROM lineitem l
        WHERE l.l_orderkey = o.o_orderkey
    )
 )
 FROM orders o
 WHERE o.o_orderkey IN (SELECT l_orderkey FROM lineitem LIMIT 10000000);
```

```sql
SELECT
 order_id,
 jsonb_path_query_first(item, '$.product_id')::int AS 
product_id,
 jsonb_path_query_first(item, '$.quantity')::numeric 
AS quantity
 FROM simple_json_orders,
 jsonb_array_elements(order_info->'items') AS item
 WHERE order_info @? '$.items[*] ? (@.quantity == 1)';
```

```
"Nested Loop  (cost=0.01..709325.94 rows=17677900 width=40) (actual time=99.333..3589.817 rows=954827 loops=1)"
"  ->  Seq Scan on simple_json_orders  (cost=0.00..178988.94 rows=176779 width=439) (actual time=99.281..2642.763 rows=192170 loops=1)"
"        Filter: (order_info @? '$.""items""[*]?(@.""quantity"" == 1)'::jsonpath)"
"        Rows Removed by Filter: 2307985"
"  ->  Function Scan on jsonb_array_elements item  (cost=0.01..1.00 rows=100 width=32) (actual time=0.001..0.001 rows=5 loops=192170)"
"Planning Time: 0.251 ms"
"JIT:"
"  Functions: 6"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.993 ms (Deform 0.306 ms), Inlining 25.435 ms, Optimization 46.935 ms, Emission 26.784 ms, Total 100.147 ms"
"Execution Time: 3630.107 ms"
```

Ускорьте работу запроса с помощью GIN индекса.

Пробовал такой индекс:

```sql
CREATE INDEX idx_partial_jsonpath_quantity_13
ON simple_json_orders
USING GIN (order_info)
WHERE order_info @? '$.items[*] ? (@.quantity == 1)';
```
но результата не получил:

```
"Nested Loop  (cost=0.01..936611.94 rows=25254100 width=40) (actual time=96.410..3635.341 rows=954827 loops=1)"
"  ->  Seq Scan on simple_json_orders  (cost=0.00..178988.94 rows=252541 width=440) (actual time=96.353..2681.756 rows=192170 loops=1)"
"        Filter: (order_info @? '$.""items""[*]?(@.""quantity"" == 1)'::jsonpath)"
"        Rows Removed by Filter: 2307985"
"  ->  Function Scan on jsonb_array_elements item  (cost=0.01..1.00 rows=100 width=32) (actual time=0.001..0.001 rows=5 loops=192170)"
"Planning Time: 0.886 ms"
"JIT:"
"  Functions: 6"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 1.125 ms (Deform 0.396 ms), Inlining 21.232 ms, Optimization 44.222 ms, Emission 30.813 ms, Total 97.391 ms"
"Execution Time: 3677.304 ms"
```

ВОПРОС Как будет выглядеть правильный индекс?

# Задание 4. VIEW

```sql
CREATE TABLE simple_json_orders (
 order_id integer PRIMARY KEY,
 customer_id integer,
 order_info jsonb
 );
 INSERT INTO simple_json_orders
 SELECT
 o.o_orderkey as order_id,
 o.o_custkey as customer_id,
 jsonb_build_object(
    'order_date', o.o_orderdate,
    'total_amount', o.o_totalprice,
    'items', (
        SELECT jsonb_agg(jsonb_build_object(
            'product_id', l.l_partkey,
            'quantity', l.l_quantity,
            'price', l.l_extendedprice
        ))
        FROM lineitem l
        WHERE l.l_orderkey = o.o_orderkey
    )
 )
 FROM orders o
 WHERE o.o_orderkey IN (SELECT l_orderkey FROM lineitem LIMIT 1000000);
```

Ускорьте работу запроса с помощью материализованного представления. Пользователь может менять 1 на другое число.

```sql
SELECT
 order_id,
 jsonb_path_query_first(item, '$.product_id')::int AS 
product_id,
 jsonb_path_query_first(item, '$.quantity')::numeric 
AS quantity
 FROM simple_json_orders,
 jsonb_array_elements(order_info->'items') AS item
 WHERE order_info @? '$.items[*] ? (@.quantity == 1)';
```

Время выполнения запроса у нас сохраняется с предыдущего задания. Создаем materialized view с денормализованными данными:

```sql
CREATE MATERIALIZED VIEW mv_simple_json_orders_items AS
SELECT
  o.order_id,
  (item->>'product_id')::int AS product_id,
  (item->>'quantity')::numeric AS quantity
FROM simple_json_orders o,
  jsonb_array_elements(o.order_info->'items') AS item;

SELECT *
FROM mv_simple_json_orders_items
WHERE quantity = 1;
```

```
"Gather  (cost=1000.00..126290.64 rows=191317 width=13) (actual time=4.977..423.042 rows=200098 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Parallel Seq Scan on mv_simple_json_orders_items  (cost=0.00..106158.94 rows=79715 width=13) (actual time=4.409..381.941 rows=66699 loops=3)"
"        Filter: (quantity = '1'::numeric)"
"        Rows Removed by Filter: 3266635"
"Planning Time: 0.123 ms"
"JIT:"
"  Functions: 6"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 1.584 ms (Deform 0.617 ms), Inlining 0.000 ms, Optimization 1.101 ms, Emission 11.655 ms, Total 14.339 ms"
"Execution Time: 431.900 ms"
```
