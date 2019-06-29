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

#### Save and Restart Docker Images

- All in one

```
docker save $(docker images | sed '1d' | awk '{print $1 ":" $2 }') -o allinone.tar

docker load -i allinone.tar
```

Reference > https://stackoverflow.com/questions/35575674/how-to-save-all-docker-images-and-copy-to-another-machine
