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
CREATE INDEX idx_customer_c_mktsegment_c_acctbal ON customer (c_mktsegment, c_acctbal); -- 15ms
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
CREATE INDEX idx_1 ON customer (c_mktsegment, c_acctbal, c_phone);
```

```sql
CREATE INDEX idx_2 ON customer (c_mktsegment, c_acctbal, c_phone) INCLUDE (c_nationkey);
```

Причина в c_acctbal::numeric?

</details>

* Какие соображения доступны до проверки скорости работы на практике?
  * Воспользуйтесь сведениями о распределении данных в колонке
  * Определите селективность каждого условия (сколько строчек отфильтровывает каждое условие)
