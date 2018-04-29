
#### Document Objective
- Build Redis cluster on distributed VMs

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
    deploy:
        placement:
            constraints: [node.labels.host==02]
        replicas: 1

networks:
  redis:
#   external:
#     name: "host"
```
