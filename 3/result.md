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
 FROM orders o
 WHERE o.o_orderkey IN (SELECT l_orderkey FROM lineitem LIMIT 100);
```

Сравните производительность денормализованной таблицы с обычным JOIN по нормализованным таблицам.

```sql
SELECT order_id, item.product_id, item.quantity
FROM simple_denormalized_orders,
     unnest((order_info).items) AS item
WHERE item.quantity > 30;
```

# Задание 3. JSON

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

Ускорьте работу запроса с помощью GIN индекса.

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
