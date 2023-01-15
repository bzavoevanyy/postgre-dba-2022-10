## Механизм блокировок

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.

    ```
    postgres=# alter system set log_lock_waits = on;
    ALTER SYSTEM
    postgres=# show deadlock_timeout ;
     deadlock_timeout
    ------------------
     1s
    (1 row)
    
    postgres=# alter system set deadlock_timeout = 200;
    ALTER SYSTEM
    
    postgres=# show log_lock_waits ;
     log_lock_waits
    ----------------
     on
    (1 row)
    
    postgres=# show deadlock_timeout ;
     deadlock_timeout
    ------------------
     200ms
    (1 row)
    ```
    
    Создадим базу и простую таблицу
    ```
    postgres=# create database locks;
    CREATE DATABASE
    postgres=# \c locks ;
    You are now connected to database "locks" as user "postgres".
    locks=# create table test (c1 integer);
    CREATE TABLE
    locks=# insert into test values (1);
    INSERT 0 1
    locks=# select * from test ;
     c1
    ----
      1
    (1 row)
    ```
    
    Создадим два подключения к БД
    
    Session 1:
    ```
    locks=# begin;
    BEGIN
    locks=*# lock table test ;
    LOCK TABLE
    ```
    
    Session 2:
    ```
    locks=# select * from test ;
    --- запрос ожидает снятие блокировки ---
    ```
    
    Session 1:
    ```
    locks=*# commit;
    COMMIT
    ```
    
    Session 2:
    ```
     c1
    ----
      1
    (1 row)
    ```
    
    Смотрим логи:
    ```
    bogdanzavoevanyy@postgres:~$ sudo tail -n 10 /var/log/postgresql/postgresql-15-main.log
    2023-01-08 12:54:19.948 UTC [6432] postgres@locks LOG:  process 6432 still waiting for AccessShareLock on relation 16389 of database 16388 after 200.146 ms at character 15
    2023-01-08 12:54:19.948 UTC [6432] postgres@locks DETAIL:  Process holding the lock: 6291. Wait queue: 6432.
    2023-01-08 12:54:19.948 UTC [6432] postgres@locks STATEMENT:  select * from test ;
    2023-01-08 12:54:31.604 UTC [6432] postgres@locks LOG:  process 6432 acquired AccessShareLock on relation 16389 of database 16388 after 11856.258 ms at character 15
    2023-01-08 12:54:31.604 UTC [6432] postgres@locks STATEMENT:  select * from test ;
    ```

2. Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.

   Создадим нужное представление для таблицы test
   ```
   CREATE VIEW locks_v AS
   SELECT pid,
   locktype,
   CASE locktype
   WHEN 'relation' THEN relation::regclass::text
   WHEN 'transactionid' THEN transactionid::text
   WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
   END AS lockid,
   mode,
   granted
   FROM pg_locks
   WHERE locktype in ('relation','transactionid','tuple')
   AND (locktype != 'relation' OR relation = 'test'::regclass);
   ```
   
   Выполняем команды update

   ```
   Session 1:
   locks=# begin;
   BEGIN
   locks=*# SELECT txid_current(), pg_backend_pid();
   txid_current | pg_backend_pid
   --------------+----------------
   770 |          21304
   (1 row)
   
   locks=*# update test set c1 = 2 where c1 = 1;
   UPDATE 1
   locks=*#
   
   Session 2:
   locks=# begin;
   BEGIN
   locks=*# SELECT txid_current(), pg_backend_pid();
   txid_current | pg_backend_pid
   --------------+----------------
   771 |          21308
   (1 row)
   
   locks=*# update test set c1 = 3 where c1 = 1;
   --- ждет снятия блокировки ---
   
   Session 3:
   locks=# begin;
   BEGIN
   locks=*# SELECT txid_current(), pg_backend_pid();
   txid_current | pg_backend_pid
   --------------+----------------
   772 |          21318
   (1 row)
   
   locks=*# update test set c1 = 4 where c1 = 1;
   --- ждет снятия блокировки ---
   ```
   
   Смотрим наше представление и определяем, какие блокировки образовались

   ```
   locks=*# select * from locks_v;
   pid  |   locktype    | lockid |       mode       | granted
   -------+---------------+--------+------------------+---------
   21304 | relation      | test   | RowExclusiveLock | t
   21318 | relation      | test   | RowExclusiveLock | t
   21308 | relation      | test   | RowExclusiveLock | t
   21318 | tuple         | test:1 | ExclusiveLock    | f
   21308 | transactionid | 771    | ExclusiveLock    | t
   21318 | transactionid | 772    | ExclusiveLock    | t
   21308 | transactionid | 770    | ShareLock        | f
   21304 | transactionid | 770    | ExclusiveLock    | t
   21308 | tuple         | test:1 | ExclusiveLock    | t
   (9 rows)
   ```

   Session 1 (pid 21304):
      - Блокировка relation RowExclusiveLock на таблицу test
      - Блокировка transactionid ExclusiveLock на строку
      - у обоих блокировок granted true
   
   Session 2 (pid 21308):
      - Блокировка relation RowExclusiveLock на таблицу test (granted true)
      - Блокировка transactionid ExclusiveLock на строку (granted true)
      - Блокировка transitionid ShareLock на строку с granted false (заблокирована txid 770 - session 1)
      - Создается tuple test:1 c ExclusiveLock
      - сессия ждет 

   Session 3 (pid 21318):
      - Блокировка relation RowExclusiveLock на таблицу test (granted true)
      - Блокировка transactionid ExclusiveLock на строку (granted true)
      - Блокировка tuple test:1 granted false
      - сессия ждет

   После того, как в Session 1 сделаем commit или rollback проснется Session 2. Далее, в Session 2 commit или rollback - отработает update в Session 3.

