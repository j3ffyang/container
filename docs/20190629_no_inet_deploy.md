# Install Kubernetes without Internet Connection or Behind Proxy

#### Document Objective
- Ubuntu local repository
- apt install through proxy
- curl/ wget through proxy
- Docker through proxy
- Kubernetes through proxy  

## Ubuntu 18.04 LTS Repo

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

## DNS

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

## SMTP
https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-as-a-send-only-smtp-server-on-ubuntu-14-04

https://mailu.io/1.6/kubernetes/mailu/index.html

https://github.com/tomav/docker-mailserver/wiki/Using-in-Kubernetes


## Save and Restore Docker Images

#### Tar all images in one from an existing working env

```
docker save $(docker images | sed '1d' | awk '{print $1 ":" $2 }') -o allinone.tar

docker load -i allinone.tar
```

Reference > https://stackoverflow.com/questions/35575674/how-to-save-all-docker-images-and-copy-to-another-machine

#### Copy and split tar files

https://superuser.com/questions/362177/how-to-split-big-files-on-mac

```
mac:sg jeff$ split -b 2000000000 allinone.tar "allinone.part"
mac:sg jeff$ ls -la
total 26497152
drwxrwxr-x  11 jeff  staff         352 Jul  1 20:20 .
drwxr-xr-x  10 jeff  staff         320 Jul  1 15:32 ..
-rw-rw-r--   1 jeff  staff        4478 Jun 29 11:26 allinone.lst
-rw-r--r--   1 jeff  staff  2000000000 Jul  1 20:20 allinone.partaa
-rw-r--r--   1 jeff  staff  2000000000 Jul  1 20:20 allinone.partab
-rw-r--r--   1 jeff  staff  1899848704 Jul  1 20:20 allinone.partac
-rw-------   1 jeff  staff  5899848704 Jun 29 11:25 allinone.tar

mac:sg jeff$ mkdir split_dir
mac:sg jeff$ cd split_dir/
mac:split_dir jeff$ ls
mac:split_dir jeff$ cat ../allinone.parta* > a.tar
mac:split_dir jeff$ ls
a.tar
mac:split_dir jeff$ md5 a.tar
MD5 (a.tar) = f874fb298c6818fce938b836441c33d3
mac:split_dir jeff$ md5 ../allinone.tar
MD5 (../allinone.tar) = f874fb298c6818fce938b836441c33d3
mac:split_dir Jeff$
```

#### Copy image tar to all nodes then Restore all images
```
docker load -i allinone.tar
```

## Setup proxy

Reference > http://www.iasptk.com/ubuntu-server-behind-proxy-firewall/

Switch to China docker repo

Configure proxy for the following

#### ```apt```

```
cat /etc/apt/apt.conf.d/proxy

Acquire::http::Proxy  "http://ip:3128";
Acquire::https::Proxy "http://ip:3128";
```

#### ```curl```
```
cat ~/.curlrc

proxy = ip:3128
```

#### ```wget```
```
cat ~/.wgetrc

http_proxy = http://ip:3128
https_proxy = http://ip:3128
```

For example, ```wget``` without checking SSL
```
wget https://kubernetes-charts.storage.googleapis.com/index.yaml --no-check-certificate
```

#### ```docker```

```
cat /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://ip:3128"
```

```
systemctl daemon-reload; systemctl restart docker.service
```

Reference > https://stackoverflow.com/questions/23111631/cannot-download-docker-images-behind-a-proxy#28093517

#### ```gradle```
```
cat ~/.gradle/gradle.properties

systemProp.http.proxyHost=ip
systemProp.http.proxyPort=3128
systemProp.https.proxyHost=ip
systemProp.https.proxyPort=3128

gitUsername=j3ffyang
gitPassword=token
```

#### ```kubeadm```

Check no_proxy

```
env | grep -i _proxy=
```

Set no_proxy
http://xmodulo.com/how-to-configure-http-proxy-exceptions.html

```
sudo swapoff -a
```

## Install specific version of ```kubelet```/ ```kubectl```/ ```kubeadm```

```
apt install -qy kubelet=1.14.1-00 kubectl=1.14.1-00 kubeadm=1.14.1-00
https://stackoverflow.com/questions/49721708/how-to-install-specific-version-of-kubernetes
```

```
sudo kubeadm init —apiserver-advertise-address=10.216.78.16 —pod-network-cidr=10.244.0.0/16 —image-repository registry.cn-hangzhou.aliyuncs.com/google_containers —kubernetes-version “1.14.1"
```

#### Save all images in one tar

```
docker save $(docker images | sed '1d' | awk '{print $1 ":" $2 }') -o allinone.tar
```

#### Load/ restore all images on each of docker nodes
```
for i in {17..21}; do ssh 10.$i 'docker load -i base_img.tar'; done
```

Reference > https://stackoverflow.com/questions/35575674/how-to-save-all-docker-images-and-copy-to-another-machine

## Product

#### Setup proxy for git

```
git config --global http.proxy http://10.216.1.213:3128
```

Update

```
~/targetCluster/cluster.properties
~/.gradle/gradle.properties
```

#### Create a specific branch under ```~/targetCluster```

```
cd targetCluster/
git checkout -b sg  # sg = value of Pcluster
```

```
./gradlew configureClient -Pcluster=sg -Dhttps.proxyHost=xxxx \
  -Dhttps.proxyPort=3128 \
  -Dhttps.protocols="TLSv1,TLSv1.1,TLSv1.2"
```

There might be up to 50~ 100 dependency files

#### Configure tiller

Reference > https://rancher.com/docs/rancher/v2.x/en/installation/ha/helm-init/

```
helm init -c --skip-refresh
```
