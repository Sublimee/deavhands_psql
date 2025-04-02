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
* Для сортировки используется буфер: set work_mem='4MB';
  * Насколько можно увеличить этот буфер?
* Как влияет на использование памяти и производительность количество параллельных воркеров max_parallel_workers_per_gather? 
