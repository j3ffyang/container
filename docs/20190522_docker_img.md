# Create Custom Docker Image

#### Doc Objective
- Create a custom docker image

> Reference > https://linoxide.com/linux-how-to/2-ways-create-docker-base-image/

## Create a full image using tar

```
sudo debootstrap xenial xenial > /dev/null
sudo tar -C xenial -c . | docker import - xenial
```

```
docker run xenial cat /etc/lsb-release
```

```
docker run -it xenial /bin/bash
```

## Creating Base Image using Scratch

```
tar cv --files-from /dev/null | docker import - scratch
```
