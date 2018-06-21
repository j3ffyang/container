## Document Objective
- MongoDB in docker
- Clustering across 3 VMs

#### Reference
- [https://docs.mongodb.com/manual/tutorial/force-member-to-be-primary/](https://docs.mongodb.com/manual/tutorial/force-member-to-be-primary/)
- [https://stackoverflow.com/questions/16331276/how-to-convert-secondary-node-to-primary-if-maximum-of-node-down-in-a-replica-se](https://stackoverflow.com/questions/16331276/how-to-convert-secondary-node-to-primary-if-maximum-of-node-down-in-a-replica-se)

- [http://qqbuby.github.io/docker/2017/09/08/how-to-deploy-mongodb-with-docker.html](http://qqbuby.github.io/docker/2017/09/08/how-to-deploy-mongodb-with-docker.html)

- [https://medium.com/lucjuggery/mongodb-replica-set-on-swarm-mode-45d66bc9245](https://medium.com/lucjuggery/mongodb-replica-set-on-swarm-mode-45d66bc9245)

#### Pre- requisite
- Attention to ```constraints``` in yaml, which has been to be set by ```label-add```.

## Steps

<center><img src=../imgs/20171018_mongo_cluster.png width="500px"></center>

#### Launch Mongo containers through compose yaml

```shell
docker stack deploy -c mongo.yaml mongo
```

```yaml
version: "3.3"

# networks:
#   mongonet2:

services:
  mongo1:
    image: mongo:3.4
    networks:
      - outside
#    ports:
#      - '27017:27017'
    hostname: "mongo1"
    volumes:
      - mongodata:/data/db
      - mongoconfigdb:/data/configdb
      - mongokeyfile:/opt/keyfile
    environment:
      - keyFile:/opt/keyfile/mongodb-keyfile
    command: ["mongod", "--replSet", "rs0"]
    deploy:
      placement:
        constraints: [node.labels.host==6]
      replicas: 1

  mongo2:
    image: mongo:3.4
    networks:
      - outside
#    ports:
#      - '27017:27017'
    hostname: "mongo2"
    volumes:
      - mongodata:/data/db
      - mongoconfigdb:/data/configdb
      - mongokeyfile:/opt/keyfile
    environment:
      - keyFile:/opt/keyfile/mongodb-keyfile
    command: ["mongod", "--replSet", "rs0"]
    deploy:
      placement:
        constraints: [node.labels.host==5]
      replicas: 1

  mongo3:
    image: mongo:3.4
    networks:
      - outside
#    ports:
#      - '27017:27017'
    hostname: "mongo3"
    volumes:
      - mongodata:/data/db
      - mongoconfigdb:/data/configdb
      - mongokeyfile:/opt/keyfile
    environment:
      - keyFile:/opt/keyfile/mongodb-keyfile
    command: ["mongod", "--replSet", "rs0"]
    deploy:
      placement:
        constraints: [node.labels.host==4]
      replicas: 1

networks:
  outside:
    external:
      name: "host"

volumes:
  mongodata:
  mongoconfigdb:
  mongokeyfile:
```

> Note: since the network in the stack is using ```outside``` network, which is on container's host respectively, Mongo replicaSet ```rs0``` requires entries in ```/etc/hosts``` for naming resolve. Therefore ```/etc/hosts``` needs modified on __all__ container hosts, such as

```
ubuntu@host6:/data/yaml$ cat /etc/hosts
127.0.0.1  localhost  localhost.localdomain  VM-0-2-ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.0.0.2	host0	gfs00	firstbox
10.0.1.2	host1	ldap
10.0.1.184	host2	gfs02
10.0.1.198	host3	gfs03
10.0.1.119	host4	gfs04	fabric	mongo3
10.0.1.31	host5	mongo2
10.0.1.43	host6	mongo1
```

#### Create an admin user.

- Log into mongo shell on ```primary``` container
```shell
docker exec -it $(docker ps -qf name=mongo) mongo
```

- Initiate replicaSet and create primary

```
> rs.initiate()
```

- Switch to the admin user

```
> use admin
switched to db admin
```

- Create a new site admin user

```
> db.createUser( {
     user: "siteUserAdmin",
     pwd: "password",
     roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
   });
```

You should see a successful message
```
Successfully added user: {
"user" : "siteUserAdmin",
"roles" : [
         {
              "role" : "userAdminAnyDatabase",
              "db" : "admin"
         }
      ]
}
```

- Create a root user

```
> db.createUser( {
     user: "siteRootAdmin",
     pwd: "password",
     roles: [ { role: "root", db: "admin" } ]
   });
```

- On ```primary```, connect to the replica set and configure it.

```shell
docker exec -it $(docker ps -qf name=mongo) mongo
```

- Switch to the admin user

```
> use admin
```

- Authenticate the password we set earlier

```
> db.auth("siteRootAdmin", "password");
1
```

- Initiate the replica set

```
> rs.initiate()
```

- Verify
```
> rs0:PRIMARY> rs.conf()
```

- Add the other 2 nodes into the replica set, on ```primary```
```
rs0:PRIMARY> rs.add("secondary1")
rs0:PRIMARY> rs.add("secondary2")
```

- Validate by

```
rs0:PRIMARY> rs.status()
```

- View logs

```shell
docker logs -ft $(docker ps -qf name=mongo)
```

#### Usage in Nodejs

- Create user for database __gps__

```shell
rs0:PRIMARY> use gps
switched to db gps
> db.createUser( {
   user: "gps",
   pwd: "password",
   roles: [ { role: "readWrite", db: "gps" } ]
 });
```
- add config into Nodejs configuration

```yaml
mongo:
  url: 'mongodb://fabric/gps'
  options:
    user: 'gps'
    pass: 'password'
    server:
      poolSize: 5
    db:
      native_parser: true
```

#### Force secondary node to primary

- Log into container and Mongo shell on the secondary node which to become primary

```shell
docker exec -it $(docker ps -qf name=mongo) mongo
```

- Get admin access

```
> use admin
> db.auth("siteRootAdmin", "password")
> rs.status()
```

where ```siteRootAdmin``` was created previously

- Update mongo member

```
> cfg = rs.conf()
> cfg.members = [cfg.members[0]]
> rs.reconfig(cfg, {force : true})
```

where ```cfg.members[0]``` is the secondary ```node_id```

- You might need to re- add other nodes into cluster

```
rs0:PRIMARY> rs.add("secondary1")
rs0:PRIMARY> rs.add("secondary2")
```

#### Operation Hints

- Connect and Use database

```
rs0:PRIMARY> show databases;
admin       0.000GB
dl_loc      0.027GB
dl_loc_env  0.000GB
local       0.105GB
rs0:PRIMARY> use dl_loc
switched to db dl_loc

rs0:PRIMARY> show collections
loc_current
loc_record

rs0:PRIMARY> db.loc_record.find({})
```
