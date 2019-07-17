# Kubernetes and Vantiq Product Deployment on Private Cloud behind a Proxy

#### Document Objective
- Deployment occurs on a OpenStack based private cloud
- Everything is behind __proxy__. Again, everything is behind a __proxy__

## Highlighted Steps

#### Environment Preparation on Master

- Update ```/etc/hosts``` and hostname all nodes
- Create ```ubuntu``` user
- ```ssh-copy-id {ubuntu, root}@workers```

#### Environment Preparation on all Nodes

- ```usermod -aG {docker,sudo} ubuntu```
- distribute the following onto all node when having a proxy:
  * ```/etc/apt/source.list```
  * ```/etc/apt/apt.conf.d/proxy``` # This configuration enables ```apt install``` behind a proxy. Check ```proxy``` file in Appendix
  * ```~/.curlrc``` # This configuration enable ```curl``` behind a proxy. Check its content in Appendix and note access within local subnet should be avoided or have ```~/.curlrc``` turned-off

- ```hostnamectl set-hostname $workers```

- ```swapoff -a```

#### Install and configure Docker and K8S on all other nodes

- ```apt install docker-ce```
- configure docker ```/etc/systemd/system/docker.service.d/{http-proxy.conf,override.conf}``` # ```http-proxy.conf``` one enables ```docker pull``` behind a proxy and ```override.conf``` enables docker managed by ```systemd```. The samples are available in Appendix
- ```systemctl daemon-reload; systemctl restart docker.service```

#### Distribute all images to all workers

- ```docker save $(docker images | sed '1d' | awk '{print $1 ":" $2 }') -o all_optimized.tar```
- rsync image.tar to all nodes then untar and extract ```docker -load -i base.tar```

#### Install K8S on all workers, with specific version K8S
- ```apt install -qy {kubelet,kubectl,kubeadm}=1.14-00```

#### ```kubeadm init``` with specific ver of K8S

```
kubeadm init —pod-network-cidr=10.244.0.0/16 \
  —image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
  —kubernetes-version “1.14.1"
```

#### Apply flannel.yaml network plugin manually
```
kubectl apply -f flannel.yaml
```

#### ```kubeadm join``` on all workers respectively

---

## Install Vantiq

#### Install Java

```
apt install opened-8-jre-headless
```

#### Configure ```git``` on Master

```
http_proxy=http://ip:3128
https_proxy=http://ip:3128
git config —global http.proxy $http_proxy
git config —global https.proxy $http_proxy
```

#### Configure ```gradle```

- Configure gradle proxy in ```~/k8sdeploy_tools/.gradle/gradle.properties``` # Sample is available in Appendix

- ```./gradlew -Pcluster=jd configureClient```    # targetCluster is created

- The above proc will fail and ```~/k8sdeploy_tools/targetCluster``` directory is created

- Modify ```~/k8sdeploy_tools/targetCluster/cluster.properties```

  ```
  requireRemote=false
  vantiq_system_release=1.1.5
  deployment=production
  vantiq.installation=eda    # this value will be used for namespace to hold Vantiq
  excludeKeycloak=true
  ```

- After modifying ```cluster.properties```, create a branch # important

  ```
  cd ~/k8sdeploy_tools/targetCluster/; git checkout -b jd    # jd = -Pcluster’s value
  ```

- rerun >

  ```
  ./gradlew -Pcluster=jd configureClient    # targetCluster is created
  ```

- At this moment, the access to http://github.com/Vantiq/k8sdeploy_clusters.git is required in this session where to continue installation and configuration

  ```
  cd /tmp/
  git clone http://github.com/Vantiq/k8sdeploy_clusters.git
  ```

  Then come back to ```~/k8sdeploy_tools/targetCluster```, keep running ```./gradlew -Pcluster=jd configureClient```till the proc succeeds (download all dependencies of gradle)
  The expected output looks when configuring helm repo
  ```
  Adding stable repo with URL:
  ...
  ```

  ```cd ~/k8sdeploy_tools/.gradle; du -ksh    # at least 561MB```

