# Stress-Test of Vantiq Product

## Project Overview

To simulate several business scenarios to generate workload, in order to monitor and capture the response behaviors of our product which has been deployed in production-grade cluster

- Monitoring spec
  - product
  - system
- Capacity
- Bottleneck
- Performance tuning

## Environment and Architecture

- 6 VMs as K8S work nodes (-1 as of master)
- 4vCPU, 16G memory, each node
- 3* MongoDB (1 arbitor, 1 primary, 1 secondary), standard disk
- 3* Vantiq-server (1.25.13)
- 3* Keycloak

## Configuration

#### System Env Setting
```
ubuntu@vantiq2-test01:~/stress_test/gatlingTestInfra3/loadTest$ cat ~/.vantiq/profile
 {
// url = 'http://eda-dev.profile_name.com:8080'
url = 'https://eda-dev.profile_name.com'
// user 'test' in system
// vantiq -s docs1 load document /tmp/vantiq-docs
token = 'kxJDu_UDE7pHHGmBsdW6mMk7YOVL5ZneRs4ZqRU9TvA='
}
```

#### Configure Products

Operations > Administer > Organizations > Actions > Configure Products

```
{
    "Pronto": {
        "enabled": true,
        "maxEventCatalogs": 1
    },
    "Modelo": {
        "enabled": true
    }
}
```

#### Code Change
-  Modify Quota: Operations > Administer > Organizations > Actions > Edit Quotas (default: percentage = 20)
```
{
    "rates": {
        "execution": 20010,
        "receiveMessage": 20000
    },
    "credit": {
        "default": {
            "percentage": 100,
            "queueRatio": 10
        }
    }
}
```

#### Grafana dataSources

Create the following dataSources

- mongoDB
<center><img src="../imgs/20190604_grafana_datasource_mongodb.png" width="550px"></center>

- systemDB
  - Name: systemDB
  - URL: http://influxdb-influxdb:8086
  - influxDB Details > Database = system

- vantiqServer
  - Name: vantiqServer
  - URL: http://influxdb-influxdb:8086
  - influxDB Details > Database = vantiq_server

- kubernetes
  - Name: kubernetes
  - URL: http://influxdb-influxdb:8086
  - influxDB Details > Database = kubernetes

- internals
  - Name: internals
  - URL: http://influxdb-influxdb:8086
  - influxDB Details > Database = \_internal

After finishing, the home dashboard looks like

<center><img src="../imgs/20190604_grafana_dashboard.png" width="900px"></center>

<div style="page-break-after: always;"></div>


## Test-Case 1: LimitLiftsSim

#### Description:
- 5K/sec update for status update
- 1K/min upsert for status history storage
- 500 users, 3 event / user
- 1,500 elevators/ second, upsert/ second in collection ```realtimeData``` (value the same as ```StatisticData_Derived```)
- 1,500 insert/ minute in ```realtimeData_his``` (history)

#### Script

Simulation code is located

```
~/stress_test/gatlingTestInfra3/loadTest/src/gatling/resources/namespaces
```

```
cd ~/stress_test/gatlingTestInfra3/loadTest

./gradlew --console=plain gatlingRun-LimitLiftsSim -Pvantiq.system=profile_name \
  -Pgatling.users=500 -Pgatling.duration="10 minutes" \
  -Pvantiq.namespace.create=false -Pvantiq.namespace.save=true
```

#### Test Output
```
================================================================================
---- Global Information --------------------------------------------------------
> request count                                     822825 (OK=822825 KO=0     )
> min response time                                      0 (OK=0      KO=-     )
> max response time                                    713 (OK=713    KO=-     )
> mean response time                                    17 (OK=17     KO=-     )
> std deviation                                         28 (OK=28     KO=-     )
> response time 50th percentile                          6 (OK=6      KO=-     )
> response time 75th percentile                         18 (OK=18     KO=-     )
> response time 95th percentile                         67 (OK=67     KO=-     )
> response time 99th percentile                        132 (OK=132    KO=-     )
> mean requests/sec                                1371.375 (OK=1371.375 KO=-  )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                        822825 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================
```

<div style="page-break-after: always;"></div>

###### Statistic from Grafana
<center><img src="../imgs/20190618_vtq_resource.png"></center>

