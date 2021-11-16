
# MongoDB Operation Practice

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Backup on Kubernetes](#backup-on-kubernetes)
    - [Kubernetes Cronjob](#kubernetes-cronjob)
    - [Cronjob to delete old MongoDB dump files](#cronjob-to-delete-old-mongodb-dump-files)
- [Export and Import `collections` from MongoDB](#export-and-import-collections-from-mongodb)
- [Force MongoDB member to become primary](#force-mongodb-member-to-become-primary)
    - [MongoDB Backup and Restore with Docker](#mongodb-backup-and-restore-with-docker)

<!-- /code_chunk_output -->


## Backup on Kubernetes

#### Kubernetes Cronjob

- Attach a raw disk to one of worker nodes in cluster. In our scenario, the disk is attached to `node7`, which is `metrics-collector`

```sh
ubuntu@node7:~$ hostname
node7
ubuntu@node7:~$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  40G  0 disk
└─vda1 252:1    0  40G  0 part /
vdb    252:16   0  40G  0 disk
└─vdb1 252:17   0  40G  0 part /mnt/disks-by-id/disk0
```

- Check `persistentVolume` exists

> Notice: the output is after `persistentVolumeClaim` being bound.

```sh
ubuntu@node1:~$ k get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                             STORAGECLASS    REASON   AGE
...
local-pv-80492a6c   39Gi       RWO            Retain           Bound       sampleNameSpace/mongodump                     local-storage            4d23h
```

- Create `persistentVolumeClaim` and bind to `persistentVolume`

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodump
  namespace: sampleNameSpace
spec:
  volumeMode: Filesystem
  volumeName: local-pv-80492a6c
  storageClassName: local-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 38Gi
```

- Apply `cronjob` yaml

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mongodump-backup
  namespace: sampleNameSpace
spec:
  schedule: '@daily'
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          securityContext:
            fsGroup: 1001   # grant permission to create file(s) in container
          containers:
            - name: mongodump-backup
              image: bitnami/mongodb
              imagePullPolicy: "IfNotPresent"
              command : ["/bin/sh", "-c"]
              args: ["mongodump -u $mongodb_user -p $mongodb_passwd --host=sample-sub-mongodb-client.sampleNameSpace.svc.cluster.local --port=27017 --authenticationDatabase=admin -o /tmp/mongodump/$(date +\"%Y_%m_%d_%H:%M\")"]
              env:
              - name: mongodb_user
                valueFrom:
                  secretKeyRef:
                    name: mongodb
                    key: user
              - name: mongodb_passwd
                valueFrom:
                  secretKeyRef:
                    name: mongodb
                    key: password
              volumeMounts:
              - mountPath: "/tmp/mongodump/"
                name: mongodump
          restartPolicy: Never
          volumes:
          - name: mongodump
            persistentVolumeClaim:
              claimName: mongodump
```

- Define credential in YAML
`env`:`valueFrom`:`secretKeyRef` (with `yq`)

```yml
k -n sampleNameSpace get secrets mongodb -oyaml | yq eval 'del(.metadata.managedFields, .metadata.creationTimestamp, .metadata.resourceVersion, .metadata.uid)' -

apiVersion: v1
data:
  password: cXXXXXXXXXX=
  user: cXXXXXX=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"cGFzc3cwcmQ=","user":"cm9vdA=="},"kind":"Secret","metadata":{"annotations":{},"name":"mongodb","namespace":"sampleNameSpace"},"type":"Opaque"}
  name: mongodb
  namespace: sampleNameSpace
type: Opaque
```

- Check on `node7` where the `persistentVolume` is attached

```sh
ubuntu@node7:/mnt/disks-by-id/disk0$ hostname
node7
ubuntu@node7:/mnt/disks-by-id/disk0$ pwd
/mnt/disks-by-id/disk0
ubuntu@node7:/mnt/disks-by-id/disk0$ ls -la
total 32
drwxrwsr-x 5 root 1001  4096 Oct 25 20:16 .
drwxr-xr-x 3 root root  4096 Oct 20 08:18 ..
drwxrwsr-x 4 1001 1001  4096 Oct 24 08:00 2021_10_24_00:00
drwxr-sr-x 4 1001 1001  4096 Oct 25 08:00 2021_10_25_00:00
drwxrws--- 2 root 1001 16384 Oct 18 21:08 lost+found
ubuntu@node7:/mnt/disks-by-id/disk0$ cd 2021_10_25_00\:00/
ubuntu@node7:/mnt/disks-by-id/disk0/2021_10_25_00:00$ du -ksh
7.5G	.
```

#### Cronjob to delete old MongoDB dump files

The objective of this is to delete old archive of `mongodump` if there is disk space limit pressure

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mongodump-backup-rm-oldfile
  namespace: sampleNameSpace
spec:
  schedule: '@daily'
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: mongodump-backup-rm-oldfile
              image: busybox
              args:
              - /bin/sh
              - -c
              - cd /tmp/mongodump/; find . -type d -mtime +2 -exec rm -fr {} \;
              volumeMounts:
              - mountPath: "/tmp/mongodump/"
                name: mongodump
          restartPolicy: OnFailure
          volumes:
          - name: mongodump
            persistentVolumeClaim:
              claimName: mongodump
```

Command `cd /tmp/mongodump/; find . -type d -mtime +3 -exec rm -fr {} \;`

- `cd /tmp/mongodump/; find ...` = avoid deleting parent directory
- `-type d` = `directory`
- `-mtime +3` = older than 3 days

## Export and Import `collections` from MongoDB

The pre-requisite is to enable a service port-forward in Kubernetes

```sh
kubectl -n sampleNameSpace port-forward svc/sample-sub-mongodb 27017:27017 &
```

For example, to export `addVisitor_KE` collection in `DEV_KE_GROUP` namespace

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:latest mongoexport --host=127.0.0.1 --port=27017   \
  -u root -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase "admin" \
  --db ars02 --collection addVisitor_KE__DEV_KE_GROUP \
  --out=/app/addVisitor_KE__DEV_KE_GROUP.json --type json
```

To import a json. Notice the `--collection=addVisitor_KE__Testing_Property_KeGroup` which means `collectionName=addVisitor_KE` under `nameSpace=Testing_Property_KeGroup`

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:latest mongoimport --host=127.0.0.1 --port=27017 \
  -u root -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase "admin" \
  --db ars02 --collection addVisitor_KE__Testing_Property_KeGroup \
  --file=app/addVisitor_KE__DEV_KE_GROUP.json --type json
```

Pay attention to the formatType of import and export

## Force MongoDB member to become primary

Check status

```sh
//-----------------------------
//check the status of the replication
//-----------------------------
rs.status()
```

```sh
{
    "set" : "Sample",
    "date" : ISODate("2021-04-26T14:59:40.263Z"),
    "myState" : 1,
    "members" : [
        {
            "_id" : 0,
            "name" : "mongo:37001",
            "health" : 1,
            "state" : 1,
            "stateStr" : "PRIMARY",
            "uptime" : 1022,
            "optime" : Timestamp(1461678841, 1),
            "optimeDate" : ISODate("2020-04-26T13:54:01Z"),
            "electionTime" : Timestamp(1461682464, 1),
            "electionDate" : ISODate("2020-04-26T14:54:24Z"),
            "configVersion" : 239068,
            "self" : true
        },
        {
            "_id" : 1,
            "name" : "mongo:37002",
            "health" : 1,
            "state" : 2,
            "stateStr" : "SECONDARY",
            "uptime" : 1022,
            "optime" : Timestamp(1461678841, 1),
            "optimeDate" : ISODate("2020-04-26T13:54:01Z"),
            "lastHeartbeat" : ISODate("2020-04-26T14:59:40.206Z"),
            "lastHeartbeatRecv" : ISODate("2020-04-26T14:59:40.179Z"),
            "pingMs" : 0,
            "configVersion" : 239068
        },
        {
            "_id" : 2,
            "name" : "mongo:37003",
            "health" : 1,
            "state" : 2,
            "stateStr" : "SECONDARY",
            "uptime" : 1022,
            "optime" : Timestamp(1461678841, 1),
            "optimeDate" : ISODate("2020-04-26T13:54:01Z"),
            "lastHeartbeat" : ISODate("2020-04-26T14:59:40.238Z"),
            "lastHeartbeatRecv" : ISODate("2020-04-26T14:59:39.593Z"),
            "pingMs" : 0,
            "configVersion" : 239068
        }
    ],
    "ok" : 1
}
```

Logged into node2 and node3 run the following command on then, one after another:

```sh
rs.stepDown()
```

Bring up the cluster

```sh
cfg = rs.conf()
printjson(cfg)
cfg.members = [cfg.members[0] , cfg.members[1] , cfg.members[2]]

rs.reconfig(cfg, {force : true})
```

> Reference > https://dba.stackexchange.com/questions/136621/how-to-set-a-mongodb-node-to-return-as-the-primary-of-a-replication-set

#### MongoDB Backup and Restore with Docker


- Figure out MongoDB's root passwd

```sh
kubectl -n sampleNameSpace get secret mongodb -o jsonpath="{.data.password}" | base64 --decode
```

```sh
export MONGODB_ROOT_PASSWORD=$(kubectl -n sampleNameSpace get secret mongodb -o \
  jsonpath="{.data.password}" | base64 --decode)

echo $MONGODB_ROOT_PASSWORD
```

- Export MongoDB's service port

```sh
kubectl -n sampleNameSpace get svc

kubectl -n sampleNameSpace port-forward svc/sample-sub-mongodb 27017:27017 &
```

- Create mongoDB backup directory

```sh
mkdir mongo_backup; chmod o+w mongo_backup/
cd mongo_backup/
```

- Execute backup

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:latest mongodump --host=127.0.0.1 --port=27017 \
  -u root -p $MONGODB_ROOT_PASSWORD -o /app
```

and if you want a specific version of MongoDB image to match the server

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:3.6.8 mongodump --host=127.0.0.1 --port=27017 \
  -u root -p $MONGODB_ROOT_PASSWORD -o /app
```

- Execute restore

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
	bitnami/mongodb:latest mongorestore -u root -p $MONGODB_ROOT_PASSWORD /app
```

> Ref > https://docs.bitnami.com/tutorials/backup-restore-data-mongodb-kubernetes/
