<!--
<img src=../imgs/vantiq_logo2.png width=200px align="right">
<br>
<br>
-->

# Deploy Kubernetes on Private OpenStack Cloud

<!-- TOC depthFrom:2 depthTo:4 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Existing Environment](#existing-environment)
- [Security Hardening](#security-hardening)
		- [Operating System on Virtual Machine](#operating-system-on-virtual-machine)
		- [Update Hostname](#update-hostname)
		- [Turn-on Firewall](#turn-on-firewall)
		- [Harden SSHd, then ```systemctl restart sshd.service```](#harden-sshd-then-systemctl-restart-sshdservice)
		- [Create a non-root user and grant it ```sudo``` permission](#create-a-non-root-user-and-grant-it-sudo-permission)
- [Update other Nodes in the Environment](#update-other-nodes-in-the-environment)
		- [Network Topology](#network-topology)
		- [Update ```/etc/hosts``` on 1st host](#update-etchosts-on-1st-host)
		- [Setup all hostnames by using ```hostnamectl set-hostname```](#setup-all-hostnames-by-using-hostnamectl-set-hostname)
		- [Setup network to allow other hosts go internet](#setup-network-to-allow-other-hosts-go-internet)
- [Docker](#docker)
		- [Install](#install)
		- [Use China local image repo. Modify ```/etc/docker/daemon.json```](#use-china-local-image-repo-modify-etcdockerdaemonjson)
		- [Grant non-root to control Docker](#grant-non-root-to-control-docker)
- [Kubernetes](#kubernetes)
		- [Using China local repo for install](#using-china-local-repo-for-install)
		- [Pull images while using local repo](#pull-images-while-using-local-repo)
		- [Start K8S by root](#start-k8s-by-root)
		- [Grant non-root user to control K8S](#grant-non-root-user-to-control-k8s)
		- [Check the status by non-root user](#check-the-status-by-non-root-user)
		- [Install ```flannel``` network](#install-flannel-network)
		- [Install and configure Docker and K8S on other hosts](#install-and-configure-docker-and-k8s-on-other-hosts)
- [Appendix](#appendix)
		- [Output of ```kubeadm init```](#output-of-kubeadm-init)

<!-- /TOC -->

## Existing Environment

CPU 核 | Memory (G) 内存 | OS Disk (G) | IP | External Disk (G) 外挂磁盘
-- | -- | -- | -- | --
4 | 16 | 40 | 10.100.100.11 |
4 | 16 | 40 | 10.100.100.12 | 500 (MongoDB)
4 | 16 | 40 | 10.100.100.13 | 500 (MongoDB)
4 | 16 | 40 | 10.100.100.14 | 10 (Grafana)/ 10 (GrafanaDB)/ 150 (InfluxDB) 分别生成，共3个
4 | 16 | 40 | 10.100.100.15 |
4 | 16 | 40 | 10.100.100.16 |

## Security Hardening

#### Operating System on Virtual Machine

```
root@vantiq01:~# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=18.04
DISTRIB_CODENAME=bionic
DISTRIB_DESCRIPTION="Ubuntu 18.04.1 LTS"
```

#### Update Hostname
```
hostnamectl set-hostname vantiq01
```

#### Turn-on Firewall
```
ufw allow ssh
ufw enable
```

#### Harden SSHd, then ```systemctl restart sshd.service```
```
PermitRootLogin prohibit-password

PasswordAuthentication no
PermitEmptyPasswords no
```

#### Create a non-root user and grant it ```sudo``` permission
```
adduser ubuntu
usermod -aG sudo ubuntu
```

## Update other Nodes in the Environment

#### Network Topology
We're given total 6 virtual machines and one of 6 has access to internet

<img src="../imgs/20190327_cp_nw_top.png">

#### Update ```/etc/hosts``` on 1st host
```
10.100.100.11	vantiq01	vantiq01
10.100.100.12	vantiq02	vantiq02
10.100.100.13	vantiq03	vantiq03
10.100.100.14	vantiq04	vantiq04
10.100.100.15	vantiq05	vantiq05
10.100.100.16	vantiq06	vantiq06
```

#### Setup all hostnames by using ```hostnamectl set-hostname```

In Bash shell, run
```
for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem "hostnamectl set-hostname vantiq0$i"; done
```

```
for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem "hostname"; done
```

#### Setup network to allow other hosts go internet
Since the 1st host is the only one allowed to access internet and others are off the world, here are the steps to route network requests from other hosts through the 1st one to internet

On the 1st host as ```gateway```, use ```POSTROUTING``` rule to ```MASQUERADE``` all traffic from ```10.100.100.0/24``` pass through ```10.100.100.11``` to reach outside. If ```ufw``` is on, need to define a rule for incoming traffic from ```10.100.100.0/24```

```
iptables -t nat -A POSTROUTING -s 10.100.100.0/24 -j MASQUERADE
ufw allow in on eth0 from 10.100.100.0/24
```

Update 1st host's ```/etc/default/ufw``` configuration to allow forward, then ```systemctl restart ufw.service```
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

On other hosts
```
ip r add default via 10.100.100.11 dev eth0
```

> Note: you might have to delete the pre-configured default gateway. Caution: be careful when you do this and you know what exactly you're doing!

## Docker

#### Install
> Reference > https://docs.docker.com/install/linux/docker-ce/ubuntu/

```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce
```

#### Use China local image repo. Modify ```/etc/docker/daemon.json```
> Reference > https://www.docker-cn.com/registry-mirror
```
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

#### Using ```systemd``` to control the Docker daemon
Overriding Defaults for the Docker Daemon

```
sudo systemctl edit docker
```

The above command generates ```/etc/systemd/system/docker.service.d``` and ```override.conf``` under it

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

```
systemctl daemon-reload
systemctl restart docker.service
```

Bonus: if you prefer ```vi``` as default editor on Ubuntu, run ```update-alternatives --config editor```

#### Grant non-root to control Docker
```
sudo gpasswd -a $USER docker
```

## Kubernetes

> Reference > https://kubernetes.io/docs/reference/kubectl/cheatsheet/

#### Using China local repo for install
```
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

#### Pull images while using local repo (optional if you don't have this challenge)

```
images=(
    kube-apiserver:v1.13.4
    kube-controller-manager:v1.13.4
    kube-scheduler:v1.13.4
    kube-proxy:v1.13.4
    pause:3.1
    etcd:3.2.24
    coredns:1.2.6
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

#### Start K8S by root

Check network pre-requisite before proceeding
> Reference > https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md
and https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

#### Grant non-root user access to control K8S
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- ```kubectl``` AutoComplete
```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### Check the status by non-root user
```
kubectl get pods -o wide --all-namespaces
```

And the expected output looks like
```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
kube-system   coredns-86c58d9df4-k9s4z           0/1     Pending   0          12m   <none>          <none>     <none>           <none>
kube-system   coredns-86c58d9df4-nqs4d           0/1     Pending   0          12m   <none>          <none>     <none>           <none>
kube-system   etcd-vantiq01                      1/1     Running   0          11m   10.100.100.11   vantiq01   <none>           <none>
kube-system   kube-apiserver-vantiq01            1/1     Running   0          11m   10.100.100.11   vantiq01   <none>           <none>
kube-system   kube-controller-manager-vantiq01   1/1     Running   0          10m   10.100.100.11   vantiq01   <none>           <none>
kube-system   kube-proxy-4dghc                   1/1     Running   0          12m   10.100.100.11   vantiq01   <none>           <none>
kube-system   kube-scheduler-vantiq01            1/1     Running   0          11m   10.100.100.11   vantiq01   <none>           <none>
```

#### Install ```flannel``` network
> Reference > https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

- Pre-requisite
	- For flannel to work correctly, you MUST pass ```--pod-network-cidr=10.244.0.0/16``` to ```kubeadm init```.
	- Set ```/proc/sys/net/bridge/bridge-nf-call-iptables``` to ```1``` by running ```sysctl net.bridge.bridge-nf-call-iptables=1``` to pass bridged IPv4 traffic to iptables’ chains. ...

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

- After install, check status
```
kubectl get pods --all-namespaces
```

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-d56gz           1/1     Running   1          91m
kube-system   coredns-86c58d9df4-vkshj           1/1     Running   1          91m
kube-system   etcd-vantiq01                      1/1     Running   0          90m
kube-system   kube-apiserver-vantiq01            1/1     Running   0          90m
kube-system   kube-controller-manager-vantiq01   1/1     Running   0          90m
kube-system   kube-flannel-ds-amd64-kc8m2        1/1     Running   0          6m10s
kube-system   kube-proxy-sbsp6                   1/1     Running   0          91m
kube-system   kube-scheduler-vantiq01            1/1     Running   0          90m
```

#### Install and Configure Docker and K8S on other hosts

- Instasll Docker

```
for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem "apt install apt-transport-https ca-certificates curl software-properties-common"; done

for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem "curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -"; done

for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"'; done

for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'hostname; apt install docker-ce'; done
```

> Note: you should NOT pass ```-y``` when installing to prevent automatically upgrade without your permission

- Check ```docker --version```

```
for i in {1..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'hostname; docker --version'; done
```

- Copy ```/etc/docker/daemon.json``` from host01 to all other hosts then restart docker
```
for i in {2..6}; do scp -i ~/.ssh/Vantiq-key.pem /etc/docker/daemon.json root@vantiq0$i:/etc/docker/; done
for i in {1..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'hostname; systemctl restart docker.service'; done
```

- Copy ```/etc/apt/sources.list.d/kubernetes.list``` to other hosts
```
for i in {2..6}; do scp -i ~/.ssh/Vantiq-key.pem kubernetes.list root@vantiq0$i:/etc/apt/sources.list.d/; done
for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -'; done
for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'apt update'; done
```

- Install K8S
```
for i in {2..6}; do ssh root@vantiq0$i -i ~/.ssh/Vantiq-key.pem 'apt install kubelet kubeadm kubectl'; done
```

- Modify ```ufw``` on 1st host
```
sudo ufw allow from 10.100.100.0/24 to any port 6443
```

where ```10.100.100.0``` is the network in which K8S resides and port ```6443``` is for cluster

- Join K8S cluster




## Appendix

#### Output of ```kubeadm init```
```
root@vantiq01:~# kubeadm init --pod-network-cidr=10.244.0.0/16
I0326 20:17:20.716172   24509 version.go:94] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://storage.googleapis.com/kubernetes-release/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I0326 20:17:20.716238   24509 version.go:95] falling back to the local client version: v1.13.4
[init] Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.3. Latest validated version: 18.06
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [vantiq01 localhost] and IPs [10.100.100.11 127.0.0.1 ::1]
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [vantiq01 localhost] and IPs [10.100.100.11 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [vantiq01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.100.100.11]
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 17.501788 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "vantiq01" as an annotation
[mark-control-plane] Marking the node vantiq01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node vantiq01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: jx059c.wrud9vy63apx8lg3
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.100.100.11:6443 --token jx059c.wrud9vy63apx8lg3 --discovery-token-ca-cert-hash sha256:1af79ce9f381c5dec72b0d9df0e68ad6d91641ae14e823360d46646b187fd491
```
