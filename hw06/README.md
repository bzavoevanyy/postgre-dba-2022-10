## Работа с журналами

1. Настраиваем выполнение контрольной точки раз в 30 секунд и перезапускаем кластер

2. Включаем логирование контрольных точек

    ```
    postgres=# alter system set log_checkpoints = on;
    ALTER SYSTEM
    postgres=# select pg_reload_conf();
     pg_reload_conf
    ----------------
     t
    (1 row)
    ```

3. Собираем начальную информацию

   ```
    postgres=# SELECT pg_stat_reset_shared('bgwriter');
    pg_stat_reset_shared
    ----------------------
    
    (1 row)
    
    postgres=# select * from pg_stat_bgwriter \gx
    postgres=# select * from pg_stat_bgwriter \gx
    -[ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 0
    checkpoints_req       | 0
    checkpoint_write_time | 0
    checkpoint_sync_time  | 0
    buffers_checkpoint    | 0
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 0
    buffers_backend_fsync | 0
    buffers_alloc         | 6
    stats_reset           | 2023-01-05 13:06:43.956668+00
   ```
   
   ```
    postgres=# select * from pg_ls_waldir() order by modification;
    name           |   size   |      modification
    --------------------------+----------+------------------------
    000000010000000000000001 | 16777216 | 2023-01-05 13:06:42+00
    (1 row)
   ```
   
   ```
    postgres=# select pg_current_wal_insert_lsn();
    pg_current_wal_insert_lsn
    ---------------------------
    0/16F5440
    (1 row)
   ```
    
4. Запускаем pg_bench

   ```
    bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -P 60 -T 600 -U postgres postgres
    pgbench (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
    starting vacuum...end.
    progress: 60.0 s, 872.3 tps, lat 1.146 ms stddev 1.054
    progress: 120.0 s, 861.5 tps, lat 1.160 ms stddev 0.176
    progress: 180.0 s, 855.5 tps, lat 1.169 ms stddev 0.159
    progress: 240.0 s, 886.8 tps, lat 1.127 ms stddev 0.269
    progress: 300.0 s, 882.2 tps, lat 1.133 ms stddev 0.162
    progress: 360.0 s, 899.7 tps, lat 1.111 ms stddev 0.163
    progress: 420.0 s, 838.4 tps, lat 1.192 ms stddev 0.166
    progress: 480.0 s, 893.8 tps, lat 1.118 ms stddev 0.185
    progress: 540.0 s, 895.6 tps, lat 1.116 ms stddev 0.155
    progress: 600.0 s, 879.0 tps, lat 1.137 ms stddev 0.153
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 525890
    latency average = 1.141 ms
    latency stddev = 0.374 ms
    initial connection time = 2.349 ms
    tps = 876.485840 (without initial connection time)
   ```
   
