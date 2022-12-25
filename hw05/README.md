## Настройка autovacuum с учетом оптимальной производительности

1. Создаем GCE инстанс типа e2-medium и диском 10GB
2. Применяем настройки кластера из задания

   ```
   max_connections = 40
   shared_buffers = 1GB
   effective_cache_size = 3GB
   maintenance_work_mem = 512MB
   checkpoint_completion_target = 0.9
   wal_buffers = 16MB
   default_statistics_target = 500
   random_page_cost = 4
   effective_io_concurrency = 2
   work_mem = 6553kB
   min_wal_size = 4GB
   max_wal_size = 16GB
   ```
   Перезагружаем кластер
   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pg_ctlcluster 14 main restart
   Warning: the cluster will not be running as a systemd service. Consider using systemctl:
   sudo systemctl restart postgresql@14-main
   ```
3. Настраиваем pgbench

    ```bash
    bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -i postgres
    dropping old tables...
    NOTICE:  table "pgbench_accounts" does not exist, skipping
    NOTICE:  table "pgbench_branches" does not exist, skipping
    NOTICE:  table "pgbench_history" does not exist, skipping
    NOTICE:  table "pgbench_tellers" does not exist, skipping
    creating tables...
    generating data (client-side)...
    100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
    vacuuming...
    creating primary keys...
    done in 0.52 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.37 s, vacuum 0.09 s, primary keys 0.05 s).
    ```

4. Запускаем pgbech на дефолтных настройках автовакуума

      ```
   postgres=# SELECT name, setting, context, short_desc FROM pg_settings WHERE name like 'autovacuum%';
                 name                  |  setting  |  context   |                                        short_desc
   ---------------------------------------+-----------+------------+-------------------------------------------------------------------------------------------
   autovacuum                            | on        | sighup     | Starts the autovacuum subprocess.
   autovacuum_analyze_scale_factor       | 0.1       | sighup     | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples.
   autovacuum_analyze_threshold          | 50        | sighup     | Minimum number of tuple inserts, updates, or deletes prior to analyze.
   autovacuum_freeze_max_age             | 200000000 | postmaster | Age at which to autovacuum a table to prevent transaction ID wraparound.
   autovacuum_max_workers                | 3         | postmaster | Sets the maximum number of simultaneously running autovacuum worker processes.
   autovacuum_multixact_freeze_max_age   | 400000000 | postmaster | Multixact age at which to autovacuum a table to prevent multixact wraparound.
   autovacuum_naptime                    | 60        | sighup     | Time to sleep between autovacuum runs.
   autovacuum_vacuum_cost_delay          | 2         | sighup     | Vacuum cost delay in milliseconds, for autovacuum.
   autovacuum_vacuum_cost_limit          | -1        | sighup     | Vacuum cost amount available before napping, for autovacuum.
   autovacuum_vacuum_insert_scale_factor | 0.2       | sighup     | Number of tuple inserts prior to vacuum as a fraction of reltuples.
   autovacuum_vacuum_insert_threshold    | 1000      | sighup     | Minimum number of tuple inserts prior to vacuum, or -1 to disable insert vacuums.
   autovacuum_vacuum_scale_factor        | 0.2       | sighup     | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.
   autovacuum_vacuum_threshold           | 50        | sighup     | Minimum number of tuple updates or deletes prior to vacuum.
   autovacuum_work_mem                   | -1        | sighup     | Sets the maximum memory to be used by each autovacuum worker process.
   (14 rows)
   ```

    ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
   pgbench (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   starting vacuum...end.
   progress: 60.0 s, 920.3 tps, lat 8.682 ms stddev 5.071
   progress: 120.0 s, 914.7 tps, lat 8.737 ms stddev 5.196
   progress: 180.0 s, 917.6 tps, lat 8.710 ms stddev 5.243
   progress: 240.0 s, 892.6 tps, lat 8.954 ms stddev 5.236
   progress: 300.0 s, 552.1 tps, lat 14.447 ms stddev 23.955
   progress: 360.0 s, 554.7 tps, lat 14.403 ms stddev 24.226
   progress: 420.0 s, 544.5 tps, lat 14.661 ms stddev 24.360
   progress: 480.0 s, 545.3 tps, lat 14.651 ms stddev 24.557
   progress: 540.0 s, 534.0 tps, lat 14.962 ms stddev 24.321
   progress: 600.0 s, 534.8 tps, lat 14.936 ms stddev 24.551
   transaction type: <builtin: TPC-B (sort of)>
   scaling factor: 1
   query mode: simple
   number of clients: 8
   number of threads: 1
   duration: 600 s
   number of transactions actually processed: 414647
   latency average = 11.561 ms
   latency stddev = 17.398 ms
   initial connection time = 12.949 ms
   tps = 691.069007 (without initial connection time)
    ```
   
