# Работа с индексами

### 1. Подготовка БД

Запустим `postgres` в docker

```pwsh
docker run --name postgres-local -e POSTGRES_PASSWORD=123123 -d postgres
```

Подключимся к кластеру `postgres` и посмотрим список БД

```pwsh
docker exec -it postgres-local psql --username postgres
```

```sql
postgres=# \l
                                                      List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
(3 rows)
```

Создадим БД test

```sql
postgres=# create database test;
CREATE DATABASE
postgres=# \l
                                                      List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 test      | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
(4 rows)
```

Подключимся к БД и создадим таблицу persons

```sql
postgres=# \c test
You are now connected to database "test" as user "postgres".

test=# create table persons(id serial, first_name text, second_name text, birthday date);
CREATE TABLE

test=# \dt
         List of relations
 Schema |  Name  | Type  |  Owner
--------+--------+-------+----------
 public | persons | table | postgres
(1 row)

test=# select * from persons;
 id | first_name | second_name | birthday
----+------------+-------------+----------
(0 rows)
```

Заполним таблицу данными

```sql
test=# insert into persons (first_name, second_name, birthday)
values ('Petr', 'Petrov', '04.16.1985');
INSERT 0 1

test=# insert into persons (first_name, second_name, birthday)
values ('Ivan', 'Ivanov', '12.11.1995'), ('Tigryan', 'Aronyan', '02.23.1895'), ('Katya', 'Rubtsova', '03.06.1999');
INSERT 0 3
```

### 2. Обычный индекс

