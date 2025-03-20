--установил, настроил и проверил статус pgbouncer

sergey@szvm:/etc/pgbouncer$ sudo systemctl status pgbouncer
sergey@szvm:/etc/pgbouncer$ sudo systemctl status pgbouncer
● pgbouncer.service - connection pooler for PostgreSQL
     Loaded: loaded (/usr/lib/systemd/system/pgbouncer.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-03-20 17:07:29 MSK; 2s ago
       Docs: man:pgbouncer(1)
             https://www.pgbouncer.org/
   Main PID: 7187 (pgbouncer)
      Tasks: 2 (limit: 4610)
     Memory: 1.2M (peak: 1.5M)
        CPU: 4ms
     CGroup: /system.slice/pgbouncer.service
             └─7187 /usr/sbin/pgbouncer /etc/pgbouncer/pgbouncer.ini

Mar 20 17:07:29 szvm systemd[1]: Starting pgbouncer.service - connection pooler for PostgreSQL...
Mar 20 17:07:29 szvm pgbouncer[7187]: kernel file descriptor limit: 1024 (hard: 524288); max_client_conn: 100, max exp>
Mar 20 17:07:29 szvm pgbouncer[7187]: listening on 0.0.0.0:6432
Mar 20 17:07:29 szvm pgbouncer[7187]: listening on [::]:6432
Mar 20 17:07:29 szvm pgbouncer[7187]: listening on unix:/tmp/.s.PGSQL.6432
Mar 20 17:07:29 szvm pgbouncer[7187]: process up: PgBouncer 1.24.0, libevent 2.1.12-stable (epoll), adns: c-ares 1.27.>
Mar 20 17:07:29 szvm systemd[1]: Started pgbouncer.service - connection pooler for PostgreSQL.

--workload состоит из двух запросов
postgres@szvm:/home/sergey/Desktop$ cat > ~/workload.sql << EOL
\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r + 1;
EOL

--Для каждого режима выполнял по два замера. Режим указывал для БД thai. 
--Перед каждым запуском проверял pool_mode
show pools;
 database  |   user    | cl_active | cl_waiting | cl_active_cancel_req | cl_waiting_cancel_req | sv_active | sv_active_cancel | sv_being_canceled | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us | pool_mode | load_balance_hosts 
-----------+-----------+-----------+------------+----------------------+-----------------------+-----------+------------------+-------------------+---------+---------+-----------+----------+---------+------------+-----------+--------------------
 pgbouncer | pgbouncer |         1 |          0 |                    0 |                     0 |         0 |                0 |                 0 |       0 |       0 |         0 |        0 |       0 |          0 | statement | 
 thai      | postgres  |         8 |          0 |                    0 |                     0 |         8 |                0 |                 0 |       0 |       0 |         0 |        0 |       0 |          0 | statement | 
(2 rows)

Ниже результаты

--8 клиентов

--без баунсера через сокет
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
tps = 16273.880528 (without initial connection time)
tps = 16542.378970 (without initial connection time)

--без баунсера через TCP
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432 -h 127.0.0.1 thai
tps = 11381.273153 (without initial connection time)
tps = 11396.726776 (without initial connection time)

--session
tps = 8958.670160 (without initial connection time)
tps = 8870.608372 (without initial connection time)

--transaction
tps = 7623.708784 (without initial connection time)
tps = 8076.041004 (without initial connection time)

--statement
tps = 8462.582098 (without initial connection time)
tps = 9360.774501 (without initial connection time)

--90 клиентов

--без баунсера через сокет
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 90 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
tps = 10546.402175 (without initial connection time)
tps = 10086.295884 (without initial connection time)

--без баунсера через TCP
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 90 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432 -h 127.0.0.1 thai
tps = 6751.358790 (without initial connection time)
tps = 6834.600781 (without initial connection time)

--session
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 90 -j 4 -T 10 -f ~/workload.sql -n -U postgres -d thai -h 127.0.0.1 -p 6432
tps = 8675.020649 (without initial connection time)
tps = 9407.656680 (without initial connection time)

--transaction
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 90 -j 4 -T 10 -f ~/workload.sql -n -U postgres -d thai -h 127.0.0.1 -p 6432
tps = 6314.139800 (without initial connection time)
tps = 7367.251321 (without initial connection time)

--statement
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 90 -j 4 -T 10 -f ~/workload.sql -n -U postgres -d thai -h 127.0.0.1 -p 6432
tps = 6727.697015 (without initial connection time)
tps = 7335.724155 (without initial connection time)