5. Проведем тест с выключенным автовакуумом
   ```
   postgres=# alter system set autovacuum = off;
   ALTER SYSTEM     
   ```
   Перезагрузим кластер и запустим тест

   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
   pgbench (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   starting vacuum...end.
   progress: 60.0 s, 878.9 tps, lat 9.090 ms stddev 4.692
   progress: 120.0 s, 879.7 tps, lat 9.085 ms stddev 4.673
   progress: 180.0 s, 889.1 tps, lat 8.988 ms stddev 4.585
   progress: 240.0 s, 825.2 tps, lat 9.685 ms stddev 10.042
   progress: 300.0 s, 539.0 tps, lat 14.819 ms stddev 24.829
   progress: 360.0 s, 527.5 tps, lat 15.135 ms stddev 24.930
   progress: 420.0 s, 532.2 tps, lat 15.011 ms stddev 24.848
   progress: 480.0 s, 529.5 tps, lat 15.087 ms stddev 24.817
   progress: 540.0 s, 526.4 tps, lat 15.172 ms stddev 25.114
   progress: 600.0 s, 528.8 tps, lat 15.104 ms stddev 24.733
   transaction type: <builtin: TPC-B (sort of)>
   scaling factor: 1
   query mode: simple
   number of clients: 8
   number of threads: 1
   duration: 600 s
   number of transactions actually processed: 399378
   latency average = 12.002 ms
   latency stddev = 18.047 ms
   initial connection time = 13.509 ms
   tps = 665.624043 (without initial connection time)
   ```   

6. Применим более агрессивные настройки автовакуума

   ```
   postgres=# SELECT name, setting, context, short_desc FROM pg_settings WHERE name like 'autovacuum%';
   name                  |  setting  |  context   |                                        short_desc
   ---------------------------------------+-----------+------------+-------------------------------------------------------------------------------------------
   autovacuum                            | off       | sighup     | Starts the autovacuum subprocess.
   autovacuum_analyze_scale_factor       | 0.01      | sighup     | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples.
   autovacuum_analyze_threshold          | 50        | sighup     | Minimum number of tuple inserts, updates, or deletes prior to analyze.
   autovacuum_freeze_max_age             | 200000000 | postmaster | Age at which to autovacuum a table to prevent transaction ID wraparound.
   autovacuum_max_workers                | 3         | postmaster | Sets the maximum number of simultaneously running autovacuum worker processes.
   autovacuum_multixact_freeze_max_age   | 400000000 | postmaster | Multixact age at which to autovacuum a table to prevent multixact wraparound.
   autovacuum_naptime                    | 10        | sighup     | Time to sleep between autovacuum runs.
   autovacuum_vacuum_cost_delay          | 2         | sighup     | Vacuum cost delay in milliseconds, for autovacuum.
   autovacuum_vacuum_cost_limit          | -1        | sighup     | Vacuum cost amount available before napping, for autovacuum.
   autovacuum_vacuum_insert_scale_factor | 0.2       | sighup     | Number of tuple inserts prior to vacuum as a fraction of reltuples.
   autovacuum_vacuum_insert_threshold    | 1000      | sighup     | Minimum number of tuple inserts prior to vacuum, or -1 to disable insert vacuums.
   autovacuum_vacuum_scale_factor        | 0.02      | sighup     | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.
   autovacuum_vacuum_threshold           | 50        | sighup     | Minimum number of tuple updates or deletes prior to vacuum.
   autovacuum_work_mem                   | -1        | sighup     | Sets the maximum memory to be used by each autovacuum worker process.
   (14 rows)
   ```
   Перезагрузим кластер и запустим тест
   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
   pgbench (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   starting vacuum...end.
   progress: 60.0 s, 965.5 tps, lat 8.276 ms stddev 4.112
   progress: 120.0 s, 957.5 tps, lat 8.349 ms stddev 3.951
   progress: 180.0 s, 949.2 tps, lat 8.421 ms stddev 4.069
   progress: 240.0 s, 935.5 tps, lat 8.544 ms stddev 4.240
   progress: 300.0 s, 731.8 tps, lat 10.918 ms stddev 16.927
   progress: 360.0 s, 565.2 tps, lat 14.138 ms stddev 23.969
   progress: 420.0 s, 575.0 tps, lat 13.890 ms stddev 23.806
   progress: 480.0 s, 566.8 tps, lat 14.086 ms stddev 23.998
   progress: 540.0 s, 550.9 tps, lat 14.498 ms stddev 24.886
   progress: 600.0 s, 583.1 tps, lat 13.699 ms stddev 23.442
   transaction type: <builtin: TPC-B (sort of)>
   scaling factor: 1
   query mode: simple
   number of clients: 8
   number of threads: 1
   duration: 600 s
   number of transactions actually processed: 442839
   latency average = 10.826 ms
   latency stddev = 16.315 ms
   initial connection time = 14.019 ms
   tps = 738.045020 (without initial connection time)
   ```
7. Построим графики

<img src="Screenshot 2022-12-25 at 19.52.37.png"/>
