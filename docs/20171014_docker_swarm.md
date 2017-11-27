# Setup Docker and Enable Swarm mode

## Document Objective
- Install docker-ce on all hosts
- Enable docker swarm
- Tweak docker

## Steps

```
for i in {2..6}; do ssh root@cm0$i "apt-get install curl \
  apt-transport-https ca-certificates software-properties-common -y; \
  echo cm0$i"; done
```

```
for i in {2..6}; do ssh root@cm0$i "curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg | apt-key add -"; done
```

```
for i in {2..6}; do ssh root@host0$i 'add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable"'; echo host0$i; done
```

```
for i in {2..6}; do ssh root@cm0$i "apt-get update; \
  apt-get install docker-ce -y"; done
```

Install docker-composer

Edit ```/etc/docker/daemon.json``` on each docker host

```
sudo cat /etc/docker/daemon.json
{"registry-mirrors": ["http://54e31bff.m.daocloud.io"],
  "graph":"/data/docker",
  "hosts":  ["fd://", "tcp://172.29.167.171:2375"]
}
```

Install and configure Swarm

Add label on __all__ hosts. This is an example for one host

```
docker node update --label-add host=0 $HOST0_ID
```

You might want to

```
sudo chown -R ubuntu:ubuntu /data
```

where ```/data``` is default location for docker

Grant ```ubuntu``` user access to manage Docker

```
sudo gpasswd -a $USER docker
```
