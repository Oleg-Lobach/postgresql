# Домашнее задание
## Реализовать свой миникластер на 3 ВМ.

### 1. Выполним подготовительные работы по разертыванию и настройке миникластера 
#### 1.1 Создадим 4 ВМ c установленной ОС Ubuntu 20.04 LTS
```
gcloud compute instances create instance-12-1 --project=postgres2021-lobach-02 --zone=us-central1-c --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=180845300769-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-12-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20211212,mode=rw,size=10,type=projects/postgres2021-lobach-02/zones/us-central1-c/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
Created [https://www.googleapis.com/compute/v1/projects/postgres2021-lobach-02/zones/us-central1-c/instances/instance-12-1].
NAME: instance-12-1
ZONE: us-central1-c
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.10
EXTERNAL_IP: 35.232.155.64
STATUS: RUNNING
```
```
gcloud compute instances create instance-12-2 --project=postgres2021-lobach-02 --zone=us-central1-c --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=180845300769-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-12-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20211212,mode=rw,size=10,type=projects/postgres2021-lobach-02/zones/us-central1-c/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
Created [https://www.googleapis.com/compute/v1/projects/postgres2021-lobach-02/zones/us-central1-c/instances/instance-12-2].
NAME: instance-12-2
ZONE: us-central1-c
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.11
EXTERNAL_IP: 34.72.177.109
STATUS: RUNNING
```
```
gcloud compute instances create instance-12-3 --project=postgres2021-lobach-02 --zone=us-central1-c --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=180845300769-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-12-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20211212,mode=rw,size=10,type=projects/postgres2021-lobach-02/zones/us-central1-c/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
Created [https://www.googleapis.com/compute/v1/projects/postgres2021-lobach-02/zones/us-central1-c/instances/instance-12-3].
NAME: instance-12-3
ZONE: us-central1-c
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.12
EXTERNAL_IP: 35.188.149.73
STATUS: RUNNING
```
```
gcloud compute instances create instance-12-4 --project=postgres2021-lobach-02 --zone=us-central1-c --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=180845300769-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-12-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20211212,mode=rw,size=10,type=projects/postgres2021-lobach-02/zones/us-central1-c/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
Created [https://www.googleapis.com/compute/v1/projects/postgres2021-lobach-02/zones/us-central1-c/instances/instance-12-4].
NAME: instance-12-4
ZONE: us-central1-c
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.13
EXTERNAL_IP: 35.192.28.127
STATUS: RUNNING
```
#### 1.2 Установим Postgresql 14 на четырех ВМ
```
    sudo sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt -y update &&  sudo apt -y install postgresql-14

```
#### 1.3 Проверим, что кластер успешно развернут и работает на каждой из четырех ВМ
```
OLobach@instance-12-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

OLobach@instance-12-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

OLobach@instance-12-3:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

OLobach@instance-12-4:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```
#### 1.4 Выполним настройку доступа по паролю к БД ВМ1  для логической репликции
##### 1.4.1 Разрешим подключение к ВМ1 на внешний ip-адрес
``` 
echo "listen_addresses = '10.128.0.10'" >> /etc/postgresql/14/main/postgresql.conf 
```
##### 1.4.2 Установим режим логической репликации 
```
echo "wal_level = logical" >>  /etc/postgresql/14/main/postgresql.conf
```
##### 1.4.3 Настроим доступ для репликации для пользовате repuser c  ВМ2 и ВМ3
```
echo "host repdb repuser 10.128.0.11/32 md5" >> /etc/postgresql/14/main/pg_hba.conf
echo "host repdb repuser 10.128.0.12/32 md5" >> /etc/postgresql/14/main/pg_hba.conf
```
##### 1.4.4 Настроим firewall для доступа с ВМ2 и ВМ3 к ВМ1 на порт 5432
```
sudo ufw allow from 10.128.0.11 to any port 5432
Rules updated
sudo ufw allow 10.128.0.12 to any port 5432
Rules updated
```
##### 1.4.5 Выполним перезапуск кластера на ВМ1 и убедимся, что он корректно запущен
```
sudo pg_ctlcluster 14 main restart && sudo pg_lsclusters

OLobach@instance-12-1:~$ sudo pg_ctlcluster 14 main restart && sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```
#### 1.5 Выполним настройку ВМ2  для логической репликции
##### 1.5.1 Разрешим подключение к ВМ2 на внешний ip-адрес
``` 
echo "listen_addresses = '10.128.0.11'" >> /etc/postgresql/14/main/postgresql.conf 
```
##### 1.5.2 Установим режим логической репликации 
```
echo "wal_level = logical" >>  /etc/postgresql/14/main/postgresql.conf
```
##### 1.5.3 Настроим доступ для репликации для пользовате repuser c  ВМ1 и ВМ3
```
echo "host repdb repuser 10.128.0.10/32 md5" >> /etc/postgresql/14/main/pg_hba.conf &&
echo "host repdb repuser 10.128.0.12/32 md5" >> /etc/postgresql/14/main/pg_hba.conf
```
##### 1.5.4 Настроим firewall для доступа с ВМ1 и ВМ3 к ВМ2 на порт 5432
```
sudo ufw allow from 10.128.0.10 to any port 5432
Rules updated
sudo ufw allow 10.128.0.12 to any port 5432
Rules updated
```
##### 1.5.5 Выполним перезапуск кластера на ВМ2 и убедимся, что он корректно запущен
```
sudo pg_ctlcluster 14 main restart &&
sudo pg_lsclusters
OLobach@instance-12-2:~$ sudo pg_ctlcluster 14 main restart &&
> sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

### 2. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
#### 2.1 Создадим БД repdb на ВМ1
```
create database repdb;
```
#### 2.2 Создадим таблицы test и test2 в БД repdb на ВМ1
```
postgres=# create database repdb;
CREATE DATABASE
postgres=# \c repdb
You are now connected to database "repdb" as user "postgres".
repdb=# create table test (id serial primary key, name text, age integer);
CREATE TABLE
repdb=# create table test2 (id serial primary key, city text);
CREATE TABLE

