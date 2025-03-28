--отключим jit компиляцию, чтобы не мешалась
alter system set jit = off;
select pg_reload_conf();


--смотрим план выполнения
--таблицы без индексов на fk считывают последовательно (seq scan)
--время выполнения чуть менее 2000ms

postgres@szvm:/home/sergey/Desktop$ psql -d thai -c "explain (analyze, verbose, buffers, settings)
WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,  
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;">>/home/postgres/1.sql



 ?column? 
----------
        1
(1 row)

                                                                                                    QUERY PLAN                                                                                                    
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=333761.74..333761.77 rows=10 width=56) (actual time=1951.538..1951.635 rows=10 loops=1)
   Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
   Buffers: shared hit=10282 read=50789, temp read=11467 written=19003
   ->  Sort  (cost=333761.74..334126.55 rows=145923 width=56) (actual time=1951.537..1951.632 rows=10 loops=1)
         Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         Buffers: shared hit=10282 read=50789, temp read=11467 written=19003
         ->  Group  (cost=328054.75..330608.40 rows=145923 width=56) (actual time=1911.125..1934.936 rows=144000 loops=1)
               Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               Buffers: shared hit=10279 read=50789, temp read=11467 written=19003
               ->  Sort  (cost=328054.75..328419.55 rows=145923 width=56) (actual time=1911.120..1919.187 rows=144000 loops=1)
                     Output: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id)), r.startdate
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7864kB
                     Buffers: shared hit=10279 read=50789, temp read=11467 written=19003
                     ->  Hash Join  (cost=261134.57..310547.31 rows=145923 width=56) (actual time=1641.615..1885.280 rows=144000 loops=1)
                           Output: r.id, ((bs.city || ', '::text) || bs.name), (count(t.id)), (count(s_1.id)), r.startdate
                           Inner Unique: true
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           Buffers: shared hit=10273 read=50789, temp read=10484 written=18017
                           ->  Nested Loop  (cost=261129.46..309104.85 rows=145923 width=84) (actual time=1641.563..1861.847 rows=144000 loops=1)
                                 Output: r.id, r.startdate, r.fkbus, bs.city, bs.name, (count(t.id))
                                 Inner Unique: true
                                 Buffers: shared hit=10271 read=50789, temp read=10484 written=18017
                                 ->  Hash Join  (cost=261129.32..305632.17 rows=145923 width=24) (actual time=1641.537..1830.532 rows=144000 loops=1)
                                       Output: r.id, r.startdate, r.fkbus, br.fkbusstationfrom, (count(t.id))
                                       Inner Unique: true
                                       Hash Cond: (s.fkroute = br.id)
                                       Buffers: shared hit=10251 read=50789, temp read=10484 written=18017
                                       ->  Hash Join  (cost=261126.97..305219.71 rows=145923 width=24) (actual time=1641.518..1815.841 rows=144000 loops=1)
                                             Output: r.id, r.startdate, r.fkbus, s.fkroute, (count(t.id))
                                             Inner Unique: true
                                             Hash Cond: (r.fkschedule = s.id)
                                             Buffers: shared hit=10250 read=50789, temp read=10484 written=18017
                                             ->  Merge Join  (cost=261083.57..304792.14 rows=145923 width=24) (actual time=1641.272..1798.614 rows=144000 loops=1)
                                                   Output: r.id, r.startdate, r.fkschedule, r.fkbus, (count(t.id))
                                                   Inner Unique: true
                                                   Merge Cond: (r.id = t.fkride)
                                                   Buffers: shared hit=10239 read=50789, temp read=10484 written=18017
                                                   ->  Index Scan using ride_pkey on book.ride r  (cost=0.42..4555.42 rows=144000 width=16) (actual time=0.011..15.629 rows=144000 loops=1)
                                                         Output: r.id, r.startdate, r.fkbus, r.fkschedule
                                                         Buffers: shared hit=1175
                                                   ->  Finalize GroupAggregate  (cost=261083.15..298052.68 rows=145923 width=12) (actual time=1641.256..1760.302 rows=144000 loops=1)
                                                         Output: count(t.id), t.fkride
                                                         Group Key: t.fkride
                                                         Buffers: shared hit=9064 read=50789, temp read=10484 written=18017
                                                         ->  Gather Merge  (cost=261083.15..295134.22 rows=291846 width=12) (actual time=1641.247..1726.611 rows=432000 loops=1)
                                                               Output: t.fkride, (PARTIAL count(t.id))
                                                               Workers Planned: 2
                                                               Workers Launched: 2
                                                               Buffers: shared hit=9064 read=50789, temp read=10484 written=18017
                                                               ->  Sort  (cost=260083.12..260447.93 rows=145923 width=12) (actual time=1631.599..1640.953 rows=144000 loops=3)
                                                                     Output: t.fkride, (PARTIAL count(t.id))
                                                                     Sort Key: t.fkride
                                                                     Sort Method: external merge  Disk: 3672kB
                                                                     Buffers: shared hit=9064 read=50789, temp read=10484 written=18017
                                                                     Worker 0:  actual time=1629.577..1637.114 rows=144000 loops=1
                                                                       Sort Method: external merge  Disk: 3672kB
                                                                       Buffers: shared hit=3101 read=16993, temp read=3552 written=6189
                                                                     Worker 1:  actual time=1624.349..1633.987 rows=144000 loops=1
                                                                       Sort Method: external merge  Disk: 3672kB
                                                                       Buffers: shared hit=2954 read=16880, temp read=3395 written=5655
                                                                     ->  Partial HashAggregate  (cost=222203.18..245071.19 rows=145923 width=12) (actual time=1279.667..1548.260 rows=144000 loops=3)
                                                                           Output: t.fkride, PARTIAL count(t.id)
                                                                           Group Key: t.fkride
                                                                           Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27416kB
                                                                           Buffers: shared hit=9050 read=50789, temp read=9107 written=16637
                                                                           Worker 0:  actual time=1277.626..1546.081 rows=144000 loops=1
                                                                             Batches: 5  Memory Usage: 8241kB  Disk Usage: 27400kB
                                                                             Buffers: shared hit=3094 read=16993, temp read=3093 written=5729
                                                                           Worker 1:  actual time=1274.517..1541.219 rows=144000 loops=1
                                                                             Batches: 5  Memory Usage: 8241kB  Disk Usage: 24048kB
                                                                             Buffers: shared hit=2947 read=16880, temp read=2936 written=5195
                                                                           ->  Parallel Seq Scan on book.tickets t  (cost=0.00..81761.59 rows=2192259 width=12) (actual time=0.027..312.221 rows=1753580 loops=3)
                                                                                 Output: t.id, t.fkride, t.fio, t.contact, t.fkseat
                                                                                 Buffers: shared hit=9050 read=50789
                                                                                 Worker 0:  actual time=0.028..315.986 rows=1767363 loops=1
                                                                                   Buffers: shared hit=3094 read=16993
                                                                                 Worker 1:  actual time=0.024..295.942 rows=1740405 loops=1
                                                                                   Buffers: shared hit=2947 read=16880
                                             ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.235..0.236 rows=1440 loops=1)
                                                   Output: s.id, s.fkroute
                                                   Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                   Buffers: shared hit=11
                                                   ->  Seq Scan on book.schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.003..0.121 rows=1440 loops=1)
                                                         Output: s.id, s.fkroute
                                                         Buffers: shared hit=11
                                       ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.010..0.011 rows=60 loops=1)
                                             Output: br.id, br.fkbusstationfrom
                                             Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                             Buffers: shared hit=1
                                             ->  Seq Scan on book.busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.002..0.005 rows=60 loops=1)
                                                   Output: br.id, br.fkbusstationfrom
                                                   Buffers: shared hit=1
                                 ->  Memoize  (cost=0.15..0.36 rows=1 width=68) (actual time=0.000..0.000 rows=1 loops=144000)
                                       Output: bs.city, bs.name, bs.id
                                       Cache Key: br.fkbusstationfrom
                                       Cache Mode: logical
                                       Hits: 143990  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB
                                       Buffers: shared hit=20
                                       ->  Index Scan using busstation_pkey on book.busstation bs  (cost=0.14..0.35 rows=1 width=68) (actual time=0.002..0.002 rows=1 loops=10)
                                             Output: bs.city, bs.name, bs.id
                                             Index Cond: (bs.id = br.fkbusstationfrom)
                                             Buffers: shared hit=20
                           ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.041..0.042 rows=5 loops=1)
                                 Output: (count(s_1.id)), s_1.fkbus
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 Buffers: shared hit=2
                                 ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.038..0.039 rows=5 loops=1)
                                       Output: count(s_1.id), s_1.fkbus
                                       Group Key: s_1.fkbus
                                       Batches: 1  Memory Usage: 24kB
                                       Buffers: shared hit=2
                                       ->  Seq Scan on book.seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.008..0.018 rows=200 loops=1)
                                             Output: s_1.id, s_1.fkbus, s_1.place, s_1.fkseatcategory
                                             Buffers: shared hit=2
 Settings: jit = 'off'
 Planning:
   Buffers: shared hit=281
 Planning Time: 0.930 ms
 Execution Time: 1955.721 ms