3. Воспроизведем взаимоблокировку трех транзакций.

   ```
   Session 1:
   
   locks=# begin;
   BEGIN
   locks=*# SELECT txid_current(), pg_backend_pid();
   txid_current | pg_backend_pid
   --------------+----------------
   786 |          21304
   (1 row)
   
   locks=*# update test set c1 = 2 where c1 = 1;
   UPDATE 1
   
   Session 2:
   
   locks=# begin;
   BEGIN
   locks=*# SELECT txid_current(), pg_backend_pid();
   txid_current | pg_backend_pid
   --------------+----------------
   787 |          21308
   (1 row)
   
   locks=*# update test set c1 = 3 where c1 = 1;
   --- ждет снятия блокировки ---
   
   Session 3:
   
   locks=# begin;
   BEGIN
   locks=*# SELECT txid_current(), pg_backend_pid();
   txid_current | pg_backend_pid
   --------------+----------------
   788 |          21318
   (1 row)
   
   locks=*# update test set c1 = 3 where c1 = 2;
   UPDATE 1
   locks=*# update test set c1 = 3 where c1 = 1;
   --- ждет снятия блокировки ---
   
   Session 1:
   
   locks=*# update test set c1 = 2 where c1 = 2;
   ERROR:  deadlock detected
   DETAIL:  Process 21304 waits for ShareLock on transaction 788; blocked by process 21318.
   Process 21318 waits for ExclusiveLock on tuple (0,1) of relation 16389 of database 16388; blocked by process 21308.
   Process 21308 waits for ShareLock on transaction 786; blocked by process 21304.
   HINT:  See server log for query details.
   CONTEXT:  while updating tuple (0,9) in relation "test"
   locks=!#
   ```
   
   Посмотрим логи кластера:

   ```
   bogdanzavoevanyy@postgres:~$ tail -n 40 /var/log/postgresql/postgresql-15-main.log
   2023-01-15 10:36:25.765 UTC [21308] postgres@locks LOG:  process 21308 still waiting for ShareLock on transaction 786 after 200.285 ms
   2023-01-15 10:36:25.765 UTC [21308] postgres@locks DETAIL:  Process holding the lock: 21304. Wait queue: 21308.
   2023-01-15 10:36:25.765 UTC [21308] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "test"
   2023-01-15 10:36:25.765 UTC [21308] postgres@locks STATEMENT:  update test set c1 = 3 where c1 = 1;
   2023-01-15 10:37:35.379 UTC [21318] postgres@locks LOG:  process 21318 still waiting for ExclusiveLock on tuple (0,1) of relation 16389 of database 16388 after 200.195 ms
   2023-01-15 10:37:35.379 UTC [21318] postgres@locks DETAIL:  Process holding the lock: 21308. Wait queue: 21318.
   2023-01-15 10:37:35.379 UTC [21318] postgres@locks STATEMENT:  update test set c1 = 3 where c1 = 1;
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks LOG:  process 21304 detected deadlock while waiting for ShareLock on transaction 788 after 200.183 ms
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks DETAIL:  Process holding the lock: 21318. Wait queue: .
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks CONTEXT:  while updating tuple (0,9) in relation "test"
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks STATEMENT:  update test set c1 = 2 where c1 = 2;
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks ERROR:  deadlock detected
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks DETAIL:  Process 21304 waits for ShareLock on transaction 788; blocked by process 21318.
   Process 21318 waits for ExclusiveLock on tuple (0,1) of relation 16389 of database 16388; blocked by process 21308.
   Process 21308 waits for ShareLock on transaction 786; blocked by process 21304.
   Process 21304: update test set c1 = 2 where c1 = 2;
   Process 21318: update test set c1 = 3 where c1 = 1;
   Process 21308: update test set c1 = 3 where c1 = 1;
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks HINT:  See server log for query details.
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks CONTEXT:  while updating tuple (0,9) in relation "test"
   2023-01-15 10:38:01.033 UTC [21304] postgres@locks STATEMENT:  update test set c1 = 2 where c1 = 2;
   2023-01-15 10:38:01.034 UTC [21308] postgres@locks LOG:  process 21308 acquired ShareLock on transaction 786 after 95469.093 ms
   2023-01-15 10:38:01.034 UTC [21308] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "test"
   2023-01-15 10:38:01.034 UTC [21308] postgres@locks STATEMENT:  update test set c1 = 3 where c1 = 1;
   2023-01-15 10:38:01.034 UTC [21318] postgres@locks LOG:  process 21318 acquired ExclusiveLock on tuple (0,1) of relation 16389 of database 16388 after 25855.178 ms
   2023-01-15 10:38:01.034 UTC [21318] postgres@locks STATEMENT:  update test set c1 = 3 where c1 = 1;
   2023-01-15 10:38:01.234 UTC [21318] postgres@locks LOG:  process 21318 still waiting for ShareLock on transaction 787 after 200.165 ms
   2023-01-15 10:38:01.234 UTC [21318] postgres@locks DETAIL:  Process holding the lock: 21308. Wait queue: 21318.
   2023-01-15 10:38:01.234 UTC [21318] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "test"
   2023-01-15 10:38:01.234 UTC [21318] postgres@locks STATEMENT:  update test set c1 = 3 where c1 = 1;
   ```
   
   ### Просмотрев журнал сообщений можно постфактум разобраться в причинах возникновения deadlock. Здесь указывается подробная информация с указанием всех необходимых данных.

4. Воспроизведем ситуацию, при которой две команды update (без where) заблокируют друг друга.

   Создадим таблицу и заполним ее данными:

   ```
   locks=# create table test(id integer primary key generated always as identity, n integer);
   CREATE TABLE
   locks=# insert into test(n) select random() from generate_series(1,1000000);
   INSERT 0 1000000
   ```
   
   Выполняем следующие команды update:
   ```
   Session 1:
   locks=# begin;
   BEGIN
   locks=*# UPDATE test SET n = (select id from test order by id asc limit 1 for update);
   ```
   
   ```
   Session 2:
   locks=*# UPDATE test SET n = (select id from test order by id desc limit 1 for update);
   UPDATE 1000000
   ```
   
   В Session 1 ошибка deadlock detected:
   ```
   ERROR:  deadlock detected
   DETAIL:  Process 21304 waits for ShareLock on transaction 804; blocked by process 21308.
   Process 21308 waits for ShareLock on transaction 803; blocked by process 21304.
   HINT:  See server log for query details.
   CONTEXT:  while updating tuple (4424,176) in relation "test"
   ```