## Домашнее задание

Работа с журналами

**Цель:**
* уметь работать с журналами и контрольными точками;
* уметь настраивать параметры журналов;

---

1. Настраиваю выполнение контрольной точки раз в 30 секунд
```bash
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

2. Создал testdb и проверил lsn

```bash
testdb=# select pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 1/68D14BF8
(1 row)
```

3.  Десять минут c помощью утилиты pgbench подавал нагрузку
```bash
postgres@ub24-server:~$ pgbench -c 10 -P 20 -j 4 -T 600 testdb
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 20.0 s, 2114.5 tps, lat 4.725 ms stddev 3.317, 0 failed
progress: 40.0 s, 2604.0 tps, lat 3.839 ms stddev 2.946, 0 failed
progress: 60.0 s, 2904.9 tps, lat 3.441 ms stddev 2.686, 0 failed
progress: 80.0 s, 2955.9 tps, lat 3.381 ms stddev 2.627, 0 failed
progress: 100.0 s, 2712.6 tps, lat 3.684 ms stddev 2.868, 0 failed
progress: 120.0 s, 2644.3 tps, lat 3.781 ms stddev 2.976, 0 failed
progress: 140.0 s, 2562.5 tps, lat 3.901 ms stddev 3.002, 0 failed
progress: 160.0 s, 2649.5 tps, lat 3.773 ms stddev 3.000, 0 failed
progress: 180.0 s, 2627.8 tps, lat 3.804 ms stddev 2.977, 0 failed
progress: 200.0 s, 2576.5 tps, lat 3.880 ms stddev 3.050, 0 failed
progress: 220.0 s, 2733.7 tps, lat 3.656 ms stddev 2.859, 0 failed
progress: 240.0 s, 2983.9 tps, lat 3.350 ms stddev 2.622, 0 failed
progress: 260.0 s, 2870.1 tps, lat 3.483 ms stddev 2.758, 0 failed
progress: 280.0 s, 2736.7 tps, lat 3.652 ms stddev 2.875, 0 failed
progress: 300.0 s, 2937.9 tps, lat 3.402 ms stddev 2.649, 0 failed
progress: 320.0 s, 2852.9 tps, lat 3.504 ms stddev 2.676, 0 failed
progress: 340.0 s, 2671.4 tps, lat 3.742 ms stddev 2.902, 0 failed
progress: 360.0 s, 2574.9 tps, lat 3.882 ms stddev 2.958, 0 failed
progress: 380.0 s, 2557.8 tps, lat 3.908 ms stddev 2.969, 0 failed
progress: 400.0 s, 2579.7 tps, lat 3.875 ms stddev 2.934, 0 failed
progress: 420.0 s, 2464.9 tps, lat 4.055 ms stddev 3.079, 0 failed
progress: 440.0 s, 2419.2 tps, lat 4.132 ms stddev 3.126, 0 failed
progress: 460.0 s, 2430.9 tps, lat 4.112 ms stddev 3.075, 0 failed
progress: 480.0 s, 2431.5 tps, lat 4.111 ms stddev 3.103, 0 failed
progress: 500.0 s, 2460.6 tps, lat 4.063 ms stddev 3.061, 0 failed
progress: 520.0 s, 2398.9 tps, lat 4.167 ms stddev 3.101, 0 failed
progress: 540.0 s, 2521.9 tps, lat 3.964 ms stddev 3.017, 0 failed
progress: 560.0 s, 2521.7 tps, lat 3.964 ms stddev 3.004, 0 failed
progress: 580.0 s, 2561.1 tps, lat 3.903 ms stddev 2.974, 0 failed
progress: 600.0 s, 2698.2 tps, lat 3.705 ms stddev 2.875, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1575215
number of failed transactions: 0 (0.000%)
latency average = 3.807 ms
latency stddev = 2.942 ms
initial connection time = 8.165 ms
tps = 2625.368012 (without initial connection time)
```

4. Проверил lsn и разницу
```bash
testdb=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 1/A8A945F8
(1 row)

testdb=# SELECT pg_wal_lsn_diff('1/A8A945F8', '1/68D14BF8');
 pg_wal_lsn_diff
-----------------
      1071118848
(1 row)

