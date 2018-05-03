
#### Document Objective
- Build Redis master- slave cluster on distributed VMs

#### Document Reference
https://hub.docker.com/r/bitnami/redis/

## Deploy

#### Topology

1 master + 2 slaves on 3 VMs, managed by Docker Swarm. Actually you can have as many of slaves as you want.

#### For master
```
docker run -d --name redis-master \
  -e REDIS_REPLICATION_MODE=master \
  -e REDIS_PASSWORD=mysecret \
  -p 6379:6379 \
  bitnami/redis:latest
```

#### For slave(s)
```
docker run -d --name redis-slave \
  -e REDIS_REPLICATION_MODE=slave \
  -e REDIS_MASTER_HOST=10.0.1.6 \
  -e REDIS_MASTER_PORT_NUMBER=6379 \
  -e REDIS_MASTER_PASSWORD=mysecret \
  -e REDIS_PASSWORD=mysecret
  bitnami/redis:latest
```

> If the Redis master goes down you can reconfigure a slave to become a master using:
> ```docker exec redis-slave redis-cli -a $REDIS_PASSWORD SLAVEOF NO ONE```
> And other slave(s) has to be aware of the change


```
version: '3.1'

services:
  redis-master:
    image: bitnami/redis:latest
    hostname: redis-master
    environment:
      - REDIS_REPLICATION_MODE=master
      - REDIS_PASSWORD=mysecret
    volumes:
      - /data/redis/redis.conf:/bitnami/redis/conf/redis.conf
#      - /data/redis/data:/bitnami/redis/data
    ports:
      - 6379:6379
    networks:
      - redis
    deploy:
        placement:
            constraints: [node.labels.host==04]
        replicas: 1

  redis-slave1:
    image: bitnami/redis:latest
    hostname: redis-slave1
    networks:
      - redis
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=10.0.1.6
      - REDIS_MASTER_PORT_NUMBER=6379
      - REDIS_MASTER_PASSWORD=mysecret
      - REDIS_PASSWORD=mysecret
    volumes:
      - /data/redis/redis.conf:/bitnami/redis/conf/redis.conf
#      - /data/redis/data:/bitnami/redis/data
    deploy:
        placement:
            constraints: [node.labels.host==03]
        replicas: 1

  redis-slave2:
    image: bitnami/redis:latest
    hostname: redis-slave2
    networks:
      - redis
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=10.0.1.6
      - REDIS_MASTER_PORT_NUMBER=6379
      - REDIS_MASTER_PASSWORD=mysecret
      - REDIS_PASSWORD=mysecret
    volumes:
      - /data/redis/redis.conf:/bitnami/redis/conf/redis.conf
#      - /data/redis/data:/bitnami/redis/data
    deploy:
        placement:
            constraints: [node.labels.host==02]
        replicas: 1

networks:
  redis:
#   external:
#     name: "host"
```

#### Modify kernel to avoid potential memory leak issues

```
30:S 02 May 02:40:16.222 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
30:S 02 May 02:40:16.222 # Server initialized
30:S 02 May 02:40:16.222 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
30:S 02 May 02:40:16.222 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
30:S 02 May 02:40:16.222 * Ready to accept connections
```
