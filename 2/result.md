# Задача 1. Низкая селективность по нескольким условиям

```sql
select count(c_mktsegment) from customer WHERE c_mktsegment = 'FURNITURE' and c_acctbal BETWEEN 4500 and 6700;
```

* **Оцените селективность каждого из условий**

Селективность условия WHERE c_mktsegment = 'FURNITURE': 355478 / 1800496 = 0,197

```sql
explain select * from customer WHERE c_mktsegment = 'FURNITURE'; -- 355478
explain select * from customer; -- 1800496
```

Селективность условия c_acctbal BETWEEN 4500 and 6700: 363175 / 1800496 = 0,202

```sql
explain SELECT * FROM customer WHERE c_acctbal BETWEEN 4500 AND 6700; -- 363175
explain select * from c_acctbal; -- 1800496
```

* **Какое поле лучше поставить первым в индексе?**

c_mktsegment

* **Добейтесь максимальной производительности запрос, используя btree индекс**

```sql
CREATE INDEX idx_customer_c_mktsegment_c_acctbal ON customer (c_mktsegment, c_acctbal); -- время выполнения select составило 15ms
```

<details>

<summary>Вопросы</summary>
Как обычно применяют термин селективность: селективность условия/ селективность колонки / селективность индекса?

Индекс будет работать тем эффективнее, чем ниже селективность условия в том смысле, что условие отбирает меньшую долю строк?

</details>

# Задача 2. Три поля в индексе

```sql
select c_custkey, c_nationkey from customer WHERE c_mktsegment = 'FURNITURE' and c_acctbal > 6000 and c_phone LIKE '1%';
```

* Подберите максимально подходящий индекс для запроса
  * Индекс может содержать как 3 так и меньше столбцов
  * Можно использовать несколько индексов
  * Можно не покрывать все поля индексами

```sql
CREATE INDEX idx_customer_mktsegment_acctbal ON customer (c_mktsegment, c_acctbal);
```

<details>

<summary>Вопросы</summary>

Почему не подошли индексы?

```sql
CREATE INDEX idx_1 ON customer (c_mktsegment, c_acctbal, c_phone); -- LIKE '1%' плохо фильтрует строки?
```

```sql
CREATE INDEX idx_2 ON customer (c_mktsegment, c_acctbal, c_phone) INCLUDE (c_nationkey); -- слишком жирные страницы в комбинации с предыдущим?
```

Причина в c_acctbal::numeric?

</details>

* Какие соображения доступны до проверки скорости работы на практике?
  * Воспользуйтесь сведениями о распределении данных в колонке

```sql
SELECT c_mktsegment, COUNT(*) 
FROM customer 
GROUP BY c_mktsegment;

-- "AUTOMOBILE"	360211
-- "BUILDING"	  360231
-- "FURNITURE"	 359251
-- "HOUSEHOLD"	 359789
-- "MACHINERY"	 360518
```

```sql
WITH acctbal_stats AS (
 SELECT MIN(c_acctbal) AS min_val, MAX(c_acctbal) AS max_val FROM customer
 ),
 histogram AS (
 SELECT
 width_bucket(c_acctbal, min_val, max_val, 10) AS bucket,
 COUNT(*) AS freq
 FROM
 customer,
 acctbal_stats
 GROUP BY bucket ORDER BY bucket
 )
 SELECT
 bucket,
 min_val + (max_val - min_val) * (bucket - 1) / 10 AS bucket_start,
 min_val + (max_val - min_val) * bucket / 10 AS bucket_end,
 freq
 FROM
 histogram,
 acctbal_stats;

-- "bucket"	"bucket_start"	"bucket_end"	"freq"
-- 1	-999.99000000000000000000	100.0080000000000000	180085
-- 2	100.0080000000000000	1200.0060000000000000	180135
-- 3	1200.0060000000000000	2300.0040000000000000	180786
-- 4	2300.0040000000000000	3400.0020000000000000	179754
-- 5	3400.0020000000000000	4500.0000000000000000	180375
-- 6	4500.0000000000000000	5599.9980000000000000	178913
-- 7	5599.9980000000000000	6699.9960000000000000	179950
-- 8	6699.9960000000000000	7799.9940000000000000	179614
-- 9	7799.9940000000000000	8899.9920000000000000	179907
-- 10	8899.9920000000000000	9999.9900000000000000	180478
-- 11	9999.9900000000000000	11099.988000000000	3
```

  * Определите селективность каждого условия (сколько строчек отфильтровывает каждое условие)

