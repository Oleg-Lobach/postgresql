# Домашнее задание
## Работа c PostgreSQL в Kubernetes
**Развернуть CitusDB в GKE, залить 72 Гб (все данные) чикагского такси. Шардировать. Оценить производительность. Описать проблемы, с которыми столкнулись**
## 1. Выполним подготовительные работы по разертыванию и настройке кластера
### 1.1 Создадим кластер GKE из трех нод

```bash

gcloud beta container --project "pgdba-340419" clusters create "citus" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.21.6-gke.1503" --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "120" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --max-pods-per-node "110" --preemptible --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/pgdba-340419/global/networks/default" --subnetwork "projects/pgdba-340419/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "us-central1-c"
...

Created [https://container.googleapis.com/v1beta1/projects/pgdba-340419/zones/us-central1-c/clusters/citus].

To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-central1-c/citus?project=pgdba-340419

kubeconfig entry generated for citus.

NAME   LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS

citus  us-central1-c  1.21.6-gke.1503  34.66.195.128  e2-medium     1.21.6-gke.1503  3          RUNNING

```

### 1.2 Используем версию citus_10.1pg12. Изменим лимиты

#### 1.2.1 Для мастера установим 100GB PVC

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: citus-master-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: **100Gi**
```            

### 1.2.2 Для workers установим 100GB PVC и 2 реплики
```
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: **100Gi**
--
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-worker
spec:
  selector:
    matchLabels:
      app: citus-workers
  serviceName: citus-workers
  replicas: **2**
```

### 1.3 Применим манифесты и проверим статус
```
c:\Projects\PG29\pgGKE\citus_10.1pg12>kubectl get all

NAME                                READY   STATUS    RESTARTS   AGE
pod/citus-master-78ff549b8f-z9zz5   1/1     Running   0          4m32s

pod/citus-worker-0                  1/1     Running   0          2m52s

pod/citus-worker-1                  1/1     Running   0          2m27s

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE

service/citus-master    NodePort    10.80.14.15   <none>        5432:32425/TCP   4m32s

service/citus-workers   ClusterIP   None          <none>        5432/TCP         2m53s

service/kubernetes      ClusterIP   10.80.0.1     <none>        443/TCP          8m18s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE

deployment.apps/citus-master   1/1     1            1           4m33s

NAME                                      DESIRED   CURRENT   READY   AGE

replicaset.apps/citus-master-78ff549b8f   1         1         1       4m33s

NAME                            READY   AGE

statefulset.apps/citus-worker   2/2     2m53s
```

**Вывод:** Кластер успешно развернут

## 2. Загрузим все данные по такси Чикаго из публичного бакета
### 2.1. Поключимся к мастеру и проверим размер свободного места на диске

```bash
c:\Projects\PG29\pgGKE\citus_10.1pg12>kubectl exec -it pod/citus-master-78ff549b8f-z9zz5  -- bash

root@citus-master-78ff549b8f-z9zz5:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         114G  3.1G  111G   3% /
tmpfs            64M     0   64M   0% /dev
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
shm              64M  8.0K   64M   1% /dev/shm
/dev/sda1       114G  3.1G  111G   3% /etc/hosts
/dev/sdb         98G   40M   98G   1% /var/lib/postgresql/data
tmpfs           2.0G   12K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           2.0G     0  2.0G   0% /proc/acpi
tmpfs           2.0G     0  2.0G   0% /proc/scsi
tmpfs           2.0G     0  2.0G   0% /sys/firmware
root@citus-master-78ff549b8f-z9zz5:/#
```
### 2.2 Загрузим все данные по такси Чикаго
```bash
root@citus-master-78ff549b8f-z9zz5:/tmp/taxi_data# apt-get update
root@citus-master-78ff549b8f-z9zz5:/tmp/taxi_data# apt-get install wget
root@citus-master-78ff549b8f-z9zz5: wget https://storage.googleapis.com/msk_dba_bucket/taxi_000000000{000..298}.csv
root@citus-master-78ff549b8f-z9zz5:/tmp/taxi_data#  wget https://storage.googleapis.com/msk_dba_bucket/taxi_000000000{000..298}.csv

taxi_000000000298.csv         100%[=================================================>] 254.91M  71.7MB/s    in 3.6s
2022-03-11 19:55:00 (71.7 MB/s) - ‘taxi_000000000298.csv’ saved [267294112/267294112]
FINISHED --2022-03-11 19:55:00--
Total wall clock time: 15m 52s
Downloaded: 299 files, 74G in 15m 20s (82.8 MB/s)

