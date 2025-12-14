### Домашнее задание

Установка и настройка PostgreSQL

**Цель:**

Создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему  
Переносить содержимое базы данных PostgreSQL на дополнительный диск  
(*) Переносить содержимое БД PostgreSQL между виртуальными машинами 

---

1. Установил виртуальную машину Ubuntu 24 (server) с помощью vmware
2. Выполнил установку Postgresql 17
```bash
axelstx@ub24-server:~$ sudo apt install postgresql-17
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
...
```

3. Проверил что кластер запущен
```bash
axelstx@ub24-server:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

4. Зашел под пользователем postgres
```bash
axelstx@ub24-server:~$ sudo -i -u postgres
```

5. Зашел в psql и создал таблицу с произвольным содержимым
```bash
postgres@ub24-server:~$ psql -U postgres
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=#
```

6. Остановил postgres
```bash
axelstx@ub24-server:~$ sudo systemctl stop postgresql@17-main
```

7. Создал диск размером 10GB. Добавил его к ВМ. Проинициализировал диск согласно инструкции (https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux), создал файловую систему и добавил его в /etc/fstab, чтобы сохранилось монтирование.
```bash
axelstx@ub24-server:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   30G  0 disk
├─sda1   8:1    0    1M  0 part
└─sda2   8:2    0   30G  0 part /
sdb      8:16   0   10G  0 disk
└─sdb1   8:17   0   10G  0 part /mnt/data
sr0     11:0    1  3.1G  0 rom
```

8. Сделал пользователя postgres владельцем /mnt/data
```bash
axelstx@ub24-server:~$ sudo chown -R postgres:postgres /mnt/data/
```
```bash
axelstx@ub24-server:/mnt$ ls -la
total 12
drwxr-xr-x  3 root     root     4096 Dec 14 15:02 .
drwxr-xr-x 23 root     root     4096 Dec 14 12:28 ..
drwxr-xr-x  3 postgres postgres 4096 Dec 14 14:59 data
```

9. Перенес содержимое /var/lib/postgres/17 в /mnt/data/
```bash
axelstx@ub24-server:/var/lib/postgresql/17$ sudo mv /var/lib/postgresql/17 /mnt/data/
```

10. Выполнил
```bash
axelstx@ub24-server:~$ sudo systemctl start postgresql@17-main
```

**Возникла ошибка, потому что postgres ищет данные по старому пути, а не на новом диске**

11. В файле `/etc/postgresql/17/main/postgresql.conf` поменял путь в переменной `data_directory` на новый, и после этого выполнил:
```bash
axelstx@ub24-server:~$ sudo systemctl start postgresql@17-main
```

**Сервис запустился без ошибок, потому что путь в конфиге был изменен на актуальный**

12. Вошел в psql и проверил содержимое ранее созданной таблицы
```bash
postgres@ub24-server:~$ psql -U postgres
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)
```

**Данные на месте**