```sql
EXPLAIN ANALYZE select * from customer; -- 1800000
EXPLAIN ANALYZE select * from customer WHERE c_mktsegment = 'FURNITURE'; -- 355200
EXPLAIN ANALYZE select * from customer WHERE c_acctbal > 6000; -- 654586
EXPLAIN ANALYZE select * from customer WHERE c_phone LIKE '1%'; -- 727273 
```

В результате выполнения запросов поняли, что данные распределены достаточно равномерно. Также нужно учитывать размеры полей, которые попадают в индекс: жирные поля -- меньше гибкость при добавлении полей в индекс.

# Задача 3. Сортировка по индексу

```sql
select * from customer WHERE c_mktsegment = 'FURNITURE' and c_acctbal BETWEEN 4500 and 6700 ORDER BY c_acctbal;
```

* Можно ли использовать индексы, которые вы уже создали в этой практике для ускорения запроса с сортировкой?

Да, индекс:

```sql
CREATE INDEX idx_customer_mktsegment_acctbal ON customer (c_mktsegment, c_acctbal);
```

подойдет. Сортировка с помощью индекса (чтение строк из индекса в сортированном виде) будет выполнена, если мы будем выбирать только те поля, которые добавляли в индекс:

```sql
explain analyze select c_mktsegment,c_acctbal from customer WHERE c_mktsegment = 'FURNITURE' and c_acctbal BETWEEN 4500 and 6700 ORDER BY c_acctbal;
```

```
"Index Only Scan using idx_customer_mktsegment_acctbal on customer  (cost=0.43..3142.26 rows=70633 width=17) (actual time=0.040..10.266 rows=71545 loops=1)"
"  Index Cond: ((c_mktsegment = 'FURNITURE'::bpchar) AND (c_acctbal >= '4500'::numeric) AND (c_acctbal <= '6700'::numeric))"
"  Heap Fetches: 0"
"Planning Time: 0.130 ms"
"Execution Time: 12.449 ms"
```

Иначе с помощью индекса будет выполнен поиск, а сортировка в памяти после прохода по Bitmap. Это подтверждается результатом выполнения:

```sql
explain analyze select * from customer WHERE c_mktsegment = 'FURNITURE' and c_acctbal BETWEEN 4500 and 6700 ORDER BY c_acctbal;
```

```
"QUERY PLAN"
"Sort  (cost=59594.17..59770.75 rows=70633 width=158) (actual time=937.796..958.102 rows=71545 loops=1)"
"  Sort Key: c_acctbal"
"  Sort Method: external merge  Disk: 12040kB"
"  ->  Bitmap Heap Scan on customer  (cost=2301.00..48350.87 rows=70633 width=158) (actual time=16.080..185.468 rows=71545 loops=1)"
"        Recheck Cond: ((c_mktsegment = 'FURNITURE'::bpchar) AND (c_acctbal >= '4500'::numeric) AND (c_acctbal <= '6700'::numeric))"
"        Heap Blocks: exact=35147"
"        ->  Bitmap Index Scan on idx_customer_mktsegment_acctbal  (cost=0.00..2283.34 rows=70633 width=0) (actual time=24.393..24.395 rows=71545 loops=1)"
"              Index Cond: ((c_mktsegment = 'FURNITURE'::bpchar) AND (c_acctbal >= '4500'::numeric) AND (c_acctbal <= '6700'::numeric))"
"Planning Time: 0.166 ms"
"Execution Time: 258.523 ms"
```

* Определите производительность запроса, если индекс не будет использоваться
  * Используйте pg_hint_plan или drop в транзакции + rollback

Предварительно удалил все ранее созданные индексы.

```sql
EXPLAIN ANALYZE select * from customer WHERE c_mktsegment = 'FURNITURE' and c_acctbal BETWEEN 4500 and 6700 ORDER BY c_acctbal;
```