```
## 3. Загрузим все данные по такси Чикаго в БД PostgreSQL
### 3.1 Создадим БД taxi на мастере и воркерах
```sql
postgres=# create database taxi;
NOTICE:  Citus partially supports CREATE DATABASE for distributed databases
DETAIL:  Citus does not propagate CREATE DATABASE command to workers
HINT:  You can manually create a database and its extensions on workers.
CREATE DATABASE
```
### 3.2 Подключим расширение citus на мастере и воркерах для БД taxi
```sql
postgres=# \c taxi;
taxi=# CREATE EXTENSION citus;
CREATE EXTENSION
```
### 3.3 Подключим воркеры
```
taxi=# SELECT * FROM master_get_active_worker_nodes();
node_name | node_port
-----------+-----------
(0 rows)
```
```
taxi=# SELECT * FROM master_add_node('citus-worker-0.citus-workers', 5432);
nodeid | groupid |           nodename           | nodeport | noderack | hasmetadata | isactive | noderole | nodecluster
--------+---------+------------------------------+----------+----------+-------------+----------+----------+------------
-

      1 |       1 | citus-worker-0.citus-workers |     5432 | default  | f           | t        | primary  | default

(1 row)

taxi=# SELECT * FROM master_add_node('citus-worker-1.citus-workers', 5432);
nodeid | groupid |           nodename           | nodeport | noderack | hasmetadata | isactive | noderole | nodecluster
--------+---------+------------------------------+----------+----------+-------------+----------+----------+------------
-
      2 |       2 | citus-worker-1.citus-workers |     5432 | default  | f           | t        | primary  | default
(1 row)
taxi=# SELECT * FROM master_get_active_worker_nodes();

          node_name           | node_port

------------------------------+-----------

citus-worker-0.citus-workers |      5432

citus-worker-1.citus-workers |      5432

