Развернул VM на VirtualBox, установил Ubuntu

Установил PG 17.4

Залил перевозки

Выполнил SELECT

```
postgres@szvm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
postgres@szvm:~$ psql
psql (17.4 (Ubuntu 17.4-1.pgdg24.04+2))
Type "help" for help.

postgres=# \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 thai      | postgres | UTF8     | libc            | C.UTF-8     | C.UTF-8     |        |           | 
(4 rows)

postgres=# \c thai 
You are now connected to database "thai" as user "postgres".
thai=# select count(*) from book.tickets;
  count  
---------
 5185505
(1 row)

thai=# 
```