```
"Gather Merge  (cost=61778.47..68645.94 rows=58860 width=158) (actual time=166.888..199.346 rows=71545 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Sort  (cost=60778.44..60852.02 rows=29430 width=158) (actual time=162.712..168.288 rows=23848 loops=3)"
"        Sort Key: c_acctbal"
"        Sort Method: external merge  Disk: 4240kB"
"        Worker 0:  Sort Method: external merge  Disk: 3688kB"
"        Worker 1:  Sort Method: external merge  Disk: 4136kB"
"        ->  Parallel Seq Scan on customer  (cost=0.00..56277.00 rows=29430 width=158) (actual time=0.033..140.388 rows=23848 loops=3)"
"              Filter: ((c_acctbal >= '4500'::numeric) AND (c_acctbal <= '6700'::numeric) AND (c_mktsegment = 'FURNITURE'::bpchar))"
"              Rows Removed by Filter: 576152"
"Planning Time: 0.113 ms"
"Execution Time: 191.536 ms"
```

Запрос без индекса выполнился быстрее по причине необходимости в работе с Bitmap и Heap при наличии индекса.

# Задача 4. Сортировка без индекса

```sql
explain analyze select * from lineitem ORDER BY l_discount DESC, l_extendedprice LIMIT 10000;
```

* Добейтесь повышения производительности запроса без использования индексов

Лучшее время выполнения запроса до проведения оптимизаций:

```sql
"Limit  (cost=3793818.58..3794985.33 rows=10000 width=117) (actual time=7984.026..8000.297 rows=10000 loops=1)"
"  ->  Gather Merge  (cost=3793818.58..10792856.60 rows=59987566 width=117) (actual time=7978.443..7993.818 rows=10000 loops=1)"
"        Workers Planned: 2"
"        Workers Launched: 2"
"        ->  Sort  (cost=3792818.56..3867803.01 rows=29993783 width=117) (actual time=7952.484..7953.648 rows=3581 loops=3)"
"              Sort Key: l_discount DESC, l_extendedprice"
"              Sort Method: top-N heapsort  Memory: 3741kB"
"              Worker 0:  Sort Method: top-N heapsort  Memory: 3736kB"
"              Worker 1:  Sort Method: top-N heapsort  Memory: 3724kB"
"              ->  Parallel Seq Scan on lineitem  (cost=0.00..1650105.83 rows=29993783 width=117) (actual time=0.087..3726.594 rows=23995026 loops=3)"
"Planning Time: 0.128 ms"
"JIT:"
"  Functions: 1"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.154 ms (Deform 0.000 ms), Inlining 0.053 ms, Optimization 2.544 ms, Emission 2.971 ms, Total 5.722 ms"
"Execution Time: 8001.589 ms"
```

Мне повезло и обращений к диску вообще не было (Sort Method: top-N heapsort Memory). Больше такого короткого времени получить не удавалось.

Лучший результат, который удалось получить при SET max_parallel_workers_per_gather = 4; сразу после выполнения запроса на двух воркерах:

```
"Limit  (cost=2816758.41..2817955.75 rows=10000 width=117) (actual time=14518.412..14601.361 rows=10000 loops=1)"
"  ->  Gather Merge  (cost=2816758.41..11435866.08 rows=71985080 width=117) (actual time=14511.857..14594.201 rows=10000 loops=1)"
"        Workers Planned: 4"
"        Workers Launched: 4"
"        ->  Sort  (cost=2815758.35..2860749.02 rows=17996270 width=117) (actual time=11681.971..11683.115 rows=2293 loops=5)"
"              Sort Key: l_discount DESC, l_extendedprice"
"              Sort Method: top-N heapsort  Memory: 3749kB"
"              Worker 0:  Sort Method: top-N heapsort  Memory: 3750kB"
"              Worker 1:  Sort Method: top-N heapsort  Memory: 3742kB"
"              Worker 2:  Sort Method: top-N heapsort  Memory: 3744kB"
"              Worker 3:  Sort Method: external merge  Disk: 354176kB"
"              ->  Parallel Seq Scan on lineitem  (cost=0.00..1530130.70 rows=17996270 width=117) (actual time=0.052..5893.188 rows=14397015 loops=5)"
"Planning Time: 0.294 ms"
"JIT:"
"  Functions: 1"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.213 ms (Deform 0.000 ms), Inlining 0.069 ms, Optimization 3.490 ms, Emission 2.982 ms, Total 6.754 ms"
"Execution Time: 14602.386 ms"
```

