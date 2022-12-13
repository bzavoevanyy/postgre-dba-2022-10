## Работа с базами данных, пользователями и правами

1. На ранее созданный инстанс c Ubuntu 20.04 LTS устанавливаем PostgreSQL 14

    ```bash
    sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
    
    bogdanzavoevanyy@postgres:~$ pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
    ```
2. Заходим под пользователем postgres

    ```bash
    bogdanzavoevanyy@postgres:~$ sudo -u postgres psql
    psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
    Type "help" for help.
    
    postgres=#
    ```
   
3. Создаем новую базу данных testdb

    ```bash
    postgres=# create database testdb;
    CREATE DATABASE
    postgres=#
    ```

4. Создаем новую схему testnm в бд testdb

   ```bash
   testdb=# create schema testnm;
   CREATE SCHEMA
   testdb=#
   ```
   
5. Создаем новую таблицу t1 с одной колонкой c1 типа integer

   ```bash
   testdb=# create table t1(c1 integer);
   CREATE TABLE
   testdb=#
   ```
   
6. Вставляем строку со значением c1=1

   ```bash
   testdb=# insert into t1 values (1);
   INSERT 0 1
   testdb=#
   ```
   
7. Создаем новую роль readonly
   
   ```bash
   testdb=# create role readonly;
   CREATE ROLE
   testdb=#
   ```
   
8. Даем новой роли право на подключение к базе данных testdb
   
   ```bash
   testdb=# grant CONNECT on DATABASE testdb to readonly ;
   GRANT
   testdb=#
   ```
   
9. Даем новой роли право на использование схемы testnm

   ```bash
   testdb=# grant USAGE on SCHEMA testnm to readonly ;
   GRANT
   testdb=#
   ```

10. Даем новой роли право на select для всех таблиц схемы testnm

   ```bash
   testdb=# grant SELECT on ALL tables in schema testnm to readonly ;
   GRANT
   testdb=#
   ```

11. Создаем пользователя testread с паролем test123

   ```bash
   testdb=# create user testread with password 'test123';
   CREATE ROLE
   testdb=#
   ```
   
12. Добавляем роль readonly пользователю testread

   ```bash
   testdb=# grant readonly to testread ;
   GRANT ROLE
   testdb=#
   ```
   
13. Заходим под пользователем testread в базу данных testdb

   ```bash
   bogdanzavoevanyy@postgres:~$ psql -h localhost -U testread -d testdb -W
   Password:
   psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
   Type "help" for help.
   
   testdb=>
   ```
   
14. Делаем SELECT из t1

   ```bash
   testdb=> select * from t1;
   ERROR:  permission denied for table t1
   testdb=>
   ```
   ```bash
   testdb=> \dt
        List of relations
   Schema | Name | Type  |  Owner
   --------+------+-------+----------
   public | t1   | table | postgres
   (1 row)
   
   testdb=>
   ```
   
   ### Таблица t1 создана в схеме public, а у роли readonly есть права чтения только на схему testnm

15. Подключаемся к бд testdb пользователем postgres

   ```bash
   testdb=> \c testdb postgres
   Password for user postgres:
   SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
   You are now connected to database "testdb" as user "postgres".
   testdb=#
   ```
   
16. Удаляем таблицу t1

   ```bash
   testdb=# drop table t1 ;
   DROP TABLE
   testdb=#
   ```
   
17. Создаем таблицу t1 в схеме testnm и вставляем в нее данные

   ```bash
   testdb=# create table testnm.t1(c1 integer);
   CREATE TABLE
   testdb=#
   
   testdb=# insert INTO testnm.t1 values (1);
   INSERT 0 1
   testdb=#
   ```
   
18. Подключаемся пользователем testread и пробуем получить данные из таблицы t1

   ```bash
   bogdanzavoevanyy@postgres:~$ psql -h localhost -U testread -d testdb -W
   Password:
   psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
   Type "help" for help.
   
   testdb=> select * from testnm.t1;
   ERROR:  permission denied for table t1
   testdb=>
   ```
   ### Доступа запрещен, потому что права выдавались до того, как создали таблицу и на текущую таблицу они не распространяются
   ### Изменим default privileges роли readonly

   ```bash
   testdb=> \c testdb postgres
   Password for user postgres:
   SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
   You are now connected to database "testdb" as user "postgres".
   testdb=# alter default privileges in schema testnm grant select on tables to readonly ;
   ALTER DEFAULT PRIVILEGES
   testdb=# exit
   
   bogdanzavoevanyy@postgres:~$ psql -h localhost -U testread -d testdb -W
   Password:
   psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
   Type "help" for help.
   
   testdb=> select * from testnm.t1;
   c1
   ----
   1
   (1 row)
   
   testdb=>
   ```
   ### Теперь доступно чтение таблицы

19. Создаем таблицу пользователем testread;

   ```bash
   testdb=> create table t2 (c1 integer);
   CREATE TABLE
   testdb=> insert into t2 values (1);
   INSERT 0 1
   testdb=>
   ```
   ```bash 
   testdb=> show search_path ;
   search_path
   -----------------
   "$user", public
   (1 row)
   
   testdb=>
   ```
   ### Таблицу удалось создать, поскольку она создается в public (схемы $user нет). Всем новым пользователям дается роль public и с этой ролью можно создавать объекты в схеме public

20. Удалим права у роли public

   ```bash
   testdb=> \c testdb postgres
   Password for user postgres:
   SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
   You are now connected to database "testdb" as user "postgres".
   testdb=# revoke CREATE on SCHEMA public from PUBLIC ;
   REVOKE
   testdb=# revoke ALL on DATABASE testdb from PUBLIC ;
   REVOKE
   testdb=#
   ```
   
21. Пробуем создать таблицу пользователем testread в схеме public

   ```bash
   testdb=> create table t3 (c1 integer);
   ERROR:  permission denied for schema public
   LINE 1: create table t3 (c1 integer);
   ^
   testdb=>
   ```
   
   ### Теперь доступ для роли public запрещен