5. Смотрим статистику

    ```
   postgres=# select * from pg_stat_bgwriter \gx
    postgres=# select * from pg_stat_bgwriter \gx
    -[ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 27
    checkpoints_req       | 0
    checkpoint_write_time | 538943
    checkpoint_sync_time  | 107
    buffers_checkpoint    | 42327
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 5089
    buffers_backend_fsync | 0
    buffers_alloc         | 5506
    stats_reset           | 2023-01-05 13:06:43.956668+00
    ```
   
    ```
    postgres=# select pg_current_wal_insert_lsn();
    pg_current_wal_insert_lsn
    ---------------------------
    0/20605FC0
    (1 row)
   ```
    Смотрим объем данных за время выполнения pg_bench
   ```
    postgres=# SELECT '0/20605FC0'::pg_lsn - '0/16F5440'::pg_lsn;
    ?column?
    -----------
    519113600
    (1 row)
   ```
   Файлы wal, 4 файла, т.к. min_wal_size = 80MB
   ```
    postgres=# select * from pg_ls_waldir() order by modification;
    name           |   size   |      modification
    --------------------------+----------+------------------------
    000000010000000000000021 | 16777216 | 2023-01-05 13:19:12+00
    000000010000000000000022 | 16777216 | 2023-01-05 13:19:46+00
    000000010000000000000023 | 16777216 | 2023-01-05 13:20:13+00
    000000010000000000000020 | 16777216 | 2023-01-05 13:22:18+00
   ```
   
    Логи:

    ```
   ...
   
    2023-01-05 13:10:12.701 UTC [65974] LOG:  checkpoint starting: time
    2023-01-05 13:10:39.084 UTC [65974] LOG:  checkpoint complete: wrote 1710 buffers (10.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.363 s, sync=0.010 s, total=26.384 s; sync files=48, longest=0.004 s, average=0.001 s; distance=12821 kB, estimate=12821 kB
    2023-01-05 13:10:42.085 UTC [65974] LOG:  checkpoint starting: time
    2023-01-05 13:11:09.061 UTC [65974] LOG:  checkpoint complete: wrote 2093 buffers (12.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.962 s, sync=0.005 s, total=26.977 s; sync files=14, longest=0.003 s, average=0.001 s; distance=24556 kB, estimate=24556 kB
    2023-01-05 13:11:12.064 UTC [65974] LOG:  checkpoint starting: time
    2023-01-05 13:11:39.043 UTC [65974] LOG:  checkpoint complete: wrote 1950 buffers (11.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.962 s, sync=0.010 s, total=26.980 s; sync files=9, longest=0.003 s, average=0.002 s; distance=24236 kB, estimate=24524 kB
    2023-01-05 13:11:42.045 UTC [65974] LOG:  checkpoint starting: time
    2023-01-05 13:12:09.021 UTC [65974] LOG:  checkpoint complete: wrote 2260 buffers (13.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.964 s, sync=0.005 s, total=26.977 s; sync files=14, longest=0.003 s, average=0.001 s; distance=25791 kB, estimate=25791 kB
   
   ...
   ```
   
### Вывод: За время выполнения теста было сгенерировано 500MB данных в wal файлах, создано 34 wal файла. На одну контрольную точку приходится около 25MB
### Все контрольные точки выполнялись по расписанию, т.к. объем данных между checkpoint не превышает max_wal_size = 1GB