```

### Исходя из полученных данных, на одну контрольную точку в среднем приходится ~54MB

5. Проверяю данные статистики

```bash
testdb=# select num_timed, num_requested, write_time, sync_time, buffers_written from pg_stat_checkpointer;
-[ RECORD 1 ]---+-------
num_timed       | 23
num_requested   | 0
write_time      | 566711
sync_time       | 70
buffers_written | 55439
```

### Судя по статистике, все контрольные точки выполнялись по расписанию, всё Ок


6. Сравниваю **tps** в **синхронном**/**асинхронном** режиме утилитой pgbench

**Синхронный**
```bash
testdb=# alter system set synchronous_commit = on;
ALTER SYSTEM
testdb=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres@ub24-server:~$ pgbench -c 10 -P 10 -j 4 -T 60 testdb
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 2702.9 tps, lat 3.694 ms stddev 2.886, 0 failed
progress: 20.0 s, 2520.5 tps, lat 3.966 ms stddev 2.998, 0 failed
progress: 30.0 s, 2678.5 tps, lat 3.732 ms stddev 2.889, 0 failed
progress: 40.0 s, 2588.7 tps, lat 3.861 ms stddev 2.994, 0 failed
progress: 50.0 s, 2619.5 tps, lat 3.815 ms stddev 2.930, 0 failed
progress: 60.0 s, 3461.9 tps, lat 2.888 ms stddev 2.262, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 165728
number of failed transactions: 0 (0.000%)
latency average = 3.618 ms
latency stddev = 2.837 ms
initial connection time = 9.388 ms
tps = 2762.390512 (without initial connection time)
```

**Асинхронный**
```bash
testdb=# alter system set synchronous_commit = off;
ALTER SYSTEM
testdb=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres@ub24-server:~$ pgbench -c 10 -P 10 -j 4 -T 60 testdb
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 5321.1 tps, lat 1.876 ms stddev 1.594, 0 failed
progress: 20.0 s, 5179.9 tps, lat 1.929 ms stddev 1.609, 0 failed
progress: 30.0 s, 5368.0 tps, lat 1.861 ms stddev 1.570, 0 failed
progress: 40.0 s, 5246.8 tps, lat 1.904 ms stddev 1.585, 0 failed
progress: 50.0 s, 5333.2 tps, lat 1.873 ms stddev 1.512, 0 failed
progress: 60.0 s, 5299.7 tps, lat 1.886 ms stddev 1.523, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 317495
number of failed transactions: 0 (0.000%)
latency average = 1.888 ms
latency stddev = 1.566 ms
initial connection time = 8.279 ms
tps = 5291.998667 (without initial connection time)
```

### В асинхронном режиме tps выше, потому что pg не ждёт записи wal на диск перед подтверждением транзакции


7.  Выполнил порядок действий:
* *Создал новый кластер с включенной контрольной суммой страниц.*
* *Создал таблицу.*
* *Вставил несколько значений.*
* *Выключил кластер.*
* *Изменил пару байт в таблице.*
* *Включил кластер и сделал выборку из таблицы.*

```bash
axelstx@ub24-server:~$ sudo pg_createcluster 17 checks -- --data-checksums
...
```

```bash
postgres@ub24-server:~$ psql -p 5433 -U postgres
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

postgres=# create table test_table (id int, data text);
CREATE TABLE
postgres=# insert into test_table values (1, 'test1'), (2, 'test2'), (3, 'test3');
INSERT 0 3
```

```bash
axelstx@ub24-server:~$ sudo pg_ctlcluster 17 checks stop
```

```bash
axelstx@ub24-server:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/17/checks/base/5/16388 bs=1 count=8 oflag=dsync conv=notrunc
8+0 records in
8+0 records out
8 bytes copied, 0.000804583 s, 9.9 kB/s

axelstx@ub24-server:~$ sudo pg_ctlcluster 17 checks start
```

```bash
postgres=# select * from test_table;
WARNING:  page verification failed, calculated checksum 4225 but expected 25735
ERROR:  invalid page in block 0 of relation base/5/16388
```

### При попытке чтения поврежденной страницы pg обнаружил несоответствие контрольной суммы.

8. Проигнорировать ошибку можно с помощью **ignore_checksum_failure**
```bash
postgres=# set ignore_checksum_failure = on;
SET
postgres=# select * from test_table;
WARNING:  page verification failed, calculated checksum 4225 but expected 25735
 id | data
----+-------
  1 | test1
  2 | test2
  3 | test3
(3 rows)
```
