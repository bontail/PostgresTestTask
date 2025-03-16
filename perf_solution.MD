# Решение

### Пункт 1

Ускорить простой запроc, добиться времени выполнения < 10ms

```postgresql
select name from t1 where id = 50000;
```

Текущее время

```postgresql
EXPlAIN ANALYZE select name from t1 where id = 50000;
```

```text
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..208392.00 rows=1 width=30) (actual time=43.438..1919.198 rows=1 loops=1)
   Filter: (id = 50000)
   Rows Removed by Filter: 9999999
 Planning Time: 2.382 ms
 JIT:
   Functions: 4
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 4.116 ms, Inlining 0.000 ms, Optimization 7.700 ms, Emission 25.653 ms, Total 37.469 ms
 Execution Time: 1924.823 ms
(9 rows)
```

Запрос выполняется медленно из-за того, что запрос берет все записи, а потом уже отфильтровывает по одному значению<br>
Создадим индекс

```postgresql
CREATE INDEX ON t1 (id)
```

И замерим время

```postgresql
EXPlAIN ANALYZE select name from t1 where id = 50000;
```

```text
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Index Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=30) (actual time=0.099..0.103 rows=1 loops=1)
   Index Cond: (id = 50000)
 Planning Time: 0.279 ms
 Execution Time: 0.170 ms
(4 rows)
```

### Пункт 2

Ускорить запрос "max + left join", добиться времени выполнения < 10ms

```postgresql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

Текущее время

```postgresql
EXPlAIN ANALYZE select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

```text
                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=385682.77..385682.78 rows=1 width=32) (actual time=7552.516..7552.540 rows=1 loops=1)
   ->  Hash Left Join  (cost=218335.76..373182.96 rows=4999923 width=9) (actual time=2048.411..6574.408 rows=5000000 loops=1)
         Hash Cond: (t2.t_id = t1.id)
         ->  Seq Scan on t2  (cost=0.00..81871.23 rows=4999923 width=13) (actual time=10.541..1030.058 rows=5000000 loops=1)
         ->  Hash  (cost=208392.00..208392.00 rows=606061 width=4) (actual time=2034.580..2034.581 rows=624526 loops=1)
               Buckets: 262144  Batches: 4  Memory Usage: 7536kB
               ->  Seq Scan on t1  (cost=0.00..208392.00 rows=606061 width=4) (actual time=0.524..1746.364 rows=624526 loops=1)
                     Filter: (name ~~ 'a%'::text)
                     Rows Removed by Filter: 9375474
 Planning Time: 48.442 ms
 JIT:
   Functions: 13
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 2.135 ms, Inlining 0.000 ms, Optimization 1.244 ms, Emission 9.634 ms, Total 13.013 ms
 Execution Time: 7555.578 ms
(15 rows)
```

Заметим, что у нас используется LEFT JOIN, а значит в max() попадет вся таблица t2, так что использовать JOIN не имеет смысла<br>
Создадим index и сделаем запрос без JOIN

```postgresql
CREATE INDEX ON t2 (day);
EXPlAIN ANALYZE select max(t2.day) from t2;
```

```text
                                                                      QUERY PLAN                                                                      
------------------------------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.45..0.46 rows=1 width=32) (actual time=1.501..1.581 rows=1 loops=1)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.45 rows=1 width=9) (actual time=1.347..1.348 rows=1 loops=1)
           ->  Index Only Scan Backward using t2_day_idx on t2  (cost=0.43..104784.44 rows=5000000 width=9) (actual time=1.346..1.346 rows=1 loops=1)
                 Index Cond: (day IS NOT NULL)
                 Heap Fetches: 0
 Planning Time: 7.478 ms
 Execution Time: 1.869 ms
(8 rows)
```

### Пункт 3

Ускорить запрос "anti-join", добиться времени выполнения < 10sec

```postgresql
select day from t2 where t_id not in ( select t1.id from t1 );
```

Текущее время

```postgresql
EXPlAIN ANALYZE select day from t2 where t_id not in ( select t1.id from t1 );
```

```text
Очень-очень долго
```

Запрос выполняется неэффективно, потому что мы для каждой записи из t2 будем проверять, находится t_id этой записи в таблице t1 колонке t1.id<br>
Сложность O(N * M), где N - количество записей в t2, M - количество записей в t1<br>
Поэтому будем использовать JOIN, чтобы PostgreSQL применила хеширование и сложность была O(N + M)

```postgresql
EXPlAIN ANALYZE select day from t2 left join t1 on t2.t_id = t1.id where t1.id is NULL;
```

