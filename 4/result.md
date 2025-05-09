# Задание 1. Взаимная блокировка

* Запустите параллельно обе команды в SSH.
  * Дождитесь взаимных блокировок
  * Прекратите выполнение команд с помощью ctrl+c
* Найдите последний deadlock в логе БД /var/log/postgresql/postgresql-17-main.log
* Поменяйте транзакции чтобы избегать взаимных блокировок, но сохранить старую функциональность.

```bash
while sleep 0.01; do echo "BEGIN; UPDATE customer SET c_comment = 'Updated by A' WHERE c_custkey = 100; UPDATE orders SET o_comment = 'Updated by A' WHERE o_orderkey = 36422; COMMIT;"; done | psql -U tpch

while sleep 0.01; do echo "BEGIN; UPDATE orders SET o_comment = 'Updated by B' WHERE o_orderkey = 36422; UPDATE customer SET c_comment = 'Updated by B' WHERE c_custkey = 100; COMMIT;"; done | psql -U tpch
```

```
2025-04-12 13:23:26.039 MSK [444671] tpch@tpch ERROR:  deadlock detected
2025-04-12 13:23:26.039 MSK [444671] tpch@tpch DETAIL:  Process 444671 waits for ShareLock on transaction 1049; blocked by process 445106.
        Process 445106 waits for ShareLock on transaction 1048; blocked by process 444671.
        Process 444671: UPDATE orders SET o_comment = 'Updated by A' WHERE o_orderkey = 36422;
        Process 445106: UPDATE customer SET c_comment = 'Updated by B' WHERE c_custkey = 100;
2025-04-12 13:23:26.039 MSK [444671] tpch@tpch HINT:  See server log for query details.
2025-04-12 13:23:26.039 MSK [444671] tpch@tpch CONTEXT:  while updating tuple (763,60) in relation "orders"
2025-04-12 13:23:26.039 MSK [444671] tpch@tpch STATEMENT:  UPDATE orders SET o_comment = 'Updated by A' WHERE o_orderkey = 36422;
```

```bash
while sleep 0.01; do echo "BEGIN; UPDATE orders SET o_comment = 'Updated by A' WHERE o_orderkey = 36422; UPDATE customer SET c_comment = 'Updated by A' WHERE c_custkey = 100; COMMIT;"; done | psql -U tpch

while sleep 0.01; do echo "BEGIN; UPDATE orders SET o_comment = 'Updated by B' WHERE o_orderkey = 36422; UPDATE customer SET c_comment = 'Updated by B' WHERE c_custkey = 100; COMMIT;"; done | psql -U tpch
```

# Задание 2. VACUUM

Найти количество строк, которые можно заморозить в таблице customer:

```sql
VACUUM (verbose, freeze) customer;
```

```
INFO:  aggressively vacuuming "tpch.public.customer"
INFO:  launched 1 parallel vacuum worker for index cleanup (planned: 1)
INFO:  finished vacuuming "tpch.public.customer": index scans: 0
pages: 0 removed, 43152 remain, 43152 scanned (100.00% of total)
tuples: 6 removed, 1800000 remain, 0 are dead but not yet removable
removable cutoff: 1080, which was 0 XIDs old when operation ended
new relfrozenxid: 1080, which is 181 XIDs ahead of previous value
frozen: 43109 pages from table (99.90% of total) had 1800000 tuples frozen
index scan bypassed: 29 pages from table (0.07% of total) have 39 dead item identifiers
avg read rate: 94.948 MB/s, avg write rate: 94.904 MB/s
buffer usage: 43203 hits, 43133 misses, 43113 dirtied
WAL usage: 86190 records, 43112 full page images, 354215048 bytes
system usage: CPU: user: 0.44 s, system: 1.61 s, elapsed: 3.54 s
VACUUM
```

Все 1 800 000 строк в таблице были заморожены.

# Задание 3. VACUUM и параллельные транзакции

* Найдите количество замороженных строк, если после начала транзакции провести модификацию таблицы и выполнить очистку.

```sql
begin; set transaction isolation level repeatable read; select * from customer WHERE c_custkey = 100;
```

Во втором соединении, не завершая первую транзакцию:

```sql
UPDATE customer SET c_comment = 'Updated by B' WHERE c_custkey = 100;
vacuum (verbose,freeze) customer;
```

```
INFO:  aggressively vacuuming "tpch.public.customer"
INFO:  launched 1 parallel vacuum worker for index cleanup (planned: 1)
INFO:  "customer": stopping truncate due to conflicting lock request
INFO:  finished vacuuming "tpch.public.customer": index scans: 0
pages: 0 removed, 43152 remain, 243 scanned (0.56% of total)
tuples: 0 removed, 1800001 remain, 1 are dead but not yet removable
removable cutoff: 1080, which was 2 XIDs old when operation ended
frozen: 0 pages from table (0.00% of total) had 0 tuples frozen
index scan bypassed: 29 pages from table (0.07% of total) have 39 dead item identifiers
avg read rate: 0.263 MB/s, avg write rate: 0.002 MB/s
buffer usage: 311 hits, 170 misses, 1 dirtied
WAL usage: 1 records, 1 full page images, 7293 bytes
system usage: CPU: user: 0.00 s, system: 0.01 s, elapsed: 5.05 s
VACUUM
```

PostgreSQL не смог заморозить ни одну строку, так как первая транзакция должна видеть старую версию строки, и PostgreSQL обязан её сохранить.

* Сравните с read committed.

```
INFO:  aggressively vacuuming "tpch.public.customer"
INFO:  launched 1 parallel vacuum worker for index cleanup (planned: 1)
INFO:  finished vacuuming "tpch.public.customer": index scans: 0
pages: 0 removed, 43109 remain, 201 scanned (0.47% of total)
tuples: 1 removed, 1800000 remain, 0 are dead but not yet removable
removable cutoff: 1084, which was 0 XIDs old when operation ended
new relfrozenxid: 1084, which is 2 XIDs ahead of previous value
frozen: 1 pages from table (0.00% of total) had 1 tuples frozen
index scan bypassed: 30 pages from table (0.07% of total) have 41 dead item identifiers
avg read rate: 0.000 MB/s, avg write rate: 0.000 MB/s
buffer usage: 396 hits, 0 misses, 0 dirtied
WAL usage: 3 records, 0 full page images, 314 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.02 s
VACUUM
```

Одна старая строка была успешно удалена (1 removed). Одна вновь вставленная строка была заморожена (1 frozen). Первая транзакция не держит "старый" snappshot.