(2 rows)
```
### 3.4 Создадим распределенную таблицу для данных
```
taxi=# create table taxi_trips (
taxi(#  unique_key text,
taxi(#  taxi_id text,
taxi(#  trip_start_timestamp TIMESTAMP,
taxi(#  trip_end_timestamp TIMESTAMP,
taxi(#  trip_seconds bigint,
taxi(#  trip_miles numeric,
taxi(#  pickup_census_tract bigint,
taxi(#  dropoff_census_tract bigint,
taxi(#  pickup_community_area bigint,
taxi(#  dropoff_community_area bigint,
taxi(#  fare numeric,
taxi(#  tips numeric,
taxi(#  tolls numeric,
taxi(#  extras numeric,
taxi(#  trip_total numeric,
taxi(#  payment_type text,
taxi(#  company text,
taxi(#  pickup_latitude numeric,
taxi(#  pickup_longitude numeric,
taxi(#  pickup_location text,
taxi(#  dropoff_latitude numeric,
taxi(#  dropoff_longitude numeric,
taxi(#  dropoff_location text
taxi(#  );
CREATE TABLE
```

 ```
taxi=# select create_distributed_table('taxi_trips', 'unique_key');
create_distributed_table
--------------------------
(1 row)
```
### 3.5 Выполним загрузку данных в БД
```bash
root@citus-master-78ff549b8f-z9zz5:/tmp/taxi_data# date && for f in *.csv; do psql -U postgres -c "\COPY taxitrips FROM
PROGRAM 'cat $f' CSV HEADER"; done && date
Fri Mar 11 20:57:58 UTC 2022
COPY 656957
...
```
## 4. Оценим скорость выполнения запроса для двух воркеров
```sql
taxi=# select payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c from taxi_trips group by payment_type order by 3;
payment_type | tips_percent |     c
--------------+--------------+-----------
Way2ride     |           14 |       142
Prepaid      |            0 |      1812
Split        |           17 |      3442
Pcard        |            2 |     36874
Dispute      |            0 |     83492
Mobile       |           16 |    725417
No Charge    |            4 |    817856
Unknown      |            1 |    963245
Prcard       |            1 |   1000985
Credit Card  |           17 |  80510921
Cash         |            0 | 114956910
(11 rows)
Time: 3811746.677 ms (01:03:31.747)
```
## 5. Изменим конфигурация кластера. Добавим 3 воркера и повторим выборку.
### 5.1 Изменим конфигурацию - увеличим количество реплик до 5
```bash
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-worker
spec:
  selector:
    matchLabels:
      app: citus-workers
  serviceName: citus-workers
  replicas: **5**
```
```bash
c:\Projects\PG29\pgGKE\citus_10.1pg12>kubectl apply -f workers.yaml

service/citus-workers unchanged

statefulset.apps/citus-worker configured

c:\Projects\PG29\pgGKE\citus_10.1pg12>kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/citus-master-78ff549b8f-z9zz5   1/1     Running   0          13h
pod/citus-worker-0                  1/1     Running   0          13h
pod/citus-worker-1                  1/1     Running   0          13h
pod/citus-worker-2                  1/1     Running   0          77s
pod/citus-worker-3                  1/1     Running   0          54s
pod/citus-worker-4                  1/1     Running   0          40s

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/citus-master    NodePort    10.80.14.15   <none>        5432:32425/TCP   13h
service/citus-workers   ClusterIP   None          <none>        5432/TCP         13h
service/kubernetes      ClusterIP   10.80.0.1     <none>        443/TCP          13h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/citus-master   1/1     1            1           13h
NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/citus-master-78ff549b8f   1         1         1       13h
NAME                            READY   AGE
statefulset.apps/citus-worker   5/5     13h

C:\Users\olobach\AppData\Local\Google\Cloud SDK>kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP          NODE                                   NOMINATED NODE   READINESS GATES
citus-master-78ff549b8f-z9zz5   1/1     Running   0          14h   10.76.1.3   gke-citus-default-pool-74c38644-vnr1   <none>           <none>
citus-worker-0                  1/1     Running   0          14h   10.76.1.4   gke-citus-default-pool-74c38644-vnr1   <none>           <none>
citus-worker-1                  1/1     Running   0          14h   10.76.1.5   gke-citus-default-pool-74c38644-vnr1   <none>           <none>
citus-worker-2                  1/1     Running   0          92m   10.76.2.4   gke-citus-default-pool-74c38644-b24t   <none>           <none>
citus-worker-3                  1/1     Running   0          92m   10.76.2.5   gke-citus-default-pool-74c38644-b24t   <none>           <none>
citus-worker-4                  1/1     Running   0          91m   10.76.1.6   gke-citus-default-pool-74c38644-vnr1   <none>           <none>

C:\Users\olobach\AppData\Local\Google\Cloud SDK>kubectl get pv -o wide

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS   REASON   AGE   VOLUMEMODE
pvc-1b792710-a256-4758-ad36-f9d20828c54a   100Gi      RWO            Delete           Bound    default/storage-citus-worker-2   standard                93m   Filesystem
pvc-3b4b20ff-a033-429a-a8c3-eecff9f1f83e   100Gi      RWO            Delete           Bound    default/citus-master-pvc         standard                15h   Filesystem
pvc-43b00610-540f-4c9a-a915-3480c2ec1704   100Gi      RWO            Delete           Bound    default/storage-citus-worker-4   standard                93m   Filesystem
pvc-6740db94-5044-4719-8f4f-17256d9c0059   100Gi      RWO            Delete           Bound    default/storage-citus-worker-3   standard                93m   Filesystem

pvc-972fbf9a-6abb-4066-aa17-a154abc749a4   100Gi      RWO            Delete           Bound    default/storage-citus-worker-0   standard                14h   Filesystem
pvc-e3eb264b-82ba-4544-96d2-b6f692c149bb   100Gi      RWO            Delete           Bound    default/storage-citus-worker-1   standard                14h   Filesystem
```

### 5.2 Создадим БД taxi на новых воркерах и добавим расширение citus. Проверим статус
### 5.2.1 Пробуем выполнить ребаланс
```
taxi=# SELECT * FROM master_get_active_worker_nodes();
          node_name           | node_port
------------------------------+-----------
citus-worker-2.citus-workers |      5432
citus-worker-3.citus-workers |      5432
citus-worker-0.citus-workers |      5432
citus-worker-4.citus-workers |      5432
citus-worker-1.citus-workers |      5432
(5 rows)
```
```
taxi=# SELECT rebalance_table_shards('taxitrips');
ERROR:  function rebalance_table_shards(unknown) does not exist
LINE 1: SELECT rebalance_table_shards('taxitrips');
               ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
taxi=#

```
**Вывод:** На текущей версии мастера ребаланс не поддерживается.
### 5.2.2 Удалим таблицу и очистим теневые данные
```sql
taxi=# SELECT undistribute_table('taxit_rips');
ERROR:  function undistribute_table(unknown) does not exist
LINE 1: SELECT undistribute_table('taxitrips');^

taxi=#SELECT truncate_local_data_after_distributing_table($$public.taxi_trips$$)
ERROR:  function truncate_local_data_after_distributing_table(unknown) does not exist
LINE 1: SELECT truncate_local_data_after_distributing_table($$public...
.
```
**Вывод** На текущей версии мастера функции не поддерживается
```
taxi=# drop table taxi_trips;
DROP TABLE
```
### 5.2.3 Проверим доступность места на диске
```
root@citus-master-78ff549b8f-z9zz5:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         114G   78G   37G  68% /
tmpfs            64M     0   64M   0% /dev
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
shm              64M  8.0K   64M   1% /dev/shm
/dev/sda1       114G   78G   37G  68% /etc/hosts
/dev/sdb         98G   65M   **98G**   1% /var/lib/postgresql/data
tmpfs           2.0G   12K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           2.0G     0  2.0G   0% /proc/acpi
tmpfs           2.0G     0  2.0G   0% /proc/scsi
tmpfs           2.0G     0  2.0G   0% /sys/firmware
```

## 5.3 Создадим распределенную таблицу в БД postgres
```
postgres=# create table taxi_trips (
postgres(#  unique_key text,
postgres(#  taxi_id text,
postgres(#  trip_start_timestamp TIMESTAMP,
postgres(#  trip_end_timestamp TIMESTAMP,
postgres(#  trip_seconds bigint,
postgres(#  trip_miles numeric,
postgres(#  pickup_census_tract bigint,
postgres(#  dropoff_census_tract bigint,
postgres(#  pickup_community_area bigint,
postgres(#  dropoff_community_area bigint,
postgres(#  fare numeric,
postgres(#  tips numeric,
postgres(#  tolls numeric,
postgres(#  extras numeric,
postgres(#  trip_total numeric,
postgres(#  payment_type text,
postgres(#  company text,
postgres(#  pickup_latitude numeric,
postgres(#  pickup_longitude numeric,
postgres(#  pickup_location text,
postgres(#  dropoff_latitude numeric,
postgres(#  dropoff_longitude numeric,
postgres(#  dropoff_location text
postgres(#  );
CREATE TABLE

postgres=# select create_distributed_table('taxi_trips', 'unique_key');
create_distributed_table
--------------------------
(1 row)
```
## 5.4 Выполним загрузку данных в БД
```
root@citus-master-78ff549b8f-z9zz5:/tmp/taxi_data# date && for f in *.csv; do psql -U postgres -c "\COPY taxi_trips FROM
PROGRAM 'cat $f' CSV HEADER"; done && date
Sat Mar 12 09:22:47 UTC 2022
COPY 656957
...
COPY 656943
Sat Mar 12 11:59:12 UTC 2022
```

## 5.5 Проверим на одном из воркеров наличие данных
```
C:\Users\olobach\AppData\Local\Google\Cloud SDK>kubectl exec -it pod/citus-worker-4  -- bash
root@citus-worker-4:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         114G   79G   36G  69% /
tmpfs            64M     0   64M   0% /dev
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
shm              64M  8.3M   56M  13% /dev/shm
/dev/sda1       114G   79G   36G  69% /etc/hosts
/dev/sde         98G   **16G**   83G  17% /var/lib/postgresql/data
tmpfs           2.0G   12K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           2.0G     0  2.0G   0% /proc/acpi
tmpfs           2.0G     0  2.0G   0% /proc/scsi
tmpfs           2.0G     0  2.0G   0% /sys/firmware

root@citus-worker-4:/# psql -U postgres
psql (12.8 (Debian 12.8-1.pgdg100+1))
Type "help" for help.

postgres=# \dt+
                           List of relations
Schema |       Name        | Type  |  Owner   |  Size   | Description
--------+-------------------+-------+----------+---------+-------------
public | taxi_trips_102044 | table | postgres | 2548 MB |
public | taxi_trips_102049 | table | postgres | 2546 MB |
public | taxi_trips_102054 | table | postgres | 2547 MB |
public | taxi_trips_102059 | table | postgres | 2548 MB |
public | taxi_trips_102064 | table | postgres | 2546 MB |
public | taxi_trips_102069 | table | postgres | 2546 MB |
(6 rows)
```
## 5.6 Оценим скорость выполнения запроса для пяти воркеров
```sql
postgres=# \timing
Timing is on.
postgres=# select payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c from taxi_trips group by payment_type order by 3;
payment_type | tips_percent |     c
--------------+--------------+-----------
Way2ride     |           14 |       142
Prepaid      |            0 |      1812
Split        |           17 |      3442
Pcard        |            2 |     36874
Dispute      |            0 |     83492
Mobile       |           16 |    725417
No Charge    |            4 |    817856
Unknown      |            1 |    963245
Prcard       |            1 |   1000985
Credit Card  |           17 |  80510921
Cash         |            0 | 114956910
(11 rows)
Time: 1176506.893 ms (19:36.507)
```
**Вывод:** Увеличение количества реплик в 2,5 раза позволило повысить производительность примерно в 3811746 / 1176506 = 3,24 раза.
**Горизонтальное масштабирование работает.**