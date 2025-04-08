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
```

Измерить время выполнения запросов без использования индексов:

```sql
SELECT * FROM customer_c ORDER BY c_name DESC
SELECT * FROM customer_unicode ORDER BY c_name DESC
SELECT * FROM customer_icu ORDER BY c_name DESC
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
