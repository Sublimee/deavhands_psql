# Задача 1

**Проведите полное сканирование таблицы part (вместо orders)**

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT * FROM part;
```

* **Какое время выполнения?** 137298.100 ms (Execution Time)

* **Сколько строчек сканируется?** 2 400 000 (Actual Rows)

* **Какое количество страниц считывается?** 103723 (Shared Read Blocks)

  * **Сколько из них считалось из кеша?** 0 (Shared Hit Blocks)

<details>

<summary>Вопросы</summary>
При повторном запуске время выполнения запроса сокращается на порядок, но значения:

Shared Hit Blocks	70

Shared Read Blocks	103653

не подтвеждают активное использование кэша. Влияет кэш операционной системы? Команда

```
sudo sync; sudo sysctl -w vm.drop_caches=3
```

и рестарт контейнера не помогли.

</details>

# Задача 2

* **Найдите поле даты создания заказа в таблице orders:** o_orderdate

* **Проведите полное сканирование таблицы orders, фильтруя записи после 1996 года**

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT * FROM orders WHERE o_orderdate > '1996-01-01';
```

* **Какое время выполнения?** 59058.100 ms (Execution Time)

* **Сколько строчек сканируется?** 18000000 (7 060 826 (rows) + 10939174 (Rows Removed by Filter))

* **Какое количество страниц считывается?** 313496 (313464 (Buffers: shared read) + 32 (Buffers: shared hit))

  * **Сколько из них считалось из кеша?** 32 (Buffers: shared hit)

# Задача 3

* **Сравните время выполнения при выборке всех полей и 3-х полей:**

```sql
SELECT * FROM lineitem; # Execution Time: 85088.664 ms, width=117

SELECT l_orderkey, l_partkey, l_quantity FROM lineitem; # Execution Time: 60000.594 ms, width=13
```

* **На сколько байт уменьшается длина строки (width) при выборке 3-х полей?** 104

<details>

<summary>Вопросы</summary>

Можно ли говорить, что выполнение этих двух запросов последовательно показательно с учетом прогрева кэша первым запросом?

</details>

# Задача 4

* **Найти поля с типом хранения plain/main в таблице part:**

```shell
psql \d+ part # p_partkey p_size p_retailprice
```

* Сделать выборку первых 10000 строк результата
  * используя только поля main+plain
  * добавив к этим полям p_doc::text

```sql
SELECT
    p_partkey,
    p_size,
    p_retailprice,
    p_doc::text as p_doc
FROM part
LIMIT 10000;
```

# Задача 5

* **Определить время исполнения и план запроса поиска всех заказов из таблицы orders, сделанных в 1997-12-25. Перед выполнением запроса, запустите на сервере:**

```shell
stress --cpu 7
```

```sql
EXPLAIN (ANALYZE, TIMING, BUFFERS) SELECT * FROM ORDERS WHERE o_orderdate = '1997-12-25';
```

```
"QUERY PLAN"
"Gather  (cost=1000.00..408981.69 rows=7450 width=107) (actual time=21.955..1890.237 rows=7479 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared hit=481 read=313015"
"  ->  Parallel Seq Scan on orders  (cost=0.00..407236.69 rows=3104 width=107) (actual time=11.537..1797.913 rows=2493 loops=3)"
"        Filter: (o_orderdate = '1997-12-25'::date)"
"        Rows Removed by Filter: 5997507"
"        Buffers: shared hit=481 read=313015"
"Planning Time: 0.759 ms"
"JIT:"
"  Functions: 6"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 1.956 ms (Deform 0.628 ms), Inlining 0.000 ms, Optimization 2.083 ms, Emission 26.228 ms, Total 30.267 ms"
"Execution Time: 1893.022 ms"
```

* **Как меняется время выполнения, если запретить параллельное выполнение запроса с помощью:**

```sql
SET max_parallel_workers_per_gather = 0;
```

```
"QUERY PLAN"
"Gather  (cost=1000.00..408981.69 rows=7450 width=107) (actual time=9.948..1080.551 rows=7479 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared hit=1057 read=312439"
"  ->  Parallel Seq Scan on orders  (cost=0.00..407236.69 rows=3104 width=107) (actual time=10.354..1032.884 rows=2493 loops=3)"
"        Filter: (o_orderdate = '1997-12-25'::date)"
"        Rows Removed by Filter: 5997507"
"        Buffers: shared hit=1057 read=312439"
"Planning Time: 0.086 ms"
"JIT:"
"  Functions: 6"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 1.036 ms (Deform 0.465 ms), Inlining 0.000 ms, Optimization 0.949 ms, Emission 26.241 ms, Total 28.225 ms"
"Execution Time: 1082.002 ms"
```

Время выполнения в конкретных прогонах уменьшилось, хотя все зависит от прогретости кэша: при повторных прогонах получаю приблизительно равные показатели времени ыполнения (800-1100 ms).

# Задача 6

Ускорьте запрос, добавив индекс нужного типа (известно, что приложение делает выборки только с операцией равенство):

```sql
select l_orderkey from lineitem where l_shipdate = '1996-12-01';
```

```sql
EXPLAIN (ANALYZE, TIMING, BUFFERS) select l_orderkey from lineitem where l_shipdate = '1996-12-01';
```

```
"QUERY PLAN"
"Gather  (cost=1000.00..1728924.92 rows=28495 width=4) (actual time=85.717..20729.047 rows=30101 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared hit=96 read=1350072"
"  ->  Parallel Seq Scan on lineitem  (cost=0.00..1725075.42 rows=11873 width=4) (actual time=192.039..20566.698 rows=10034 loops=3)"
"        Filter: (l_shipdate = '1996-12-01'::date)"
"        Rows Removed by Filter: 23984992"
"        Buffers: shared hit=96 read=1350072"
"Planning Time: 0.624 ms"
"JIT:"
"  Functions: 12"
"  Options: Inlining true, Optimization true, Expressions true, Deforming true"
"  Timing: Generation 4.542 ms (Deform 1.548 ms), Inlining 300.259 ms, Optimization 152.917 ms, Emission 92.081 ms, Total 549.799 ms"
"Execution Time: 20746.216 ms"
```

```sql
CREATE INDEX shipdate_idx ON lineitem USING HASH (l_shipdate);
```

```
"QUERY PLAN"
"Bitmap Heap Scan on lineitem  (cost=916.84..101843.23 rows=28496 width=4) (actual time=17.610..206.555 rows=30101 loops=1)"
"  Recheck Cond: (l_shipdate = '1996-12-01'::date)"
"  Heap Blocks: exact=29343"
"  Buffers: shared read=29417"
"  ->  Bitmap Index Scan on shipdate_idx  (cost=0.00..909.72 rows=28496 width=0) (actual time=7.816..7.817 rows=30101 loops=1)"
"        Index Cond: (l_shipdate = '1996-12-01'::date)"
"        Buffers: shared read=74"
"Planning Time: 0.108 ms"
"JIT:"
"  Functions: 4"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 0.603 ms (Deform 0.302 ms), Inlining 0.000 ms, Optimization 0.426 ms, Emission 3.717 ms, Total 4.746 ms"
"Execution Time: 209.481 ms"
```
