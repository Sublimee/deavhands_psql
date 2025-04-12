# Задание 1. 

* Запустите параллельно обе команды в SSH.
  * Дождитесь взаимных блокировок
  * Прекратите выполнение команд с помощью ctrl+c
* Найдите последний deadlock в логе БД /var/log/postgresql/postgresql-17-main.log
* Поменяйте транзакции чтобы избегать взаимных блокировок, но сохранить старую функциональность.

```sql
while sleep 0.01; do echo "BEGIN; UPDATE customer SET c_comment = 'Updated by A' WHERE c_custkey =
100; UPDATE orders SET o_comment = 'Updated by A' WHERE o_orderkey = 36422; COMMIT;"; done | psql -U tpch

while sleep 0.01; do echo "BEGIN; UPDATE orders SET o_comment = 'Updated by B' WHERE o_orderkey =
36422; UPDATE customer SET c_comment = 'Updated by B' WHERE c_custkey = 100; COMMIT;"; done | psql -U tpch
```
