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

подойдет. Поиск будет выполнен с помощью индекса, а сортировка в памяти после прохода по Bitmap. Это подтверждается результатом выполнения:

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