```text
                                                         QUERY PLAN                                                         
----------------------------------------------------------------------------------------------------------------------------
 Hash Right Anti Join  (cost=168787.00..592220.00 rows=1 width=9) (actual time=9394.654..9394.701 rows=0 loops=1)
   Hash Cond: (t1.id = t2.t_id)
   ->  Seq Scan on t1  (cost=0.00..183392.00 rows=10000000 width=4) (actual time=0.352..1731.882 rows=10000000 loops=1)
   ->  Hash  (cost=81872.00..81872.00 rows=5000000 width=13) (actual time=3346.807..3346.811 rows=5000000 loops=1)
         Buckets: 262144  Batches: 64  Memory Usage: 5734kB
         ->  Seq Scan on t2  (cost=0.00..81872.00 rows=5000000 width=13) (actual time=57.282..962.836 rows=5000000 loops=1)
 Planning Time: 0.838 ms
 JIT:
   Functions: 10
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 2.844 ms, Inlining 4.500 ms, Optimization 30.573 ms, Emission 21.868 ms, Total 59.785 ms
 Execution Time: 9398.103 ms
(12 rows)
```

### Пункт 4

Ускорить запрос "semi-join", добиться времени выполнения < 10sec

```postgresql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```

Текущее время

```postgresql
EXPlAIN ANALYZE select day from t2 where t_id in (select t1.id from t1, t2 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```

```text
Очень много
```

Запрос, как и в прошлом пункте, выполняется неэффективно N * M<br>
Используем JOIN, но перед этим создадим индекс

```postgresql
CREATE INDEX ON t2 (day);
EXPlAIN ANALYZE select day from t2 join t1 on t2.t_id = t1.id and day > to_char(date_trunc('day', now() - '1 months'::interval),'yyyymmdd');
```

```text
                                                                 QUERY PLAN                                                                 
--------------------------------------------------------------------------------------------------------------------------------------------
 Hash Join  (cost=33364.45..343481.44 rows=745699 width=9) (actual time=574.311..6990.882 rows=772313 loops=1)
   Hash Cond: (t1.id = t2.t_id)
   ->  Seq Scan on t1  (cost=0.00..183392.00 rows=10000000 width=4) (actual time=17.130..1834.981 rows=10000000 loops=1)
   ->  Hash  (cost=20401.21..20401.21 rows=745699 width=13) (actual time=553.931..553.932 rows=772313 loops=1)
         Buckets: 262144  Batches: 8  Memory Usage: 6582kB
         ->  Index Scan using t2_day_idx on t2  (cost=0.44..20401.21 rows=745699 width=13) (actual time=0.499..231.012 rows=772313 loops=1)
               Index Cond: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
 Planning Time: 15.433 ms
 JIT:
   Functions: 13
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 5.129 ms, Inlining 0.000 ms, Optimization 2.108 ms, Emission 15.062 ms, Total 22.299 ms
 Execution Time: 7028.361 ms
```

### Пункт 5

Ускорить работу "savepoint + update", добиться постоянной во времени производительности (число транзакций в секунду)

Создание кастомной функции random

```shell
psql -X -q <<'EOF'
create or replace function random(left bigint, right bigint) returns bigint
as $$
 select trunc(random.left + random()*(random.right - random.left))::bigint;
$$                                                
language sql;
EOF
```

Создание нагрузочного файла

```shell
psql -X -q > generate_100_subtrans.sql <<EOF
select '\\set id random(1,10000000)'
union all
select 'BEGIN;'
union all
select 'savepoint v' || v.id || ';'                || E'\n' 
    || 'update t1 set name = name where id = :id;' || E'\n'
from generate_series(1,100) v(id)
union all
select E'COMMIT;\n' \g (tuples_only=true format=unaligned)
EOF
```

Запускаем тест

```shell
# execute long running transaction
psql -c 'select txid_current(); select pg_sleep(3600);' &
# check using pgbench
pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql 2>&1 > generate_100_subtrans_pgbench.log
```

Получаем
```text
progress: 1.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 2.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 3.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 4.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 5.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 6.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 7.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 8.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 9.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 10.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
```

Решение<br>
Создаем индекс
```postgresql
CREATE INDEX ON t1 (id);
```

Запускаем тест

```shell
# execute long running transaction
psql -c 'select txid_current(); select pg_sleep(3600);' &
# check using pgbench
pgbench -p 5432 -rn -P1 -c1 -T3600 -M prepared -f generate_5_subtrans.sql 2>&1 > generate_5_subtrans_pgbench.log
```

Получаем

```text
progress: 1.0 s, 269.0 tps, lat 35.349 ms stddev 32.143, 0 failed
progress: 2.0 s, 302.0 tps, lat 33.003 ms stddev 14.812, 0 failed
progress: 3.0 s, 364.0 tps, lat 27.511 ms stddev 3.029, 0 failed
progress: 4.0 s, 346.0 tps, lat 28.792 ms stddev 4.170, 0 failed
progress: 5.0 s, 347.0 tps, lat 28.753 ms stddev 3.724, 0 failed
progress: 6.0 s, 343.0 tps, lat 29.225 ms stddev 3.601, 0 failed
progress: 7.0 s, 340.0 tps, lat 29.568 ms stddev 3.749, 0 failed
progress: 8.0 s, 355.0 tps, lat 27.934 ms stddev 3.011, 0 failed
progress: 9.0 s, 326.0 tps, lat 30.728 ms stddev 6.144, 0 failed
progress: 10.0 s, 351.0 tps, lat 28.606 ms stddev 4.516, 0 failed
```