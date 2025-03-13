--1. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк
```SQL
DROP TABLE IF EXISTS free_text;
CREATE TABLE IF NOT EXISTS free_text(info text); 
--заполняем произвольным символом от a до z
INSERT INTO free_text(info) 
SELECT 
  chr(trunc(random() * 26 + 97)::integer) 
FROM 
  generate_series(1,1e6) AS g(i);
```

--2. Посмотреть размер файла с таблицей
```SQL
SELECT pg_total_relation_size('free_text');
```
--36290560
--здесь и далее TOAST нас не интересует, тк строки слишком короткие



--3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```SQL
BEGIN;
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
END;
```

--4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```SQL
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'free_text';
```
/*
relname        |free_text|
n_dead_tup     |5000000  |
n_live_tup     |1000000  |
last_autovacuum|         |
*/

--5. Подождать некоторое время, проверяя, пришел ли автовакуум
```SQL
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'free_text';
```
/*
relname        |free_text                    |
n_dead_tup     |0                            |
n_live_tup     |1000000                      |
last_autovacuum|2025-03-13 09:06:54.589 +0300|
*/

--6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```SQL
BEGIN;
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
UPDATE free_text SET info = info || chr(trunc(random() * 26 + 97)::integer);
END;
```

--7. Посмотреть размер файла с таблицей
```SQL
SELECT pg_total_relation_size('free_text');
```
--249724928

--8. Отключить Автовакуум на конкретной таблице
```SQL
ALTER TABLE free_text SET (autovacuum_enabled=OFF);
```


--9. 10 раз обновить все строчки и добавить к каждой строчке любой символ
```SQL
UPDATE free_text SET info = info || 'a';
UPDATE free_text SET info = info || 'b';
UPDATE free_text SET info = info || 'c';
UPDATE free_text SET info = info || 'd';
UPDATE free_text SET info = info || 'e';
UPDATE free_text SET info = info || 'a';
UPDATE free_text SET info = info || 'b';
UPDATE free_text SET info = info || 'c';
UPDATE free_text SET info = info || 'd';
UPDATE free_text SET info = info || 'e';
```

--10. Посмотреть размер файла с таблицей
```SQL
SELECT pg_total_relation_size('free_text');
```
--740032512


--11. Объясните полученный результат
В п.4 из-за 5-ти обновлений подряд, размер таблицы вырос ~ в 6 раз, кол-во мертвых строк стало равно 5 миллионам + 1 млн живых строк.
В п.5 пришел вакуум, почистим мертвые строки, но не вернул место ОС, размер таблицы остался прежний
В п.7 после обновлений размер таблицы увеличился, но уже не так значительно, тк после вакуума занимаемое место было переиспользовано.
В п.9 размер таблицы увеличился более чем в 2 раза, тк до обновление место под таблицу было заполнено мертвыми строками и 
каждое обновление создавало новые страницы. 

Общий вывод: при работающем автовакууме (особенно при агрессивных настройках), место занимаемое таблицей может быть переиспользовано, при отключенном - нет

--12. Не забудьте включить автовакуум
```SQL
ALTER TABLE free_text SET (autovacuum_enabled=ON);
```


--Задание со *:
--Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. 
--Не забыть вывести номер шага цикла.
DROP TABLE IF EXISTS free_text;
CREATE TABLE IF NOT EXISTS free_text(info text); 
INSERT INTO free_text(info) SELECT chr(trunc(random() * 26 + 97)::integer) FROM generate_series(1,1e5) AS g(i);

```SQL
DO 
$$
DECLARE 
	step_count int = 10; 
	is_commit_on_every_step boolean = TRUE;
	execution_ts timestamptz;
	i int;
BEGIN 
	RAISE NOTICE '----';
	execution_ts = clock_timestamp();
	RAISE NOTICE '%step_count = % is_commit_on_every_step = %',execution_ts, step_count, is_commit_on_every_step;
	FOR i in 1..step_count LOOP
		UPDATE free_text 
		SET info = info || chr(trunc(random() * 26 + 97)::integer);
	 	IF is_commit_on_every_step THEN
	 		COMMIT;
	 	END IF;
	 	RAISE NOTICE '% step = %',clock_timestamp(), i;
	END LOOP;
	RAISE NOTICE '% total_time = %',clock_timestamp(),clock_timestamp() - execution_ts;	
    RAISE NOTICE '----';
END $$;
```
```
/*
----
2025-03-13 09:33:47.81873+03step_count = 10 is_commit_on_every_step = t
2025-03-13 09:33:48.142399+03 step = 1
2025-03-13 09:33:48.641469+03 step = 2
2025-03-13 09:33:49.256879+03 step = 3
2025-03-13 09:33:49.715521+03 step = 4
2025-03-13 09:33:50.170343+03 step = 5
2025-03-13 09:33:50.690136+03 step = 6
2025-03-13 09:33:51.119556+03 step = 7
2025-03-13 09:33:51.418703+03 step = 8
2025-03-13 09:33:51.891397+03 step = 9
2025-03-13 09:33:52.448419+03 step = 10
2025-03-13 09:33:52.448701+03 total_time = 00:00:04.630013
----*/
```
