# Install Kubernetes without Internet Connection



#### Ubuntu 18.04 LTS Repo

- Install
```
apt install apache2 apt-mirror
```

- Create repo location

```
mkdir -p /var/www/debs
chown -R ubuntu.ubuntu /var/www/debs
```

- Start Apache2
```
systemctl restart apache2
```

Verify in command line and you should see the front-page
```
lynx http://[IP]
```

- Modify ```/etc/apt/mirror.list``` and change value of ```set base_path``` to wherever the repo resides. ```deb-src``` can be commented-out if source code isn't needed to save the disk size

```
ubuntu@vantiq2-test01:/var/www/debs$ cat /etc/apt/mirror.list
############# config ##################
#
set base_path    /var/www/debs
#
# set mirror_path  $base_path/mirror
# set skel_path    $base_path/skel
# set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
# set defaultarch  <running host architecture>
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     20
set _tilde 0
#
############# end config ##############

deb http://archive.ubuntu.com/ubuntu bionic main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-security main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-updates main restricted universe multiverse
#deb http://archive.ubuntu.com/ubuntu bionic-proposed main restricted universe multiverse
#deb http://archive.ubuntu.com/ubuntu bionic-backports main restricted universe multiverse

#deb-src http://archive.ubuntu.com/ubuntu bionic main restricted universe multiverse
#deb-src http://archive.ubuntu.com/ubuntu bionic-security main restricted universe multiverse
#deb-src http://archive.ubuntu.com/ubuntu bionic-updates main restricted universe multiverse
#deb-src http://archive.ubuntu.com/ubuntu bionic-proposed main restricted universe multiverse
#deb-src http://archive.ubuntu.com/ubuntu bionic-backports main restricted universe multiverse
```

Download is about 108GB in size

- Create repo

```
sudo apt-mirror
```

- Update path

Create a soft-link under ```/var/www/html/``` that points to the actual location. Eg,

```
ubuntu@vantiq2-test03:/var/www/html$ pwd
/var/www/html
ubuntu@vantiq2-test03:/var/www/html$ ls -la debs
lrwxrwxrwx 1 root root 27 Jun 29 10:48 debs -> /mnt/disks-by-id/disk0/debs
```

- Update apt client ```sudo vi /etc/apt/sources.list``` and add

```
deb http://10.100.102.13/ubuntu bionic universe
deb http://10.100.102.13/ubuntu bionic main restricted
deb http://10.100.102.13/ubuntu bionic-updates main restricted
```

Reference >
- https://www.unixmen.com/setup-local-repository-in-ubuntu-15-04/
- https://www.maketecheasier.com/setup-local-repository-ubuntu/
- https://askubuntu.com/questions/170348/how-to-create-a-local-apt-repository

#### Save and Restore Docker Images

- All in one

```
docker save $(docker images | sed '1d' | awk '{print $1 ":" $2 }') -o allinone.tar

docker load -i allinone.tar
```

Reference > https://stackoverflow.com/questions/35575674/how-to-save-all-docker-images-and-copy-to-another-machine

#### DNS

+ Create and Launch

```
docker pull sameersbn/bind:latest
docker run -d --name=bind --dns=127.0.0.1 --publish=10.100.102.11:53:53/udp \
 --publish=10.100.102.11:10000:10000 --env='ROOT_PASSWORD=secret' sameersbn/bind:latest
```

Reference > https://linoxide.com/containers/setting-dns-server-docker/

http://www.damagehead.com/blog/2015/04/28/deploying-a-dns-server-using-docker/

+ Configure DNS entry
https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-14-04

#### SMTP
https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-as-a-send-only-smtp-server-on-ubuntu-14-04

https://mailu.io/1.6/kubernetes/mailu/index.html

https://github.com/tomav/docker-mailserver/wiki/Using-in-Kubernetes