При потворных вызовах время выполнения увеличивалось:

```
"Limit  (cost=2816758.41..2817955.75 rows=10000 width=117) (actual time=58990.831..59347.031 rows=10000 loops=1)"
"  ->  Gather Merge  (cost=2816758.41..11435866.08 rows=71985080 width=117) (actual time=58977.853..59333.389 rows=10000 loops=1)"
"        Workers Planned: 4"
"        Workers Launched: 4"
"        ->  Sort  (cost=2815758.35..2860749.02 rows=17996270 width=117) (actual time=53041.959..53043.064 rows=2329 loops=5)"
"              Sort Key: l_discount DESC, l_extendedprice"
"              Sort Method: top-N heapsort  Memory: 3737kB"
"              Worker 0:  Sort Method: external merge  Disk: 936968kB"
"              Worker 1:  Sort Method: external merge  Disk: 911096kB"
"              Worker 2:  Sort Method: external merge  Disk: 950080kB"
"              Worker 3:  Sort Method: external merge  Disk: 921424kB"
"              ->  Parallel Seq Scan on lineitem  (cost=0.00..1530130.70 rows=17996270 width=117) (actual time=0.477..8991.115 rows=14397015 loops=5)"
"Planning Time: 0.317 ms"
"JIT:"
"  Functions: 1"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.812 ms (Deform 0.000 ms), Inlining 0.129 ms, Optimization 4.954 ms, Emission 7.867 ms, Total 13.762 ms"
"Execution Time: 59349.507 ms"
```

Это выглядит несколько странно, так как у нас уже получалось не делать обращений к диску. ВОзможно, влияет количество воркеров. 

При SET max_parallel_workers_per_gather = 3; время выполнения составляло ~ 50 секунд. После возвращения к двум воркерам сначала удалось выполнить запрос за 26 секунда, а затем:

```
"Limit  (cost=3793818.58..3794985.33 rows=10000 width=117) (actual time=178450.927..179200.400 rows=10000 loops=1)"
"  ->  Gather Merge  (cost=3793818.58..10792856.60 rows=59987566 width=117) (actual time=178444.165..179192.967 rows=10000 loops=1)"
"        Workers Planned: 2"
"        Workers Launched: 2"
"        ->  Sort  (cost=3792818.56..3867803.01 rows=29993783 width=117) (actual time=176549.084..176552.282 rows=3580 loops=3)"
"              Sort Key: l_discount DESC, l_extendedprice"
"              Sort Method: external merge  Disk: 3046912kB"
"              Worker 0:  Sort Method: external merge  Disk: 3088024kB"
"              Worker 1:  Sort Method: external merge  Disk: 3044632kB"
"              ->  Parallel Seq Scan on lineitem  (cost=0.00..1650105.83 rows=29993783 width=117) (actual time=0.068..13906.250 rows=23995026 loops=3)"
"Planning Time: 0.106 ms"
"JIT:"
"  Functions: 1"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.767 ms (Deform 0.000 ms), Inlining 0.086 ms, Optimization 2.707 ms, Emission 3.949 ms, Total 7.509 ms"
"Execution Time: 179900.533 ms"
```

При следующих запросах время увеличивалось еще до 191 секунды. Странно, что с каждым запросом наблюдаем деградацию.

При SET work_mem = '128MB'; и двух воркерах возвращаемся к одному из лучших результатов:

