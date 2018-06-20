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

Install ```pip``` and ```docker-compose```

```
for i in {1..8}; do ssh host0$i -t "sudo apt-get install python-pip -y"; done
for i in {1..8}; do ssh host0$i -t "sudo pip install docker-compose"; done
```

Edit ```/etc/docker/daemon.json``` on each docker host

```
sudo cat /etc/docker/daemon.json
{
  "registry-mirrors": ["http://54e31bff.m.daocloud.io"],
  "graph":"/data/docker"
}
```

where ```/data/docker``` holds all docker data

Create ```/etc/systemd/system/docker.service.d/docker.conf```

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

Restart
```
systemctl daemon-reload
systemctl restart docker.service
```

Grant a non- root user to operate docker
```
sudo usermod -aG docker your-user
```

Chown

```
for i in {1..8}; do ssh host0$i -t "sudo chown -R ubuntu:ubuntu /data/"; done
for i in {1..8}; do ssh host0$i -t "sudo systemctl daemon-reload; sudo systemctl restart docker.service"; done
```

Install and configure Swarm

Add label on __all__ hosts. This is an example for one host

```
docker node update $NODE_ID --label-add host=$ID
```

Grant ```ubuntu``` user access to manage Docker

```
sudo gpasswd -a $USER docker
```