Построим обычный индекс по полю `first_name` в таблице `persons` и изучим план через [`explain`](https://www.postgresql.org/docs/current/sql-explain.html)

> **Примечание**: Важно понимать, что `explain analyze` выполняет команду и возвращает реальный план, а просто `explain` ожидаемый план выполнения. В общем, такое же поведение как `ssms` у `microsoft`

Для начала посмотрим, как выглядит план без индекса. Для сравнения выполняю `explain` и `explain analyze`, чтобы посмотреть чем отличаются визуально. В результате видно, что в итоге будет произведен последовательное сканирование всей таблицы для поиска записи по условию `where first_name = 'Ivan'`

```sql
test=# explain select * from persons where first_name = 'Ivan';
                       QUERY PLAN
--------------------------------------------------------
 Seq Scan on persons  (cost=0.00..20.12 rows=4 width=72)
   Filter: (first_name = 'Ivan'::text)
(2 rows)

test=# explain analyze select * from persons where first_name = 'Ivan';
                                            QUERY PLAN
--------------------------------------------------------------------------------------------------
 Seq Scan on persons  (cost=0.00..20.12 rows=4 width=72) (actual time=0.016..0.017 rows=1 loops=1)
   Filter: (first_name = 'Ivan'::text)
   Rows Removed by Filter: 3
 Planning Time: 0.032 ms
 Execution Time: 0.026 ms
(5 rows)
```

Создадим индекс по [документации](https://www.postgresql.org/docs/current/sql-createindex.html)

```sql
test=# create index persons_first_name_idx on persons (first_name);
CREATE INDEX
```

Посмотрим на план еще раз. Видно, что `cost` запроса значительно уменьшился с `cost=0.00..20.12 rows=4 width=72` до `cost=0.00..1.05 rows=1 width=72`, однако странно, что все еще используется `Seq Scan` вместо `Index Scan`. Немного поизучав этот вопрос, я [выяснил](https://stackoverflow.com/a/5203827), что когда возвращается много записей относительно всей таблицы, то выгоднее делать `Seq Scan` вместо `Index Scan`, потому что там последовательное физически на диске чтение, которое значительное быстрее чтение индекса

```sql
test=# explain select * from persons where first_name = 'Ivan';
                      QUERY PLAN
-------------------------------------------------------
 Seq Scan on persons  (cost=0.00..1.05 rows=1 width=72)
   Filter: (first_name = 'Ivan'::text)
(2 rows)
```

Проверим, что появится `Index Scan`, если отключить `Seq Scan`

```sql
test=# SET enable_seqscan = OFF;
SET

test=# explain select * from persons where first_name = 'Ivan';
                                       QUERY PLAN
----------------------------------------------------------------------------------------
 Index Scan using persons_first_name_idx on persons  (cost=0.14..8.46 rows=18 width=21)
   Index Cond: (first_name = 'Ivan'::text)
(2 rows)
```

### 3. Индекс для полнотекстового поиска

Создадим таблицу `books`

```sql
test=# create table books(id serial, name text, content text);
CREATE TABLE
```

Вставим данные в таблицу

```sql
test=# insert into books (name, content)
test-# values ('First', 'The sharp steps of the directors began to rap through the hallway behind me. I had a vision of myself seizing control and forcing them to help. We could still help.'),
test-# ('Second', 'One last moment of silence from this loquacious caller. She must have been able to hear the howl of the wind, the creaking of the timber board. She must have known before she called.'),
test-# ('Third', 'A few clicks and the distinct figure of a tall, redheaded woman in military garb appeared on screen. She was standing at attention before a gate, eyes locked ahead in terror as others streamed past her.');
INSERT 0 3
```

Посмотрим как выглядит план без индекса. Ничего особенного, просто идет последовательное сканирование таблицы

```sql
test=# explain select * from books where to_tsvector('english', content) @@ to_tsquery('english', 'force');
                                  QUERY PLAN
-------------------------------------------------------------------------------
 Seq Scan on books  (cost=10000000000.00..10000000233.12 rows=4 width=68)
   Filter: (to_tsvector('english'::regconfig, content) @@ '''forc'''::tsquery)
 JIT:
   Functions: 2
   Options: Inlining true, Optimization true, Expressions true, Deforming true
(5 rows)
```

> **Примечание**: Тут `cost` такой большой, потому что я выполнил `SET enable_seqscan = OFF;`, чтобы заставить планировщик выбрать индекс, потому что на малом количестве данных часто выбирается `Seq Scan`

Создадим индекс для полнотекстового поиска по [статье](https://www.postgresql.org/docs/current/textsearch-tables.html)

```sql
test=# CREATE INDEX books_content_idx ON books USING GIN (to_tsvector('english', content));
CREATE INDEX
```

Посмотрим план с индексом. Теперь используется `Index Scan` и `cost` значительно меньше

```sql
test=# explain select * from books where to_tsvector('english', content) @@ to_tsquery('english', 'force');
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Bitmap Heap Scan on books  (cost=12.82..22.29 rows=4 width=68)
   Recheck Cond: (to_tsvector('english'::regconfig, content) @@ '''forc'''::tsquery)
   ->  Bitmap Index Scan on books_content_idx  (cost=0.00..12.82 rows=4 width=0)
         Index Cond: (to_tsvector('english'::regconfig, content) @@ '''forc'''::tsquery)
(4 rows)
```

### 3. Фильтрованный индекс

Создадим фильтрованный по `birthday` индекс в `persons`

```sql
test=# create index on persons (birthday) where birthday > '1900-01-01';
CREATE INDEX
```

Посмотрим как выглядят планы для данных входящих в фильтр индекса и не входящих

```sql
test=# explain analyze select * from persons where birthday = '1995-12-11';
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Index Scan using persons_birthday_idx on persons  (cost=0.13..8.15 rows=1 width=72) (actual time=0.011..0.012 rows=1 loops=1)
   Index Cond: (birthday = '1995-12-11'::date)
 Planning Time: 0.155 ms
 Execution Time: 0.018 ms
(4 rows)

test=# explain analyze select * from persons where birthday = '1895-02-23';
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Seq Scan on persons  (cost=10000000000.00..10000000001.05 rows=1 width=72) (actual time=17.456..17.457 rows=1 loops=1)
   Filter: (birthday = '1895-02-23'::date)
   Rows Removed by Filter: 3
 Planning Time: 0.073 ms
 JIT:
   Functions: 2
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.200 ms, Inlining 1.807 ms, Optimization 10.101 ms, Emission 5.537 ms, Total 17.645 ms
 Execution Time: 17.680 ms
(9 rows)
```

> **Примечание**: Тут `cost` такой большой, потому что я выполнил `SET enable_seqscan = OFF;`, чтобы заставить планировщик выбрать индекс, потому что на малом количестве данных часто выбирается `Seq Scan`

Из планов хорошо видно, что для данных, которые не входили в область фильтрации фильтра используется `Seq scan` вместо `Index Scan`

> **Примечание**: Мне стало интересно, что будет, если период выборки начинается за фильтрованным индексом. Мое предположение было, что он просто будет всю таблицу сканить. Такое же поведение при секционировании вроде, поэтому так подумал. Проверим:
>
>```
>test=# explain analyze select * from persons where birthday > '1895-02-23' and birthday < '1995-02-23';
>                                                       QUERY PLAN
>------------------------------------------------------------------------------------------------------------------------
> Seq Scan on persons  (cost=10000000000.00..10000000001.83 rows=1 width=21) (actual time=18.608..18.612 rows=1 loops=1)
>   Filter: ((birthday > '1895-02-23'::date) AND (birthday < '1995-02-23'::date))
>   Rows Removed by Filter: 54
> Planning Time: 0.095 ms
> JIT:
>   Functions: 2
>   Options: Inlining true, Optimization true, Expressions true, Deforming true
>   Timing: Generation 0.171 ms, Inlining 1.567 ms, Optimization 11.052 ms, Emission 5.981 ms, Total 18.771 ms
> Execution Time: 18.806 ms
>(9 rows)
>```
> Как я и думал, он использует последовательное сканирование всей таблицы