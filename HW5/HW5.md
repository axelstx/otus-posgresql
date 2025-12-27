## Домашнее задание

Настройка PostgreSQL

**Цель:**
* поработать с параметрами конфигурации PostgreSQL  
* понимать разницу между различными группами параметров  
* выбирать оптимальное значение для параметров

---

VM: Ubuntu 24 server, PostgreSQL17, CPU = 8 cores, RAM = 8GB.

1. Выполнил инициализацию pgbench
```bash
postgres@ub24-server:~$ pgbench -i testdb
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.13 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.06 s, vacuum 0.04 s, primary keys 0.03 s).
```

2.  Запуск бенчмарка без настроек
```bash
postgres@ub24-server:~$ pgbench -c 50 -j 2 -P 10 -T 60 testdb
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 1855.5 tps, lat 26.705 ms stddev 34.217, 0 failed
progress: 20.0 s, 1898.6 tps, lat 26.342 ms stddev 36.470, 0 failed
progress: 30.0 s, 1853.8 tps, lat 26.883 ms stddev 34.829, 0 failed
progress: 40.0 s, 1830.0 tps, lat 27.372 ms stddev 36.542, 0 failed
progress: 50.0 s, 1901.3 tps, lat 26.307 ms stddev 32.359, 0 failed
progress: 60.0 s, 1920.1 tps, lat 26.016 ms stddev 33.708, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 112642
number of failed transactions: 0 (0.000%)
latency average = 26.618 ms
latency stddev = 34.744 ms
initial connection time = 54.912 ms
tps = 1877.120138 (without initial connection time)
```

3. **CPU.** Изменил значения **max_parallel_workers_per_gather** и **max_parallel_maintenance_workers** на рекомендуемые для моего конфига и перезапустил кластер
```bash
postgres=# show max_worker_processes;
 max_worker_processes
----------------------
 8
(1 row)

postgres=# show max_parallel_workers;
 max_parallel_workers
----------------------
 8
(1 row)

postgres=# show max_parallel_workers_per_gather;
 max_parallel_workers_per_gather
---------------------------------
 2
(1 row)

postgres=# show max_parallel_maintenance_workers;
 max_parallel_maintenance_workers
----------------------------------
 2
(1 row)

postgres=# alter system set max_parallel_workers_per_gather = 4;
ALTER SYSTEM
postgres=# alter system set max_parallel_maintenance_workers = 4;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

```

4. Запустил тест. Видим что tps умеренно вырос
```bash
postgres@ub24-server:~$ pgbench -c 50 -j 2 -P 10 -T 60 testdb
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 2041.3 tps, lat 24.257 ms stddev 31.379, 0 failed
progress: 20.0 s, 2015.2 tps, lat 24.853 ms stddev 30.425, 0 failed
progress: 30.0 s, 2148.5 tps, lat 23.267 ms stddev 27.848, 0 failed
progress: 40.0 s, 2225.6 tps, lat 22.458 ms stddev 27.199, 0 failed
progress: 50.0 s, 2284.8 tps, lat 21.876 ms stddev 25.184, 0 failed
progress: 60.0 s, 2230.7 tps, lat 22.421 ms stddev 27.939, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 129510
number of failed transactions: 0 (0.000%)
latency average = 23.147 ms
latency stddev = 28.333 ms
initial connection time = 53.854 ms
tps = 2159.024895 (without initial connection time)
```

