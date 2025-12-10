### Домашнее задание

Установка и настройка PostgteSQL в контейнере Docker

**Цель:**  
Установить PostgreSQL в Docker контейнере  
Настроить контейнер для внешнего подключения

---

1. Установка postgresql в docker c помощью docker-compose на виртуальной машине с Ubuntu 24(vmware). Конфиг:
```
version: "3.9"
services:
postgres:
image: postgres:17.7-alpine3.23
ports:
- "5432:5432"
environment:
POSTGRES_USER: ${POSTGRES_USER}
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
POSTGRES_DB: ${POSTGRES_DB}
volumes:
- pgdata:/var/lib/postgresql/data
restart: unless-stopped

volumes:
pgdata:
```
**В папку pgdata с помощью volumes пробрасываются данные из /var/lib/postgresql**

2. Сбилдил и поднял контейнер с помощью:

```bash
docker-compose build
```
```bash
docker-compose up -d
```
>Creating network "postgesql_default" with the default driver
> 
>Creating postgesql_postgres_1 ... done

3. Зашел в контейнер:
```bash
docker exec -it postgesql_postgres_1 bash
```
```
alex@ub24:~/OTUS/PostgeSQL$ docker exec -it postgesql_postgres_1 bash
2d3474d5b2d0:/#
```

4. Подключился к БД:
```bash
psql -h localhost -U otususer -d otusdb
```
```
2d3474d5b2d0:/# psql -h localhost -U otususer -d otusdb
psql (17.7)
Type "help" for help.

otusdb=#
```

5. Создал таблицу и внес в неё запись:
```sql
create table test_table(id serial, name text); insert into test_table(name) values('test');
```
```
otusdb=# create table test_table(id serial, name text); insert into test_table(name) values('test');
CREATE TABLE
INSERT 0 1

otusdb=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner   
--------+------------+-------+----------
 public | persons    | table | otususer
 public | test_table | table | otususer
(2 rows)

otusdb=# select * from test_table;
 id | name 
----+------
  1 | test
(1 row)
```

6. Подключение к контейнеру postgresql 17 на виртуальной машине c Ubuntu 24(vmware)
> На хосте с windows 11 установил pgAdmin 4
> 
> В терминале виртуальной машины набрал команду - hostname -I, которая выдала ip адрес
> 
> Используя данный ip и данные для подключения(ip, port, user, password, db) c помощью psql tool workspace(pgAdmin 4)
> выполнил подключение к БД в контейнере docker на виртуальной машине с Ubuntu 24
> 
> Подключение прошло успешно

7. Удаление и создание контейнера:
```
alex@ub24:~/OTUS/PostgeSQL$ docker ps -a
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS                     PORTS     NAMES
9bcdae3b432b   postgres:17.7-alpine3.23   "docker-entrypoint.s…"   14 seconds ago   Exited (0) 4 seconds ago             postgesql_postgres_1

alex@ub24:~/OTUS/PostgeSQL$ docker rm 9bcdae3b432b
9bcdae3b432b

alex@ub24:~/OTUS/PostgeSQL$ docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                   PORTS     NAMES

alex@ub24:~/OTUS/PostgeSQL$ docker-compose up -d
Creating postgesql_postgres_1 ... done
```
8. Проверка данных:
```
alex@ub24:~/OTUS/PostgeSQL$ docker exec -it postgesql_postgres_1 bash
0ab9bcd47188:/# psql -h localhost -U otususer -d otusdb
psql (17.7)
Type "help" for help.

otusdb=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner   
--------+------------+-------+----------
 public | persons    | table | otususer
 public | test_table | table | otususer
(2 rows)

otusdb=# select * from test_table;
 id | name 
----+------
  1 | test
(1 row)

otusdb=# 
```

**Все данные остались на месте**