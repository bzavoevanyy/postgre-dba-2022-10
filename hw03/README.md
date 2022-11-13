## Установка и настройка PostgreSQL, работа с дополнительными дисками

1. На ранее созданный инстанс c Ubuntu 20.04 LTS устанавливаем PostgreSQL 14

    ```
    sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
    
    bogdanzavoevanyy@postgres:~$ pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
    ```
   
2. Создаем простую таблицу с данными

    ```
    bogdanzavoevanyy@postgres:~$ sudo -u postgres psql

    psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
    Type "help" for help.
    
    postgres=#
   
    postgres=# create table test(c1 text);
    CREATE TABLE
    postgres=# insert into test (c1) values ('1');
    INSERT 0 1
    postgres=#
    ```
   
3. Останавливаем кластер

   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pg_ctlcluster 14 main stop
   bogdanzavoevanyy@postgres:~$ pg_lsclusters
   Ver Cluster Port Status Owner    Data directory              Log file
   14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
   ```
   
4. Создаем дополнительный диск 

   ```
   gcloud compute disks create disk-1 --project=postgres2022-01 --type=pd-ssd --size=10GB --zone=europe-west4-a
   
   Created [https://www.googleapis.com/compute/v1/projects/postgres2022-01/zones/europe-west4-a/disks/disk-1].
   NAME    ZONE            SIZE_GB  TYPE    STATUS
   disk-1  europe-west4-a  10       pd-ssd  READY
   
   New disks are unformatted. You must format and mount a disk before it
   can be used. You can find instructions on how to do this at:
   
   https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting
   ```
   
5. Добавляем новый диск к инстансу postgres

   ```
   gcloud compute instances attach-disk postgres --disk disk-1
   No zone specified. Using zone [europe-west4-a] for instance: [postgres].
   Updated [https://www.googleapis.com/compute/v1/projects/postgres2022-01/zones/europe-west4-a/instances/postgres].
   
   bogdanzavoevanyy@postgres:~$ lsblk
   NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   loop0     7:0    0  55.6M  1 loop /snap/core18/2566
   loop1     7:1    0  63.2M  1 loop /snap/core20/1623
   loop2     7:2    0 301.2M  1 loop /snap/google-cloud-cli/77
   loop3     7:3    0  67.8M  1 loop /snap/lxd/22753
   loop4     7:4    0    48M  1 loop /snap/snapd/17029
   sda       8:0    0    10G  0 disk
   ├─sda1    8:1    0   9.9G  0 part /
   ├─sda14   8:14   0     4M  0 part
   └─sda15   8:15   0   106M  0 part /boot/efi
   
   Новый диск
   sdb       8:16   0    10G  0 disk  
   ```
   
6. Форматируем и монтируем новый диск

   ```
   bogdanzavoevanyy@postgres:~$  sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
   mke2fs 1.45.5 (07-Jan-2020)
   Discarding device blocks: done
   Creating filesystem with 2621440 4k blocks and 655360 inodes
   Filesystem UUID: 1a6488b8-c4d2-48e8-9ab6-923306f43750
   Superblock backups stored on blocks:
   32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632
   
   Allocating group tables: done
   Writing inode tables: done
   Creating journal (16384 blocks): done
   Writing superblocks and filesystem accounting information: done
   
   bogdanzavoevanyy@postgres:~$ sudo mkdir /mnt/data
   bogdanzavoevanyy@postgres:~$ sudo mount -o discard,defaults /dev/sdb /mnt/data
   bogdanzavoevanyy@postgres:~$ lsblk
   NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   loop0     7:0    0  55.6M  1 loop /snap/core18/2566
   loop1     7:1    0  63.2M  1 loop /snap/core20/1623
   loop2     7:2    0 301.2M  1 loop /snap/google-cloud-cli/77
   loop3     7:3    0  67.8M  1 loop /snap/lxd/22753
   loop4     7:4    0    48M  1 loop /snap/snapd/17029
   sda       8:0    0    10G  0 disk
   ├─sda1    8:1    0   9.9G  0 part /
   ├─sda14   8:14   0     4M  0 part
   └─sda15   8:15   0   106M  0 part /boot/efi
   sdb       8:16   0    10G  0 disk /mnt/data
   ```
   
7. Добавляем конфигурацию, что бы диск монтировался автоматически при рестарте инстанса

   ```
   bogdanzavoevanyy@postgres:~$ sudo cp /etc/fstab /etc/fstab.backup
   bogdanzavoevanyy@postgres:~$ sudo blkid /dev/sdb
   /dev/sdb: UUID="1a6488b8-c4d2-48e8-9ab6-923306f43750" TYPE="ext4"
   ```
   
   Добавляем запись в /etc/fstab
   ```
   sudo sh -c "echo \"UUID=1a6488b8-c4d2-48e8-9ab6-923306f43750 /mnt/data ext4 discard,defaults,nofail 0 2\" >> /etc/fstab"
   ```
   
   Проверяем
   ```
   bogdanzavoevanyy@postgres:~$ cat /etc/fstab
   LABEL=cloudimg-rootfs	/	 ext4	defaults	0 1
   LABEL=UEFI	/boot/efi	vfat	umask=0077	0 1
   UUID=1a6488b8-c4d2-48e8-9ab6-923306f43750 /mnt/data ext4 discard,defaults,nofail 0 2
   ```
   
8. Перезагружаем инстанс и проверяем, что диск монтируется

   ```
   gcloud compute instances stop postgres 
   No zone specified. Using zone [europe-west4-a] for instance: [postgres].
   Stopping instance(s) postgres...done.
   Updated [https://compute.googleapis.com/compute/v1/projects/postgres2022-01/zones/europe-west4-a/instances/postgres].
   ```
   ```
   gcloud compute instances start postgres
   No zone specified. Using zone [europe-west4-a] for instance: [postgres].
   Starting instance(s) postgres...done.
   Updated [https://compute.googleapis.com/compute/v1/projects/postgres2022-01/zones/europe-west4-a/instances/postgres].
   Instance internal IP is 10.164.0.4
   Instance external IP is 34.90.144.120
   ```
   ```
   bogdanzavoevanyy@postgres:~$ lsblk
   NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   loop0     7:0    0  55.6M  1 loop /snap/core18/2566
   loop1     7:1    0  63.2M  1 loop /snap/core20/1623
   loop2     7:2    0 301.2M  1 loop /snap/google-cloud-cli/77
   loop3     7:3    0  67.8M  1 loop /snap/lxd/22753
   loop4     7:4    0    48M  1 loop /snap/snapd/17029
   sda       8:0    0    10G  0 disk
   ├─sda1    8:1    0   9.9G  0 part /
   ├─sda14   8:14   0     4M  0 part
   └─sda15   8:15   0   106M  0 part /boot/efi
   sdb       8:16   0    10G  0 disk /mnt/data
   ```
   
9. Делаем пользователя postgres владельцем /mnt/data

   ```
   sudo chown -R postgres:postgres /mnt/data/
   ```
   
10. Переносим содержимое /var/lib/postgres/14 в /mnt/data

   ```
   sudo -u postgres mv /var/lib/postgresql/14 /mnt/data
   ```
   
11. Пробуем запустить кластер

   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pg_ctlcluster 14 main start
   Error: /var/lib/postgresql/14/main is not accessible or does not exist
   ```
   
12. Ищем нужный параметр в конфиг-файле и меняем его

   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres cat /etc/postgresql/14/main/postgresql.conf | grep /var/lib
   data_directory = '/var/lib/postgresql/14/main'		# use data in another directory    
   ```
   
   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres sed -i "s|"/var/lib/postgresql"|"/mnt/data"|" /etc/postgresql/14/main/postgresql.conf
   ```
   
   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres cat /etc/postgresql/14/main/postgresql.conf | grep data_directory
   data_directory = '/mnt/data/14/main'		# use data in another directory
   ```
   
13. Запускаем кластер, подключаемся и проверяем наличие нашей таблицы

   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres pg_ctlcluster 14 main start
   
   bogdanzavoevanyy@postgres:~$ pg_lsclusters
   Ver Cluster Port Status Owner    Data directory    Log file
   14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log
   ```
   
   ```
   bogdanzavoevanyy@postgres:~$ sudo -u postgres psql
   psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   Type "help" for help.
   
   postgres=# show data_directory;
   data_directory
   -------------------
   /mnt/data/14/main
   (1 row)
   
   postgres=# \d
   List of relations
   Schema | Name | Type  |  Owner
   --------+------+-------+----------
   public | test | table | postgres
   (1 row)
   
   postgres=# select * from test ;
   c1
   ----
   1
   (1 row)
   
   postgres=#
   ```
   
14. Создаем второй инстанс postgres2, устанавливаем postgres и редактируем конфиг файл

   ```
   bogdanzavoevanyy@postgres2:~$ sudo systemctl status postgresql
   ● postgresql.service - PostgreSQL RDBMS
   Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
   Active: active (exited) since Sun 2022-11-13 15:41:27 UTC; 11min ago
   Main PID: 10497 (code=exited, status=0/SUCCESS)
   Tasks: 0 (limit: 4694)
   Memory: 0B
   CGroup: /system.slice/postgresql.service
   ```

   ```
   bogdanzavoevanyy@postgres2:~$ sudo -u postgres sed -i "s|"/var/lib/postgresql"|"/mnt/data"|" /etc/postgresql/14/main/postgresql.conf
   bogdanzavoevanyy@postgres2:~$ sudo -u postgres cat /etc/postgresql/14/main/postgresql.conf | grep data_directory
   data_directory = '/mnt/data/14/main'		# use data in another directory
   ```
   
15. Останавливаем postgres на первом инстансе, удаляем у него дополнительный диск, добавляем диск ко второму инстансу

   ```
   sudo -u postgres pg_ctlcluster 14 main stop
   ```

   ```
   bogdanzavoevanyy@postgres:~$ sudo umount /mnt/data
   ```
   
   ```
   gcloud compute instances detach-disk postgres --disk disk-1
   No zone specified. Using zone [europe-west4-a] for instance: [postgres].
   Updated [https://www.googleapis.com/compute/v1/projects/postgres2022-01/zones/europe-west4-a/instances/postgres].
   ```
   
   ```
   gcloud compute instances attach-disk postgres2 --disk disk-1
   No zone specified. Using zone [europe-west4-a] for instance: [postgres2].
   Updated [https://www.googleapis.com/compute/v1/projects/postgres2022-01/zones/europe-west4-a/instances/postgres2].
   ```
   
16. Монтируем диск во втором инстансе (настраиваем монтирование при загрузке системы согласно п.7)

   ```
   bogdanzavoevanyy@postgres2:~$ sudo mkdir /mnt/data
   bogdanzavoevanyy@postgres2:~$ sudo mount -o discard,defaults /dev/sdb /mnt/data
   bogdanzavoevanyy@postgres2:~$ lsblk
   NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   loop0     7:0    0  55.6M  1 loop /snap/core18/2566
   loop1     7:1    0  63.2M  1 loop /snap/core20/1623
   loop2     7:2    0 301.2M  1 loop /snap/google-cloud-cli/77
   loop3     7:3    0  67.8M  1 loop /snap/lxd/22753
   loop4     7:4    0    48M  1 loop /snap/snapd/17029
   sda       8:0    0    10G  0 disk
   ├─sda1    8:1    0   9.9G  0 part /
   ├─sda14   8:14   0     4M  0 part
   └─sda15   8:15   0   106M  0 part /boot/efi
   sdb       8:16   0    10G  0 disk /mnt/data
   ```
   
17. Перезапускаем кластер и проверяем работу

   ```
   bogdanzavoevanyy@postgres2:~$ sudo systemctl restart postgresql@14-main
   bogdanzavoevanyy@postgres2:~$ sudo -u postgres psql
   psql (14.6 (Ubuntu 14.6-1.pgdg20.04+1))
   Type "help" for help.
   
      
   postgres=# show data_directory;
   data_directory
   -------------------
   /mnt/data/14/main
   (1 row)
   
   postgres=# \d
   List of relations
   Schema | Name | Type  |  Owner
   --------+------+-------+----------
   public | test | table | postgres
   (1 row)
   
   postgres=# select * from test;
   c1
   ----
   1
   (1 row)
   
   postgres=# insert into test(c1) values ('2');
   INSERT 0 1
   postgres=# select * from test;
   c1
   ----
   1
   2
   (2 rows)
   
   postgres=#
   ```
   
   