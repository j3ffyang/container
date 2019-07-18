# Kubernetes and Vantiq Product Deployment on Private Cloud behind a Proxy

#### Document Summary
- Deployment occurs on a OpenStack based private cloud
- Everything is behind __proxy__. Again, everything is behind a __proxy__
- No KeyCloak installed, as well as PostgreSQL for KeyCloak

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
Org Name: fqdn.com
Org Unit Name: fqdn.com
Common Name (e.g. server FQDN): eda2.fqdn.com
```

```
openssl req -new -x509 -nodes -out ingress.crt -keyout ingress.key -days 9999 \
  -subj /CN=hostname.fqdn.com
```

```
cp {key,cert}.pem ~/k8sdeploy_tools/targetCluster/deploy/vantiq/certificates/
cp {key,cert}.pem ~/k8sdeploy_tools/targetCluster/deploy/certificates/
```

#### Modify, pay attention to FQDN's value

- ```~/k8sdeploy_tools/targetCluster/deploy.yaml```
- ```~/k8sdeploy_tools/targetCluster/cluster.properties```

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

#### ```docker image ls```

Up to 20190717, the latest used images

```
ubuntu@vantiq2-test01:~$ docker image ls | sort
bitnami/mongodb                                                  4.0.3-debian-9      b0bfbf15c8ae        6 months ago        382MB
gcr.io/kubernetes-helm/tiller                                    v2.11.0             ac5f7ee9ae7e        9 months ago        71.8MB
gcr.io/kubernetes-helm/tiller                                    v2.12.2             e3e79362c9d6        6 months ago        81.4MB
grafana/grafana                                                  5.4.3               d0454da13c84        6 months ago        240MB
influxdb                                                         1.7.4-alpine        1063411109c2        4 months ago        117MB
k8s.gcr.io/coredns                                               1.3.1               eb516548c180        6 months ago        40.3MB
k8s.gcr.io/defaultbackend                                        1.4                 846921f0fe0e        21 months ago       4.84MB
k8s.gcr.io/etcd                                                  3.3.10              2c4adeb21b4f        7 months ago        258MB
k8s.gcr.io/kube-apiserver                                        v1.14.1             cfaa4ad74c37        3 months ago        210MB
k8s.gcr.io/kube-controller-manager                               v1.14.1             efb3887b411d        3 months ago        158MB
k8s.gcr.io/kube-proxy                                            v1.14.1             20a2d7035165        3 months ago        82.1MB
k8s.gcr.io/kube-scheduler                                        v1.14.1             8931473d5bdb        3 months ago        81.6MB
k8s.gcr.io/pause                                                 3.1                 da86e6ba6ca1        19 months ago       742kB
mysql                                                            5.7.14              4b3b6b994512        2 years ago         385MB
quay.io/coreos/flannel                                           v0.11.0-amd64       ff281650a721        5 months ago        52.6MB
quay.io/coreos/kube-rbac-proxy                                   v0.3.0              543e2018dcac        15 months ago       40.2MB
quay.io/external_storage/local-volume-provisioner                v2.2.0              a17d656ccc81        11 months ago       289MB
quay.io/kubernetes-ingress-controller/nginx-ingress-controller   0.20.0              a3f21ec4bd11        9 months ago        513MB
quay.io/prometheus/node-exporter                                 v0.15.2             ff5ecdcfc4a2        19 months ago       22.8MB
telegraf                                                         1.9.4-alpine        07140bfde316        5 months ago        74.1MB
vantiq/helm-kubectl                                              2.11.0              a92e4902c4bb        5 months ago        166MB
vantiq/vantiq-server                                             1.25.13             ed619139fc57        6 weeks ago         622MB
```

#### SSL Keys in K8S

- List in ```default``` namespace
```
kubectl -n default get secrets
```

- Describe the secret, named ```vantiq-cert``` in ```default``` namespace

```
kubectl -n default get secret vantiq-cert -o yaml
```

Re-generate secret by using new {cert,key}.pem files

```
kubectl -n default create secret tls --cert cert.pem --key key.pem \
  vantiq-cert --dry-run -o yaml | kubectl apply -f -
```

Reference > https://developer.ibm.com/recipes/tutorials/changing-the-tls-certificate-in-ingress-in-information-server-kubernetes-deployments/

In case, you need to create a CSR
```
openssl req -new -newkey rsa:2048 -nodes -keyout ingress.key -out ingress.csr
```

Convert p12 to pem
```
openssl pkcs7 -print_certs -in certificate.p7b -out ingress.pem  
```

To inspect cert.pem
```
openssl x509 -in ingress.pem -text -noout
```

#### Add a DNS entry in lb-nginx-ingress-controller

```
kubectl -n default edit daemonsets.apps lb-nginx-ingress-controller
```

Sample

```
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox
    command:
    - cat
    args:
    - "/etc/hosts"
```

Reference > https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/