```
"Limit  (cost=3793818.58..3794985.33 rows=10000 width=117) (actual time=8430.175..8445.664 rows=10000 loops=1)"
"  ->  Gather Merge  (cost=3793818.58..10792856.60 rows=59987566 width=117) (actual time=8425.276..8439.909 rows=10000 loops=1)"
"        Workers Planned: 2"
"        Workers Launched: 2"
"        ->  Sort  (cost=3792818.56..3867803.01 rows=29993783 width=117) (actual time=8400.198..8401.242 rows=3594 loops=3)"
"              Sort Key: l_discount DESC, l_extendedprice"
"              Sort Method: top-N heapsort  Memory: 4548kB"
"              Worker 0:  Sort Method: top-N heapsort  Memory: 4536kB"
"              Worker 1:  Sort Method: top-N heapsort  Memory: 4545kB"
"              ->  Parallel Seq Scan on lineitem  (cost=0.00..1650105.83 rows=29993783 width=117) (actual time=0.064..3763.042 rows=23995026 loops=3)"
"Planning Time: 0.097 ms"
"JIT:"
"  Functions: 1"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.497 ms (Deform 0.000 ms), Inlining 0.041 ms, Optimization 1.919 ms, Emission 2.926 ms, Total 5.384 ms"
"Execution Time: 8447.220 ms"
```

При поднятии воркеров до трех и четырех получались примерно те же лучшие и средние результаты:

```
"Limit  (cost=3242252.39..3243435.73 rows=10000 width=117) (actual time=11878.014..11901.401 rows=10000 loops=1)"
"  ->  Gather Merge  (cost=3242252.39..11485705.26 rows=69662982 width=117) (actual time=11869.515..11891.986 rows=10000 loops=1)"
"        Workers Planned: 3"
"        Workers Launched: 3"
"        ->  Sort  (cost=3241252.35..3299304.84 rows=23220994 width=117) (actual time=11808.655..11809.528 rows=2769 loops=4)"
"              Sort Key: l_discount DESC, l_extendedprice"
"              Sort Method: top-N heapsort  Memory: 4553kB"
"              Worker 0:  Sort Method: top-N heapsort  Memory: 4546kB"
"              Worker 1:  Sort Method: top-N heapsort  Memory: 4548kB"
"              Worker 2:  Sort Method: top-N heapsort  Memory: 4537kB"
"              ->  Parallel Seq Scan on lineitem  (cost=0.00..1582377.94 rows=23220994 width=117) (actual time=0.208..7490.570 rows=17996269 loops=4)"
"Planning Time: 0.252 ms"
"JIT:"
"  Functions: 1"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 0.443 ms (Deform 0.000 ms), Inlining 0.075 ms, Optimization 3.331 ms, Emission 5.080 ms, Total 8.930 ms"
"Execution Time: 11903.053 ms"
```

При повышении числа воркеров до 4 и work_mem до 512MB наблюдаю снижение времени выполнения до ~5.5 секунд. При повышении числа воркеров до 5/6 получаем ~5 секунд.
При повышении числа воркеров до 7: ~4.5 секунд. Далее при повышении до 15 воркеров остаемся примерно в том же диапазоне ~5 секунд.

<details>

<summary>Вопросы</summary>

Почему при росте скорости выполнения при увеличении work_mem у воркеров не растет Memory: 4546kB?

</details>

* Для сортировки используется буфер: set work_mem='4MB';
  * Насколько можно увеличить этот буфер?

Все зависит от общей нагрузки и свободной RAM на машине. Приблизительно можно рассчитать так, оставив некоторый запас: свободная RAM / max_connections / max_parallel_workers_per_gather 

* Как влияет на использование памяти и производительность количество параллельных воркеров max_parallel_workers_per_gather?

Объем памяти увеличивается. До определнного количества воркеров увеличивается и производительность.

# Задача 5.Соединение

```sql
SELECT o.o_orderkey AS document_id, jsonb_build_object('order', jsonb_build_object('orderkey', o.o_orderkey, 'custkey', o.o_custkey, 'orderstatus', o.o_orderstatus, 'totalprice', o.o_totalprice, 'orderdate', o.o_orderdate, 'orderpriority', o.o_orderpriority, 'clerk', o.o_clerk, 'shippriority', o.o_shippriority, 'comment', o.o_comment), 'customer', jsonb_build_object('custkey', c.c_custkey, 'name', c.c_name, 'address', c.c_address, 'nationkey', c.c_nationkey, 'phone', c.c_phone, 'acctbal', c.c_acctbal, 'mktsegment', c.c_mktsegment, 'comment', c.c_comment)) AS document_data FROM orders o JOIN customer c ON o.o_custkey = c.c_custkey WHERE c.c_custkey IN (SELECT c_custkey FROM customer ORDER BY c_custkey LIMIT 5);
```

