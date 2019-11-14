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

#### Install and configure Docker and K8S on all nodes

- ```apt install docker-ce```
- Grant ```$USER``` to control docker ```sudo gpasswd -a $USER docker```
- configure docker ```/etc/systemd/system/docker.service.d/{http-proxy.conf,override.conf}``` # ```http-proxy.conf``` one enables ```docker pull``` behind a proxy and ```override.conf``` enables docker managed by ```systemd```. The samples are available in Appendix
- ```systemctl daemon-reload; systemctl restart docker.service```

#### Distribute all images to all workers

- ```docker save $(docker images | sed '1d' | awk '{print $1 ":" $2 }') -o all_optimized.tar```
- rsync image.tar to all nodes then untar and extract ```docker load -i base.tar```

#### Install K8S on all workers, with specific version K8S
- ```apt install -qy {kubelet,kubectl,kubeadm}=1.14-00```

#### ```kubeadm init``` with specific ver of K8S

```bash
kubeadm init —pod-network-cidr=10.244.0.0/16 \
  —image-repository gcr.azk8s.cn/google_containers \
  —kubernetes-version “1.14.1"
```

Alternative registry > https://github.com/Azure/container-service-for-azure-china/tree/master/aks

#### Apply ```flannel.yaml``` network plugin manually

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

#### ```kubeadm join``` on all workers respectively

---

## Install Vantiq Product

#### Pre-requisite

##### Install Java

```bash
apt install opened-8-jre-headless
```

##### Configure ```git``` on Master, if there is a proxy

```bash
http_proxy=http://ip:3128
https_proxy=http://ip:3128
git config —global http.proxy $http_proxy
git config —global https.proxy $http_proxy
```

##### Configure ```gradle```

- Configure gradle proxy in ```~/k8sdeploy_tools/.gradle/gradle.properties``` # Sample is available in Appendix

  ```bash
  gitUsername=MY_GITHUB_ID
  gitPassword=MY_TOKEN
  ```

#### Setup Vantiq Env

- Run to create ```targetCluster``` and associated files # jd = -Pcluster's value as example

  ```bash
  ./gradlew -Pcluster=jd configureClient
  ```

- The above process will fail, as expected, and ```~/k8sdeploy_tools/targetCluster``` directory is created

- ```  cp ~/.kube/config ~/k8sdeploy_tools/targetCluster/kubeconfig```

- Modify ```~/k8sdeploy_tools/targetCluster/cluster.properties```

  ```bash
  requireRemote=false
  provider=kubeadm
  vantiq_system_release=1.1.5
  deployment=production
  vantiq.installation=eda    # used for namespace to hold Vantiq and the same as 1st part of FQDN!!!
  excludeKeycloak=true       # this excludes KeyCloak install
  ```

  __Attention__ to setting of ```vantiq.installation```, which will be used as Vantiq namespace in K8S and must match the 1st part of FQDN, being used to sign SSL Key.
<br>

- After modifying ```cluster.properties```, create a Git branch # important

  ```bash
  cd ~/k8sdeploy_tools/targetCluster/; git checkout -b jd    # jd = -Pcluster’s value
  ```

  __Attention__: the Git branch must match the value of ```-Pcluster``` which will be used in service def in K8S
<br>  

- At this moment, the access to http://github.com/Vantiq/k8sdeploy_clusters.git is required to get granted in current shell session where to continue installation and configuration

  ```bash
  cd /tmp/; git clone http://github.com/Vantiq/k8sdeploy_clusters.git
  ```

- Then come back to ```~/k8sdeploy_tools/```, keep running

  ```./gradlew -Pcluster=jd configureClient```

  till the proc succeeds (download all dependencies of gradle). The expected output looks when configuring helm repo

  ```bash
  Adding stable repo with URL:
  ...
  ```

  ```bash
  cd ~/k8sdeploy_tools/.gradle; du -ksh    # about 561MB
  ```

- Check helm repo. If it doesn't have ```vantiq``` repo, check "update helm repo" part in this doc

  ```bash
  cd ~/k8sdeploy_tools/; ./helm repo list
  ```

- ```./gradlew -Pcluster=jd clusterInfo```


- ```kubectl apply -f rbac-config.yaml```
  Important or access-control err challenging later. You'd see

  ```bash
  kubectl apply -f rbac-config.yaml
  serviceaccount/tiller created
  clusterrolebinding.rbac.authorization.k8s.io/tiller created
  ```

- ```./gradlew -Pcluster=jd setupCluster```

