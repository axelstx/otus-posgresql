## Домашнее задание

Блокировки

**Цель:**
* понять как работают блокировки;
* научиться находить проблемные места;

---

1. Настроил сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд
```bash
postgres=# alter system set log_lock_waits = on;
ALTER SYSTEM
postgres=# alter system set deadlock_timeout = '200ms';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)
```


2. Создал таблицу и вставил данные
```bash
postgres=# create table test_locks (id int primary key, value text);
CREATE TABLE
postgres=# insert into test_locks values (1, 'test');
INSERT 0 1
```


3. В разных сеансах:

**Сеанс 1**
```bash
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
           4007
(1 row)

postgres=# begin;
BEGIN
postgres=*# update  test_locks set value = 'test1' WHERE id = 1;
UPDATE 1
```

**Сеанс 2**
```bash
postgres=# begin;
BEGIN
postgres=*# update  test_locks set value = 'test2' WHERE id = 1;
```

**Журнал:**
```bash
2026-01-06 14:45:53.926 UTC [4050] postgres@postgres LOG:  process 4050 still waiting for ShareLock on transaction 751 after 200.316 ms
2026-01-06 14:45:53.926 UTC [4050] postgres@postgres DETAIL:  Process holding the lock: 4007. Wait queue: 4050.
2026-01-06 14:45:53.926 UTC [4050] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test_locks"
2026-01-06 14:45:53.926 UTC [4050] postgres@postgres STATEMENT:  update  test_locks set value = 'test2' WHERE id = 1;
```


4. Смоделировал ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах

**Сеанс 1**
```bash
postgres=# begin;
BEGIN
postgres=*# update test_locks set value = 'update1' where id = 1;
UPDATE 1
```

**Сеанс 2**
```bash
postgres=# begin;
BEGIN
postgres=*# update test_locks set value = 'update2' where id = 1;
```

**Сеанс 3**
```bash
postgres=# begin;
BEGIN
postgres=*# update test_locks set value = 'update3' where id = 1;
```


5. Вывел представление по блокировкам

```bash
 pid  |   locktype    |   table_name    | page | tuple | transactionid |       mode       | granted |                         query                         |        state
------+---------------+-----------------+------+-------+---------------+------------------+---------+-------------------------------------------------------+---------------------
 4007 | relation      | test_locks_pkey |      |       |               | RowExclusiveLock | t       | update test_locks set value = 'update1' where id = 1; | idle in transaction
 4007 | relation      | test_locks      |      |       |               | RowExclusiveLock | t       | update test_locks set value = 'update1' where id = 1; | idle in transaction
 4007 | transactionid |                 |      |       |           757 | ExclusiveLock    | t       | update test_locks set value = 'update1' where id = 1; | idle in transaction
 4007 | virtualxid    |                 |      |       |               | ExclusiveLock    | t       | update test_locks set value = 'update1' where id = 1; | idle in transaction
 4050 | relation      | test_locks      |      |       |               | RowExclusiveLock | t       | update test_locks set value = 'update2' where id = 1; | active
 4050 | relation      | test_locks_pkey |      |       |               | RowExclusiveLock | t       | update test_locks set value = 'update2' where id = 1; | active
 4050 | transactionid |                 |      |       |           757 | ShareLock        | f       | update test_locks set value = 'update2' where id = 1; | active
 4050 | transactionid |                 |      |       |           758 | ExclusiveLock    | t       | update test_locks set value = 'update2' where id = 1; | active
 4050 | tuple         | test_locks      |    0 |     3 |               | ExclusiveLock    | t       | update test_locks set value = 'update2' where id = 1; | active
 4050 | virtualxid    |                 |      |       |               | ExclusiveLock    | t       | update test_locks set value = 'update2' where id = 1; | active
 4100 | relation      | test_locks      |      |       |               | RowExclusiveLock | t       | update test_locks set value = 'update3' where id = 1; | active
 4100 | relation      | test_locks_pkey |      |       |               | RowExclusiveLock | t       | update test_locks set value = 'update3' where id = 1; | active
 4100 | transactionid |                 |      |       |           759 | ExclusiveLock    | t       | update test_locks set value = 'update3' where id = 1; | active
 4100 | tuple         | test_locks      |    0 |     3 |               | ExclusiveLock    | f       | update test_locks set value = 'update3' where id = 1; | active
 4100 | virtualxid    |                 |      |       |               | ExclusiveLock    | t       | update test_locks set value = 'update3' where id = 1; | active
```

