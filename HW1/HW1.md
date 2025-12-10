### Домашнее задание

Работа с уровнями изоляции транзакции в PostgreSQL

**Цель:**  
Научиться управлять уровнем изоляции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read.

---

1. Установка postgresql в docker c помощью docker-compose. Конфиг:
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
2. Сбилдил и поднял контейнер с помощью: 

```bash
docker-compose build
```
```bash
docker-compose up -d
```

3. Зашел в контейнер под разными сессиями из 2х терминалов:
```bash
docker exec -it postgesql_postgres_1 bash
```
4. В каждом ввёл:
```bash
psql -h localhost -U otususer -d otusdb
```
5. Выключил autocommit и проверил, что он выключился:
```
\set AUTOCOMMIT off
```
```
\echo :AUTOCOMMIT
```
6. В первой сессии создал таблицу и наполнил её данными:
```sql
create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```
7. Посмотрел уровень изоляции:
```
otusdb=# show transaction isolation level;
transaction_isolation
-----------------------
read committed
(1 row)
```
8. В первой сессии добавил новую запись
```
otusdb=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
9. Выполнил запрос во второй сессии:
```
otusdb=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

**Новая запись отсутствует, потому что в первой сессии не закончена транзакция.**

10. Выполнил коммит в первой сессии:
```
otusdb=*# commit;
COMMIT
```
11. Сделал запрос во второй сессии:
```
otusdb=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
**Теперь новая запись видна, так как транзакция в перовй сессии завершилась.**

12. Запустил в сессиях новые транзакции. Поставил уровень repeatable read.
```
otusdb=*# set transaction isolation level repeatable read;
SET
```
13. В первой сессии добавил новую запись:
```
otusdb=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
14. Во второй сессии выполнил:
```
otusdb=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

**Запись не видна, так как стоит уровень изоляции repeatable read**

15. В первой сессии выполнил коммит тем самым завершив транзакцию:
```
otusdb=*# commit;
COMMIT
```
16. Во второй сессии выполнил:
```
otusdb=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

**Запись также не видна.**

17. Во второй сессии завершил транзакцию и выполнил select:
```
otusdb=*# commit;
COMMIT

otusdb=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

**После успешного завершения транзакции и во второй сессии, запись стала видна.**