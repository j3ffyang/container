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
-  Modify Quota: Operations > Administer > Organizations > Actions > Edit Quotas
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

## Stress-Test, scenario: LimitLiftsSim

Description:
- 5K update per second for status update
- 1K upsert per minute for status history storage

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
> mean requests/sec                                1371.375 (OK=1371.375 KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                        822825 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================
```

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
log into database | mongo ars02 -u ars -p ars
list database | show dbs
use database | use ars02
list all tables | show collections
query table | vantiq:PRIMARY> db.realtimeData_his__myfirstnamespace.count() = 105335
