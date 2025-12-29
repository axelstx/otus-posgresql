## ## Домашнее задание

Настройка autovacuum с учетом особенностей производительности

**Цель:**
* запустить нагрузочный тест pgbench;
* настроить параметры autovacuum;
* проверить работу autovacuum;

---
Создал инстанс ВМ с 2 ядрами / 4 Гб ОЗУ / SSD 10GB и установил PostgreSQL17.

1.  Выполнил `pgbench -i postgres`
```bash
postgres@ub24-server:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) of pgbench_accounts done (elapsed 0.06 s, remaining 0.0                                                                                      
vacuuming...
creating primary keys...
done in 0.20 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.09 s, vacuum 0.06 s, primary keys 0.03 s).
```

2. Запустил тест производительности pgbench -c8 -P 6 -T 60 -U postgres postgres
```bash

postgres@ub24-server:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1418.8 tps, lat 5.609 ms stddev 5.141, 0 failed
progress: 12.0 s, 1363.0 tps, lat 5.863 ms stddev 6.016, 0 failed
progress: 18.0 s, 1253.7 tps, lat 6.366 ms stddev 6.069, 0 failed
progress: 24.0 s, 1358.4 tps, lat 5.876 ms stddev 6.897, 0 failed
progress: 30.0 s, 1429.4 tps, lat 5.592 ms stddev 5.894, 0 failed
progress: 36.0 s, 1381.1 tps, lat 5.780 ms stddev 5.608, 0 failed
progress: 42.0 s, 1384.3 tps, lat 5.774 ms stddev 5.476, 0 failed
progress: 48.0 s, 1288.2 tps, lat 6.202 ms stddev 5.882, 0 failed
progress: 54.0 s, 1319.3 tps, lat 6.051 ms stddev 5.681, 0 failed
progress: 60.0 s, 1451.1 tps, lat 5.501 ms stddev 5.541, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 81892
number of failed transactions: 0 (0.000%)
latency average = 5.850 ms
latency stddev = 5.836 ms
initial connection time = 16.621 ms
tps = 1364.998935 (without initial connection time)
```

3. Применил параметры настройки PostgreSQL из прикрепленного к материалам занятия файла, перечитал конфиг и перезапустил кластер.  
Выполнил тест ещё раз
```sql
postgres=# alter system set max_connections = 40;
ALTER SYSTEM
postgres=# alter system set shared_buffers = '1GB';
ALTER SYSTEM
postgres=# alter system set effective_cache_size = '3GB';
ALTER SYSTEM
postgres=# alter system set maintenance_work_mem = '512MB';
ALTER SYSTEM
postgres=# alter system set checkpoint_completion_target = 0.9;
ALTER SYSTEM
postgres=# alter system set wal_buffers = '16MB';
ALTER SYSTEM
postgres=# alter system set default_statistics_target = 500;
ALTER SYSTEM
postgres=# alter system set random_page_cost = 4;
ALTER SYSTEM
postgres=# alter system set effective_io_concurrency = 2;
ALTER SYSTEM
postgres=# alter system set work_mem = '6553kB';
ALTER SYSTEM
postgres=# alter system set min_wal_size = '4GB';
ALTER SYSTEM
postgres=# alter system set max_wal_size = '16GB';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres@ub24-server:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1367.2 tps, lat 5.823 ms stddev 5.202, 0 failed
progress: 12.0 s, 1316.9 tps, lat 6.048 ms stddev 5.615, 0 failed
progress: 18.0 s, 1311.9 tps, lat 6.100 ms stddev 5.271, 0 failed
progress: 24.0 s, 1257.2 tps, lat 6.351 ms stddev 6.151, 0 failed
progress: 30.0 s, 1121.6 tps, lat 7.121 ms stddev 6.534, 0 failed
progress: 36.0 s, 1168.7 tps, lat 6.820 ms stddev 5.963, 0 failed
progress: 42.0 s, 1215.0 tps, lat 6.586 ms stddev 6.473, 0 failed
progress: 48.0 s, 1294.0 tps, lat 6.171 ms stddev 5.881, 0 failed
progress: 54.0 s, 1350.8 tps, lat 5.912 ms stddev 5.398, 0 failed
progress: 60.0 s, 1372.2 tps, lat 5.819 ms stddev 5.184, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 76661
number of failed transactions: 0 (0.000%)
latency average = 6.249 ms
latency stddev = 5.775 ms
initial connection time = 17.287 ms
tps = 1277.571652 (without initial connection time)
```

### TPS немного упал