- ```./gradlew -Pcluster=jd clusterInfo```

- ```kubectl apply -f rbac-config.yaml```
  Important or access-control err challenging later
- ```./gradlew -Pcluster=jd setupCluster```

At this time, ```local-volume-provisioner``` are running on all workers

#### Install ```tiller```
Manually install tiller, preventing from being init’d by helm through proxy

```
cd ~/k8sdeploy_tools/
./helm init —dry-run —debug > /tmp/tiller.yaml
kubectl apply -f tiller.yaml
```

Initialize Helm client

```
./helm init —client-only —skip-refresh
```

Reference > https://whmzsu.github.io/helm-doc-zh-cn/quickstart/install_faq-zh_cn.html

> stty erase ^h    # prevent ^H when hitting BACKSPACE in openssl req cmd

#### Generate ```key.pem``` and ```cert.pem```

```
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.pem -out cert.pem
```

```
Org Name: stategrid.nx
Org Unit Name: stategrid.nx
Common Name (e.g. server FQDN): eda2
```

```
cp {key,cert}.pem ~/k8sdeploy_tools/targetCluster/deploy/vantiq/certificates/
cp {key,cert}.pem ~/k8sdeploy_tools/targetCluster/deploy/certificates/
```

#### Modify ```~/k8sdeploy_tools/targetCluster/deploy.yaml```

#### Setup private helm for Vantiq

```
cp modified {build,settings}.gradle ~/k8sdeploy_tools/
```

```
mkdir -p ~/helm_local    # for the mounted path from the below, where private charts copied to
```

```
docker run -it —rm vantiq/helm-kubectl:2.11.0 bash    # get into docker container

ip route show | awk ‘/default/ {print $3}’    # the output = 172.17.0.1    # run inside the container
```

```
docker run -d —name chart-repo -p 8879:8879 -v /home/ubuntu/helm_local/charts/:/charts \
  -h charts vantiq/helm-kubectl:2.11.0 helm serve \
  —address charts:8879 —repo-path /charts \
  —url http://172.17.0.1:8879
```

```
docker logs chart-repo    # check logs
```


#### ```deployNginx```, ```deployShared``` and ```deployVantiq```
```
./gradlew -Pcluster=jd repoUpdate

./gradlew -Pcluster=jd deployNginx

./gradlew -Pcluster=jd deployShared
```

#### The pattern of creating mount-point
* ```sudo mkdir -p /mnt/disks-by-id/disk0```
* ```sudo fdisk /dev/vdX``` > create partition then commit
* ```sudo mkfs.ext4 -j /dev/vdX1```
* ```sudo mount /dev/vdX1 /mnt/disks-by-id/disk0```

The sequence to mount:
* 50G for ```grafanadb-mysql```
* 150G for ```influxdb-influxdb```
* 50G for ```grafana```, on worker which perhaps needs to ```docker pull grafana/grafana:5.4.3```

```
./gradlew -Pcluster=jd deployVantiq
```
* Find workers that have 2* 530G block disks
* ```sudo mkdir -p /mnt/disks-by-id/disk0```
* ```sudo fdisk /dev/vdX```
* ```sudo mount /dev/vdX1 /mnt/disks-by-id/disk0```

## Appendix
#### ```/etc/apt/apt.conf.d/proxy```

```
Acquire::http::Proxy "http://ip:3128"
Acquire::https::Proxy "http://ip:3128"
```

#### ```~/.curlrc```
```
proxy = ip:3128
```

#### ```/etc/systemd/system/docker.service.d/http-proxy.conf```
```
[Service]
Environment="HTTP_PROXY=http://up:3128/"
```

#### ```/etc/systemd/system/docker.service.d/override.conf```

This file is generated by ```systemctl edit docker``` automatically
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

#### ```~/k8sdeploy_tools/.gradle/gradle.properties```

```
systemProp.http.proxyHost=ip
systemProp.http.proxyPost=3128
systemProp.https.proxyHost=ip
systemProp.https.proxyPost=3128

gitUsername=me
gitPassword=token
```
