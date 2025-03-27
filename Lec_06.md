--Делал на одной виртуалке
--Создал кластер для реплики
sudo pg_createcluster 17 slave

pg_lsclusters 
Ver Cluster Port Status Owner    Data directory               Log file
17  main    5432 online postgres /var/lib/postgresql/17/main  /var/log/postgresql/postgresql-17-main.log
17  slave   5433 online postgres /var/lib/postgresql/17/slave /var/log/postgresql/postgresql-17-slave.log

--Остановил slave
sudo pg_ctlcluster 17 slave stop

--Создал пользователя и слот репликации
postgres=# CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret123';
CREATE ROLE
postgres=# SELECT pg_create_physical_replication_slot('test');
 pg_create_physical_replication_slot 

--удалил рабочую дирректорию, запустил pg_basebackup и поправил права
rm -rf /var/lib/postgresql/17/slave
sudo pg_basebackup -h localhost -p 5432 -U replicator -R -S test -D /var/lib/postgresql/17/slave
sudo chown -R postgres:postgres /var/lib/postgresql/17/slave


--тесты, взял с лекции 
postgres@szvm:/home/sergey/Desktop$ cat > ~/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
        ceil(random()*100)
        , (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));

EOL
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Ubuntu 17.4-1.pgdg24.04+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 43278
number of failed transactions: 0 (0.000%)
latency average = 1.843 ms
initial connection time = 44.298 ms
tps = 4341.834546 (without initial connection time)
postgres@szvm:/home/sergey/Desktop$ cat > ~/workload.sql << EOL

\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

EOL
postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (17.4 (Ubuntu 17.4-1.pgdg24.04+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 291030
number of failed transactions: 0 (0.000%)
latency average = 0.275 ms
initial connection time = 24.365 ms
tps = 29141.379196 (without initial connection time)


sudo pg_ctlcluster 17 slave start

postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Ubuntu 17.4-1.pgdg24.04+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 31958
number of failed transactions: 0 (0.000%)
latency average = 2.503 ms
initial connection time = 15.783 ms
tps = 3195.709242 (without initial connection time)

postgres@szvm:/home/sergey/Desktop$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (17.4 (Ubuntu 17.4-1.pgdg24.04+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 310509
number of failed transactions: 0 (0.000%)
latency average = 0.257 ms
initial connection time = 23.451 ms
tps = 31081.400178 (without initial connection time)


--На запись tps снизилось с 4341 до 3195
--на чтение увеличилось! Скорее всего помогло кеширование. 


На всякий случай проверил режим режим работы
psql -c "SHOW synchronous_commit;"
 synchronous_commit 
--------------------
 on
 
postgres=# SELECT * FROM pg_stat_replication \gx
...
sync_state       | async