```
#### 2.3 Создадим пользовтеля repuser и настроим его права для выполнения репликации на ВМ1
```

repdb=# create role repuser with replication login password 'rep123';
CREATE ROLE
repdb=# grant all privileges on database repdb to repuser;
GRANT
repdb=# grant all privileges on all tables in schema public to repuser;
GRANT
```

### 3. Создаем публикацию таблицы test на ВМ1

```
repdb=# create publication pub_test;
CREATE PUBLICATION
repdb=# alter publication pub_test add table test;
ALTER PUBLICATION
```
Проверим создание публикации
```

repdb=# \dRp+
                            Publication pub_test
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"

```

### 4. На ВМ2 создаем таблицы test2 для записи, test для запросов на чтение. 
#### 4.1 Создадим БД repdb на ВМ2
```
create database repdb;
```
#### 4.2 Создадим таблицы test и test2 в БД repdb на ВМ2
```
postgres=# \c repdb
You are now connected to database "repdb" as user "postgres".
repdb=# create table test (id serial primary key, name text, age integer);
CREATE TABLE
repdb=# create table test2 (id serial primary key, city text);
CREATE TABLE
```
#### 4.3 Создадим пользовтеля repuser и настроим его права для выполнения репликации
```
repdb=# create role repuser with replication login password 'rep123';
CREATE ROLE
repdb=# grant all privileges on database repdb to repuser;
GRANT
repdb=# grant all privileges on all tables in schema public to repuser;
GRANT

```
### 5. Создаем публикацию таблицы test2 на ВМ2 и подписываемся на публикацию таблицы test с ВМ №1. 
#### 5.1 Создадим публикацию таблицы test2 на ВМ2
```
repdb=# create publication pub_test2;
CREATE PUBLICATION
repdb=# alter publication pub_test2 add table test2;
ALTER PUBLICATION
```
Проверим создание публикации
```
repdb=# \dRp+
                           Publication pub_test2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```
#### 5.2 Создаем подписку  на публикацию таблицы test c ВМ №1
```
repdb=# create subscription sub_test connection 'host=10.128.0.10 port=5432 password=rep123 user=repuser dbname=repdb' PUBLICATION pub_test;
NOTICE:  created replication slot "sub_test" on publisher
CREATE SUBSCRIPTION
```
Проверим создание подписки
```
\dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication
-----------+----------+---------+-------------
 sub_test | postgres | t       | {pub_test}
(1 row)
```
#### 6 Создаем подписку  на публикацию pub_test2 таблицы test2 c ВМ №2. 
```

repdb=# create subscription sub_test2 connection 'host=10.128.0.11 port=5432 password=rep123 user=repuser dbname=repdb' PUBLICATION pub_test2;
NOTICE:  created replication slot "sub_test2" on publisher
CREATE SUBSCRIPTION