```
"Hash Join  (cost=129987.02..690959.58 rows=86 width=36) (actual time=1021.708..4756.340 rows=42 loops=1)"
"  Hash Cond: (o.o_custkey = c.c_custkey)"
"  ->  Seq Scan on orders o  (cost=0.00..493478.12 rows=17998212 width=107) (actual time=0.089..2336.033 rows=18000000 loops=1)"
"  ->  Hash  (cost=129986.96..129986.96 rows=5 width=162) (actual time=968.160..968.349 rows=5 loops=1)"
"        Buckets: 1024  Batches: 1  Memory Usage: 9kB"
"        ->  Hash Semi Join  (cost=64109.90..129986.96 rows=5 width=162) (actual time=698.415..968.313 rows=5 loops=1)"
"              Hash Cond: (c.c_custkey = customer.c_custkey)"
"              ->  Seq Scan on customer c  (cost=0.00..61152.00 rows=1800000 width=158) (actual time=0.031..241.761 rows=1800000 loops=1)"
"              ->  Hash  (cost=64109.84..64109.84 rows=5 width=4) (actual time=583.396..583.581 rows=5 loops=1)"
"                    Buckets: 1024  Batches: 1  Memory Usage: 9kB"
"                    ->  Limit  (cost=64109.25..64109.84 rows=5 width=4) (actual time=583.311..583.498 rows=5 loops=1)"
"                          ->  Gather Merge  (cost=64109.25..239121.47 rows=1500000 width=4) (actual time=282.948..283.133 rows=5 loops=1)"
"                                Workers Planned: 2"
"                                Workers Launched: 2"
"                                ->  Sort  (cost=63109.23..64984.23 rows=750000 width=4) (actual time=244.798..244.801 rows=5 loops=3)"
"                                      Sort Key: customer.c_custkey"
"                                      Sort Method: top-N heapsort  Memory: 25kB"
"                                      Worker 0:  Sort Method: top-N heapsort  Memory: 25kB"
"                                      Worker 1:  Sort Method: top-N heapsort  Memory: 25kB"
"                                      ->  Parallel Seq Scan on customer  (cost=0.00..50652.00 rows=750000 width=4) (actual time=75.391..215.372 rows=600000 loops=3)"
"Planning Time: 0.241 ms"
"JIT:"
"  Functions: 25"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 2.629 ms (Deform 1.620 ms), Inlining 217.755 ms, Optimization 187.018 ms, Emission 121.702 ms, Total 529.103 ms"
"Execution Time: 4759.011 ms"
```

* Добавьте минимальное количество индексов для ускорения запроса

```sql
 -- Ускоряет соединение между orders и customer
CREATE INDEX idx_orders_o_custkey ON orders (o_custkey);

-- Ускоряет соединение и фильтрацию в подзапросе
CREATE INDEX idx_customer_c_custkey ON customer (c_custkey);
```

```
"Nested Loop  (cost=1.44..51.75 rows=86 width=36) (actual time=0.078..0.604 rows=42 loops=1)"
"  ->  Nested Loop  (cost=1.00..42.90 rows=5 width=162) (actual time=0.033..0.046 rows=5 loops=1)"
"        ->  HashAggregate  (cost=0.57..0.62 rows=5 width=4) (actual time=0.022..0.024 rows=5 loops=1)"
"              Group Key: customer.c_custkey"
"              Batches: 1  Memory Usage: 24kB"
"              ->  Limit  (cost=0.43..0.56 rows=5 width=4) (actual time=0.015..0.016 rows=5 loops=1)"
"                    ->  Index Only Scan using idx_c_custkey on customer  (cost=0.43..46802.43 rows=1800000 width=4) (actual time=0.014..0.015 rows=5 loops=1)"
"                          Heap Fetches: 0"
"        ->  Index Scan using idx_c_custkey on customer c  (cost=0.43..8.45 rows=1 width=158) (actual time=0.003..0.003 rows=1 loops=5)"
"              Index Cond: (c_custkey = customer.c_custkey)"
"  ->  Index Scan using idx_orders_o_custkey on orders o  (cost=0.44..1.47 rows=17 width=107) (actual time=0.003..0.013 rows=8 loops=5)"
"        Index Cond: (o_custkey = c.c_custkey)"
"Planning Time: 0.567 ms"
"Execution Time: 0.652 ms"
```