5. **RAM: Shared memory.** Изменил значения **shared_buffers** и **effective_cache_size** на рекомендуемые и перезапустил кластер
```bash
postgres=# show shared_buffers;
 shared_buffers
----------------
 128MB
(1 row)

postgres=# show effective_cache_size;
 effective_cache_size
----------------------
 4GB
(1 row)

postgres=# alter system set shared_buffers = '1024MB';
ALTER SYSTEM
postgres=# alter system set effective_cache_size = '6GB';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

6. Запустил тест. Видим что tps даже снизился
```bash
postgres@ub24-server:~$ pgbench -c 50 -j 2 -P 10 -T 60 testdb
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 1921.2 tps, lat 25.837 ms stddev 30.905, 0 failed
progress: 20.0 s, 1958.3 tps, lat 25.528 ms stddev 31.724, 0 failed
progress: 30.0 s, 1988.9 tps, lat 25.077 ms stddev 29.713, 0 failed
progress: 40.0 s, 1995.9 tps, lat 25.094 ms stddev 30.302, 0 failed
progress: 50.0 s, 1923.2 tps, lat 25.965 ms stddev 30.823, 0 failed
progress: 60.0 s, 1857.2 tps, lat 26.856 ms stddev 32.951, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 116497
number of failed transactions: 0 (0.000%)
latency average = 25.731 ms
latency stddev = 31.121 ms
initial connection time = 56.014 ms
tps = 1942.240935 (without initial connection time)
```

7. **RAM: Backend memory.** Изменил значения **temp_buffers**, **work_mem** и **maintenance_work_mem** на рекомендуемые и перезапустил кластер
```bash
postgres=# show temp_buffers;
 temp_buffers
--------------
 8MB
(1 row)

postgres=# show work_mem;
 work_mem
----------
 4MB
(1 row)

postgres=# show maintenance_work_mem;
 maintenance_work_mem
----------------------
 64MB
(1 row)

postgres=# alter system set work_mem = '32MB';
ALTER SYSTEM
postgres=# alter system set maintenance_work_mem = '320MB';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

8. Запустил тест. Видим что tps немного вырос
```bash
postgres@ub24-server:~$ pgbench -c 50 -j 2 -P 10 -T 60 testdb -U postgres
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 2226.3 tps, lat 22.259 ms stddev 27.123, 0 failed
progress: 20.0 s, 2059.8 tps, lat 24.255 ms stddev 29.564, 0 failed
progress: 30.0 s, 2094.8 tps, lat 23.883 ms stddev 29.503, 0 failed
progress: 40.0 s, 2157.9 tps, lat 23.182 ms stddev 28.820, 0 failed
progress: 50.0 s, 2097.8 tps, lat 23.798 ms stddev 29.115, 0 failed
progress: 60.0 s, 2102.7 tps, lat 23.788 ms stddev 29.159, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 127443
number of failed transactions: 0 (0.000%)
latency average = 23.518 ms
latency stddev = 28.883 ms
initial connection time = 60.965 ms
tps = 2125.022304 (without initial connection time)
```

9. **Настройки дисковой подсистемы.** Выключил **fsync** и **synchronous_commit** и перезапустил кластер
```bash
postgres=# show fsync;
 fsync
-------
 on
(1 row)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

postgres=# alter system set fsync = '0';
ALTER SYSTEM
postgres=# alter system set synchronous_commit = '0';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

10. Запустил тест. Видим, что tps **значительно** вырос, практически в **2** раза по сравнению с базовым
```bash
postgres@ub24-server:~$ pgbench -c 50 -j 2 -P 10 -T 60 testdb -U postgres
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 3534.3 tps, lat 14.030 ms stddev 17.803, 0 failed
progress: 20.0 s, 3475.5 tps, lat 14.375 ms stddev 17.818, 0 failed
progress: 30.0 s, 3642.8 tps, lat 13.733 ms stddev 17.555, 0 failed
progress: 40.0 s, 3601.3 tps, lat 13.867 ms stddev 17.421, 0 failed
progress: 50.0 s, 3660.1 tps, lat 13.653 ms stddev 17.598, 0 failed
progress: 60.0 s, 3476.6 tps, lat 14.385 ms stddev 18.407, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 213955
number of failed transactions: 0 (0.000%)
latency average = 14.006 ms
latency stddev = 17.772 ms
initial connection time = 60.167 ms
tps = 3567.740912 (without initial connection time)
```

**Вывод:
Изменение настроек CPU и RAM не дало заметного прироста производительности, что объясняется характером нагрузки в тесте pgbench 
и небольшим размером тестовой базы. Отключение fsync и synchronous_commit, дало резкий скачок TPS за счёт снижения затрат 
на дисковую запись, но одновременно уменьшило гарантии надежности и сохранности данных при сбоях.**