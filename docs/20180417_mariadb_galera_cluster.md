
#### Document Objective
- Official MariaDB image
- Built- in Galera plugin

#### Reference

[Galera cluster, MariaDB, CoreOS and Docker](https://withblue.ink/2016/03/09/galera-cluster-mariadb-coreos-and-docker-part-1.html)

## Deployment

#### Assumption and Pre- requisite
- Must read the reference doc first!
- MariaDB persistent volume mount = ```/data/mariadb/mysql``` in this doc
- A configuration file of Galera ```mysql_server.cnf``` is copied to ```/data/mariadb/etc/mysql.conf.d```
- ```mariadb-node-0,mariadb-node-1,mariadb-node-2``` (they're nodes to host MariaDB cluster) in ```mysql.conf.d/mysql_server.cnf``` need to reflect your actual domain resolving

Configuration File

- ```mysql-server.cnf```

```
#
# Galera Cluster: mandatory settings
#

[server]
bind-address=0.0.0.0
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
query_cache_size=0
query_cache_type=0

[galera]
wsrep_on=ON
wsrep_provider="/usr/lib/galera/libgalera_smm.so"
wsrep_cluster_address="gcomm://mariadb-node-0,mariadb-node-1,mariadb-node-2"
wsrep-sst-method=rsync

#
# Optional setting
#

# Tune this value for your system, roughly 2x cores; see https://mariadb.com/kb/en/mariadb/galera-cluster-system-variables/#wsrep_slave_threads
# wsrep_slave_threads=1

# innodb_flush_log_at_trx_commit=0
```

#### On 1st node

```
docker run \
  --name mariadb-0 -d \
  -v /data/mariadb/etc/mysql.conf.d:/etc/mysql/conf.d \
  -v /data/mariadb/maria_data/:/var/lib/mysql \
  -e MYSQL_INITDB_SKIP_TZINFO=yes \
  -e MYSQL_ROOT_PASSWORD=mysecret \
  -p 13306:3306 \
  -p 4567-4568:4567-4568 \
  -p 4444:4444 \
  mariadb:latest \
  --wsrep-new-cluster \
  --wsrep_node_address=$(ip -4 addr ls eth0 | awk '/inet / {print $2}' | cut -d"/" -f1)
```

> Note: credit to referenced doc
Please observe the ```--wsrep-new-cluster```, which is used to bootstrap the Galera Cluster. This flag has to be used once and only once. If you need to restart this container in the future, after the cluster has been initialized, you must __omit__ the ```--wsrep-new-cluster``` flag.

#### On 2nd node and others

> Note: In ```--name mariadb-X```, ```X``` reflect 2nd, 3rd nodes in cluster and so on

```
docker run \
  --name mariadb-1 -d \
  -v /data/mariadb/etc/mysql.conf.d:/etc/mysql/conf.d \
  -v /data/mariadb/maria_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=mysecret \
  -p 13306:3306 \
  -p 4567:4567/udp \
  -p 4567-4568:4567-4568 \
  -p 4444:4444 mariadb:latest --wsrep_node_address=$(ip -4 addr ls eth0 | awk '/inet / {print $2}' | cut -d"/" -f1)
```

#### Verify the cluster

On each of node, log into ```mariadb``` (mysql actually)

```
MariaDB [(none)]> show status like 'wsrep_%';
+------------------------------+--------------------------------------------+
| Variable_name                | Value                                      |
+------------------------------+--------------------------------------------+
| wsrep_apply_oooe             | 0.000000                                   |
| wsrep_apply_oool             | 0.000000                                   |
| wsrep_apply_window           | 1.000000                                   |
| wsrep_causal_reads           | 0                                          |
| wsrep_cert_deps_distance     | 1.000000                                   |
| wsrep_cert_index_size        | 3753                                       |
| wsrep_cert_interval          | 0.000000                                   |
| wsrep_cluster_conf_id        | 3                                          |
| wsrep_cluster_size           | 3                                          |
| wsrep_cluster_state_uuid     | 4c90025e-41e6-11e8-aee1-5b87fd582640       |
| wsrep_cluster_status         | Primary                                    |
...
| wsrep_gcomm_uuid             | aad3574b-41f7-11e8-a743-1be43ad13318       |
| wsrep_incoming_addresses     | 10.0.1.6:3306,10.0.1.2:3306,10.0.1.10:3306 |
```
