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