- Reduced concurrent user from 1000 down to 500
- Network ~450K/s input and output. Busy
- Vantiq Resources reached to ~80%. Quite heavy

<center><img src="../imgs/20190618_vtq_mongo.png"></center>

- MongoDB: 49 connections and up to 4.5G memory usage (busy). Transaction per second:
  ```
  query= 369
  insert= 39
  update= 83
  ```
- realtimeData table in MongoDB:
  ```vantiq:PRIMARY> db.realtimeData_his__myfirstnamespace.count() = 105335```
  (not that high as expected. Some cap possibly limits this)

###### Statistic from Gatling
<center><img src="../imgs/20190618_vtq_gatling.png"></center>

- Total 822,825 requests completed 100% in success, including 500 token accesses, about 1,371 req/s (not bad)
- Response-time: 99th pct = 132 ms (good enough for production. The less the better)

<center><img src="../imgs/20190618_vtq_gatling2.png"></center>

###### Statistic from MongoDB

```
vantiq:PRIMARY> db.realtimeData_his__myfirstnamespace.count()
105335
```

#### Check MongoDB

action | command
-- | --
log into mongo | kubectl -n eda-dev exec -it vantiq-eda-dev-mongodb-primary-0 /bin/bash
log into database on primary | mongo ars02 -u ars -p ars
list database | show dbs
use database | use ars02
list all tables | show collections
query table | db.realtimeData_his__myfirstnamespace.count() = 105335

## Test-Case 2: DBInsertSim

#### Description:
- 50 concurrent users
- insert 1000 rows into MongoDB over and over
- 10 minutes running

#### Script
```
ubuntu@vantiq2-test01:~/stress_test/gatlingTestInfra3/loadTest$ pwd
/home/ubuntu/stress_test/gatlingTestInfra3/loadTest

../gradlew --console=plain gatlingRun-DB_InsertSim -Pvantiq.system=cptheat -Pgatling.users=50 \
  -Pgatling.duration="10 minutes" -Pvantiq.namespace.create=false -Pvantiq.namespace.save=true
```

#### Test Output

```
================================================================================
---- Global Information --------------------------------------------------------
> request count                                       3823 (OK=3823   KO=0     )
> min response time                                      7 (OK=7      KO=-     )
> max response time                                  23298 (OK=23298  KO=-     )
> mean response time                                  7625 (OK=7625   KO=-     )
> std deviation                                       6442 (OK=6442   KO=-     )
> response time 50th percentile                       4403 (OK=4403   KO=-     )
> response time 75th percentile                      13806 (OK=13806  KO=-     )
> response time 95th percentile                      19891 (OK=19891  KO=-     )
> response time 99th percentile                      21387 (OK=21387  KO=-     )
> mean requests/sec                                  6.361 (OK=6.361  KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                            44 (  1%)
> 800 ms < t < 1200 ms                                  50 (  1%)
> t > 1200 ms                                         3729 ( 98%)
> failed                                                 0 (  0%)
================================================================================
```


#### Statistic of Vantiq_Resource

<img src="../imgs/20190619_dbinsert_vantiq_resource.png">

- CPU up to 100% for Vantiq-servers (very busy)
- Req 35 ops (operation/ second)

#### Statistic of MongoDB
<img src="../imgs/20190619_dbinsert_mongo.png">

- Insert __3,150/sec__ and Query __2,150/sec__ (slower than on AWS)
- 174 connections (looks good)
- Memory usage 5.48G
- Network IO close to 3MB/s (busy)

#### Statistic of Vantiq Resource_Usage > API > vail
<img src="../imgs/20190619_dbinsert_resource_uage_api_vail.png">

- insert p99 avg = 50ms (very good)

#### Statistic of Gatling

<img src="../imgs/20190619_dbinsert_gatling_stat.png" width="750px">

<img src="../imgs/20190619_dbinsert_gatling_response_time_distribution.png">

<img src="../imgs/20190619_dbinsert_gatling_response_time_percentile_over_time.png">

- 95% of response completed within 20kms = 20 second, with avg 40 concurrent users!!! (__quite busy__)

<img src="../imgs/20190619_dbinsert_gatling_number_of_req_per_second.png">

- Avg ~= 10 req

<img src="../imgs/20190619_dbinsert_gatling_number_of_response_per_sec.png">

- Avg = 10 response/ second