```
Проверим создание подписки
```
repdb=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication
-----------+----------+---------+-------------
 sub_test2 | postgres | t       | {pub_test2}
(1 row)

```
### 7.  ВМ3 использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). 
### 7.1 Создадим БД repdb и таблицы test и test2 на ВМ3
```
create database repdb;
```
#### 7.2 Создадим таблицы test и test2 в БД repdb на ВМ3
```
\c repdb
create table test (id serial primary key, name text, age integer);
create table test2 (id serial primary key, city text);
```
#### 7.3 Создадим подписку на публикации pub_test c ВМ1 и pub_test2 c ВМ2 
```
repdb=# create subscription sub_test3 connection 'host=10.128.0.10 port=5432 password=rep123 user=repuser dbname=repdb' PUBLICATION pub_test;
NOTICE:  created replication slot "sub_test3" on publisher
CREATE SUBSCRIPTION


repdb=# create subscription sub_test4 connection 'host=10.128.0.11 port=5432 password=rep123 user=repuser dbname=repdb' PUBLICATION pub_test2;
NOTICE:  created replication slot "sub_test4" on publisher
CREATE SUBSCRIPTION

```
Проверим создание подписки
```
repdb=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication
-----------+----------+---------+-------------
 sub_test3 | postgres | t       | {pub_test}
 sub_test4 | postgres | t       | {pub_test2}
(2 rows)

```
### 8. Небольшое описание, того, что получилось.
В результате описанных действий реализована следующая схема логической репликации:
- Изменяемая на ВМ1 таблица test логически реплицируется на ВМ2 и ВМ3
- Изменяемая на ВМ2 таблица test2 логически реплицируется на ВМ1 и ВМ3

Проиллистрируем эти факты на конкретных примерах
#### 8.1 Добавим новые данные в таблицу test на ВМ1. Убедимся, что эти данные атоматически появились в таблице test на ВМ2 и ВМ3:
```
ВМ1:
repdb=# insert into test values (1, 'Владимир', 32), (2, 'Николай', 53);
INSERT 0 2
ВМ1:
ВМ2:
ВМ3:
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
(2 rows)

```  

### 9. Реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. 
#### 9.1 Выполним настройку ВМ3 для физической репликции
##### 9.1.1 Разрешим подключение к ВМ3 на внешний ip-адрес
``` 
postgres@instance-12-3:/home/OLobach$ echo "listen_addresses = '10.128.0.12'" >> /etc/postgresql/14/main/postgresql.conf 
```

##### 9.1.2 Настроим доступ для репликации для пользователя repstreamuser c  ВМ4
```
postgres@instance-12-3:/home/OLobach$ echo "host replication repstreamuser 10.128.0.13/32 md5" >> /etc/postgresql/14/main/pg_hba.conf

```
##### 9.1.3 Настроим firewall для доступа с ВМ4  к ВМ3 на порт 5432
```
OLobach@instance-12-3:~$ sudo ufw allow from 10.128.0.13 to any port 5432
Rules updated

```
##### 9.1.4 Создадим пользователя от имени которого будет выполняться физическая репликация 
ВМ3:
```
postgres=# create role repstreamuser with replication password '123stream' login;
CREATE ROLE
```
##### 9.1.5 Убедимся, что параметры настройки физической репликации содержат корректные значения
ВМ3:
```
postgres=# show wal_level;
 wal_level
-----------
 replica
(1 row)

postgres=# show max_wal_senders;
 max_wal_senders
-----------------
 10
(1 row)

postgres=# show max_replication_slots;
 max_replication_slots
-----------------------
 10
(1 row)

postgres=# show wal_sender_timeout;
 wal_sender_timeout
--------------------
 1min