### Блокировки:
**RowExclusiveLock** блокировка на уровне таблицы, получают все три сеанса  
**ExclusiveLock** на **transactionid**, значит что каждая транзакция блокирует свой id  
**ExclusiveLock** на **tuple**, первый в очереди идет сеанс 2, а сеанс 3 в ожидании  
**ShareLock** на **transactionid**, значит что сеанс 2 ждет завершения сеанса 1  


6. Воспроизвел взаимоблокировку трех транзакций.
```bash
postgres=# create table test_table (id int primary key, num numeric);
CREATE TABLE
postgres=# insert into test_table values (1, 1000), (2, 1000), (3, 1000);
INSERT 0 3
```

**Сеанс 1:**
```bash
postgres=# begin;
BEGIN
postgres=*# update test_table set num = num - 100 where id = 1;
UPDATE 1
```

**Сеанс 2:**
```bash
postgres=# begin;
BEGIN
postgres=*# update test_table set num = num - 100 where id = 2;
UPDATE 1
```

**Сеанс 3:**
```bash
postgres=# begin;
BEGIN
postgres=*# update test_table set num = num - 100 where id = 3;
UPDATE 1
```

**Сеанс 1:**
```bash
postgres=*# update test_table set num = num + 100 where id = 2;
```

**Сеанс 2:**
```bash
postgres=*# update test_table set num = num + 100 where id = 3;
```

**Сеанс 3:**
```bash
postgres=*# update test_table set num = num + 100 where id = 1;
ERROR:  deadlock detected
DETAIL:  Process 4100 waits for ShareLock on transaction 762; blocked by process 4007.
Process 4007 waits for ShareLock on transaction 763; blocked by process 4050.
Process 4050 waits for ShareLock on transaction 764; blocked by process 4100.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "test_table"
```

**Журнал:**
```bash
2026-01-06 16:07:47.621 UTC [4100] postgres@postgres ERROR:  deadlock detected
2026-01-06 16:07:47.621 UTC [4100] postgres@postgres DETAIL:  Process 4100 waits for ShareLock on transaction 762; blocked by process 4007.
        Process 4007 waits for ShareLock on transaction 763; blocked by process 4050.
        Process 4050 waits for ShareLock on transaction 764; blocked by process 4100.
        Process 4100: update test_table set num = num + 100 where id = 1;
        Process 4007: update test_table set num = num + 100 where id = 2;
        Process 4050: update test_table set num = num + 100 where id = 3;
2026-01-06 16:07:47.621 UTC [4100] postgres@postgres HINT:  See server log for query details.
2026-01-06 16:07:47.621 UTC [4100] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test_table"
2026-01-06 16:07:47.621 UTC [4100] postgres@postgres STATEMENT:  update test_table set num = num + 100 where id = 1;
2026-01-06 16:07:47.622 UTC [4050] postgres@postgres LOG:  process 4050 acquired ShareLock on transaction 764 after 6089.378 ms
2026-01-06 16:07:47.622 UTC [4050] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "test_table"
2026-01-06 16:07:47.622 UTC [4050] postgres@postgres STATEMENT:  update test_table set num = num + 100 where id = 3;
```
### В итоге 3 ждет 1, 1 ждет 2, 2 ждет 3, postgres обнаружил deadlock и отменил одну транзакцию


7. Вопрос: *Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?*

### Могут, потому что postgres блокирует строки последовательно, и порядок блокировки не гарантирован, что может привести к взаимной блокировке