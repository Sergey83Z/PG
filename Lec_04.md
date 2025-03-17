postgres=# CREATE TABLE accounts(id integer, amount numeric);
CREATE TABLE
postgres=# INSERT INTO accounts(id,amount) values(1,100),(2,200),(3,300),(4,400),(5,500);
INSERT 0 5
postgres=# SELECT deadlocks FROM pg_stat_database WHERE datname = current_database();
 deadlocks 
-----------
         0
(1 row)
postgres=# CREATE OR REPLACE PROCEDURE sp_deadlock(p_delay_sec int,p_is_order_asc bool)
LANGUAGE plpgsql
AS 
$sp_deadlock$
DECLARE
        rs record; 
BEGIN
        RAISE NOTICE '----';
        RAISE NOTICE 'p_delay = % p_order = %', p_delay_sec, p_is_order_asc;
        FOR rs IN EXECUTE(FORMAT('SELECT id, amount FROM accounts ORDER BY ID %s', CASE WHEN p_is_order_asc THEN 'ASC' ELSE 'DESC' END))
        LOOP
                UPDATE 
                        accounts 
                SET 
                        amount = amount + 100
                WHERE 
                        id = rs.id;
                PERFORM pg_sleep(p_delay_sec);
                RAISE NOTICE 'id= %', rs.id;
        END LOOP;
    RAISE NOTICE '----';
END;
$sp_deadlock$
;
CREATE PROCEDURE


postgres=# CALL sp_deadlock(2,true);
NOTICE:  ----
NOTICE:  p_delay = 2 p_order = t
NOTICE:  id= 1
NOTICE:  id= 2
NOTICE:  id= 3
ERROR:  deadlock detected
DETAIL:  Process 8379 waits for ShareLock on transaction 10867; blocked by process 8396.
Process 8396 waits for ShareLock on transaction 10866; blocked by process 8379.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,4) in relation "accounts"
SQL statement "UPDATE 
			accounts 
		SET 
			amount = amount + 100
		WHERE 
			id = rs.id"
PL/pgSQL function sp_deadlock(integer,boolean) line 9 at SQL statement

postgres=# CALL sp_deadlock(2,false);
NOTICE:  ----
NOTICE:  p_delay = 2 p_order = f
NOTICE:  id= 5
NOTICE:  id= 4
NOTICE:  id= 3
NOTICE:  id= 2
NOTICE:  id= 1
NOTICE:  ----
CALL
postgres=# 

postgres=# SELECT deadlocks FROM pg_stat_database WHERE datname = current_database();
 deadlocks 
-----------
         1
(1 row)

postgres@szvm:/home/sergey/Desktop$ tail /var/log/postgresql/postgresql-17-main.log
	SQL statement "UPDATE 
				accounts 
			SET 
				amount = amount + 100
			WHERE 
				id = rs.id"
	PL/pgSQL function sp_deadlock(integer,boolean) line 9 at SQL statement
2025-03-17 14:49:37.362 MSK [8379] postgres@postgres STATEMENT:  CALL sp_deadlock(2,true);
2025-03-17 14:50:13.229 MSK [1153] LOG:  checkpoint starting: time
2025-03-17 14:50:16.385 MSK [1153] LOG:  checkpoint complete: wrote 32 buffers (0.2%); 0 WAL file(s) added, 0 removed, 0 recycled; write=3.120 s, sync=0.027 s, total=3.156 s; sync files=27, longest=0.005 s, average=0.001 s; distance=126 kB, estimate=126 kB; lsn=0/1EE450F8, redo lsn=0/1EE450A0