* Получить распределение значений c.c_custkey, изменить запрос, чтобы генерировать документы для 30% от всех клиентов

```
SELECT width_bucket(c_custkey, (SELECT MIN(c_custkey) FROM customer), (SELECT MAX(c_custkey) FROM customer), 10) AS bucket,
       COUNT(*) AS count
FROM customer
GROUP BY bucket
ORDER BY bucket;
```

```
"bucket"	"count"
1	180000
2	180000
3	180000
4	180000
5	180000
6	180000
7	180000
8	180000
9	180000
10	179999
11	1
```

```sql
EXPLAIN ANALYZE WITH total AS (
  SELECT COUNT(*) AS total_customers FROM customer
)
SELECT
  o.o_orderkey AS document_id,
  jsonb_build_object(
    'order', jsonb_build_object(
      'orderkey', o.o_orderkey,
      'custkey', o.o_custkey,
      'orderstatus', o.o_orderstatus,
      'totalprice', o.o_totalprice,
      'orderdate', o.o_orderdate,
      'orderpriority', o.o_orderpriority,
      'clerk', o.o_clerk,
      'shippriority', o.o_shippriority,
      'comment', o.o_comment
    ),
    'customer', jsonb_build_object(
      'custkey', c.c_custkey,
      'name', c.c_name,
      'address', c.c_address,
      'nationkey', c.c_nationkey,
      'phone', c.c_phone,
      'acctbal', c.c_acctbal,
      'mktsegment', c.c_mktsegment,
      'comment', c.c_comment
    )
  ) AS document_data
FROM orders o
LEFT JOIN customer c
  ON o.o_custkey = c.c_custkey
  LIMIT (SELECT total_customers * 0.3 FROM total);
```

```
"Limit  (cost=39187.70..244413.61 rows=1800000 width=36) (actual time=192.517..10991.629 rows=540000 loops=1)"
"  InitPlan 1"
"    ->  Subquery Scan on total  (cost=39177.64..39177.66 rows=1 width=32) (actual time=174.710..174.932 rows=1 loops=1)"
"          ->  Finalize Aggregate  (cost=39177.64..39177.65 rows=1 width=8) (actual time=174.684..174.903 rows=1 loops=1)"
"                ->  Gather  (cost=39177.43..39177.64 rows=2 width=8) (actual time=174.478..174.883 rows=3 loops=1)"
"                      Workers Planned: 2"
"                      Workers Launched: 2"
"                      ->  Partial Aggregate  (cost=38177.43..38177.44 rows=1 width=8) (actual time=128.728..128.737 rows=1 loops=3)"
"                            ->  Parallel Index Only Scan using idx_c_custkey on customer  (cost=0.43..36302.43 rows=750000 width=0) (actual time=0.100..100.283 rows=600000 loops=3)"
"                                  Heap Fetches: 0"
"  ->  Merge Right Join  (cost=10.04..2052269.13 rows=18000000 width=36) (actual time=0.202..10735.635 rows=540000 loops=1)"
"        Merge Cond: (c.c_custkey = o.o_custkey)"
"        ->  Index Scan using idx_c_custkey on customer c  (cost=0.43..89913.05 rows=1800000 width=158) (actual time=0.049..41.728 rows=54058 loops=1)"
"        ->  Index Scan using idx_orders_o_custkey on orders o  (cost=0.44..1597856.08 rows=18000000 width=107) (actual time=0.060..4189.910 rows=540000 loops=1)"
"Planning Time: 1.399 ms"
"JIT:"
"  Functions: 17"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 10.084 ms (Deform 6.626 ms), Inlining 0.000 ms, Optimization 1.729 ms, Emission 23.556 ms, Total 35.369 ms"
"Execution Time: 11042.053 ms"
```

