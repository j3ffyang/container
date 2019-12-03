# Create Custom Docker Image

#### Doc Objective
- Create a custom docker image

> Reference > https://linoxide.com/linux-how-to/2-ways-create-docker-base-image/

## Create a full image using tar

```bash
sudo debootstrap xenial xenial > /dev/null
sudo tar -C xenial -c . | docker import - xenial
```

```bash
docker run xenial cat /etc/lsb-release
```

```bash
docker run -it xenial /bin/bash
```

## Creating Base Image using Scratch

```bash
tar cv --files-from /dev/null | docker import - scratch
```

## Save and Load Image in Docker

On node where an image resides, for example,

```bash
root@ubuntu05:~# docker image list | grep ubuntu
ubuntu/ubuntu-server      1.24.12             ab30f4dfd278        6 weeks ago         601MB
ubuntu/keycloak           4.2.1.Final         0c41c64e19a4        6 weeks ago         801MB
root@ubuntu05:~# docker save -o ./ubuntu-server.tar  ubuntu/ubuntu-server:1.25.6
```
Copy the tar file to the target node. On node where to load the image,
```bash
root@ubuntu06:~# docker load -i ./ubuntu-server.tar
```
