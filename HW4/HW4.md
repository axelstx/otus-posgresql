## Домашнее задание

Работа с базами данных, пользователями и правами

**Цель:**
* создание новой базы данных, схемы и таблицы
* создание роли для чтения данных из созданной схемы созданной базы данных
* создание роли для чтения и записи из созданной схемы созданной базы данных

---

1. Зашел под пользователя **postgres** и подключился к **Postgres 17**
```bash
postgres@ub24-server:~$ psql -U postgres
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.
```

2. Зашел в **psql** под **postgres**
* создал БД **testdb**
* зашёл в **testdb** под **postgres**
* создал cхему **testnm**
* создал таблицу **t1** с одной колонкой **c1** типа **integer**
* вставил строку со значением **c1=1**
```bash
postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb postgres
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table t1 (c1 integer);
CREATE TABLE
testdb=# insert into t1 values (1);
INSERT 0 1
testdb=#
```

3. Создал новую роль **readonly**
* дал новой роли право на подключение к базе данных **testdb**
* дал новой роли право на использование схемы **testnm**
* дал новой роли право на **select** для всех таблиц схемы **testnm**
* дал новой роли право на **select** для всех **будущих таблиц**
```bash
testdb=# create role readonly nologin;
CREATE ROLE
testdb=# grant connect on database testdb to readonly;
GRANT
testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
testdb=#
```

**Дал новой роли право на select для всех будущих таблиц**

4. Создал пользователя **testread** с паролем **test123** и дал роль **readonly** пользователю **testread**
```bash
testdb=# create user testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
testdb=#
```

5.  Зашел под пользователем **testread** в базу данных testdb
```bash
postgres@ub24-server:~$ psql -U testread -d testdb -h localhost
Password for user testread:
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
```

6. Сделал select * from t1;
```bash
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=>
```

**Таблица в схеме public, а права были выданы на testnm**

7.  Зашел в **testdb** под **postgres**
```bash
postgres@ub24-server:~$ psql -U postgres -d testdb
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

testdb=#
```

8.  Удалил таблицу **t1**, а потом создал ее в схеме **testnm** и вставил туда значение
```bash
testdb=# drop table t1;
DROP TABLE
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
testdb=# insert into testnm.t1 values (1);
INSERT 0 1
testdb=#
```

9. Зашел под пользователем **testread** в базу данных **testdb** и выполнил **select**
```bash
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```

**Всё ок!**

10.  При выполнении **create table t2(c1 integer);** возникла **ошибка**, потому что у пользователя *нет наследуемых прав на создание по умолчанию* (начиная с версии **PostgreSQL15**)
```bash
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
```

Права можно выдать с помощью команды
```bash
grant create on schema testnm to readonly;
```

И соответственно создать таблицу в схеме **testnm**:
```bash
testdb=> create table testnm.t2(c1 integer);
CREATE TABLE
```

Для того чтобы забрать права на создание, нужно выполнить:
```bash
revoke create on schema testnm from readonly;
```