## Работа с уровнями изоляции транзакции в PostgreSQL

1. Создание нового проекта в GCP

    ```
    gcloud projects create postgres2022-01 --name="Otus homework"
    
    Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/postgres2022-01].
    Waiting for [operations/cp.6784463166638061634] to finish...done.
    Enabling service [cloudapis.googleapis.com] on project [postgres2022-01]...
    Operation "operations/acat.p2-883923616787-92cd8586-c285-4a37-a798-ed0e99860fbe" finished successfully.
    ```

2. Создание VM Instance

    ```
    gcloud beta compute instances create postgres \
    --project=postgres2022-01 \
    --zone=europe-west4-a \
    --machine-type=e2-medium \
    --subnet=default \
    --network-tier=PREMIUM \
    --maintenance-policy=MIGRATE \
    --service-account=883923616787-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=10GB \
    --boot-disk-type=pd-ssd \
    --boot-disk-device-name=postgres \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --reservation-affinity=any
    ```
    
    ```
    Created [https://www.googleapis.com/compute/beta/projects/postgres2022-01/zones/europe-west4-a/instances/postgres].
    NAME      ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    postgres  europe-west4-a  e2-medium                  10.164.0.2   35.204.168.19  RUNNING
    ```

3. Добавляем публичный ключ для инстанса

    ```
    gcloud compute project-info add-metadata \
    --metadata ssh-keys="$(gcloud compute project-info describe \
    --format="value(commonInstanceMetadata.items.filter(key:ssh-keys).firstof(value))")
    $(whoami):$(cat ~/.ssh/id_rsa.pub)"
    
    ---------------------------------------------
    Проверяем:
    
    gcloud compute project-info describe \
    --format="value(commonInstanceMetadata.items.filter(key:ssh-keys).firstof(value))"
    
    ```

4. Подключаемся к инстансу

    ```
    gcloud compute ssh postgres
    
    или
    
    ssh 35.204.168.19
    ```

5. Устанавливаем postgres-15

    ```
    sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
    ```
    
   ```
   bogdanzavoevanyy@postgres:~$ pg_lsclusters
   Ver Cluster Port Status Owner    Data directory              Log file
   15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
   ```

6. Запускаем две сессии psql

   ```
   sudo -u postgres psql
   ```
   
7. Выключаем autocommit в обоих сессиях

   ```
   postgres=# \set AUTOCOMMIT OFF
   ```
   
8. Создаем таблицу person и вставляем данные
   
   ```
   create table persons(id serial, first_name text, second_name text);
   insert into persons(first_name, second_name) values('ivan', 'ivanov');
   insert into persons(first_name, second_name) values('petr', 'petrov');
   commit;
   ```
   
9. Проверяем текущий уровень изоляции транзакций

   ```
   postgres=# show transaction isolation level;
   transaction_isolation
   -----------------------
   read committed
   (1 row)
   ```
   
10. В первой сессии добавляем новую запись

   ```
   insert into person(first_name, last_name) values ('sergey','sergeev');
   ```
   
11. Проверяем наличие новой записи во второй сессии
   
   ```
   postgres=# select * from person;
   id | first_name | last_name
   ----+------------+-----------
   1 | ivan       | ivanov
   2 | peetr      | petrov
   (2 rows)
   ```
   ### Запись отсутствует, так как в первой сессии транзакция не завершена. Уровень изоляции read committed не допускает "грязное чтение"

12. Завершаем транзакцию в первой сессии и проверяем наличие записи во второй сессии

   ```
   postgres=# insert into person(first_name, last_name) values ('sergey','sergeev');
   INSERT 0 1
   postgres=*# commit;
   COMMIT
   ```
   ```
   postgres=*# select * from person;
   id | first_name | last_name
   ----+------------+-----------
   1 | ivan       | ivanov
   2 | peetr      | petrov
   3 | sergey     | sergeev
   (3 rows)
   
   postgres=*# commit;
   COMMIT
   ```
   ###

13. Устанавливаем уровень изоляции транзакций repeatable read в обоих сессиях

   ```
   postgres=# set transaction isolation level repeatable read;
   SET
   postgres=*# show transaction isolation level;
   transaction_isolation
   -----------------------
   repeatable read
   (1 row)
   ```
   
14. Делаем вставку новой записи в первой сессии

   ```
   postgres=*# insert into person (first_name, last_name) values ('sveta', 'svetova');
   INSERT 0 1
   ```
   
15. Проверяем наличие записи во второй сессии

   ```
   postgres=*# select * from person;
   id | first_name | last_name
   ----+------------+-----------
   1 | ivan       | ivanov
   2 | peetr      | petrov
   3 | sergey     | sergeev
   (3 rows)
   ```
   ### Новой записи нет, т.к. транзакция в первой сессии не завершена и "грязное чтение" не допускается в postgres

16. Завершаем транзакцию в первой сессии

   ```
   postgres=*# commit;
   COMMIT
   ```
   
17. Проверяем наличие записи во второй сессии

   ```
   postgres=*# select * from person;
   id | first_name | last_name
   ----+------------+-----------
   1 | ivan       | ivanov
   2 | peetr      | petrov
   3 | sergey     | sergeev
   (3 rows)
   ```
   ### Новой записи нет, т.к. уровень изоляции текущей транзакции repeatable read, что не допускает "фантомное чтение"

18. Завершаем транзакцию во второй сессии и проверяем наличие записи

   ```
   postgres=*# commit;
   COMMIT
   postgres=# select * from person;
   id | first_name | last_name
   ----+------------+-----------
   1 | ivan       | ivanov
   2 | peetr      | petrov
   3 | sergey     | sergeev
   4 | sveta      | svetova
   (4 rows)
   ```
   ### Новая запись есть, т.к. данные зафиксированы в бд и во второй сессии новая транзакция