<details>

<summary>Вопросы</summary>

Сравнить с:

```sql
EXPLAIN ANALYZE WITH total AS (
  SELECT COUNT(*) AS total_customers FROM customer
),
c AS (
  SELECT *
  FROM customer, total
  ORDER BY c_custkey
  LIMIT (SELECT total_customers * 0.3 FROM total)
)
SELECT
  o.o_orderkey AS document_id,
  jsonb_build_object(
    'order', jsonb_build_object(
      'orderkey', o.o_orderkey,
      'custkey', o.o_custkey,
      'orderstatus', o.o_orderstatus,
      'totalprice', o.o_totalprice,
      'orderdate', o.o_orderdate,
      'orderpriority', o.o_orderpriority,
      'clerk', o.o_clerk,
      'shippriority', o.o_shippriority,
      'comment', o.o_comment
    ),
    'customer', jsonb_build_object(
      'custkey', c.c_custkey,
      'name', c.c_name,
      'address', c.c_address,
      'nationkey', c.c_nationkey,
      'phone', c.c_phone,
      'acctbal', c.c_acctbal,
      'mktsegment', c.c_mktsegment,
      'comment', c.c_comment
    )
  ) AS document_data
FROM orders o
LEFT JOIN c ON o.o_custkey = c.c_custkey;```
```

</details>

# Задача 6. Нормализация

```sql
INSERT INTO order_details (order_id, customer_name, order_date, item1_name, item1_quantity, item1_price, 
item2_name, item2_quantity, item2_price, item3_name, item3_quantity, item3_price) VALUES
 (1, 'John Doe', '2023-10-01', 'Laptop', 1, 1200.00, 'Mouse', 2, 25.00, 'Keyboard', 1, 50.00),
 (2, 'Jane Smith', '2023-10-02', 'Monitor', 1, 300.00, 'HDMI Cable', 1, 15.00, NULL, NULL, NULL),
 (3, 'Alice Johnson', '2023-10-03', 'Printer', 1, 200.00, 'Paper', 5, 10.00, 'Ink Cartridge', 2, 40.00);
```

* Работая с базой TPCH, вы обнаружили Legacy код который делал вставку в таблицу order_details с неизвестной структурой. Воспроизведите, какая структура у таблицы order_details.

```sql
CREATE TABLE order_details (
    order_id INT PRIMARY KEY NOT NULL,
    customer_name VARCHAR(25) NOT NULL,
    order_date DATE NOT NULL,
    item1_name VARCHAR(45) NOT NULL,
    item1_quantity NUMERIC(15,2) NOT NULL,
    item1_price NUMERIC(15,2) NOT NULL,
    item2_name VARCHAR(45),
    item2_quantity NUMERIC(15,2),
    item2_price NUMERIC(15,2),
    item3_name VARCHAR(45),
    item3_quantity NUMERIC(15,2),
    item3_price NUMERIC(15,2)
);
```

* Находится ли таблица order_details в 1NF? (Обоснуйте)
  * Если нет, переведите её в 1NF

Таблица находится в 1NF, если:

 ✔ если все строки поменяют порядок нет потери информации
 
 ✔ если колонки поменять местами нет потери информации
 
 ✔ нет одинаковых строк
 
 ✔ одно значение (не массив, не сложный тип, не JSON) в ячейке
 
 ✔ одинаковый тип данных и ограничения на всю колонку

Однако таблица содержит повторяющиеся группы столбцов: item*_name, item*_quantity, item*_price, что в некотором смысле можно считать хранением массива в колонках вместо разбиения на отдельные строки.

```sql
CREATE TABLE orders_by_customer (order_id INT PRIMARY KEY, customer_name VARCHAR(25) NOT NULL, order_date DATE NOT NULL);

CREATE TABLE order_items (order_id INT REFERENCES orders_by_customer(order_id), item_name VARCHAR(45) NOT NULL, item_quantity NUMERIC(15,2) NOT NULL, item_price NUMERIC(15,2)) NOT NULL;
```

Теперь:

  * каждая товарная позиция — отдельная строка в order_items;

  * нет ограничений на количество товаров.
