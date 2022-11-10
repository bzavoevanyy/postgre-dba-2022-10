## Установка и настройка PostgteSQL в контейнере Docker

1. На подготовленном инстансе в GCP устанавливаем Docker Engine

    ```
   curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER
   ```
   
2. Создаем каталог /var/lib/postgres

   ```
   sudo mkdir /var/lib/postgres
   ```
   
3. Создаем docker сеть

   ```
   sudo docker network create pg-net
   ```
   
4. Создаем контейнер-сервер с postgres-14 и монтируем к нему /var/lib/postgres

   ```
   sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
   ```
   
   ```
   sudo docker ps
   CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
   1e9bbb8a949b   postgres:14   "docker-entrypoint.s…"   28 seconds ago   Up 27 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
   ```
   
5. Запускаем контейнер-клиент и подключаемся из него к pg-server

   ```
   sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
   Password for user postgres:
   psql (14.5 (Debian 14.5-2.pgdg110+2))
   Type "help" for help.

   postgres=#
   ```
   
6. Создаем таблицу в контейнере-сервере

   ```
   postgres=# create table person(id serial, first_name text, last_name text);
   CREATE TABLE
   postgres=# insert into person (first_name, last_name) values ('Ivan', 'Ivanov');
   INSERT 0 1
   postgres=# insert into person (first_name, last_name) values ('Petr', 'Petrov');
   INSERT 0 1
   postgres=#
   ```
   
7. Подключаемся к контейнеру-серверу с локальной машины

   ```
   Добавляем firewall rule для открытия порта 5432 для внешних подключений
   
   gcloud compute firewall-rules create postgres-allow --allow tcp:5432 --source-tags=docker-postgres --source-ranges=0.0.0.0/0
   
   Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/postgres2022-01/global/firewalls/postgres-allow].
   Creating firewall...done.
   NAME            NETWORK  DIRECTION  PRIORITY  ALLOW     DENY  DISABLED
   postgres-allow  default  INGRESS    1000      tcp:5432        False
   ```

   ```
   Подключаемся клиентом psql
   
   psql -h 35.204.60.24 -U postgres
   Password for user postgres:
   psql (14.5)
   Type "help" for help.
   
   postgres=# \d
   List of relations
   Schema |     Name      |   Type   |  Owner
   --------+---------------+----------+----------
   public | person        | table    | postgres
   public | person_id_seq | sequence | postgres
   (2 rows)
   
   postgres=#
   ```

8. Удаляем контейнер-сервер

   ```
   sudo docker container stop pg-server && sudo docker container prune
   pg-server
   WARNING! This will remove all stopped containers.
   Are you sure you want to continue? [y/N] y
   Deleted Containers:
   1e9bbb8a949b80ea13ec379c594d3f6ef18ecf25e811caa273dd8f4d8bb5c4dd
   
   Total reclaimed space: 0B
   sudo docker ps -a
   CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
   ```

9. Создаем новый контейнер-сервер и проверяем, что данные остались на месте

   ```
   sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
   f66cfd49e980238db0f1a3745ef95c8bd90812d096649b801e417fc070d017fa
   ```
   
   Запускаем контейнер-клиент
   
   ```
   sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
   Password for user postgres:
   psql (14.5 (Debian 14.5-2.pgdg110+2))
   Type "help" for help.
   
   postgres=# select * from person;
   id | first_name | last_name
   ----+------------+-----------
   1 | Ivan       | Ivanov
   2 | Petr       | Petrov
   (2 rows)
   
   postgres=#
   ```
   ### Данные на месте, т.к. они хранятся вне контейнеров и не удаляются при удалении контейнера

   