1. Сравните tps в синхронном/асинхронном режиме

    В синхронном режиме
    ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -P 1 -T 10 -U postgres postgres
    pgbench (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
    starting vacuum...end.
    progress: 1.0 s, 1083.0 tps, lat 0.921 ms stddev 0.074
    progress: 2.0 s, 1006.9 tps, lat 0.992 ms stddev 0.149
    progress: 3.0 s, 1020.1 tps, lat 0.980 ms stddev 0.125
    progress: 4.0 s, 837.0 tps, lat 1.195 ms stddev 0.140
    progress: 5.0 s, 812.0 tps, lat 1.230 ms stddev 0.185
    progress: 6.0 s, 799.1 tps, lat 1.252 ms stddev 0.152
    progress: 7.0 s, 825.9 tps, lat 1.209 ms stddev 0.145
    progress: 8.0 s, 847.0 tps, lat 1.180 ms stddev 0.127
    progress: 9.0 s, 879.1 tps, lat 1.138 ms stddev 0.088
    progress: 10.0 s, 849.0 tps, lat 1.176 ms stddev 0.118
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 10 s
    number of transactions actually processed: 8960
    latency average = 1.115 ms
    latency stddev = 0.176 ms
    initial connection time = 2.217 ms
    tps = 896.167852 (without initial connection time)
   ```
    
    В асинхронном режиме

    ```
   postgres=# ALTER SYSTEM SET synchronous_commit = on;
    ALTER SYSTEM
    postgres=# select pg_reload_conf();
    pg_reload_conf
    ----------------
    t
    (1 row)
   ```
   
    ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -P 1 -T 10 -U postgres postgres
    pgbench (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
    starting vacuum...end.
    progress: 1.0 s, 1653.9 tps, lat 0.603 ms stddev 0.072
    progress: 2.0 s, 1674.0 tps, lat 0.597 ms stddev 0.042
    progress: 3.0 s, 1655.0 tps, lat 0.604 ms stddev 0.061
    progress: 4.0 s, 1681.0 tps, lat 0.594 ms stddev 0.030
    progress: 5.0 s, 1672.0 tps, lat 0.598 ms stddev 0.047
    progress: 6.0 s, 1649.0 tps, lat 0.606 ms stddev 0.078
    progress: 7.0 s, 1630.9 tps, lat 0.613 ms stddev 0.068
    progress: 8.0 s, 1651.1 tps, lat 0.605 ms stddev 0.062
    progress: 9.0 s, 1659.9 tps, lat 0.602 ms stddev 0.048
    progress: 10.0 s, 1675.0 tps, lat 0.597 ms stddev 0.045
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 10 s
    number of transactions actually processed: 16602
    latency average = 0.602 ms
    latency stddev = 0.057 ms
    initial connection time = 2.149 ms
    tps = 1660.527124 (without initial connection time)
   ```
    
### Вывод: В асинхронном режиме получаем значительный прирост производительности, т.к. отсутствует ожидание локального сброса wal на диск

1. Перезапускаем кластер с включенной контрольной суммой страниц.

    ```
    bogdanzavoevanyy@postgres:~$ sudo -u postgres pg_ctlcluster 14 main stop
   
    bogdanzavoevanyy@postgres:~$ sudo -u postgres /usr/lib/postgresql/14/bin/pg_checksums --enable -D "/var/lib/postgresql/14/main"
    Checksum operation completed
    Files scanned:  946
    Blocks scanned: 5238
    pg_checksums: syncing data directory
    pg_checksums: updating control file
    Checksums enabled in cluster
   
    bogdanzavoevanyy@postgres:~$ sudo -u postgres pg_ctlcluster 14 main start
    Warning: the cluster will not be running as a systemd service. Consider using systemctl:
    sudo systemctl start postgresql@14-main
   
    postgres=# show data_checksums ;
    data_checksums
    ----------------
    on
    (1 row)
   ```
   
2. Создаем таблицу и вставляем значения

    ```
    postgres=# create table test(c1 int);
    CREATE TABLE
    postgres=# insert into test(c1) values (1);
    INSERT 0 1
   ```
   
3. Смотрим, где лежит файл таблицы.

    ```
    postgres=# SELECT pg_relation_filepath('test');
    pg_relation_filepath
    ----------------------
    base/13726/16433
    (1 row)
   ```
   
4. Меняем файл пробуем прочитать файл

     ```
    bogdanzavoevanyy@postgres:~$ sudo -u postgres dd if=/dev/zero of=/var/lib/postgresql/14/main/base/13726/16433 oflag=dsync conv=notrunc bs=1 count=8
    8+0 records in
    8+0 records out
    8 bytes copied, 0.00520505 s, 1.5 kB/s
    ```
   
    ```
   postgres=# select * from test;
    WARNING:  page verification failed, calculated checksum 35670 but expected 14708
    ERROR:  invalid page in block 0 of relation base/13726/16433
    postgres=#
   ```
   
    Для того, что бы прочитать строку, устанавливаем параметр ignore_checksum_failure в true

    ```
    postgres=# set ignore_checksum_failure = true;
    SET
   ```
   ```
    postgres=# select * from test;
    WARNING:  page verification failed, calculated checksum 35670 but expected 14708
    c1
    ----
    1
    (1 row)
   ```
    Или другим способом, с потерей строки

    ```
    postgres=# select * from test;
    WARNING:  page verification failed, calculated checksum 1547 but expected 59297
    ERROR:  invalid page in block 0 of relation base/13726/16433
   ```
   ```
   postgres=# SET zero_damaged_pages = on;
    SET
    postgres=# vacuum full test;
    WARNING:  page verification failed, calculated checksum 1547 but expected 59297
    WARNING:  invalid page in block 0 of relation base/13726/16433; zeroing out page
    VACUUM
    postgres=# select * from test;
    c1
    ----
    (0 rows)
   
   ```

    
