# Домашнее задание
## PostgreSQL и Azure, GCP, AWS, ЯО
```
Цель:
научиться пользоваться PostgreSQL based и like сервисами в Azure, GCP, AWS, ЯО
Описание/Пошаговая инструкция выполнения домашнего задания:
2 вариант:
Воспользоваться PostgreSQL based и like сервисами в GCP. Описать что и как делали и с какими проблемами столкнулись
```
## 1. Развернем DBAAS в GCP
```
Public IP address: 35.222.158.100
Outgoing IP address: 34.71.250.45
Connection name: pgdba-340419:us-central1:gcp-pg-1
gcloud compute instances create instance-15 --project=pgdba-340419 --zone=us-central1-c --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=13258540026-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-16,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/pgdba-340419/zones/us-central1-c/diskTypes/pd-standard --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```
## 2. Настроим публичный доступ и подключимся к кластеру
### 2.1 Подключаемся
```
OLobach@instance-15:~$ psql -h 35.222.158.100 -U postgres
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1), server 14.0)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```
### 2.2 Проверим список БД
```
postgres=> \l
                                                List of databases

     Name      |       Owner       | Encoding |  Collate   |   Ctype    |            Access privileges

---------------+-------------------+----------+------------+------------+-----------------------------------------
cloudsqladmin | cloudsqladmin     | UTF8     | en_US.UTF8 | en_US.UTF8 |
postgres      | cloudsqlsuperuser | UTF8     | en_US.UTF8 | en_US.UTF8 |
template0     | cloudsqladmin     | UTF8     | en_US.UTF8 | en_US.UTF8 | =c/cloudsqladmin                       +
               |                   |          |            |            | cloudsqladmin=CTc/cloudsqladmin
template1     | cloudsqlsuperuser | UTF8     | en_US.UTF8 | en_US.UTF8 | =c/cloudsqlsuperuser                   +

               |                   |          |            |            | cloudsqlsuperuser=CTc/cloudsqlsuperuser
(4 rows)

```
## 3. Выполним тестирование кластера
### 3.1 Оценим свободное место на диске
```
postgres=> \! df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  2.1G  7.5G  22% /
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G   28K  2.0G   1% /dev/shm
tmpfs           393M  940K  392M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       56M   56M     0 100% /snap/core18/2284
/dev/loop1       62M   62M     0 100% /snap/core20/1361
/dev/loop4       44M   44M     0 100% /snap/snapd/14978
/dev/loop3      256M  256M     0 100% /snap/google-cloud-sdk/228
/dev/loop2       68M   68M     0 100% /snap/lxd/22526
/dev/sda15      105M  5.2M  100M   5% /boot/efi
tmpfs           393M     0  393M   0% /run/user/1001
```
### 3.2 Загрузим 2.5 ГБ данных из бакета по такси Чикаго
```
postgres=> \! mkdir /tmp/taxi_data
postgres=> \! ls -l /tmp/
total 20

drwx------ 3 root    root    4096 Mar 12 10:17 snap.lxd
drwx------ 3 root    root    4096 Mar 12 10:17 systemd-private-91a09e8cbe37429b8420f56941ce41a3-chrony.service-yQJcNg
drwx------ 3 root    root    4096 Mar 12 10:17 systemd-private-91a09e8cbe37429b8420f56941ce41a3-systemd-logind.service-4ezPqg
drwx------ 3 root    root    4096 Mar 12 10:17 systemd-private-91a09e8cbe37429b8420f56941ce41a3-systemd-resolved.service-uRLkIg
drwxrwxr-x 2 OLobach OLobach 4096 Mar 12 12:47 taxi_data

postgres=> \! gsutil -m cp -R gs://msk_dba_bucket/taxi_00000000000[0-9].csv /tmp/taxi_data/;
Copying gs://msk_dba_bucket/taxi_000000000000.csv...
Copying gs://msk_dba_bucket/taxi_000000000001.csv...
Copying gs://msk_dba_bucket/taxi_000000000006.csv...
Copying gs://msk_dba_bucket/taxi_000000000008.csv...
Copying gs://msk_dba_bucket/taxi_000000000007.csv...
Copying gs://msk_dba_bucket/taxi_000000000002.csv...
Copying gs://msk_dba_bucket/taxi_000000000003.csv...
Copying gs://msk_dba_bucket/taxi_000000000004.csv...
Copying gs://msk_dba_bucket/taxi_000000000009.csv...
Copying gs://msk_dba_bucket/taxi_000000000005.csv...

| [10/10 files][  2.5 GiB/  2.5 GiB] 100% Done  95.0 MiB/s ETA 00:00:00
Operation completed over 10 objects/2.5 GiB. 
```
### 3.3 Загрузим данные в БД
#### 3.3.1 Создаем таблицу
```
postgres=> create table taxi_trips (
postgres(>  unique_key text,
postgres(>  taxi_id text,
postgres(>  trip_start_timestamp TIMESTAMP,
postgres(>  trip_end_timestamp TIMESTAMP,
tolls numeric,
extras numeric,
trip_total numeric,
payment_type text,
company text,
pickup_latitude numeric,
pickup_longitude numeric,
pickup_location text,
dropoff_latitude numeric,
dropoff_longitude numeric,
dropoff_location text
postgres(>  trip_seconds bigint,
postgres(>  trip_miles numeric,
postgres(>  pickup_census_tract bigint,
postgres(>  dropoff_census_tract bigint,
postgres(>  pickup_community_area bigint,
postgres(>  dropoff_community_area bigint,
postgres(>  fare numeric,
postgres(>  tips numeric,
postgres(>  tolls numeric,
postgres(>  extras numeric,
postgres(>  trip_total numeric,
postgres(>  payment_type text,
postgres(>  company text,
postgres(>  pickup_latitude numeric,
postgres(>  pickup_longitude numeric,
postgres(>  pickup_location text,
postgres(>  dropoff_latitude numeric,
postgres(>  dropoff_longitude numeric,
postgres(>  dropoff_location text
postgres(>  );
CREATE TABLE
```
#### 3.3.2 Пытаемся загрузить данные в БД изнутри клиентского подключения
```
postgres=> COPY taxi_trips(unique_key,
postgres(> taxi_id,
postgres(> trip_start_timestamp,
postgres(> trip_end_timestamp,
postgres(> trip_seconds,
postgres(> trip_miles,
postgres(> pickup_census_tract,
postgres(> dropoff_census_tract,
postgres(> pickup_community_area,
postgres(> dropoff_community_area,
postgres(> fare,
postgres(> tips,
postgres(> tolls,
postgres(> extras,
postgres(> trip_total,
postgres(> payment_type,
postgres(> company,
postgres(> pickup_latitude,
postgres(> pickup_longitude,
postgres(> pickup_location,
postgres(> dropoff_latitude,
postgres(> dropoff_longitude,
postgres(> dropoff_location)
postgres-> FROM PROGRAM 'awk FNR-1 /tmp/taxi_data/*.csv | cat' DELIMITER ',' CSV HEADER;

ERROR:  must be superuser or a member of the pg_execute_server_program role to COPY to or from an external program
HINT:  Anyone can COPY to stdout or from stdin. psql's \copy command also works for anyone.

postgres=>
postgres=> \! psql -U postgres -c "\COPY taxitrips FROM '/tmp/taxi_data/taxi_000000000000.csv ' CSV HEADER;"
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "postgres"
```
#### 3.3.3 Загружаем данные в БД снаружи
```
OLobach@instance-15:/tmp/taxi_data$ date && for f in *.csv; do psql -h 35.222.158.100  -U postgres -c "\COPY taxi_trips FROM  PROGRAM 'cat $f' CSV HEADER"; done && date
Sat Mar 12 13:24:24 UTC 2022
COPY 656957
COPY 659652
COPY 650472
COPY 664317
COPY 656225
COPY 660814
COPY 668161
COPY 669651
COPY 628484
COPY 652369
COPY 670690
Sat Mar 12 13:25:54 UTC 2022
```
#### 4. Проверим, что данные загружены и типовой запос работает корректно

```
OLobach@instance-15:/tmp/taxi_data$ psql -h 35.222.158.100 -U postgres
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1), server 14.0)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=> \x
postgres=> \dt+

List of relations
-[ RECORD 1 ]-+-----------
Schema        | public
Name          | taxi_trips
Type          | table
Owner         | postgres
Persistence   | permanent
Access method | heap
Size          | 3001 MB
Description   |

postgres=> select payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c from taxi_trips group by payment_type order by 3;
payment_type | tips_percent |    c
--------------+--------------+---------
Prepaid      |            0 |       5
Pcard        |            3 |     869
Prcard       |            1 |    2992
Mobile       |           15 |    3414
Dispute      |            0 |    6588
Unknown      |            5 |   19935
No Charge    |            6 |   77515
Credit Card  |           17 | 2610474
Cash         |            0 | 4516000
(9 rows)

Time: 23554.534 ms (00:23.555)

```