- At this time, ```local-volume-provisioner``` are running on all workers. If receiving ```ImagePullBackOff``` in status, go edit

  ```bash
  kubectl -n default edit daemonsets.apps local-volume-provisioner
  ```

  as the following (you'd need to manually download such docker image on each node before and image name = ```quay.io/external_storage/local-volume-provisioner```)

  ```bash
  imagePullPolicy: IfNotPresent
  ```

##### Install Tiller
Manually install tiller, preventing from being init’d by helm through proxy

> Reference > https://rancher.com/docs/rancher/v2.x/en/installation/ha/helm-init/

```bash
cd ~/k8sdeploy_tools/
./helm init —dry-run —debug > /tmp/tiller.yaml
kubectl apply -f tiller.yaml
```

##### Initialize Helm client manually

If receiving the error when running ```./gradlew -Pcluster=ins setupCluster```

```bash
./gradlew -Pcluster=ins setupCluster
serviceaccount/tiller unchanged
clusterrolebinding.rbac.authorization.k8s.io/tiller configured
configmap/local-provisioner-config created
daemonset.extensions/local-volume-provisioner created
serviceaccount/local-storage-admin created
clusterrolebinding.rbac.authorization.k8s.io/local-storage-provisioner-pv-binding created
clusterrole.rbac.authorization.k8s.io/local-storage-provisioner-node-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/local-storage-provisioner-node-binding created
storageclass.storage.k8s.io/local-storage created
storageclass.storage.k8s.io/vantiq-sc created
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Error: Looks like "https://kubernetes-charts.storage.googleapis.com" is not a valid chart repository or cannot be reached: read tcp 172.17.0.2:36652->216.58.200.48:443: read: connection reset by peer
> Task :initHelmServer FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':initHelmServer'.
> COMMAND: vantiq_helm init --service-account tiller --upgrade

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 4m 23s
5 actionable tasks: 3 executed, 2 up-to-date
```

Run

```bash
./helm init —client-only —skip-refresh
```

Reference > https://whmzsu.github.io/helm-doc-zh-cn/quickstart/install_faq-zh_cn.html

> stty erase ^h    # prevent ^H when hitting BACKSPACE in openssl req cmd

##### Update ```vantiq``` repo in Helm

```bash
./helm repo list
NAME    URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts                    
vantiq	https://vantiq.github.io/k8sdeploy/charts       
```

If ```vantiq``` repo not existing, re-run

```bash
./gradlew -Pcluster=ins configureClient
$HELM_HOME has been configured at /root/.helm.
Not installing Tiller due to 'client-only' flag having been set
Happy Helming!
"vantiq" has been added to your repositories

BUILD SUCCESSFUL in 7s
2 actionable tasks: 2 executed
```

##### (optional) Setup private helm for Vantiq to skip update from internet

```bash
cp modified {build,settings}.gradle ~/k8sdeploy_tools/
```

```bash
mkdir -p ~/helm_local    # for the mounted path from the below, where private charts copied to
```

```bash
docker run -it —rm vantiq/helm-kubectl:2.11.0 bash    # get into docker container

ip route show | awk ‘/default/ {print $3}’    # the output = 172.17.0.1    # run inside the container
```

```bash
docker run -d —name chart-repo -p 8879:8879 -v /home/ubuntu/helm_local/charts/:/charts \
  -h charts vantiq/helm-kubectl:2.11.0 helm serve \
  —address charts:8879 —repo-path /charts \
  —url http://172.17.0.1:8879
```

```bash
docker logs chart-repo    # check logs
```

##### Update stable repo in Helm (optional)

The purpose of doing this is to avoid time-out connection attempt to googleapis.com which is blocked by firewall

```bash
k8s_master:~/k8sdeploy_tools$ ./helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts                    
vantiq	https://vantiq.github.io/k8sdeploy/charts       

k8s_master:~/k8sdeploy_tools$ ./helm repo remove stable
"stable" has been removed from your repositories

k8s_master:~/k8sdeploy_tools$ ./helm repo list
NAME  	URL                                      
local 	http://127.0.0.1:8879/charts             
vantiq	https://vantiq.github.io/k8sdeploy/charts

k8s_master:~/k8sdeploy_tools$ ./helm repo add stable http://mirror.azure.cn/kubernetes/charts/
"stable" has been added to your repositories

k8s_master:~/k8sdeploy_tools$ ./helm repo list
NAME  	URL                                      
local 	http://127.0.0.1:8879/charts             
vantiq	https://vantiq.github.io/k8sdeploy/charts
stable	http://mirror.azure.cn/kubernetes/charts/

k8s_master:~/k8sdeploy_tools$ ./helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "vantiq" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

#### SSL, if self-signed

##### Generate ```key.pem``` and ```cert.pem```

```bash
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.pem -out cert.pem
```

```bash
Org Name: fqdn.com
Org Unit Name: fqdn.com
Common Name (e.g. server FQDN): eda2.fqdn.com
```

```bash
openssl req -new -x509 -nodes -out ingress.crt -keyout ingress.key -days 9999 \
  -subj /CN=hostname.fqdn.com
```

```bash
cp {key,cert}.pem ~/k8sdeploy_tools/targetCluster/deploy/vantiq/certificates/
cp {key,cert}.pem ~/k8sdeploy_tools/targetCluster/deploy/certificates/
```

##### Modify, pay attention to FQDN's value

- ```~/k8sdeploy_tools/targetCluster/deploy.yaml```
- ```~/k8sdeploy_tools/targetCluster/cluster.properties```

#### Start Install

###### ```deployNginx```, and ```deployShared```
```bash
./gradlew -Pcluster=jd repoUpdate
./gradlew -Pcluster=jd deployNginx
./gradlew -Pcluster=jd deployShared
```

##### The pattern of creating mount-point
* ```sudo mkdir -p /mnt/disks-by-id/disk0```
* ```sudo fdisk /dev/vdX``` > create partition then commit
* ```sudo mkfs.ext4 -j /dev/vdX1```
* ```sudo mount /dev/vdX1 /mnt/disks-by-id/disk0```

The sequence to mount:
* 50G for ```grafanadb-mysql```
* 160G for ```influxdb-influxdb```  # at least 156G or failed
* 50G for ```grafana```, on worker which perhaps needs to ```docker pull grafana/grafana:5.4.3```

#####  ```deployVantiq```
```bash
./gradlew -Pcluster=jd deployVantiq
```
* Find workers that have 2* 520G block disks  # at least 520G each
* ```sudo mkdir -p /mnt/disks-by-id/disk0```
* ```sudo fdisk /dev/vdX```
* ```sudo mount /dev/vdX1 /mnt/disks-by-id/disk0```

## Appendix
##### ```/etc/apt/apt.conf.d/proxy```

```bash
Acquire::http::Proxy "http://ip:3128"
Acquire::https::Proxy "http://ip:3128"
```

##### ```~/.curlrc```
```bash
proxy = ip:3128
```

##### ```/etc/systemd/system/docker.service.d/http-proxy.conf```
```bash
[Service]
Environment="HTTP_PROXY=http://up:3128/"
```

##### ```/etc/systemd/system/docker.service.d/override.conf```

This file is generated by ```systemctl edit docker``` automatically
```bash
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

##### ```~/k8sdeploy_tools/.gradle/gradle.properties```

```bash
systemProp.http.proxyHost=ip
systemProp.http.proxyPost=3128
systemProp.https.proxyHost=ip
systemProp.https.proxyPost=3128

gitUsername=me
gitPassword=token
```

##### ```docker image ls```

Up to 20190717, the latest used images

```bash
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
quay.io/kubernetes-ingress-controller/nginx-ingress-controller   0.23.0              a3f21ec4bd11        9 months ago        513MB
quay.io/prometheus/node-exporter                                 v0.15.2             ff5ecdcfc4a2        19 months ago       22.8MB
telegraf                                                         1.9.4-alpine        07140bfde316        5 months ago        74.1MB
vantiq/helm-kubectl                                              2.11.0              a92e4902c4bb        5 months ago        166MB
vantiq/vantiq-server                                             1.25.13             ed619139fc57        6 weeks ago         622MB
```

##### SSL Keys in K8S

- List in ```default``` namespace
```bash
kubectl -n default get secrets
```

- Describe the secret, named ```vantiq-cert``` in ```default``` namespace

```bash
kubectl -n default get secret vantiq-cert -o yaml
```

Re-generate secret by using new {cert,key}.pem files

```bash
kubectl -n default create secret tls --cert cert.pem --key key.pem \
  vantiq-cert --dry-run -o yaml | kubectl apply -f -
```

Reference > https://developer.ibm.com/recipes/tutorials/changing-the-tls-certificate-in-ingress-in-information-server-kubernetes-deployments/

In case, you need to create a CSR
```bash
openssl req -new -newkey rsa:2048 -nodes -keyout ingress.key -out ingress.csr
```

Convert p12 to pem
```bash
openssl pkcs7 -print_certs -in certificate.p7b -out ingress.pem  
```

To inspect cert.pem
```bash
openssl x509 -in ingress.pem -text -noout
```

##### Add a DNS entry in lb-nginx-ingress-controller

```bash
kubectl -n default edit daemonsets.apps lb-nginx-ingress-controller
```

Sample

```bash
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

##### Product Reinstall Steps

In case you want to undeploy/ re-deploy the product, here are the steps
1. undeployVantiq > undeployShared > undeployNginx > ```./gradlew -Pcluster=<cluster_name> clean```
2. Go to each of mounted disks under ```/mnt/disks-by-id/diskX```
    - Delete all in those path, then umount # You understand what you're doing
    - ```kubectl get pv``` > ```kubectl delete pv local-pv-xxx```  # assume all PVs belong to this product. And you know what you're doing
3. deployNginx > deployShared
    - Disk mount sequence > 50G (grafana mysql) > 150G (influx) > 50G (mysql)
    - You'd see ```Bound``` status in ```kubectl get pv```
4. deployVantiq
    - mount 2* 530G disks

##### Alternative Repos in China

Reference > https://github.com/Azure/container-service-for-azure-china/tree/master/aks

- Ubuntu APT
  https://mirrors.aliyun.com/docker-ce/linux/ubuntu for https://download.docker.com/linux/ubuntu
  https://mirrors.aliyun.com/kubernetes/apt/ for https://apt.kubernetes.io/


  Reference > https://www.ilanni.com/?p=14534

- Install docker-ce
  ```bash
  apt-get update && apt-get install -y apt-transport-https ca-certificates curl
  curl -s http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
  cat <<EOF >/etc/apt/sources.list.d/docker-ce.list
  deb [arch=amd64] https://mirror.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu xenial stable
  EOF
  ```


Alternative registry from ```azk8s.cn``` from https://github.com/Azure/container-service-for-azure-china/tree/master/aks

> Reference > http://mirror.azure.cn/help/gcr-proxy-cache.html

  | global | proxy in China | format | example |
  | ---- | ---- | ---- | ---- |
  | [dockerhub](hub.docker.com) (docker.io) | [dockerhub.azk8s.cn](http://mirror.azk8s.cn/help/docker-registry-proxy-cache.html) | `dockerhub.azk8s.cn/<repo-name>/<image-name>:<version>` | `dockerhub.azk8s.cn/microsoft/azure-cli:2.0.61` `dockerhub.azk8s.cn/library/nginx:1.15` |
  | gcr.io | [gcr.azk8s.cn](http://mirror.azk8s.cn/help/gcr-proxy-cache.html) | `gcr.azk8s.cn/<repo-name>/<image-name>:<version>` | `gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.13.5` |
  | quay.io | [quay.azk8s.cn](http://mirror.azk8s.cn/help/quay-proxy-cache.html) | `quay.azk8s.cn/<repo-name>/<image-name>:<version>` | `quay.azk8s.cn/deis/go-dev:v1.10.0` |
  | https://kubernetes-charts.storage.googleapis.com | http://mirror.azure.cn/kubernetes/charts/ |  | `helm repo add stable http://mirror.azure.cn/kubernetes/charts/`

  Example
  ```bash
  docker pull quay.azk8s.cn/kubernetes-ingress-controller/nginx-ingress-controller:0.23.0
  ```

##### Check logs and use labels

```bash
kubectl -n kube-system logs -l k8s-app=kube-dns
```

##### List K8S image sha256 digest

```bash
kubectl -n knative-eventing get pod $POD -o json \
  | jq '.status.containerStatuses[] | { "image": .image, "imageID": .imageID }'
```

##### List Docker image sha256 digest

```bash
docker inspect --format='{{index .RepoDigests 0}}' $IMAGE
```

##### K8S GUI

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```

```bash
ubuntu@vantiq2-test01:~/cluster_resource.yaml$ kubectl -n kubernetes-dashboard get all
NAME                                              READY   STATUS    RESTARTS   AGE
pod/kubernetes-dashboard-6f89577b77-4fldr         1/1     Running   0          8m6s
pod/kubernetes-metrics-scraper-79c9985bc6-rg84m   1/1     Running   0          8m6s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/dashboard-metrics-scraper   ClusterIP   10.96.32.200    <none>        8000/TCP   8m6s
service/kubernetes-dashboard        ClusterIP   10.111.152.50   <none>        443/TCP    8m6s

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubernetes-dashboard         1/1     1            1           8m6s
deployment.apps/kubernetes-metrics-scraper   1/1     1            1           8m6s

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/kubernetes-dashboard-6f89577b77         1         1         1       8m6s
replicaset.apps/kubernetes-metrics-scraper-79c9985bc6   1         1         1       8m6s

NAME                                                       READY   REASON   AGE
clusterchannelprovisioner.eventing.knative.dev/in-memory   True             33d
```

#### Helm install private repo (contributed by Jun Zou)

```bash
helm install --name releasename local_path --debug --timeout 6000
```