(123 rows)



--Создадим индексы
CREATE INDEX CONCURRENTLY idx_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX CONCURRENTLY idx_ride_fkbus ON book.ride(fkbus);
CREATE INDEX CONCURRENTLY idx_schedule_fkroute ON book.schedule(fkroute);
CREATE INDEX CONCURRENTLY idx_busroute_fkbusstationfrom ON book.busroute(fkbusstationfrom);
CREATE INDEX CONCURRENTLY idx_seat_fkbus ON book.seat(fkbus);
CREATE INDEX CONCURRENTLY idx_tickets_fkride ON book.tickets(fkride);

--заново запускаем 
--время выполнения примерно такое, индексы не используются (можно убедиться поиском по фразе idx_ в плане)
--не используются тк запрос выбирает все данные используемых таблиц, либо таблички маленькие (busroute, seat) и их выгоднее читать целиком.

postgres@szvm:/home/sergey/Desktop$ psql -d thai -c "explain (analyze, verbose, buffers, settings) 
....

                                                                                                    QUERY PLAN                                                                                                    
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=333737.94..333737.97 rows=10 width=56) (actual time=1930.988..1931.071 rows=10 loops=1)
   Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
   Buffers: shared hit=9714 read=51357, temp read=11781 written=19714
   ->  Sort  (cost=333737.94..334102.75 rows=145923 width=56) (actual time=1930.987..1931.069 rows=10 loops=1)
         Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         Buffers: shared hit=9714 read=51357, temp read=11781 written=19714
         ->  Group  (cost=328030.95..330584.60 rows=145923 width=56) (actual time=1893.489..1916.627 rows=144000 loops=1)
               Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               Buffers: shared hit=9711 read=51357, temp read=11781 written=19714
               ->  Sort  (cost=328030.95..328395.75 rows=145923 width=56) (actual time=1893.485..1901.386 rows=144000 loops=1)
                     Output: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id)), r.startdate
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7864kB
                     Buffers: shared hit=9711 read=51357, temp read=11781 written=19714
                     ->  Hash Join  (cost=261110.77..310523.51 rows=145923 width=56) (actual time=1611.007..1869.815 rows=144000 loops=1)
                           Output: r.id, ((bs.city || ', '::text) || bs.name), (count(t.id)), (count(s_1.id)), r.startdate
                           Inner Unique: true
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           Buffers: shared hit=9705 read=51357, temp read=10798 written=18728
                           ->  Nested Loop  (cost=261105.66..309081.05 rows=145923 width=84) (actual time=1610.955..1844.309 rows=144000 loops=1)
                                 Output: r.id, r.startdate, r.fkbus, bs.city, bs.name, (count(t.id))
                                 Inner Unique: true
                                 Buffers: shared hit=9703 read=51357, temp read=10798 written=18728
                                 ->  Hash Join  (cost=261105.52..305608.37 rows=145923 width=24) (actual time=1610.928..1811.147 rows=144000 loops=1)
                                       Output: r.id, r.startdate, r.fkbus, br.fkbusstationfrom, (count(t.id))
                                       Inner Unique: true
                                       Hash Cond: (s.fkroute = br.id)
                                       Buffers: shared hit=9683 read=51357, temp read=10798 written=18728
                                       ->  Hash Join  (cost=261103.17..305195.91 rows=145923 width=24) (actual time=1610.903..1796.431 rows=144000 loops=1)
                                             Output: r.id, r.startdate, r.fkbus, s.fkroute, (count(t.id))
                                             Inner Unique: true
                                             Hash Cond: (r.fkschedule = s.id)
                                             Buffers: shared hit=9682 read=51357, temp read=10798 written=18728
                                             ->  Merge Join  (cost=261059.77..304768.34 rows=145923 width=24) (actual time=1610.131..1778.104 rows=144000 loops=1)
                                                   Output: r.id, r.startdate, r.fkschedule, r.fkbus, (count(t.id))
                                                   Inner Unique: true
                                                   Merge Cond: (r.id = t.fkride)
                                                   Buffers: shared hit=9671 read=51357, temp read=10798 written=18728
                                                   ->  Index Scan using ride_pkey on book.ride r  (cost=0.42..4555.42 rows=144000 width=16) (actual time=0.018..15.530 rows=144000 loops=1)
                                                         Output: r.id, r.startdate, r.fkbus, r.fkschedule
                                                         Buffers: shared hit=1175
                                                   ->  Finalize GroupAggregate  (cost=261059.35..298028.88 rows=145923 width=12) (actual time=1610.109..1737.724 rows=144000 loops=1)
                                                         Output: count(t.id), t.fkride
                                                         Group Key: t.fkride
                                                         Buffers: shared hit=8496 read=51357, temp read=10798 written=18728
                                                         ->  Gather Merge  (cost=261059.35..295110.42 rows=291846 width=12) (actual time=1610.100..1702.803 rows=432000 loops=1)
                                                               Output: t.fkride, (PARTIAL count(t.id))
                                                               Workers Planned: 2
                                                               Workers Launched: 2
                                                               Buffers: shared hit=8496 read=51357, temp read=10798 written=18728
                                                               ->  Sort  (cost=260059.32..260424.13 rows=145923 width=12) (actual time=1603.361..1612.581 rows=144000 loops=3)
                                                                     Output: t.fkride, (PARTIAL count(t.id))
                                                                     Sort Key: t.fkride
                                                                     Sort Method: external merge  Disk: 3672kB
                                                                     Buffers: shared hit=8496 read=51357, temp read=10798 written=18728
                                                                     Worker 0:  actual time=1600.155..1607.964 rows=144000 loops=1
                                                                       Sort Method: external merge  Disk: 3672kB
                                                                       Buffers: shared hit=2829 read=17045, temp read=3599 written=6239
                                                                     Worker 1:  actual time=1600.226..1609.477 rows=144000 loops=1
                                                                       Sort Method: external merge  Disk: 3672kB
                                                                       Buffers: shared hit=2787 read=17086, temp read=3596 written=6241
                                                                     ->  Partial HashAggregate  (cost=222182.15..245047.39 rows=145923 width=12) (actual time=1234.294..1516.383 rows=144000 loops=3)
                                                                           Output: t.fkride, PARTIAL count(t.id)
                                                                           Group Key: t.fkride
                                                                           Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27576kB
                                                                           Buffers: shared hit=8482 read=51357, temp read=9421 written=17348
                                                                           Worker 0:  actual time=1231.501..1509.381 rows=144000 loops=1
                                                                             Batches: 5  Memory Usage: 8241kB  Disk Usage: 27568kB
                                                                             Buffers: shared hit=2822 read=17045, temp read=3140 written=5779
                                                                           Worker 1:  actual time=1231.239..1516.229 rows=144000 loops=1
                                                                             Batches: 5  Memory Usage: 8241kB  Disk Usage: 27536kB
                                                                             Buffers: shared hit=2780 read=17086, temp read=3137 written=5781
                                                                           ->  Parallel Seq Scan on book.tickets t  (cost=0.00..81758.75 rows=2191975 width=12) (actual time=0.028..294.933 rows=1753580 loops=3)
                                                                                 Output: t.id, t.fkride, t.fio, t.contact, t.fkseat
                                                                                 Buffers: shared hit=8482 read=51357
                                                                                 Worker 0:  actual time=0.024..306.379 rows=1743936 loops=1
                                                                                   Buffers: shared hit=2822 read=17045
                                                                                 Worker 1:  actual time=0.033..293.231 rows=1747744 loops=1
                                                                                   Buffers: shared hit=2780 read=17086
                                             ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.759..0.759 rows=1440 loops=1)
                                                   Output: s.id, s.fkroute
                                                   Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                   Buffers: shared hit=11
                                                   ->  Seq Scan on book.schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.003..0.525 rows=1440 loops=1)
                                                         Output: s.id, s.fkroute
                                                         Buffers: shared hit=11
                                       ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.015..0.016 rows=60 loops=1)
                                             Output: br.id, br.fkbusstationfrom
                                             Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                             Buffers: shared hit=1
                                             ->  Seq Scan on book.busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.003..0.008 rows=60 loops=1)
                                                   Output: br.id, br.fkbusstationfrom
                                                   Buffers: shared hit=1
                                 ->  Memoize  (cost=0.15..0.36 rows=1 width=68) (actual time=0.000..0.000 rows=1 loops=144000)
                                       Output: bs.city, bs.name, bs.id
                                       Cache Key: br.fkbusstationfrom
                                       Cache Mode: logical
                                       Hits: 143990  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB
                                       Buffers: shared hit=20
                                       ->  Index Scan using busstation_pkey on book.busstation bs  (cost=0.14..0.35 rows=1 width=68) (actual time=0.002..0.002 rows=1 loops=10)
                                             Output: bs.city, bs.name, bs.id
                                             Index Cond: (bs.id = br.fkbusstationfrom)
                                             Buffers: shared hit=20
                           ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.042..0.043 rows=5 loops=1)
                                 Output: (count(s_1.id)), s_1.fkbus
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 Buffers: shared hit=2
                                 ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.038..0.039 rows=5 loops=1)
                                       Output: count(s_1.id), s_1.fkbus
                                       Group Key: s_1.fkbus
                                       Batches: 1  Memory Usage: 24kB
                                       Buffers: shared hit=2
                                       ->  Seq Scan on book.seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.007..0.017 rows=200 loops=1)
                                             Output: s_1.id, s_1.fkbus, s_1.place, s_1.fkseatcategory
                                             Buffers: shared hit=2
 Settings: jit = 'off'
 Planning:
   Buffers: shared hit=364
 Planning Time: 1.222 ms
 Execution Time: 1935.389 ms
(123 rows)

