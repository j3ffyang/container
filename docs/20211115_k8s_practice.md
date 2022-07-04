# Kubernetes Production Deployment and Operation

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Executive Summary](#executive-summary)
- [Pre-requisites for Production](#pre-requisites-for-production)
    - [Environment Pre-requisite](#environment-pre-requisite)
    - [Network Access Pre-requisite:](#network-access-pre-requisite)
- [Security and System Hardening on bastionHost](#security-and-system-hardening-on-bastionhost)
    - [Non-root User](#non-root-user)
    - [OpenSSH](#openssh)
    - [Firewall by `firewalld`](#firewall-by-firewalld)
- [Environment Preparation](#environment-preparation)
    - [`/etc/hosts` on bastionHost](#etchosts-on-bastionhost)
    - [Create `ubuntu` (nonroot) User on other VMs](#create-ubuntu-nonroot-user-on-other-vms)
    - [Add SSH key into both `root` and `ubuntu` accounts](#add-ssh-key-into-both-root-and-ubuntu-accounts)
    - [Create `ubuntu` user on each of VMs and add ssh keys](#create-ubuntu-user-on-each-of-vms-and-add-ssh-keys)
    - [Hardware Spec in Cluster](#hardware-spec-in-cluster)
- [HAProxy and keepAlived for multiple Kubernetes Multiple Kubernets Master Nodes](#haproxy-and-keepalived-for-multiple-kubernetes-multiple-kubernets-master-nodes)
- [Ansible for Docker and Kubernetes](#ansible-for-docker-and-kubernetes)
    - [Pre-requisite](#pre-requisite)
    - [Install: `ansible`, `python3`, and `pip3`](#install-ansible-python3-and-pip3)
    - [Clone the code](#clone-the-code)
    - [Prepare the env for Ansible](#prepare-the-env-for-ansible)
    - [Tune the configuration files up to your choice](#tune-the-configuration-files-up-to-your-choice)
    - [Run Ansible](#run-ansible)
    - [Verify `etcd` cluster](#verify-etcd-cluster)
    - [Verify `calico`](#verify-calico)
    - [Verify `kubectl` and AutoComplete (optional)](#verify-kubectl-and-autocomplete-optional)
    - [Add `ubuntu` user into `docker` group  (optional)](#add-ubuntu-user-into-docker-group-optional)
    - [Reset with using Ansible](#reset-with-using-ansible)
- [Disk and Volume](#disk-and-volume)
    - [Disks allocation](#disks-allocation)
    - [Format disk for MongoDB in `xfs`](#format-disk-for-mongodb-in-xfs)
    - [Mount and add mount entry in `/etc/fstab`](#mount-and-add-mount-entry-in-etcfstab)
    - [Format other 3 disks in `ext4`, then mount](#format-other-3-disks-in-ext4-then-mount)
- [Sample Application](#sample-application)
    - [Deployment Architecture](#deployment-architecture)
    - [SSL certificates and private key](#ssl-certificates-and-private-key)
- [Operation](#operation)
    - [`etcd` Backup](#etcd-backup)
    - [Kubernetes Backup `/etc/kubernetes/pki`](#kubernetes-backup-etckubernetespki)
    - [MongoDB Backup with Kubernetes Cronjob](#mongodb-backup-with-kubernetes-cronjob)
    - [Cronjob to delete old MongoDB dump files](#cronjob-to-delete-old-mongodb-dump-files)
    - [Export and Import `collections` from MongoDB](#export-and-import-collections-from-mongodb)
    - [Force MongoDB member to become primary](#force-mongodb-member-to-become-primary)
    - [Reschedule a Pod to another node](#reschedule-a-pod-to-another-node)
    - [List all selected resource per worker node in Kubernetes cluster](#list-all-selected-resource-per-worker-node-in-kubernetes-cluster)
- [Customization](#customization)
    - [Apply a customized theme for KeyCloak by `kustomize `](#apply-a-customized-theme-for-keycloak-by-kustomize)
- [Troubleshooting](#troubleshooting)
    - [`No data` for `nginx` and `mongodb` in Grafana](#no-data-for-nginx-and-mongodb-in-grafana)
- [Appendix](#appendix)
    - [MongoDB Backup and Restore with Docker](#mongodb-backup-and-restore-with-docker)

<!-- /code_chunk_output -->

<div style="page-break-after: always;"></div>

## Executive Summary

Document created: 20210830

Document changeLog

action | date | details
-- | -- | --
init | 20210830 |

## Pre-requisites for Production

#### Environment Pre-requisite

- An SMTP account that is used to send out notification email upon registration and password-reset request automatically
- A fully qualified domain name (FQDN) valid on internet, eg. eventdriven.sample.com
- 1 public floating IP and at least 1 public IPs for one or two up-front Nginx web server(s)
- 1 intranet floating IP that is used for high availability of Nginx web server
- (optional) An official certificate and key for FQDN for both up-front Nginx web server and sampleApp. If this is unavailable, a pair of self-signed certificate and key will be introduced, which doesn't reduce security but receiving SSL warning each time when browsing the service

#### Network Access Pre-requisite:

from | to | allowed
-- | -- | --
internet | bastionHost | `22/TCP`
nginx | sampleApp | all & no restriction
sampleApp | nginx | all & no restriction
sampleApp | internet | all & no restriction
internet | sampleApp | none

<div style="page-break-after: always;"></div>

## Security and System Hardening on bastionHost

#### Non-root User
- Create a non-root user: `ubuntu`
- Copy SSH public key to remove VMs
- Make `ubuntu` user in `sudoer` group

```sh
usermod -aG sudo ubuntu
```

#### OpenSSH

- Edit `/etc/ssh/sshd_config`. Disable passwd login, force public key login and disable X11Forwarding. Restart `sshd.service` to apply the change

```sh
# PermitRootLogin prohibit-password
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
```

- Create SSH key

```sh
ssh-keygen -t rsa
```

#### Firewall by `firewalld`

```sh
ubuntu@bastionhost:~$ sudo firewall-cmd --permanent --zone=public --add-port=22/tcp
success
ubuntu@bastionhost:~$ sudo firewall-cmd --reload
success
ubuntu@bastionhost:~$ sudo firewall-cmd --list-all
public
  target: default
  icmp-block-inversion: no
  interfaces:
  sources:
  services: dhcpv6-client ssh
  ports: 22/tcp
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```


## Environment Preparation


#### `/etc/hosts` on bastionHost

```sh
for i in {11..24}; do echo "10.0.10.$i    node$i" | tee -a /etc/hosts; done
```
Then edit specific hostName as you want to make `/etc/hosts` looks like eventually

```sh
ubuntu@bastionhost:/tmp$ cat /etc/hosts
127.0.0.1	localhost

# The following lines are desirable for IPv6 capable hosts
::1	localhost	ip6-localhost	ip6-loopback
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
127.0.1.1	localhost.vm	localhost
127.0.1.1	bastionhost	bastionhost

10.0.10.11    node11    sampleApp1
10.0.10.12    node12    sampleApp2
10.0.10.13    node13    sampleApp3
10.0.10.14    node14    k8smaster1
10.0.10.15    node15    k8smaster2
10.0.10.16    node16    k8smaster3
10.0.10.17    node17    metrics-collector
10.0.10.18    node18    mongo1
10.0.10.19    node19    mongo2
10.0.10.20    node20    mongo3
10.0.10.21    node21    influx
10.0.10.22    node22    keycloak3
10.0.10.23    node23    keycloak1
10.0.10.24    node24    keycloak2
```

#### Create `ubuntu` (nonroot) User on other VMs

```sh
for i in {11..24}; do ssh root@node$i "adduser ubuntu"; done

# Update ``ubuntu`` user pass
for i in {11..24}; do ssh root@node$i "hostname; passwd ubuntu"; done
```

#### Add SSH key into both `root` and `ubuntu` accounts

```sh
for i in {11..24}; do ssh-copy-id root@node$i; done
```
> For security reason: the key would be removed after installation later, to disable root access on other VMs

You could verify whether ssh public key has been copied properly

```sh
for i in {11..24}; do ssh root@node$i "hostname"; done
```

#### Create `ubuntu` user on each of VMs and add ssh keys

```sh
for i in {11..24}; do ssh-copy-id ubuntu@node$i; done

# Add ubuntu user in sudo group
for i in {11..24}; do ssh root@node$i "usermod -aG sudo ubuntu"; done

# Verify
for i in {11..24}; do ssh ubuntu@node$i "hostname"; done
```

#### Hardware Spec in Cluster

- `/proc/cpuinfo`
```sh
ubuntu@bastionhost:~/$ for i in {11..24}; do ssh ubuntu@node$i "hostname; cat /proc/cpuinfo | grep processor | wc"; done
sampleServer-1
      4      12      56
sampleServer-2
      4      12      56
sampleServer-3
      4      12      56
K8Master-1
      2       6      28
K8Master-2
      2       6      28
k8Master-3
      2       6      28
Metrics-collector1
      4      12      56
MongoDBServer-1
      4      12      56
MongoDBServer-2
      4      12      56
MongoDBServer-3
      4      12      56
monitoring-server-influxdb-1
      4      12      56
keycloak-server-nginx-grafana-mysql-1
      2       6      28
Keycloak-Server-Nginx-1
      2       6      28
Keycloak-Server-Nginx-2
      2       6      28
```

- `/proc/meminfo`

```sh
ubuntu@bastionhost:~/$ for i in {11..24}; do ssh ubuntu@node$i "hostname; cat /proc/meminfo | grep MemTotal"; done
sampleServer-1
MemTotal:        8152404 kB
sampleServer-2
MemTotal:        8152404 kB
sampleServer-3
MemTotal:        8152412 kB
K8Master-1
MemTotal:        4030220 kB
K8Master-2
MemTotal:        4030220 kB
k8Master-3
MemTotal:        4030220 kB
Metrics-collector1
MemTotal:        8152404 kB
MongoDBServer-1
MemTotal:       16397656 kB
MongoDBServer-2
MemTotal:       16397656 kB
MongoDBServer-3
MemTotal:       16397656 kB
monitoring-server-influxdb-1
MemTotal:       16397656 kB
keycloak-server-nginx-grafana-mysql-1
MemTotal:        4030228 kB
Keycloak-Server-Nginx-1
MemTotal:        4030220 kB
Keycloak-Server-Nginx-2
MemTotal:        4030220 kB
```

<div style="page-break-after: always;"></div>

## HAProxy and keepAlived for multiple Kubernetes Multiple Kubernets Master Nodes

> Note: since a floatIP is provided from Huawei Cloud platform, as the main entrance of multiple Kubernetes master nodes, this configuration won't be necessary.

The below diagram is drawn in `plantuml` format. It can be displayed in Atom or Sublime with `plantuml` plugin enabled.

```plantuml
' skinparam handwritten true

node haproxy {

package 10.0.10.15 {
  agent keepalived1
  agent haproxy1
  keepalived1 -[hidden]-> haproxy1
}

package 10.0.10.14 {
  agent keepalived0
  agent haproxy0
  keepalived0 -[hidden]-> haproxy0
}

agent vip_lb
note right of vip_lb: "10.0.10.200:8383"

haproxy0 -- vip_lb
haproxy1 -- vip_lb

10.0.10.14 -[hidden]> 10.0.10.15
}

node k8sMaster {
  package master2 {
    agent apiServer2
    agent scheduler2
    agent controllerManager2
    apiServer2 -[hidden]d-> scheduler2
    scheduler2 -[hidden]d-> controllerManager2

    note right of master2: 10.0.10.15:6443"
  }

    package master1 {
    agent apiServer1
    agent scheduler1
    agent controllerManager1
    apiServer1 -[hidden]d-> scheduler1
    scheduler1 -[hidden]d-> controllerManager1
    note right of master1: 10.0.10.14:6443"

  }

  package master3 {
    agent apiServer3
    agent scheduler3
    agent controllerManager3
    apiServer3 -[hidden]d-> scheduler3
    scheduler3 -[hidden]d-> controllerManager3

    note right of master3: 10.0.10.16:6443"
  }

  master1 -[hidden]right-> master2
  master2 -[hidden]right-> master3
}

vip_lb -d-> apiServer3
vip_lb -d-> apiServer1
vip_lb -d-> apiServer2

node etcdCluster {
  package etcd2 {
    note right of etcd2: 10.0.10.15:2379"    
  }

  package etcd3 {
    note right of etcd3: 10.0.10.16:2379"    
  }

  package etcd1 {
    note right of etcd1: 10.0.10.14:2379"    
  }

  etcd1 -[hidden]r- etcd2
  etcd2 -[hidden]r- etcd3

}

k8sMaster -[hidden]> etcdCluster

controllerManager1 -[hidden]down- etcd1
controllerManager2 -[hidden]down- etcd2
controllerManager3 -[hidden]down- etcd3
```

Description:

- `haproxy` and `keepalived` are installed on `10.0.10.14` and `10.0.10.15` respectively
- `10.0.10.200` is the floating virtual IP between `10.0.10.14` and `10.0.10.15` when failover occurs
- Load Balancing (LB) service, bound to `10.0.10.200` over port `8383` (this will be also defined and used in Ansible deployment code as well). Request hitting `10.0.10.200` over port `8383` will be forwarded to those 3 Kubernetes master nodes (`10.0.10.14/15/16`) over port `6443`


> Note: `etcd` cluster will be created by Ansible later

> Reference >
> https://kubesphere.io/docs/installing-on-linux/high-availability-configurations/set-up-ha-cluster-using-keepalived-haproxy/
> https://kubespray.io/#/docs/ha-mode

- Install

```sh
for i in {14..15}; do ssh root@node$i "apt install keepalived haproxy -y"; done
```

- `/etc/haproxy/haproxy.cfg` on `10.0.10.14` and `10.0.10.15`

They're identical on 2 HAproxy machines

```sh
ubuntu@node1:~$ cat /etc/haproxy/haproxy.cfg
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend kube-apiserver
  bind *:8383
  mode tcp
  option tcplog
  default_backend kube-apiserver

backend kube-apiserver
    mode tcp
    option tcplog
    option tcp-check      
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server kube-apiserver-1 10.0.10.14:6443 check # Replace the IP address with your own.
    server kube-apiserver-2 10.0.10.15:6443 check # Replace the IP address with your own.
    server kube-apiserver-3 10.0.10.16:6443 check # Replace the IP address with your own.
```

- `/etc/keepalived/keepalived.conf` on `10.0.10.14`

`unicast_src_ip` = the current machine
`unicast_peer`   = the remote peer(s). There could be multiple peers

```sh
ubuntu@node1:~$ cat /etc/keepalived/keepalived.conf
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}

vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface eth0                # Network card
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 10.0.10.14	# The IP address of this machine
  unicast_peer {
    10.0.10.15			# The IP address of peer machines
  }

  virtual_ipaddress {
    10.0.10.200/24              # The VIP address
  }

  track_script {
    chk_haproxy
  }
}
```

- Restart `haproxy` and `keepalived`

```sh
systemctl restart haproxy keepalived
systemctl enable haproxy keepalived
```

- Verify

```sh
systemctl stop haproxy
```

This will force the floatIP floating to the peer of `haproxy`

> Reference > https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md

<div style="page-break-after: always;"></div>

## Ansible for Docker and Kubernetes

#### Pre-requisite

- Ansible v2.9 and python-netaddr is installed on the machine that will run Ansible commands
- Jinja 2.11 (or newer) is required to run the Ansible Playbooks
- The target servers must have access to the Internet in order to pull docker images. Otherwise, additional configuration is required (See Offline Environment)
- The target servers are configured to allow IPv4 forwarding
- Your ssh key must be copied to all the servers part of your inventory
- The firewalls are not managed, you'll need to implement your own rules the way you used to. in order to avoid any issue during deployment you should disable your firewall
- If kubespray is ran from non-root user account, correct privilege escalation method should be configured in the target servers. Then the ansible_become flag or command parameters --become or -b should be specified

#### Install: `ansible`, `python3`, and `pip3`

```sh
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible-base

sudo apt install -y python3-pip
```

```sh
ubuntu@bastionhost:~/.ssh$ ansible --version
ansible 2.10.11
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/ubuntu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.8/dist-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 3.8.5 (default, Jul 28 2020, 12:59:40) [GCC 9.3.0]
```

#### Clone the code

```sh
git clone https://github.com/kubernetes-sigs/kubespray.git
```

#### Prepare the env for Ansible

```sh
# Install dependencies from ``requirements.txt``
sudo pip3 install -r requirements.txt

# Copy ``inventory/sample`` as ``inventory/mycluster``
cp -rfp inventory/sample inventory/mycluster

# Update Ansible inventory file with inventory builder
# declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
declare -a IPS=(10.0.10.14 10.0.10.15 10.0.10.16 10.0.10.11 10.0.10.12 10.0.10.13 10.0.10.17 10.0.10.18 10.0.10.19 10.0.10.20 10.0.10.21 10.0.10.22 10.0.10.23 10.0.10.24)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review and change parameters under ``inventory/mycluster/group_vars``
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

#### Tune the configuration files up to your choice

Edit `inventory/mycluster/group_vars/all/all.yml`

```sh
## External LB example config
## apiserver_loadbalancer_domain_name: "elb.some.domain"
loadbalancer_apiserver:
    address: 10.0.10.200    # the floatIP for LB from HAproxy
    port: 8383              # must be the same as incoming port of LB

loadbalancer_apiserver_localhost: false
```

and edit `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`

```sh
## Change this to use another Kubernetes version, e.g. a current beta release
kube_version: v1.20.10
kube_network_plugin: calico   # or flannel

## Container runtime
## docker for docker, crio for cri-o and containerd for containerd.
container_manager: docker     # containerd
```

> Reference > https://kubespray.io/#/docs/ha-mode

#### Run Ansible

```sh
pwd
ubuntu@bastionhost:~/kubespray$
ubuntu@bastionhost:~/kubespray$ ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml --extra-vars "ansible_sudo_pass=passSecret"

# or if you don't want to type passSecret in command line
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml --extra-vars --ask-pass --ask-become-pass
```

The script takes about 25 minutes to complete

#### Verify `etcd` cluster

```sh
ubuntu@node1:~/.kube$ sudo etcdctl --key /etc/ssl/etcd/ssl/admin-node1-key.pem --cert /etc/ssl/etcd/ssl/admin-node1.pem endpoint status --write-out=table
+----------------+-----------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT    |       ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------+-----------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 127.0.0.1:2379 | 4cf7fb4a1e9b1c6 |  3.4.13 |   14 MB |     false |      false |         5 |       6863 |               6863 |        |
+----------------+-----------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

ubuntu@node2:~$ sudo etcdctl --key /etc/ssl/etcd/ssl/admin-node1-key.pem --cert /etc/ssl/etcd/ssl/admin-node1.pem endpoint status --write-out=table
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 127.0.0.1:2379 | ce02601469562442 |  3.4.13 |   14 MB |      true |      false |         5 |       6239 |               6239 |        |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

ubuntu@node3:~$ sudo etcdctl --key /etc/ssl/etcd/ssl/admin-node1-key.pem --cert /etc/ssl/etcd/ssl/admin-node1.pem endpoint status --write-out=table
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 127.0.0.1:2379 | 8ca7b7f7a27c4def |  3.4.13 |   14 MB |     false |      false |         5 |       7283 |               7283 |        |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

#### Verify `calico`

```sh
ubuntu@node1:~$ docker ps | grep calico
1f34672aaa34   7aa1277761b5                  "start_runit"            54 minutes ago   Up 54 minutes             k8s_calico-node_calico-node-zm52b_kube-system_be10ff19-aafc-4757-b79f-af8caffe6a8e_0
2dd28d6a7a1b   k8s.gcr.io/pause:3.3          "/pause"                 55 minutes ago   Up 54 minutes             k8s_POD_calico-node-zm52b_kube-system_be10ff19-aafc-4757-b79f-af8caffe6a8e_0
```

#### Verify `kubectl` and AutoComplete (optional)

- Execute the command

```sh
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

- Append the following to `~/.bashrc`

```sh
# alias and auto-completion
alias k=kubectl
complete -F __start_kubectl k
```

#### Add `ubuntu` user into `docker` group  (optional)

Run this on bastionHost

```sh
for i in {11..24}; do ssh root@node$i "usermod -aG docker ubuntu"; done
```

#### Reset with using Ansible

> Caution: This command will destroy the deployed the entire cluster of Docker, Kubernetes and ETCd.

```sh
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root reset.yml --extra-vars "ansible_sudo_pass=passSecret"
```
> Reference> https://github.com/kubernetes-sigs/kubespray/blob/master/reset.yml

<div style="page-break-after: always;"></div>

## Disk and Volume

#### Disks allocation

```sh
ubuntu@bastionhost:~$ for i in {18..22}; do ssh ubuntu@node$i "hostname; lsblk | grep -v vda"; done
node8
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  500G  0 disk
└─vdb1 252:17   0  500G  0 part /mnt/disks-by-id/disk0
node9
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  500G  0 disk
node10
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  500G  0 disk
node11
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  150G  0 disk
node12
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  20G  0 disk
vdc    252:32   0  20G  0 disk
```

#### Format disk for MongoDB in `xfs`

```sh
ubuntu@node8:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    252:0    0   40G  0 disk
└─vda1 252:1    0   40G  0 part /
vdb    252:16   0  500G  0 disk

ubuntu@node8:~$ sudo fdisk /dev/vdb

ubuntu@node8:~$ sudo mkfs.xfs /dev/vdb1 -f
meta-data=/dev/vdb1              isize=512    agcount=4, agsize=32767936 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=131071744, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=63999, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

#### Mount and add mount entry in `/etc/fstab`

```sh
ubuntu@node8:~$ sudo mkdir -p /mnt/disks-by-id/disk0
ubuntu@node8:~$ sudo mount /dev/vdb1 /mnt/disks-by-id/disk0

## Configure `/etc/fstab` and enable auto-mount when VM restarts
ubuntu@node8:~$ cat /etc/fstab
...
/dev/vdb1   /mnt/disks-by-id/disk0 xfs defaults 0 0
```

#### Format other 3 disks in `ext4`, then mount

These 3 disks are for `influxdb`, `grafana` and `grafanaMysql`. After all disks mounted, you'd see

```sh
ubuntu@bastionhost:~$ for i in {18..22}; do ssh ubuntu@node$i "hostname; lsblk | grep -v vda"; done
node8
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  500G  0 disk
└─vdb1 252:17   0  500G  0 part /mnt/disks-by-id/disk0
node9
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  500G  0 disk
└─vdb1 252:17   0  500G  0 part /mnt/disks-by-id/disk0
node10
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  500G  0 disk
└─vdb1 252:17   0  500G  0 part /mnt/disks-by-id/disk0
node11
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  150G  0 disk
└─vdb1 252:17   0  150G  0 part /mnt/disks-by-id/disk0
node12
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vdb    252:16   0  20G  0 disk
└─vdb1 252:17   0  20G  0 part /mnt/disks-by-id/disk0
vdc    252:32   0  20G  0 disk
└─vdc1 252:33   0  20G  0 part /mnt/disks-by-id/disk1
```

> Note: `xfs`​ for all 3 MongoDB instances and `ext4` for others when formatting and defining in ``/etc/fstab​``

---

<div style="page-break-after: always;"></div>

## Sample Application


#### Deployment Architecture

```plantuml

cloud freeWorld

package publicELB {
  collections ELB
}

component bastionHost

package sampleSystem {

  note "everything \nrunning on k8s" as N1

  collections nginx_ingress
  note bottom of nginx_ingress: ingress_controller \non each worker \nfrom k8s

  package sample {
    collections sampleCluster
  }

  package mongo {
    database mongoCluster

    note "primary* 1\nsecondary* 2" as N2
    mongoCluster .[hidden].> N2
  }

  mongoCluster -[hidden]> monitoring

  package monitoring {
    agent grafana
    database influxdb
    database mysql
    grafana --> influxdb
    grafana --> mysql

    note bottom of influxdb: timeSeries\ndata
    note bottom of mysql: grafana\nconfiguration
  }


  nginx_ingress --> sampleCluster

  sampleCluster --> mongoCluster

  sampleCluster --> grafana


  package identity {
    collections keycloak
    database postgresql
    keycloak -- postgresql
  }

  note top of identity: optional in\ndeployment

  sampleCluster <-- keycloak
}

package operationSvc {

  component mongoBackup
  component kubernetesBackup
}


mongoBackup -[hidden]- kubernetesBackup

freeWorld -- publicELB
freeWorld -- bastionHost
bastionHost -- sampleSystem
publicELB --> nginx_ingress

publicELB -[hidden] bastionHost

sampleSystem -.- operationSvc
```

#### SSL certificates and private key

- Certificates of hostSelf, intermediate and CAroot, when bundled in one single __chained__ `cert.pem`

The correct order of bundled certificates is: hostSelf (top) > intermediate cert > CAroot cert

- Validate the chained certificates

```sh
ubuntu@node1:~/k8sdeploy_tools/targetCluster/sensitive$ openssl crl2pkcs7 -nocrl -certfile chain.pem | openssl pkcs7 -print_certs -noout
subject=CN = sub.domain.com

issuer=C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA


subject=C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA

issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority


subject=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

issuer=C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
```

- Verify the imported certificate

```sh
ubuntu@bastionhost:~$ openssl s_client -showcerts -connect 10.0.10.18:32160
CONNECTED(00000003)
Can't use SSL_get_servername
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA
verify return:1
depth=0 CN = sub.domain.com
verify return:1
---
Certificate chain
 0 s:CN = sub.domain.com
   i:C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA
-----BEGIN CERTIFICATE-----
MIIGSTCCBTGgAwIBAgIQQ9ytVSnDH7HqTBzJ3ZYdxTANBgkqhkiG9w0BAQsFADCB
...
```

Where `10.0.10.18` is one of the worker node with `nginx-ingress-controller` deployed and `32160` is `nodePort` configured in `service/nginx-ingress-controller`

```sh
ubuntu@node1:~/k8sdeploy_tools/$ kubectl -n shared get svc | grep nginx-ingress-controller
nginx-ingress-controller           NodePort    10.233.33.142   <none>        80:31500/TCP,443:32160/TCP   8d
```

The output of `openssl s_client -showcerts` should be _exactly_ the same as the above result from `openssl crl2pkcs7`

> Reference > https://medium.com/@superseb/get-your-certificate-chain-right-4b117a9c0fce

---

<div style="page-break-after: always;"></div>

## Operation

#### `etcd` Backup

Backup on `etcd` master (leader) only

- Get on leader

```sh
ubuntu@node1:~$ sudo etcdctl --key /etc/ssl/etcd/ssl/admin-node1-key.pem --cert /etc/ssl/etcd/ssl/admin-node1.pem endpoint status --write-out=table
[sudo] password for ubuntu:
+----------------+-----------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT    |       ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------+-----------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 127.0.0.1:2379 | 4cf7fb4a1e9b1c6 |  3.4.13 |   18 MB |      true |      false |         5 |    5215191 |            5215191 |        |
+----------------+-----------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

- `save snapshot`

```sh
ubuntu@node1:~$ cd etcd/
ubuntu@node1:~/etcd$  sudo etcdctl --key /etc/ssl/etcd/ssl/admin-node1-key.pem --cert /etc/ssl/etcd/ssl/admin-node1.pem snapshot save snapshot.db
{"level":"info","ts":1631673970.1899395,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"snapshot.db.part"}
{"level":"info","ts":"2021-09-15T10:46:10.227+0800","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1631673970.2272623,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"127.0.0.1:2379"}
{"level":"info","ts":"2021-09-15T10:46:10.867+0800","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1631673970.894913,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"127.0.0.1:2379","size":"18 MB","took":0.704905053}
{"level":"info","ts":1631673970.8949878,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"snapshot.db"}
Snapshot saved at snapshot.db
```

> Reference > https://support.coreos.com/hc/en-us/articles/115000323894-Creating-etcd-backup

- `restore snapshot`

```sh
sudo etcdctl --key /etc/ssl/etcd/ssl/admin-node1-key.pem --cert /etc/ssl/etcd/ssl/admin-node1.pem snapshot restore snapshot.db
```

#### Kubernetes Backup `/etc/kubernetes/pki`

This directory contains all keys of `kubernetes`

---

#### MongoDB Backup with Kubernetes Cronjob

- Attach a raw disk to one of worker nodes in cluster. In our scenario, the disk is attached to `node7`, which is `metrics-collector`

```sh
ubuntu@node7:~$ hostname
node7
ubuntu@node7:~$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  40G  0 disk
└─vda1 252:1    0  40G  0 part /
vdb    252:16   0  40G  0 disk
└─vdb1 252:17   0  40G  0 part /mnt/disks-by-id/disk0
```

- Check `persistentVolume` exists

> Notice: the output is after `persistentVolumeClaim` being bound.

```sh
ubuntu@node1:~$ k get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                             STORAGECLASS    REASON   AGE
...
local-pv-80492a6c   39Gi       RWO            Retain           Bound       sampleNamespace/mongodump                     local-storage            4d23h
```

- Create `persistentVolumeClaim` and bind to `persistentVolume`

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodump
  namespace: sampleNamespace
spec:
  volumeMode: Filesystem
  volumeName: local-pv-80492a6c
  storageClassName: local-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 38Gi
```

- Apply `cronjob` yaml

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mongodump-backup
  namespace: sampleNamespace
spec:
  schedule: '@daily'
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          securityContext:
            fsGroup: 1001   # grant permission to create file(s) in container
          containers:
            - name: mongodump-backup
              image: bitnami/mongodb
              imagePullPolicy: "IfNotPresent"
              command : ["/bin/sh", "-c"]
              args: ["mongodump -u $mongodb_user -p $mongodb_passwd --host=sample-sub-mongodb-client.sampleNamespace.svc.cluster.local --port=27017 --authenticationDatabase=admin -o /tmp/mongodump/$(date +\"%Y_%m_%d_%H:%M\")"]
              env:
              - name: mongodb_user
                valueFrom:
                  secretKeyRef:
                    name: mongodb
                    key: user
              - name: mongodb_passwd
                valueFrom:
                  secretKeyRef:
                    name: mongodb
                    key: password
              volumeMounts:
              - mountPath: "/tmp/mongodump/"
                name: mongodump
          restartPolicy: Never
          volumes:
          - name: mongodump
            persistentVolumeClaim:
              claimName: mongodump
```

- Define credential in YAML
`env`:`valueFrom`:`secretKeyRef` (with `yq`)

```yml
k -n sampleNamespace get secrets mongodb -oyaml | yq eval 'del(.metadata.managedFields, .metadata.creationTimestamp, .metadata.resourceVersion, .metadata.uid)' -

apiVersion: v1
data:
  password: cXXXXXXXXXX=
  user: cXXXXXX=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"cGFzc3cwcmQ=","user":"cm9vdA=="},"kind":"Secret","metadata":{"annotations":{},"name":"mongodb","namespace":"sampleNamespace"},"type":"Opaque"}
  name: mongodb
  namespace: sampleNamespace
type: Opaque
```

- Check on `node7` where the `persistentVolume` is attached

```sh
ubuntu@node7:/mnt/disks-by-id/disk0$ hostname
node7
ubuntu@node7:/mnt/disks-by-id/disk0$ pwd
/mnt/disks-by-id/disk0
ubuntu@node7:/mnt/disks-by-id/disk0$ ls -la
total 32
drwxrwsr-x 5 root 1001  4096 Oct 25 20:16 .
drwxr-xr-x 3 root root  4096 Oct 20 08:18 ..
drwxrwsr-x 4 1001 1001  4096 Oct 24 08:00 2021_10_24_00:00
drwxr-sr-x 4 1001 1001  4096 Oct 25 08:00 2021_10_25_00:00
drwxrws--- 2 root 1001 16384 Oct 18 21:08 lost+found
ubuntu@node7:/mnt/disks-by-id/disk0$ cd 2021_10_25_00\:00/
ubuntu@node7:/mnt/disks-by-id/disk0/2021_10_25_00:00$ du -ksh
7.5G	.
```

#### Cronjob to delete old MongoDB dump files

The objective of this is to delete old archive of `mongodump` with the disk space limit pressue

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mongodump-backup-rm-oldfile
  namespace: sampleNamespace
spec:
  schedule: '@daily'
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: mongodump-backup-rm-oldfile
              image: busybox
              args:
              - /bin/sh
              - -c
              - cd /tmp/mongodump/; find . -type d -mtime +2 -exec rm -fr {} \;
              volumeMounts:
              - mountPath: "/tmp/mongodump/"
                name: mongodump
          restartPolicy: OnFailure
          volumes:
          - name: mongodump
            persistentVolumeClaim:
              claimName: mongodump
```

Command `cd /tmp/mongodump/; find . -type d -mtime +3 -exec rm -fr {} \;`

- `cd /tmp/mongodump/; find ...` = avoid deleting parent directory
- `-type d` = `directory`
- `-mtime +3` = older than 3 days

#### Export and Import `collections` from MongoDB

The pre-requisite is to enable a service port-forward in Kubernetes

```sh
kubectl -n sampleNamespace port-forward svc/sampleNamespace-sub-mongodb 27017:27017 &
```

For example, to export `addVisitor_KE` collection in `DEV_KE_GROUP` namespace

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:latest mongoexport --host=127.0.0.1 --port=27017   \
  -u root -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase "admin" \
  --db ars02 --collection addVisitor_KE__DEV_KE_GROUP \
  --out=/app/addVisitor_KE__DEV_KE_GROUP.json --type json
```

To import a json. Notice the `--collection=addVisitor_KE__Testing_Property_KeGroup` which means `collectionName=addVisitor_KE` under `nameSpace=Testing_Property_KeGroup`

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:latest mongoimport --host=127.0.0.1 --port=27017 \
  -u root -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase "admin" \
  --db ars02 --collection addVisitor_KE__Testing_Property_KeGroup \
  --file=app/addVisitor_KE__DEV_KE_GROUP.json --type json
```

Pay attention to the formatType of import and export

#### Force MongoDB member to become primary

> Reference > https://dba.stackexchange.com/questions/136621/how-to-set-a-mongodb-node-to-return-as-the-primary-of-a-replication-set

#### Reschedule a Pod to another node

When we want to schedule an existing pod to a reserved worker node

- Check the pod `nodeAffinity`

```sh
kubectl -n sampleNamespace get statefulsets.apps metrics-collector -o yaml | grep -v "f:node" | grep "nodeAffinity" -A7

        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: sample.com/workload-preference
                operator: In
                values:
                - compute
```

The pod expects to be distributed to a worker node with label `sample.com/workload-preference=compute`, where key is `sample.com/workload-preference` and value is `compute`

- Create a label on reserved worker node, which is `node7` in our environment

```sh
kubectl label nodes node7 sample.com/workload-preference=compute
node/node7 labeled

ubuntu@node1:~$ k get nodes --show-labels | grep "sample.com"
node7    Ready    <none>        13d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,component=metrics-collector-server,kubernetes.io/arch=amd64,kubernetes.io/hostname=node7,kubernetes.io/os=linux,sample.com/workload-preference=compute
```

- Then re-scale this pod and check where it is scheduled

```sh
kubectl get pods --all-namespaces -o wide --sort-by="{.spec.nodeName}" | grep metrics
sampleNamespace           metrics-collector-0       1/1     Running   0          36m     10.233.95.23    node7    <none>           <none>
```

> Reference > https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity

#### List all selected resource per worker node in Kubernetes cluster

```sh
kubectl describe nodes|egrep "^Name:|instance|mongo|userdb|sampleApp|metrics|vision|influx|grafana|coredns|keycloak|nginx|telegraf|domain|Taint|role|cpu|memory"

Name:               node1
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
Taints:             <none>
  MemoryPressure       False   Fri, 17 Sep 2021 16:39:33 +0800   Thu, 02 Sep 2021 19:10:06 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  cpu:                2
  memory:             4030220Ki
  cpu:                1800m
  memory:             3403532Ki
  default                     local-volume-provisioner-vtfj8    0 (0%)        0 (0%)      0 (0%)           0 (0%)         10d
  kube-system                 coredns-bbb7d66cd-6w8rq           100m (5%)     0 (0%)      70Mi (2%)        170Mi (5%)     14d
  shared                      nginx-ingress-controller-bdk22    0 (0%)        0 (0%)      0 (0%)           0 (0%)         9d
  shared                      telegraf-ds-zz6h9                 100m (5%)     1 (55%)     256Mi (7%)       1Gi (30%)      8d
  cpu                1 (55%)          1300m (72%)
  memory             479236096 (13%)  1930257664 (55%)
Name:               node10
...
```

<div style="page-break-after: always;"></div>

## Customization

#### Apply a customized theme for KeyCloak by `kustomize `

Kustomize is Kubernetes native configuration management tool. Kustomize introduces a template-free way to customize application configuration that simplifies the use of off-the-shelf applications.

It becomes a kind of standard and the simplest way to manage configuration change in Kubernetes cluster.

##### Prepare customized theme property files for Keycloak

- Create a directory `domain2` and copy customized theme file to `~/kustomize/domain2`

```sh
ubuntu@debian:~/kustomize$ tree domain2
domain2
└── login
    ├── login.ftl
    ├── messages
    │   └── messages_en.properties
    ├── resources
    │   ├── css
    │   │   ├── login.css
    │   │   └── login.css.orig
    │   └── img
    │       ├── favicon.ico
    │       ├── feedback-error-arrow-down.png
    │       ├── feedback-error-sign.png
    │       ├── feedback-success-arrow-down.png
    │       ├── feedback-success-sign.png
    │       ├── feedback-warning-arrow-down.png
    │       ├── feedback-warning-sign.png
    │       ├── keycloak-bg.png
    │       ├── keycloak-logo.png
    │       ├── keycloak-logo-text.png
    │       ├── domain_Background.png
    │       └── domain_logo.png
    └── theme.properties

5 directories, 17 files
```

- Create a tarball for such directory

```sh
tar -cvf domain2.tar ./domain2
```

- Create a configMap from the above tarball

```sh
kubectl -n shared create configmap keycloak-tdg2 --from-file=./domain2.tar
```

##### Generate `base` resource for `kustomize`

- Extract `keycloak` `statefulSet` into `keycloak-sts.yaml`. Edit it and remove all runtime specification, such as `creationTimestamp`, `softLink`, `uid`, and `status`
```sh
kubectl -n shared get statefulsets.apps keycloak -o yaml | neat > keycloak-sts.yaml
```

- Another option to generate a clean output yaml

```sh
k -n iam get sts keycloak -oyaml | yq --yaml-roundtrip 'del(.metadata.creationTimestamp, .metadata.uid, .metadata.selfLink, .metadata.managedFields, .status)' -
```

##### Generate `overlay`patch YAML for `kustomize`

```yml
ubuntu@node1:~/kustomize$ cat keycloak-sts-patch.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: keycloak
  namespace: shared
spec:
  template:
    spec:
      volumes:
      - name: theme
        configMap:
          name: keycloak-tdg2
      containers:
      - name: keycloak
        volumeMounts:
        - name: theme
          mountPath: "/mnt"
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "tar -xvf /mnt/domain2.tar -C /opt/jboss/keycloak/themes/"]
```

##### `kustomize` patch

- Generate `kustomization.yaml` for `kustomize`

```yml
ubuntu@node1:~/kustomize$ cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: shared
resources:
  - keycloak-sts.yaml
patchesStrategicMerge:
  - keycloak-sts-patch.yaml
```      

- Apply patch

```sh
kubectl apply -k .
```

<div style="page-break-after: always;"></div>

## Troubleshooting

#### `No data` for `nginx` and `mongodb` in Grafana

Error message in `kubectl -n shared logs -f telegraf-prom-*`

```sh
2021-10-05T23:27:07Z E! [inputs.prometheus] Unable to watch resources: kubernetes api: Failure 403 pods is forbidden: User "system:serviceaccount:shared:telegraf-prom" cannot watch resource "pods" in API group "" at the cluster scope
2021-10-05T23:27:08Z E! [inputs.prometheus] Unable to watch resources: kubernetes api: Failure 403 pods is forbidden: User "system:serviceaccount:shared:telegraf-prom" cannot watch resource "pods" in API group "" at the cluster scope
2021-10-05T23:27:09Z E! [inputs.prometheus] Unable to watch resources: kubernetes api: Failure 403 pods is forbidden: User "system:serviceaccount:shared:telegraf-prom" cannot watch resource "pods" in API group "" at the cluster scope
2021-10-05T23:27:10Z E! [inputs.prometheus] Unable to watch resources: kubernetes api: Failure 403 pods is forbidden: User "system:serviceaccount:shared:telegraf-prom" cannot watch resource "pods" in API group "" at the cluster scope
```

Analysis: there is no `rolebindings` for serviceAccount `telegraf-prom`

> The following output is correct after the problem fixed

```sh
ubuntu@node1:~$ k get clusterrolebindings.rbac.authorization.k8s.io | grep telegraf-prom
telegraf-prom                                          ClusterRole/view                                                                   6h17m
```

Apply the YAML to bind `ClusterRoleBinding` for account `telegraf-prom`

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"creationTimestamp":"2020-03-27T10:45:30Z","name":"telegraf-prom","resourceVersion":"1987850","selfLink":"/apis/rbac.authorization.k8s.io/v1/clusterrolebindings/telegraf-prom","uid":"8ff9ff5d-aa8d-44a4-ae18-6d1492a6e932"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"ClusterRole","name":"view"},"subjects":[{"kind":"ServiceAccount","name":"telegraf-prom","namespace":"shared"}]}
  name: telegraf-prom
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/telegraf-prom
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: telegraf-prom
  namespace: shared
```

The log of pod `telegraf-prom-*` after problem fixed

```sh
ubuntu@node1:~$ k -n shared logs -f telegraf-prom-f6fb967d7-s4krm
2021-10-06T01:55:13Z I! Starting Telegraf 1.14.5
2021-10-06T01:55:13Z I! Using config file: /etc/telegraf/telegraf.conf
2021-10-06T01:55:13Z I! Loaded inputs: prometheus internal
2021-10-06T01:55:13Z I! Loaded aggregators:
2021-10-06T01:55:13Z I! Loaded processors: enum
2021-10-06T01:55:13Z I! Loaded outputs: influxdb health
2021-10-06T01:55:13Z I! Tags enabled: host=telegraf-polling-service
2021-10-06T01:55:13Z I! [agent] Config: Interval:30s, Quiet:false, Hostname:"telegraf-polling-service", Flush Interval:30s
2021-10-06T01:55:13Z W! [inputs.prometheus] Use of deprecated configuration: 'metric_version = 1'; please update to 'metric_version = 2'
2021-10-06T01:55:13Z I! [outputs.health] Listening on http://[::]:8888
```

<div style="page-break-after: always;"></div>

## Appendix

#### MongoDB Backup and Restore with Docker

> Ref > https://docs.bitnami.com/tutorials/backup-restore-data-mongodb-kubernetes/

- Figure out MongoDB's root passwd

```sh
kubectl -n sampleNamespace get secret mongodb -o jsonpath="{.data.password}" | base64 --decode
```

```sh
export MONGODB_ROOT_PASSWORD=$(kubectl -n sampleNamespace get secret mongodb -o \
  jsonpath="{.data.password}" | base64 --decode)

echo $MONGODB_ROOT_PASSWORD
```

- Export MongoDB's service port

```sh
kubectl -n sampleNamespace get svc

kubectl -n sampleNamespace port-forward svc/sample-sub-mongodb 27017:27017 &
```

- Create mongoDB backup directory

```sh
mkdir mongo_backup; chmod o+w mongo_backup/
cd mongo_backup/
```

- Execute backup

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:latest mongodump --host=127.0.0.1 --port=27017 \
  -u root -p $MONGODB_ROOT_PASSWORD -o /app
```

and if you want a specific version of MongoDB image to match the server

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
  bitnami/mongodb:3.6.8 mongodump --host=127.0.0.1 --port=27017 \
  -u root -p $MONGODB_ROOT_PASSWORD -o /app
```

- Execute restore

```sh
docker run --rm --name mongodb -v $(pwd):/app --net="host" \
	bitnami/mongodb:latest mongorestore -u root -p $MONGODB_ROOT_PASSWORD /app
```

> Reference >
> https://dev.to/ptuladhar3/schedule-mongodb-backup-to-s3-using-kubernetes-cronjob-2bl7
> https://medium.com/@eboye/mongodb-auto-backup-via-cronjob-737e096c8470
> https://www.percona.com/blog/2020/07/20/backup-and-restore-of-mongodb-deployment-on-kubernetes/
> https://simodev.medium.com/how-to-automate-backup-mongodb-using-kubernetes-b07c61a8a6ec