```
##### 9.1.6 Выполним перезапуск кластера на ВМ3 и убедимся, что он корректно запущен
```
OLobach@instance-12-3:~$ sudo pg_ctlcluster 14 main restart
OLobach@instance-12-3:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
#### 9.2 Выполним настройку ВМ4 в качестве приемника физической репликации
##### 9.2.1 Выполним очистку файлов кластера 
```
OLobach@instance-12-4:~$ sudo -u postgres rm -rf /var/lib/postgresql/14/main/

```
##### 9.2.2 Инициируем физическую репликации с ВМ3 на ВМ4 - выполним перевод кластера ВМ4 в режим standby
```
OLobach@instance-12-4:~$ sudo -u postgres pg_basebackup -h 10.128.0.12 -p 5432 -U repstreamuser -D /var/lib/postgresql/14/main/  -R
Password:
OLobach@instance-12-4:~$ sudo -u postgres ls -l /var/lib/postgresql/14/main
total 260
-rw------- 1 postgres postgres      3 Jan  7 15:25 PG_VERSION
-rw------- 1 postgres postgres    225 Jan  7 15:25 backup_label
-rw------- 1 postgres postgres 181313 Jan  7 15:25 backup_manifest
drwx------ 6 postgres postgres   4096 Jan  7 15:25 base
drwx------ 2 postgres postgres   4096 Jan  7 15:25 global
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_commit_ts
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_dynshmem
drwx------ 4 postgres postgres   4096 Jan  7 15:25 pg_logical
drwx------ 4 postgres postgres   4096 Jan  7 15:25 pg_multixact
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_notify
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_replslot
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_serial
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_snapshots
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_stat
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_stat_tmp
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_subtrans
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_tblspc
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_twophase
drwx------ 3 postgres postgres   4096 Jan  7 15:25 pg_wal
drwx------ 2 postgres postgres   4096 Jan  7 15:25 pg_xact
-rw------- 1 postgres postgres    334 Jan  7 15:25 postgresql.auto.conf
-rw------- 1 postgres postgres      0 Jan  7 15:25 standby.signal
```
##### 9.2.3 Для повышения надежности физической репликации настроим слот репликации
9.2.3.1 На ВМ3 создадим слот физической репликации
```
repdb=# select pg_create_physical_replication_slot('vm3_slot');
 pg_create_physical_replication_slot
-------------------------------------
 (vm3_slot,)
(1 row)

```
Проверим, что слот создан:
```
repdb=# select slot_name, slot_type, active FROM pg_replication_slots;
 slot_name | slot_type | active
-----------+-----------+--------
 vm3_slot  | physical  | f
(1 row)

```
9.2.3.2 На ВМ4 добавим данные по слоту репликации в файл postgresql.conf
```

echo "primary_slot_name = 'vm3_slot'" >>  /etc/postgresql/14/main/postgresql.conf
```
##### 9.2.4 Выполним рестарт кластера 
```
OLobach@instance-12-4:~$ sudo pg_ctlcluster 14 main restart
OLobach@instance-12-4:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
##### 9.2.5 Убедимся, что изменился статус слота репликации для ВМ3:
```
repdb=# select slot_name, slot_type, active FROM pg_replication_slots;
 slot_name | slot_type | active
-----------+-----------+--------
 vm3_slot  | physical  | t
(1 row)
```
##### 9.2.6 Убедимся, что данные с ВМ3 появились на ВМ4 
```
postgres@instance-12-4:/home/OLobach$ psql
psql (14.1 (Ubuntu 14.1-2.pgdg20.04+1))
Type "help" for help.

postgres=# \c repdb
You are now connected to database "repdb" as user "postgres".
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
(2 rows)

```

### 10 Проверим состояние репликации на ВМ1, ВМ2 и ВМ3


#### 10.1 Для ВМ1
```
repdb=#postgres-# select * from pg_stat_replication
-[ RECORD 1 ]----+------------------------------
pid              | 852
usesysid         | 16403
usename          | repuser
application_name | sub_test
client_addr      | 10.128.0.11
client_hostname  |
client_port      | 34570
backend_start    | 2022-01-08 08:14:07.910158+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/1732C88
write_lsn        | 0/1732C88
flush_lsn        | 0/1732C88
replay_lsn       | 0/1732C88
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2022-01-08 08:22:10.553972+00
-[ RECORD 2 ]----+------------------------------
pid              | 854

```
#### 10.2 Для ВМ2
```
repdb=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 695
usesysid         | 16403
usename          | repuser
application_name | sub_test2
client_addr      | 10.128.0.10
client_hostname  |
client_port      | 41864
backend_start    | 2022-01-08 08:14:10.644554+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/1734D20
write_lsn        | 0/1734D20
flush_lsn        | 0/1734D20
replay_lsn       | 0/1734D20
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2022-01-08 08:26:21.044375+00
-[ RECORD 2 ]----+------------------------------
pid              | 844


