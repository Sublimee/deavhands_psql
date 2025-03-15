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

explain (analyze, timing, buffers)
select ... from part ...;

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
