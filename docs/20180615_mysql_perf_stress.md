
## Doc Objective
- Stress test by ```sysbench``` against single MySQL node and Galera cluster

## Testcase on Fedora 28 Linux

- Install ```sysbench``` and ```mysql``` client

- Parameters of MySQL

```
innodb_buffer_pool_size=6000M
innodb_log_file_size=256M
innodb_log_buffer_size=4M
innodb_flush_log_at_trx_commit=0
innodb_thread_concurrency=8
innodb_flush_method=O_DIRECT
innodb_file_per_table

innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1

bind-address=0.0.0.0
binlog_format=row
default_storage_engine=InnoDB

# query_cache_size=1048576
query_cache_size=2097152
# query_cache_type=0
sync_binlog=0
```

- Create a database named ```test```

- Prepare database

```
sysbench --test=/usr/share/sysbench/tests/include/oltp_legacy/oltp.lua \
  --oltp-table-size=1000000 --db-driver=mysql --mysql-host=127.0.0.1 \
  --mysql-db=test --mysql-user=root --mysql-port=3306 --mysql-password=mysecret \
  prepare
```

- Run load test

```
sysbench --test=/usr/share/sysbench/tests/include/oltp_legacy/oltp.lua \
  --oltp-table-size=1000000 --db-driver=mysql --mysql-host=127.0.0.1 \
  --mysql-db=test --mysql-user=root --mysql-port=3306 --mysql-password=mysecret \
  --max-time=60 --oltp-read-only=on --num-threads=8 run
```

- Output of __read_only__

```
sysbench 1.0.12 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            1617126
        write:                           0
        other:                           231018
        total:                           1848144
    transactions:                        115509 (1924.94 per sec.)
    queries:                             1848144 (30799.09 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0044s
    total number of events:              115509

Latency (ms):
         min:                                  2.08
         avg:                                  4.15
         max:                                 40.62
         95th percentile:                      5.67
         sum:                             479757.09

Threads fairness:
    events (avg/stddev):           14438.6250/19.27
    execution time (avg/stddev):   59.9696/0.00
```

- Output of __read_and_write__

```
[jeff@f28 conf.d]$ sysbench --test=/usr/share/sysbench/tests/include/oltp_legacy/oltp.lua --oltp-table-size=1000000 --db-driver=mysql --mysql-host=127.0.0.1 --mysql-db=test --mysql-user=root --mysql-port=3306 --mysql-password=mysecret --max-time=60 --oltp-read-only=off --num-threads=8 run
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
WARNING: --max-time is deprecated, use --time instead
sysbench 1.0.12 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            1141966
        write:                           326252
        other:                           163127
        total:                           1631345
    transactions:                        81558  (1359.16 per sec.)
    queries:                             1631345 (27186.24 per sec.)
    ignored errors:                      11     (0.18 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0043s
    total number of events:              81558

Latency (ms):
         min:                                  1.64
         avg:                                  5.88
         max:                                 79.56
         95th percentile:                      9.73
         sum:                             479828.50

Threads fairness:
    events (avg/stddev):           10194.7500/30.66
    execution time (avg/stddev):   59.9786/0.00
```