```
#### 10.3 Для ВМ3
```
postgres=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 854
usesysid         | 16409
usename          | repstreamuser
application_name | 14/main
client_addr      | 10.128.0.13
client_hostname  |
client_port      | 57020
backend_start    | 2022-01-08 08:14:24.871834+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000958
write_lsn        | 0/3000958
flush_lsn        | 0/3000958
replay_lsn       | 0/3000958
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2022-01-08 08:23:38.724791+00


```
### 11 Настроим синхронный режим для репликации ВМ3->ВМ4
#### 11.1 Установим необходимые параметры
```
ostgres@instance-12-3:/home/User$ echo "synchronous_standby_names  = '*'" >>  /etc/postgresql/14/main/postgresql.conf
postgres@instance-12-3:/home/User$ echo "synchronous_commit = remote_apply" >>  /etc/postgresql/14/main/postgresql.conf

```
#### 11.2 После рестарта кластера проверим, что параметры установлены верно
```
postgres=# show synchronous_standby_names;
 synchronous_standby_names
---------------------------
 *
(1 row)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 remote_apply
(1 row)

```
#### 11.3 Убедимся, что статус репликации на ВМ3 изменился на синхронный
```

postgres=#  select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 1737
usesysid         | 16409
usename          | repstreamuser
application_name | 14/main
client_addr      | 10.128.0.13
client_hostname  |
client_port      | 57410
backend_start    | 2022-01-08 09:56:04.525641+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000FD0
write_lsn        | 0/3000FD0
flush_lsn        | 0/3000FD0
replay_lsn       | 0/3000FD0
write_lag        | 00:00:00.002155
flush_lag        | 00:00:00.002168
replay_lag       | 00:00:00.002175
sync_priority    | 1
sync_state       | sync
reply_time       | 2022-01-08 09:56:14.71666+00

```
### 12 Проверим работу настроенного механизма репликации на реальных данных
#### 12.1 Добавим данные в таблицу test на ВМ1
```
repdb=#  insert into test values  ( 3, 'Ольга', 29), ( 4, 'Андрей', 23);
INSERT 0 2
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
  3 | Ольга    |  29
  4 | Андрей   |  23
(4 rows)
```
#### 12.2 Добавим данные в таблицу test2 на ВМ2
```
repdb=# insert into test2 values (1, 'Рязань'),(2, 'Калининград');
INSERT 0 2
repdb=# select * from test2;
 id |    city
----+-------------
  1 | Рязань
  2 | Калининград
(2 rows)

```
#### 12.3 Убедимся, что всех ВМ в таблицах test и test2 присутствуют идентичные данные
##### 12.3.1 Для ВМ1
```
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
  3 | Ольга    |  29
  4 | Андрей   |  23
(4 rows)

repdb=# select * from test2;
 id |    city
----+-------------
  1 | Рязань
  2 | Калининград
(2 rows)

```
##### 12.3.2 Для ВМ2
```
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
  3 | Ольга    |  29
  4 | Андрей   |  23
(4 rows)

repdb=# select * from test2;
 id |    city
----+-------------
  1 | Рязань
  2 | Калининград
(2 rows)

```
##### 12.3.3 Для ВМ3
```
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
  3 | Ольга    |  29
  4 | Андрей   |  23
(4 rows)

repdb=# select * from test2;
 id |    city
----+-------------
  1 | Рязань
  2 | Калининград
(2 rows)

```
#### 12.3.4 Для ВМ4
```
repdb=# select * from test;
 id |   name   | age
----+----------+-----
  1 | Владимир |  32
  2 | Николай  |  53
  3 | Ольга    |  29
  4 | Андрей   |  23
(4 rows)

repdb=# select * from test2;
 id |    city
----+-------------
  1 | Рязань
  2 | Калининград
(2 rows)

```
# 13. Написать с какими проблемами столкнулись.
 - Возникла сложность с настройкой параметров доступа для пользователей. Настройки доступа для логической и физической репликации отличаются. Пришлось гуглить правильные параметры.
 - Непонятно какое имя необходимо задавать при настройке синхронной репликации для параметра synchronous_standby_names. Пришлось использовать значение '*', значение 'walreceiver' не привело к переключению режима из асинхронного в синхронный
 - В документации указано, что одновременное использование синхронного режима репликации  и слота репликации допустимо и позволяет повысить уровень надежности. Не очень понятно почему. Ведь синхронная репликация в режиме remote_apply гарантирует согласованность данных. 