4. Изменил настройки **random_page_cost** и **effective_io_concurrency**, для большего соответствия параметрам своему  
жесткому диску (*у меня **ssd**, в файле из задания настройки для **hdd***), а также понизил значения **min_wal_size**  
и **max_wal_size**, потому что они не соответствовали адекватным значениям с учетом того, что в задании максимальный  
объем диска **10GB**.

```sql
postgres=# alter system set min_wal_size = '512MB';
ALTER SYSTEM
postgres=# alter system set max_wal_size = '2GB';
ALTER SYSTEM
postgres=# alter system set random_page_cost = 1.1;
ALTER SYSTEM
postgres=# alter system set effective_io_concurrency = 200;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres@ub24-server:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1251.6 tps, lat 6.351 ms stddev 5.371, 0 failed
progress: 12.0 s, 1785.5 tps, lat 4.483 ms stddev 5.551, 0 failed
progress: 18.0 s, 1556.6 tps, lat 5.127 ms stddev 6.236, 0 failed
progress: 24.0 s, 1155.4 tps, lat 6.915 ms stddev 6.704, 0 failed
progress: 30.0 s, 1485.1 tps, lat 5.382 ms stddev 5.928, 0 failed
progress: 36.0 s, 1571.9 tps, lat 5.085 ms stddev 6.412, 0 failed
progress: 42.0 s, 1823.8 tps, lat 4.380 ms stddev 5.509, 0 failed
progress: 48.0 s, 1411.4 tps, lat 5.647 ms stddev 6.900, 0 failed
progress: 54.0 s, 1583.1 tps, lat 5.059 ms stddev 6.492, 0 failed
progress: 60.0 s, 1747.6 tps, lat 4.569 ms stddev 5.354, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 92242
number of failed transactions: 0 (0.000%)
latency average = 5.197 ms
latency stddev = 6.082 ms
initial connection time = 18.165 ms
tps = 1536.915223 (without initial connection time)
```


### После очередного теста, tps вырос


5.  Создал таблицу с текстовым полем и заполнить случайными  данным в размере 1млн строк

```bash
postgres=# create table test_table (id serial primary key, text_field text);
CREATE TABLE
postgres=# insert into test_table (text_field) select md5(random()::text) from generate_series(1, 1000000);
INSERT 0 1000000
```

6. Посмотрел её размер
```bash
postgres=# select pg_size_pretty(pg_relation_size('test_table')) as table_size;
 table_size
------------
 65 MB
(1 row)
```

7. Пять раз обновил все строчки и добавил к каждой строчке символ. Посмотрел количество мертвых строчек в таблице и когда последний раз приходил autovacuum. Спустя время ещё раз проверил autovacuum.
```bash
postgres=# update test_table set text_field = text_field || 'a';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'b';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'c';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'd';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'e';
UPDATE 1000000
postgres=# select n_live_tup, n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 'test_table';
 n_live_tup | n_dead_tup |       last_autovacuum
------------+------------+------------------------------
    1000000 |    1000000 | 2025-12-29 16:52:31.181782+00
(1 row)

postgres=# select n_live_tup, n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 'test_table';
 n_live_tup | n_dead_tup |       last_autovacuum
------------+------------+------------------------------
    1000000 |          0 | 2025-12-29 16:53:30.74047+00
(1 row)

```

8. Пять раз обновил все строчки и добавил к каждой строчке символ. Посмотрел размер файла с таблицей и отключил autovacuum на конкретной таблице.
```bash
postgres=# update test_table set text_field = text_field || 'f';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'g';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'h';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'i';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'j';
UPDATE 1000000
postgres=# select pg_size_pretty(pg_relation_size('test_table')) as table_size;
 table_size
------------
 365 MB
(1 row)

postgres=# alter table test_table set (autovacuum_enabled = false);
ALTER TABLE
```

9. Десять раз обновил все строчки и добавить к каждой строчке символ, а потом посмотрел размер файла с таблицей, а также мертвые строки
```bash
postgres=# update test_table set text_field = text_field || 'k';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'l';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'm';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'n';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'o';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'p';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'q';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 'r';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 's';
UPDATE 1000000
postgres=# update test_table set text_field = text_field || 't';
UPDATE 1000000
postgres=# select pg_size_pretty(pg_relation_size('test_table')) as table_size;
 table_size
------------
 879 MB
(1 row)

postgres=# select n_live_tup, n_dead_tup from pg_stat_user_tables where relname = 'test_table';
 n_live_tup | n_dead_tup
------------+------------
    1000000 |    9997763
(1 row)
```


### Когда avtovacuum включен, тогда размер таблицы растет медленно, т.к avtovacuum периодически очищает мертвые строки. При выключенном avtovacuum, каждое обновление создает новую копию строки, что мы наблюдаем в выводе терминала