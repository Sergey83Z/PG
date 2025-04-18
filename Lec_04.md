--Создать таблицу accounts(id integer, amount numeric);
CREATE TABLE accounts(id integer, amount numeric);

--Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).
INSERT INTO accounts(id,amount) values(1,100),(2,200),(3,300),(4,400),(5,500);

--Смотрим счетчик deadlocks по нашей БД
postgres=# SELECT deadlocks FROM pg_stat_database WHERE datname = current_database();
 deadlocks 
-----------
         0

--Создаем ХП, которая обновляет записи в разном порядке с указаной задержкой
CREATE OR REPLACE PROCEDURE sp_deadlock(p_delay_sec int,p_is_order_asc bool)
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

--Запускаем в разном порядке
--терминал 1
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
PL/pgSQL function sp_deadlock(integer,boolean) line 9 at SQL STATEMENT

--терминал 2
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

--Посмотреть логи и убедиться, что информация о дедлоке туда попала.
--Снова смотрим счетчик deadlocks по нашей БД
postgres=# SELECT deadlocks FROM pg_stat_database WHERE datname = current_database();
 deadlocks 
-----------
         1
    
--В логе
postgres@szvm:/home/sergey/Desktop$ tail /var/log/postgresql/postgresql-17-main.log
	SQL statement "UPDATE 
				accounts 
			SET 
				amount = amount + 100
			WHERE 
				id = rs.id"
	PL/pgSQL function sp_deadlock(integer,boolean) line 9 at SQL statement
2025-03-17 14:49:37.362 MSK [8379] postgres@postgres STATEMENT:  CALL sp_deadlock(2,true);


