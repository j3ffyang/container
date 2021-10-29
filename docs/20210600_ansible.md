# Ansible Deployment Workflow

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Objective](#objective)
- [Pre-requisite](#pre-requisite)
- [Architecture Overview](#architecture-overview)
- [Deployment Workflow](#deployment-workflow)
  - [Prototype Environment](#prototype-environment)
  - [System Hardening](#system-hardening)
    - [`sshd`](#sshd)
    - [`firewalld`](#firewalld)
    - [Firewall Rules for `kubeadm`](#firewall-rules-for-kubeadm)
  - [Ansible Env Setup](#ansible-env-setup)
  - [Operating System](#operating-system)
    - [Delete Unused Packages - `role: os_pkg_rm`](#delete-unused-packages-role-os_pkg_rm)
    - [Set `/etc/hosts` - `role: os_hosts_mod`](#set-etchosts-role-os_hosts_mod)
    - [Set `hostname` to All Nodes - `role: os_hostname_set`](#set-hostname-to-all-nodes-role-os_hostname_set)
    - [Create a `nonroot` user on all nodes, including `control` node - `role: os_usr_create`](#create-a-nonroot-user-on-all-nodes-including-control-node-role-os_usr_create)
  - [Docker](#docker)
    - [Install docker (containerd.io) - `role: docker_install`](#install-docker-containerdio-role-docker_install)
  - [Air-Gapped Installation (optional)](#air-gapped-installation-optional)
    - [Create Local Images (option 1)](#create-local-images-option-1)
    - [Build a Private Docker Registry (option 2)](#build-a-private-docker-registry-option-2)
    - [Tag Pulled Docker Images then Push into Local Registry](#tag-pulled-docker-images-then-push-into-local-registry)
    - [Docker Points to Local Private Registry (with Ansible)](#docker-points-to-local-private-registry-with-ansible)
    - [Remove tagged images and destroy repository](#remove-tagged-images-and-destroy-repository)
  - [Kubernetes (specific version) - `1.18.18`](#kubernetes-specific-version-11818)
    - [Disable `swap` - `role: os_swap_disable`](#disable-swap-role-os_swap_disable)
    - [Install Kubernetes - `role: k8s_install`](#install-kubernetes-role-k8s_install)
    - [Init the cluster on `master` only - `role: k8s_init`](#init-the-cluster-on-master-only-role-k8s_init)
    - [Grant Permission to `ubuntu` to Manage Kubernetes - `role: k8s_kubeconfig`](#grant-permission-to-ubuntu-to-manage-kubernetes-role-k8s_kubeconfig)
    - [Install `flannel` Network Plugin (master only) - `role: k8s_flannel`](#install-flannel-network-plugin-master-only-role-k8s_flannel)
    - [Create `kubeadm join` Command - `role: k8s_join_cmd`](#create-kubeadm-join-command-role-k8s_join_cmd)
    - [Join `node` in Kubernetes cluster - `role: k8s_join_node`](#join-node-in-kubernetes-cluster-role-k8s_join_node)
    - [Destroy Entire Kubernetes Cluster](#destroy-entire-kubernetes-cluster)
    - [(Optional) AutoComplete and Alias for `kubectl` and `kubeadm` - `role: k8s_autocompletion`](#optional-autocomplete-and-alias-for-kubectl-and-kubeadm-role-k8s_autocompletion)
    - [Custom persistentVolume](#custom-persistentvolume)
    - [Kubernetes within China](#kubernetes-within-china)
    - [Kubernetes upgrade for an existing cluster](#kubernetes-upgrade-for-an-existing-cluster)
    - [Configure Kubernetes HA](#configure-kubernetes-ha)
  - [Helm3](#helm3)
    - [Install `helm3` - `role: helm3_install`](#install-helm3-role-helm3_install)
    - [`ingress-nginx` - `role: k8s_ingress_nginx`](#ingress-nginx-role-k8s_ingress_nginx)
    - [~~`nginx_ingress` - `role: nginx_ingress`~~](#~~nginx_ingress-role-nginx_ingress~~)
    - [`cert-manager` - `role: cert-manager`](#cert-manager-role-cert-manager)
    - [Prometheus - `role: prometheus`](#prometheus-role-prometheus)
    - [~~`sealedSecrets` by `kubeseal`~~](#~~sealedsecrets-by-kubeseal~~)
    - [Upfront Nginx Web Server on VM(s)](#upfront-nginx-web-server-on-vms)
    - [Rook Storage Cluster with Ceph](#rook-storage-cluster-with-ceph)
- [Appendix](#appendix)
    - [Full HA of Nginx](#full-ha-of-nginx)
    - [Reference](#reference)
    - [Comparison between `terraform` and `ansible`](#comparison-between-terraform-and-ansible)
    - [Sample code](#sample-code)
    - [Format Raw Disk](#format-raw-disk)
    - [Global Variable](#global-variable)
    - [Template in `j2`](#template-in-j2)
    - [Task and Sub-Tasks](#task-and-sub-tasks)
    - [Rook Storage Cluster with Ceph through YAML](#rook-storage-cluster-with-ceph-through-yaml)
- [Troubleshooting](#troubleshooting)
    - [Terminating namespace gets stuck](#terminating-namespace-gets-stuck)
    - [Clean up Resources](#clean-up-resources)

<!-- /code_chunk_output -->

## Objective

- This is an environment for production and it is not an airgap scenario
- Simplify deployment of a Kubernetes platform and a bunch of common applications, such as Nginx, PostgreSQL, Prometheus, etc
- Deploy bastion-host (DMZ) and upfront Nginx webServer, manage system hardening, configure firewall
- Define a kind of standardized environment where VANTIQ product can be deployed, as well as operational and manage-able, other than conforming cloud service provider
- If an **airgapped** installation is needed, check "Air-Gapped Installation (optional)" and this part could be performance after `docker` installed before `kubernetes` started

## Pre-requisite

- An existing virtual_machine platform. Doesn't matter they're from OpenStack, or VMware, or AWS/ Azure (deploy-able by Ansible or Terraform)
- The virtual_machines are networked functionally. `bastion-host` and `vantiqSystem`, in the next figure, can be in separate sub-network (eg, `10.0.10.0/24` and `10.0.20.0/24`) or the same. If in separate networks, they must be accessible to each other
- Need several block-disks for MongoDB and others
- A DNS that can resolve service.domain.com, or a local `/etc/hosts` must be modified as well as `nginx-ingress` accordingly

## Architecture Overview

The following figure is drawn by [PlantUML](https://plantuml.com). If you use VScode or Atom.io as markdown editor, you need a plugin `markdown-preview-enhanced`(https://shd101wyy.github.io/markdown-preview-enhanced) to view it.

```plantuml

cloud freeWorld

package bastion-host {
  package webServer {
    collections nginx
    collections lb
    collections ha

    nginx -[hidden]> lb
    lb -[hidden]> ha
  }

  package vpn {
    collections wireGuard
  }

  webServer -[hidden]> vpn
}


package kubernetes {
  package vantiqSystem {

  component nginx_ingress
  component VANTIQ

  nginx_ingress -[hidden]> VANTIQ
  }

  package operation {
    component prometheus
  }
}

freeWorld -- webServer
webServer --> nginx_ingress
```


## Deployment Workflow

### Prototype Environment

name | ip | spec
-- | -- | --
master0 | 10.39.64.10 | 2c4g
worker0 | 10.39.64.20 | 2c8g
worker1 | 10.39.64.21 | 2c8g

### System Hardening

This is specifically setup for bastion-host. __Only__ bastion-host can be accessed from internet

#### `sshd`
- `/etc/ssh/sshd_config`
```sh
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
```

#### `firewalld`
```sh
  ubuntu@master0:~/ansible$ sudo firewall-cmd --list-all
  public
    target: default
    icmp-block-inversion: no
    interfaces:
    sources:
    services: ssh dhcpv6-client http https
    ports: 12345/udp
    protocols:
    masquerade: yes
    forward-ports:
    source-ports:
    icmp-blocks:
    rich rules:
```

#### Firewall Rules for `kubeadm`

```sh
ubuntu@master0:~$ firewall-cmd --get-active-zones
docker
  interfaces: docker0

ubuntu@master0:~$ firewall-cmd --get-default-zone
public

ubuntu@master0:~$ sudo firewall-cmd --permanent --zone=internal --add-interface=eth0
ubuntu@master0:~$ sudo firewall-cmd --reload
success

ubuntu@master0:~$ firewall-cmd --get-default-zone
public
ubuntu@master0:~$ firewall-cmd --get-active-zones
docker
  interfaces: docker0
internal
  interfaces: eth0

ubuntu@master0:~$ sudo firewall-cmd --permanent --zone=internal --set-target=ACCEPT
success
ubuntu@master0:~$ sudo firewall-cmd --reload
success
```

Allow all traffic for `internal` network on `eth0` interface only


- Kernel tuning and enable `ip_forward` for `masquerade`


- Port open (VM level. Still need configuration in security group from cloud service provider):
- TCP: `80`, `443`, `22`
- UDP: `12345` for wireGuard


### Ansible Env Setup

- Pre-requisite: `ansible`, `python3`, and `pip3`

```sh
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible
sudo apt install -y python3-pip
```

```sh
ubuntu@master0:~/ansible$ python3 -V
Python 3.6.9
```

- Setup accounts and access

Follow the instruction to setup a `master0` and two `worker0` and `worker1` in the environment. And copy sshKey from `master0` into two workers by `ssh-copy-id`, so you can access worker by `root`

```sh
ubuntu@master0:~$ ssh root@worker1
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1041-azure x86_64)
...
*** System restart required ***
Last login: Thu Mar 25 11:02:14 2021 from 10.39.64.10
root@worker1:~#
```

- Directory structure

tl;dr

```sh
ubuntu@master0:~/ansible$ tree
.
├── ansible.cfg
└── inventory
    └── hosts

1 directory, 2 files
```

- Basic Configurations

  - `ansible.cfg`

  ```sh
  [defaults]

  inventory               = inventory/hosts
  strategy                = debug   # display debug message while running
  # enable_task_debugger  = True
  ```

  - `inventory/hosts` - all nodes and associated IPs are here under Ansible

  ```sh
  ubuntu@master0:~/ansible$ cat inventory/hosts
  [worker]
  worker0	ansible_host=10.39.64.20
  worker1 ansible_host=10.39.64.21

  [master]
  master0 ansible_host=10.39.64.10

  [controller]
  master0

  [k8s_node]
  master0
  worker0
  worker1

  [webserver]
  webserver0 ansible_host=10.39.64.20
  webserver1 ansible_host=10.39.64.21

  [all:vars]
  ansible_python_interpreter=/usr/bin/python3
  ```

  - `global.yaml` - global variable

  ```yml
  ubuntu@master0:~/ansible$ cat global.yaml
  uusername: ubuntu
  vaulted_passwd: secret

  k8s_version: 1.18.18-00
  k8s_packages:
          - kubelet
          - kubeadm
          - kubectl
  ```

- Test Connections

```sh
ubuntu@master0:~/ansible$ ansible all -m ping -u root
worker0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
master0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> Reference > https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-18-04

- Initialize `roles`

`roles` naming pattern: `segment_target_exec`, eg. `os_host_mod`. All `roles` go to `~/ansible/roles`

Create a `role` with `task`

```sh
mkdir -p ~/ansible/roles
cd ~/ansible/roles

ansible-galaxy init os_pkg_rm
- Role roles was created successfully

ubuntu@master0:~/ansible/roles/os_pkg_rm$ tree
.
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

> Reference > https://www.decodingdevops.com/ansible-apt-module-examples/

- `main.yaml` - main playbook pattern

```yml
ubuntu@master0:~/ansible$ cat main.yaml
- hosts: all
  become: true

  roles:
    # - { role: os_hostname_display, when: "inventory_hostname in groups['all']" }
    # - { role: os_pkg_rm, when: "inventory_hostname in groups['worker']" }
    # - { role: os_hosts_mod, when: "inventory_hostname in groups['worker']" }
    # - { role: os_hosts_cp, when: "inventory_hostname in groups['worker']" }
    # - { role: os_hostname_set, when: "inventory_hostname in groups['worker']" }
    # - { role: os_ssh_auth, when: "inventory_hostname in groups['worker']" }
    #
    # - { role: docker_install, when: "inventory_hostname in groups['k8s_node']" }
    # - { role: os_usr_create, when: "inventory_hostname in groups['k8s_node']" }
    # #
    # - { role: os_swap_disable, when: "inventory_hostname in groups['k8s_node']" }
    # - { role: k8s_install, when: "inventory_hostname in groups['k8s_node']" }
    # - { role: k8s_init, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_kubeconfig, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_flannel, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_join_cmd, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_join_node, when: "inventory_hostname in groups['worker']" }
    # - { role: k8s_autocompletion, when: "inventory_hostname in groups['controller']" }
    # - { role: os_lsblk, when: "inventory_hostname in groups['k8s_node']" }
    #
    # - { role: helm3_install, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_ingress_nginx, when: "inventory_hostname in groups['master']" }
    # - { role: cert-manager, when: "inventory_hostname in groups['master']" }
    # - { role: prometheus, when: "inventory_hostname in groups['master']" }
    # - { role: kubeseal, when: "inventory_hostname in groups['master']" }
    #
    # - { role: nginx_server, when: "inventory_hostname in groups['webserver']" }
    #
    # ######## Destroy Kubernetes ########
    - { role: k8s_destroy, when: "inventory_hostname in groups['k8s_node']" }
```

- Standard playbook execution command

__Make sure uncomment the `roles` in `main.yaml` for particular action(s) you want to perform__. Uncomment all to execute all actions in one click. The `role` value in `main.yaml` must be identical to the one created by `ansible-galaxy init`

> Caution: there is a role called `k8s_destroy`, which will destroy the existing Kubernetes cluster and Docker!

```sh
ansible-playbook --extra-vars @global.yaml main.yaml
```

where `global.yaml` contains the global variables.

---

### Operating System

#### Delete Unused Packages - `role: os_pkg_rm`

  ```yml
  cat roles/os_pkg_rm/tasks/main.yml

  ---
  # tasks file for os_pkg_rm
  - name: Remove "snapd" package
    apt:
      name: snapd
      state: absent

  - name: Remove "ufw" package
    apt:
      name: ufw
      state: absent

  - name: Remove dependencies that are no longer required
    apt:
      autoremove: yes
  ```

This task won't trigger the actions of

  ```sh
  systemctl stop ufw
  systemctl disable ufw
  ```

But it doesn't matter

#### Set `/etc/hosts` - `role: os_hosts_mod`

- `ansible_host`

This file manages all hosts and their IPs

```sh
ubuntu@master0:~/ansible$ cat inventory/hosts

[worker]
worker0	ansible_host=10.39.64.20
worker1 ansible_host=10.39.64.21

[controller]
master0 ansible_host=10.39.64.10

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

- `templates` in `jinja2` format

```yml
ubuntu@master0:~/ansible$ cat roles/os_hosts_mod/templates/hosts.j2

{% for item in groups["all"] %}
{{hostvars[item]['ansible_host']}} {{hostvars[item]['inventory_hostname']}}
{% endfor %}
```

The variable of `ansible_host` requires `inventory/hosts` contains `ansible_host` values

- `tasks`

```sh
ubuntu@master0:~/ansible$ cat roles/os_hosts_mod/tasks/main.yml
---
# tasks file for os_hosts_mod

- name: Copy hosts
  template:
    src: hosts.j2
    dest: /etc/hosts
```

> Reference > https://www.howtoforge.com/ansible-guide-manage-files-using-ansible/

#### Set `hostname` to All Nodes - `role: os_hostname_set`

According to `/etc/hosts` on `control node`

  ```yml
  ubuntu@master0:~/ansible$ cat roles/os_hostname_set/tasks/main.yml
  ---
  # tasks file for os_hostname_set

  - name: hostnamectl set-hostname
    hostname:
      name: "{{inventory_hostname}}"
  ```

#### Create a `nonroot` user on all nodes, including `control` node - `role: os_usr_create`

  This supposes to create a `nonroot` user, to manage Docker and Kubernetes, in case a specific user has been occupied already in customer's environment. For example, `ubuntu` user is not allowed to be used.

  ```sh
  ubuntu@master0:~/ansible$ cat global.yaml

  uusername: nonroot
  vaulted_passwd: secret
  ```


  ```yml
  ubuntu@master0:~/ansible$ cat roles/os_usr_create/tasks/main.yml
  ---
  # tasks file for os_usr_create

  - name: Create a nonroot user
    user:
      name: "{{ uusername }}"
      password: "{{ vaulted_passwd | password_hash('sha512') }}"
      shell: /bin/bash
      groups: docker
      append: yes
      generate_ssh_key: yes
      ssh_key_bits: 2048
      ssh_key_file: .ssh/id_rsa
  ```

  ```sh
  ansible-playbook --extra-vars @global.yaml main.yaml
  ```


- ~~Setup `sshd` and `ssh-copy-id` to all worker-node~~

This action has been done before setting up Ansible as pre-requisite, unless otherwise you want to have another userId

- Kernel tuning: `inode`, `ulimit`, etc

### Docker
#### Install docker (containerd.io) - `role: docker_install`

```yml
ubuntu@master0:~/ansible$ cat roles/docker_install/tasks/main.yml
---
# tasks file for docker_install

- name: Install pre-requisite packages
  apt:
    update_cache: yes
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - software-properties-common
    state: present

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker apt repository
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release | lower }} stable
    state: present

- name: Install Docker
  apt:
    update_cache: yes
    name: "{{ packages }}"
    state: present
  vars:
    packages:
    - docker-ce
    - docker-ce-cli
    - containerd.io
    # notify:
    # - docker status
```

> Reference >
- https://horrell.ca/2020/06/18/installing-docker-on-ubuntu-with-ansible/
- https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/ # updated 20210405 for `containerd`

---

### Air-Gapped Installation (optional)

Write up this solution for an airgapped environment and use a private docker registry. This part covers the pre-requisite docker container environment setup, before VANTIQ can be deployed.

#### Create Local Images (option 1)

- List images

```sh
ubuntu@master0:~/ansible$ docker image ls | grep -v 'REPOSITORY' | awk '{print $1":"$2}'
quay.io/coreos/flannel:v0.14.0-rc1
k8s.gcr.io/kube-proxy:v1.18.18
k8s.gcr.io/kube-scheduler:v1.18.18
k8s.gcr.io/kube-controller-manager:v1.18.18
k8s.gcr.io/kube-apiserver:v1.18.18
registry:2
k8s.gcr.io/pause:3.2
k8s.gcr.io/coredns:1.6.7
k8s.gcr.io/etcd:3.4.3-0
```

- Save all images on environment with internet

```sh
mkdir -p ~/docker
ubuntu@master0:~/docker$ docker save $(docker images -q) -o k8s_img.tar
ubuntu@master0:~/docker$ ls -la
total 804656
drwxrwxr-x  2 ubuntu ubuntu      4096 May  4 09:52 .
drwxr-xr-x 14 ubuntu ubuntu      4096 May  4 09:51 ..
-rw-------  1 ubuntu ubuntu 823953408 May  4 09:52 k8s_img.tar
```

This tar file contains all pre-requisite images to build a Kubernetes cluster. You can copy `k8s_img.tar` into a thumb-drive and bring it with you to install.

#### Build a Private Docker Registry (option 2)

The objective of building a private local docker registry to provide images within an isolated network. A simplest registry should work well for this requirement. Therefore this part of document doesn't cover SSL and other hardening steps.

- Edit `/etc/hosts`, if you want to use hostname instead of IP address for `registry`

```sh
cat /etc/hosts

10.39.64.10	master0 k8s-master registry.local
10.39.64.20	worker0
10.39.64.21	worker1
```

- Enable insecure registry

```json
ubuntu@master0:~/ansible$ cat /etc/docker/daemon.json
{
  "insecure-registries" : [ "registry.local:5000" ]
}
```

```sh
sudo systemctl restart docker
```

- Launch `registry`

```sh
mkdir -p /home/ubuntu/registry

docker run -d -p 5000:5000 --name registry \
  -v /home/ubuntu/registry:/var/lib/registry \
  --restart always registry:2
```

> Reference >
> https://windsock.io/automated-docker-image-builds-with-multiple-tags/
> https://linuxhint.com/create-crt-file-linux/

#### Tag Pulled Docker Images then Push into Local Registry

- Tag and push images

```sh
ubuntu@master0:~/docker$ cat img_tag.sh
#!/bin/bash

# create /tmp/img.lst with repo:tag
docker images | sed '1d' | grep -v "registry" | awk '{print $1 ":" $2}' > /tmp/img.lst
# cat /tmp/img.lst

# replace repo with registry.local:5000
awk -F'.io' '{print $0, "registry.local:5000" $2}' /tmp/img.lst > /tmp/tag.lst
# cat /tmp/tag.lst

# read lines, then tag the imgs
while read i; do docker tag $i; done < /tmp/tag.lst

# push images into registry.local:5000
for i in `docker images | grep "registry.local" | awk '{print $1 ":"  $2}'`; do docker push $i; done

# remove all tagged img
# docker rmi `docker images | grep "registry.local" | awk '{print $1 ":"  $2}'`

# clean up tmp files
# rm -fr /tmp/img.lst
# rm -fr /tmp/tag.lst
```

> Reference > https://stackoverflow.com/questions/35575674/how-to-save-all-docker-images-and-copy-to-another-machine

- Check pushed images

```sh
ubuntu@master0:~/docker$ curl -X GET http://registry.local:5000/v2/_catalog
{"repositories":["coredns","coreos/flannel","etcd","kube-apiserver","kube-controller-manager","kube-proxy","kube-scheduler","pause","registry"]}
```

#### Docker Points to Local Private Registry (with Ansible)

- Update `/etc/hosts` on all workers
- Update `/etc/docker/daemon.json` on all workers

```yml
# tasks file for local private registry
- name: update /etc/hosts
  lineinfile:
    path: /etc/hosts
    line: 10.39.64.10	master0 k8s-master registry.local

- name: point to private registry
  copy:
    dest: "/etc/docker/daemon.json"
    content: |
      {
        registry-mirrors": ["https://registry.local:5000"]
      }
```

#### Remove tagged images and destroy repository

This step only removes the tagged images.

```sh
# stop registry
docker rm -f registry

# remove all tags
docker rmi `docker images | grep "registry.local" | awk '{print $1 ":" $2}'`

# hard delete all pushed images
sudo rm -fr /home/ubuntu/registry
```

---

### Kubernetes (specific version) - `1.18.18`

#### Disable `swap` - `role: os_swap_disable`

```yml
ubuntu@master0:~/ansible$ cat roles/os_swap_disable/tasks/main.yml
---
# tasks file for os_swap_disable
- name: Remove swapfile from /etc/fstab
  mount:
    name: "{{ item }}"
    fstype: swap
    state: absent
  with_items:
    - swap
    - none

- name: Disable swap
  command: swapoff -a
  when: ansible_swaptotal_mb > 0
```

> Reference > https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/

#### Install Kubernetes - `role: k8s_install`

First set variable

```yml
ubuntu@master0:~/ansible$ cat global.yaml
uusername: ubuntu
vaulted_passwd: secret

k8s_version: "1.18.18-00"
k8s_packages:
  - kubelet
  - kubeadm
  - kubectl
```

Then both variable of `k8s_packages` and `k8s_version` are set `~/global.yaml` and `k8s_packages` is installed in a for-loop pattern.

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_install/tasks/main.yml
---
# tasks file for k8s_install
- name: Add an apt signing key for Kubernetes
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Adding apt repository for Kubernetes
  apt_repository:
    repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
    state: present
    filename: kubernetes.list

- name: Install Kubernetes binaries
  apt:
    name: "{{ item }}={{ k8s_version }}"
    state: present
    update_cache: yes
  with_items: "{{ k8s_packages }}"

- name: Restart kubelet
  service:
    name: kubelet
    daemon_reload: yes
    state: restarted
```

You can use `--check` as `dry-run` to test the result without committing

```sh
ansible-playbook --extra-vars @global.yaml main.yaml --check -v
```

---

I leave the following code in an earlier release as a reference.

You may change the version of installed binary

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_install/tasks/main.yml
---
# tasks file for k8s_install
- name: Add an apt signing key for Kubernetes
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Adding apt repository for Kubernetes
  apt_repository:
    repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
    state: present
    filename: kubernetes.list

- name: Install Kubernetes binaries
  apt:
    name: "{{ packages }}"
    state: present
    update_cache: yes
  vars:
    packages:
      - kubelet=1.18.18-00
      - kubeadm=1.18.18-00
      - kubectl=1.18.18-00

- name: Restart kubelet
  service:
    name: kubelet
    daemon_reload: yes
    state: restarted
```

> Reference > https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04

#### Init the cluster on `master` only - `role: k8s_init`

`--apiserver-advertise-address` is `master0` IP address

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_init/tasks/main.yml
---
# tasks file for k8s_init
- name: Initialize the Kubernetes cluster using kubeadm
  command: kubeadm init --apiserver-advertise-address="10.39.64.10" --apiserver-cert-extra-sans="10.39.64.10" --node-name k8s-master --pod-network-cidr=10.244.0.0/16
```

As long as `--pod-network-cidr=10.244.0.0/16` gets set, `cni0` device IP won't be able to change until otherwise `cni0` device is deleted.

#### Grant Permission to `ubuntu` to Manage Kubernetes - `role: k8s_kubeconfig`

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_kubeconfig/tasks/main.yml
---
# tasks file for k8s_kubeconfig
# - name: Setup kubeconfig for ubuntu user
#   command: "{{ item }}"
#   with_items:
#    - mkdir -p /home/ubuntu/.kube
#    - cp -i /etc/kubernetes/admin.conf /home/ubuntu/.kube/config
#    - chown -R ubuntu:ubuntu /home/ubuntu/.kube

- name: create .kube directory
  become: yes
  become_user: ubuntu
  file:
    path: $HOME/.kube
    state: directory
    mode: 0755

- name: copy admin.conf to user's kube config
  copy:
    src: /etc/kubernetes/admin.conf
    dest: /home/ubuntu/.kube/config
    remote_src: yes
    owner: ubuntu
    mode: 0600
```

You have to logout then login again to pick up the change!

If `/home/ubuntu/.kube` isn't owned by `ubuntu.ubuntu`, but owned by `root.root` instead, the following error would appear

> Reference > https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04

```sh
ubuntu@master0:~/.kube$ kubectl get all --all-namespaces
I0412 08:13:35.339148   11404 request.go:668] Waited for 1.152966258s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/policy/v1?timeout=32s
I0412 08:13:45.339837   11404 request.go:668] Waited for 6.198896723s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/apiextensions.k8s.io/v1beta1?timeout=32s
I0412 08:13:56.739889   11404 request.go:668] Waited for 1.19751473s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/events.k8s.io/v1?timeout=32s
I0412 08:14:06.939134   11404 request.go:668] Waited for 3.191095759s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/admissionregistration.k8s.io/v1?timeout=32s
I0412 08:14:16.939176   11404 request.go:668] Waited for 4.997458384s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/scheduling.k8s.io/v1?timeout=32s
I0412 08:14:27.139494   11404 request.go:668] Waited for 6.99703677s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/networking.k8s.io/v1beta1?timeout=32s
I0412 08:14:37.739217   11404 request.go:668] Waited for 1.197000394s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/authentication.k8s.io/v1?timeout=32s
```


#### Install `flannel` Network Plugin (master only) - `role: k8s_flannel`

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_flannel/tasks/main.yml
---
# tasks file for k8s_flannel
- name: Install flannel pod network
  become: false
  command: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> Reference > https://github.com/flannel-io/flannel

#### Create `kubeadm join` Command - `role: k8s_join_cmd`

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_join_cmd/tasks/main.yml
---
# tasks file for k8s_join_cmd
- name: Generate join command
  command: kubeadm token create --print-join-command
  register: join_command

- name: Copy join command to local file
  local_action: copy content="{{ join_command.stdout_lines[0] }}" dest="/tmp/join-command"
```

#### Join `node` in Kubernetes cluster - `role: k8s_join_node`

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_join_node/tasks/main.yml
---
# tasks file for k8s_join_node
- name: Copy the join command to other workers
  copy: src=/tmp/join-command dest=/tmp/join-command.sh mode=0777

- name: Join node to cluster
  command: sh /tmp/join-command.sh
```

#### Destroy Entire Kubernetes Cluster

Run the following on __each__ of nodes

```sh
sudo kubeadm reset

# remove binaries
sudo apt purge kubectl kubeadm kubelet kubernetes-cni -y
sudo apt autoremove

# kubeadm reset should remove the following ones. But double check
sudo rm -fr /etc/kubernetes/; sudo rm -fr /var/lib/etcd; sudo rm -rf /var/lib/cni/; sudo rm -fr /etc/cni/net.d; rm -fr ~/.kube/

# unnecessary in most cases
sudo systemctl daemon-reload

# run this if firewalld and/ or ufw are running
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# remove all running docker containers for Kubernetes
docker rm -f `docker ps -a | grep "k8s_" | awk '{print $1}'`
```

If you also want to remove all pulled docker image, do

**Warning**: this would remove **ALL** images

```sh
for i in `docker image ls | grep -v "IMAGE ID" | awk '{print $3}'`; do docker image rm $i; done
```

In my case, I want to keep `registry` docker image, I run this instead

```sh
for i in `docker image ls | grep -E -v 'IMAGE ID|registry' | awk '{print $3}'`; do docker image rm $i; done
```

**Remove both `cni0` and `flannel.1` devices by restarting VM**

Consult with this reference, if you want to remove `docker` as well

> Reference > https://hiberstack.com/10677/how-to-uninstall-docker-and-kubernetes/


If you want to run the step one by one, you run the following in a batch

**Caution: this will destroy the entire Kubernetes cluster and Docker environment!**

> Note: ``{{ uusername }}`` is defined in `global.yaml`

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_destroy/tasks/main.yml
---
# tasks file for k8s_destroy
- name: destroy kubernetes cluster and docker
  shell: |
    yes 'Y' | kubeadm reset
    apt purge kubectl kubeadm kubelet kubernetes-cni -y
    rm -fr /etc/kubernetes/; rm -fr /var/lib/etcd; rm -rf /var/lib/cni/; rm -fr /etc/cni/net.d
    rm -fr /home/{{ uusername }}/.kube/
    systemctl daemon-reload
    iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
    docker rm -f `docker ps -a | grep "k8s_" | awk '{print $1}'`
    apt purge docker-ce docker-ce-cli docker-ce-rootless-extras docker-scan-plugin -y
    ip link delete cni0
    ip link delete flannel.1
    ip link delete docker0
```

> Reference >
https://stackoverflow.com/questions/24851575/ansible-how-to-pass-multiple-commands
https://stackoverflow.com/questions/41871918/how-to-let-ansible-answer-yes-to-everything-sendmailconfig-asks

#### (Optional) AutoComplete and Alias for `kubectl` and `kubeadm` - `role: k8s_autocompletion`

- Ansible code - run this on Ansible `controller` for `ubuntu` user only

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_autocompletion/tasks/main.yml
---
# tasks file for k8s_autocompletion
- name: update ~/.bashrc for kubectl & kubeadm autocompletion for ubuntu user
  blockinfile:
    dest: /home/ubuntu/.bashrc
    block: |
      source <(kubectl completion bash)
      source <(kubeadm completion bash)
      source <(helm completion bash)
      alias k=kubectl
      complete -F __start_kubectl k
    backup: yes
```

- Manual step - add the following into `~/.bashrc` on Ansible `controller` for `ubuntu`

```bash{.line-numbers}
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```bash
# alias and auto-completion
alias k=kubectl
complete -F __start_kubectl k
```

#### Custom persistentVolume

**Caution**: This part of instruction contains block disk format. Make sure you understand what you're doing and what exact disk(s) you're going to format. You may destroy your existing installation if incorrect device is chosen

The environment that I have when writing this document is

vm | device | size
-- | -- | --
master0 | |
worker0 | sda | 20g
worker1 | sdb | 20g

##### `lsblk` - `role: os_lsblk`

```yml
ubuntu@master0:~/ansible$ cat roles/os_lsblk/tasks/main.yml
---
# tasks file for os_lsblk
- name: Run lsblk on all nodes
  command: lsblk
  register: cmd_reg
- name: "lsblk stdout"
  debug:
    msg: "{{ cmd_reg.stdout.split('\n') }}"
```

Output:
```yml
TASK [os_lsblk : lsblk stdout] ***************************************************************************
ok: [master0] => {
    "msg": [
        "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT",
        "sda       8:0    0   128G  0 disk ",
        "├─sda1    8:1    0 127.9G  0 part /",
        "├─sda14   8:14   0     4M  0 part ",
        "└─sda15   8:15   0   106M  0 part /boot/efi",
        "sdb       8:16   0    16G  0 disk ",
        "└─sdb1    8:17   0    16G  0 part ",
        "sr0      11:0    1  1024M  0 rom  "
    ]
}
ok: [worker0] => {
    "msg": [
        "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT",
        "sda       8:0    0    20G  0 disk ",
        "sdb       8:16   0   128G  0 disk ",
        "├─sdb1    8:17   0 127.9G  0 part /",
        "├─sdb14   8:30   0     4M  0 part ",
        "└─sdb15   8:31   0   106M  0 part /boot/efi",
        "sr0      11:0    1  1024M  0 rom  "
    ]
}
ok: [worker1] => {
    "msg": [
        "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT",
        "sda       8:0    0   128G  0 disk ",
        "├─sda1    8:1    0 127.9G  0 part /",
        "├─sda14   8:14   0     4M  0 part ",
        "└─sda15   8:15   0   106M  0 part /boot/efi",
        "sdb       8:16   0    20G  0 disk ",
        "sr0      11:0    1  1024M  0 rom  "
    ]
}
```

If running `ansible-playbook --extra-vars @global.yaml main.yaml -v | sed 's/\\n/\n/g'`, the output looks like

```yml
ubuntu@master0:~/ansible$ ansible-playbook --extra-vars @global.yaml main.yaml -v
Using /home/ubuntu/ansible/ansible.cfg as config file

PLAY [all] ***********************************************************************************************

TASK [Gathering Facts] ***********************************************************************************
ok: [master0]
ok: [worker0]
ok: [worker1]

TASK [os_lsblk : Run lsblk on all nodes] *****************************************************************
changed: [worker0] => {"changed": true, "cmd": ["lsblk"], "delta": "0:00:00.003796", "end": "2021-04-23 03:32:19.148881", "rc": 0, "start": "2021-04-23 03:32:19.145085", "stderr": "", "stderr_lines": [], "stdout": "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT\nsda       8:0    0    20G  0 disk \nsdb       8:16   0   128G  0 disk \n├─sdb1    8:17   0 127.9G  0 part /\n├─sdb14   8:30   0     4M  0 part \n└─sdb15   8:31   0   106M  0 part /boot/efi\nsr0      11:0    1  1024M  0 rom  ", "stdout_lines": ["NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT", "sda       8:0    0    20G  0 disk ", "sdb       8:16   0   128G  0 disk ", "├─sdb1    8:17   0 127.9G  0 part /", "├─sdb14   8:30   0     4M  0 part ", "└─sdb15   8:31   0   106M  0 part /boot/efi", "sr0      11:0    1  1024M  0 rom  "]}
changed: [worker1] => {"changed": true, "cmd": ["lsblk"], "delta": "0:00:00.004000", "end": "2021-04-23 03:32:19.153084", "rc": 0, "start": "2021-04-23 03:32:19.149084", "stderr": "", "stderr_lines": [], "stdout": "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT\nsda       8:0    0   128G  0 disk \n├─sda1    8:1    0 127.9G  0 part /\n├─sda14   8:14   0     4M  0 part \n└─sda15   8:15   0   106M  0 part /boot/efi\nsdb       8:16   0    20G  0 disk \nsr0      11:0    1  1024M  0 rom  ", "stdout_lines": ["NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT", "sda       8:0    0   128G  0 disk ", "├─sda1    8:1    0 127.9G  0 part /", "├─sda14   8:14   0     4M  0 part ", "└─sda15   8:15   0   106M  0 part /boot/efi", "sdb       8:16   0    20G  0 disk ", "sr0      11:0    1  1024M  0 rom  "]}
changed: [master0] => {"changed": true, "cmd": ["lsblk"], "delta": "0:00:00.005033", "end": "2021-04-23 03:32:19.218856", "rc": 0, "start": "2021-04-23 03:32:19.213823", "stderr": "", "stderr_lines": [], "stdout": "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT\nsda       8:0    0   128G  0 disk \n├─sda1    8:1    0 127.9G  0 part /\n├─sda14   8:14   0     4M  0 part \n└─sda15   8:15   0   106M  0 part /boot/efi\nsdb       8:16   0    16G  0 disk \n└─sdb1    8:17   0    16G  0 part \nsr0      11:0    1  1024M  0 rom  ", "stdout_lines": ["NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT", "sda       8:0    0   128G  0 disk ", "├─sda1    8:1    0 127.9G  0 part /", "├─sda14   8:14   0     4M  0 part ", "└─sda15   8:15   0   106M  0 part /boot/efi", "sdb       8:16   0    16G  0 disk ", "└─sdb1    8:17   0    16G  0 part ", "sr0      11:0    1  1024M  0 rom  "]}

TASK [os_lsblk : lsblk stdout] ***************************************************************************
ok: [master0] => {
    "msg": [
        "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT",
        "sda       8:0    0   128G  0 disk ",
        "├─sda1    8:1    0 127.9G  0 part /",
        "├─sda14   8:14   0     4M  0 part ",
        "└─sda15   8:15   0   106M  0 part /boot/efi",
        "sdb       8:16   0    16G  0 disk ",
        "└─sdb1    8:17   0    16G  0 part ",
        "sr0      11:0    1  1024M  0 rom  "
    ]
}
ok: [worker0] => {
    "msg": [
        "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT",
        "sda       8:0    0    20G  0 disk ",
        "sdb       8:16   0   128G  0 disk ",
        "├─sdb1    8:17   0 127.9G  0 part /",
        "├─sdb14   8:30   0     4M  0 part ",
        "└─sdb15   8:31   0   106M  0 part /boot/efi",
        "sr0      11:0    1  1024M  0 rom  "
    ]
}
ok: [worker1] => {
    "msg": [
        "NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT",
        "sda       8:0    0   128G  0 disk ",
        "├─sda1    8:1    0 127.9G  0 part /",
        "├─sda14   8:14   0     4M  0 part ",
        "└─sda15   8:15   0   106M  0 part /boot/efi",
        "sdb       8:16   0    20G  0 disk ",
        "sr0      11:0    1  1024M  0 rom  "
    ]
}
```

- `fdisk` > `n` to create partition > `w` to write configuration
- `mkfs.xfs` for `mongoDB` and `mkfs.ext4` for others
- `mount` to `/mnt/disks-by-id/diskX`
- Update `/etc/fstab`
- `storageClass`

#### Kubernetes within China

- gcr.io (Google Container Registry) can be replaced by the following repository.

```sh
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF
```

Or

```sh
https://mirrors.aliyun.com/kubernetes/apt/
```

- List the required images

```sh
kubeadm config images list
```

- Download the required images and tag them accordingly

```sh
images=(  # 下面的镜像应该去除"k8s.gcr.io/"的前缀，版本换成上面获取到的版本
    kube-apiserver:v1.12.1
    kube-controller-manager:v1.12.1
    kube-scheduler:v1.12.1
    kube-proxy:v1.12.1
    pause:3.1
    etcd:3.2.24
    coredns:1.2.2
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

> Reference in Chinese > https://zhuanlan.zhihu.com/p/46341911

#### Kubernetes upgrade for an existing cluster

#### Configure Kubernetes HA

---

### Helm3

#### Install `helm3` - `role: helm3_install`

Up to the date that I write this document, `helm` version = `v3.5.3`

- Install `community.kubernetes` plugin
```sh
ansible-galaxy collection install community.kubernetes
```

- Ansible code

Notice that we want to use `ubuntu` user to control `helm`, there are `become` and `become_user` privilege escalation defined, as `~/ansible/main.yaml` has a global `become: true` as `root`

```yml
ubuntu@master0:~/ansible$ cat roles/helm3_install/tasks/main.yml

---
# tasks file for helm3_install
#- name: add gnupg key for codership repo
#  apt-key: keyserver=keyserver.ubuntu.com id=BC19DDBA
#
#- name: add repo
#  apt_repository: repo='deb http://ubuntu-cloud.archive.canonical.com/{{ ansible_distribution | lower }} {{ ansible_distribution_release }}-updates/liberty main' state=present

# - name: add apt-key
#   shell:
#   - /usr/bin/curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
#   - /usr/bin/sudo apt-get install apt-transport-https --yes
#
# - name: add repo
#   apt_repository: repo='deb https://baltocdn.com/helm/stable/debian/ all main' state=present
#
# - name: install helm3
#   shell: sudo apt-get update; sudo apt-get install helm
#

- name: Retrieve helm bin
  unarchive:
    src: https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz
    dest: /tmp
    creates: /usr/local/bin/helm
    remote_src: yes

- name: Move helm binary into place
  command: cp /tmp/linux-amd64/helm /usr/local/bin/helm
  args:
    creates: /usr/local/bin/helm

- name: Add a repository
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: stable
    repo_url: https://charts.helm.sh/stable
```

> Reference >
https://github.com/geerlingguy/ansible-for-devops/blob/master/kubernetes/examples/helm.yml
https://www.ansible.com/blog/automating-helm-using-ansible

#### `ingress-nginx` - `role: k8s_ingress_nginx`

```yml
ubuntu@master0:~/ansible$ cat roles/k8s_ingress_nginx/tasks/main.yml
---
# tasks file for k8s_ingress_nginx

- name: Add ingress-nginx chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: ingress-nginx
    repo_url: "https://kubernetes.github.io/ingress-nginx"

- name: Deploy latest version of ingress-nginx
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: ingress-nginx
    chart_ref: ingress-nginx/ingress-nginx
    release_namespace: ingress-nginx
    create_namespace: yes
    values:
      replicas: 1
```

Delete `ingress-nginx`

```sh
helm -n ingress-nginx uninstall ingress-nginx
```

#### ~~`nginx_ingress` - `role: nginx_ingress`~~

This project has been deprecated and replaced with https://kubernetes.github.io/ingress-nginx/

```yml
ubuntu@master0:~/ansible$ cat roles/nginx_ingress/tasks/main.yml
---
# tasks file for nginx-ingress

- name: Add nginx-ingress chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: nginx-ingress
    repo_url: "https://helm.nginx.com/stable"

- name: Deploy latest version of ingress-nginx
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: nginx-ingress
    chart_ref: nginx-stable/nginx-ingress
    release_namespace: nginx-ingress
    create_namespace: yes
    values:
      replicas: 1
```

#### `cert-manager` - `role: cert-manager`

Install with `crd` enabled

```yml
ubuntu@master0:~/ansible$ cat roles/cert-manager/tasks/main.yml
---
# tasks file for cert-manager
- name: Add cert-manager chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: jetstack
    repo_url: "https://charts.jetstack.io"

- name: Deploy latest version of cert-manager
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: cert-manager
    chart_ref: jetstack/cert-manager
    release_namespace: cert-manager
    create_namespace: yes
    release_values:
      installCRDs: true
```

> Reference > https://cert-manager.io/docs/installation/kubernetes/

Delete `cert-manager`

```sh
helm -n monitoring uninstall cert-manager
kubectl delete ns monitoring  # check whether there is no other resource in this namespace, before deleting
```

To test and verify the installation (this isn't included in code)
- Create an `issuer`

```yml
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

- Create the test resource

```sh
kubectl apply -f test-resources.yaml
```

- Check the `certificate` created

```sh
kubectl describe certificate -n cert-manager-test
```

- Clean up

```sh
kubectl delete -f test-resources.yaml
```

> Reference > https://www.fosstechnix.com/kubernetes-nginx-ingress-controller-letsencrypt-cert-managertls/

#### Prometheus - `role: prometheus`

```yml
ubuntu@master0:~/ansible$ cat roles/prometheus/tasks/main.yml
---
# tasks file for prometheus
- name: Add prometheus chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: prometheus-community
    repo_url: "https://prometheus-community.github.io/helm-charts"

- name: Deploy latest version of prometheus
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: prometheus
    chart_ref: prometheus-community/kube-prometheus-stack
    release_namespace: monitoring
    create_namespace: yes

```

> Reference > https://github.com/prometheus-community/helm-charts

Delete `prometheus`

```sh
helm -n monitoring uninstall prometheus
```

#### ~~`sealedSecrets` by `kubeseal`~~

I skip this one as VANTIQ deployment code would cover it

- Install `kubernetes.core` module, equivalent to `community.kubernetes`

```sh
ansible-galaxy collection install kubernetes.core
```

- Ansible code

```yml
ubuntu@master0:~/ansible$ cat roles/kubeseal/tasks/main.yml
---
# tasks file for kubeseal

- name: Add helm repo
  become: yes
  become_user: ubuntu
  vars:
    chart_repo_url: "https://bitnami-labs.github.io/sealed-secrets"
  kubernetes.core.helm_repository:
    name: bitnami
    repo_url: "{{ chart_repo_url }}"
```

> Reference > https://github.com/bitnami-labs/sealed-secrets


#### Upfront Nginx Web Server on VM(s)

Define `webserver` group in `inventory/hosts`

```yml
ubuntu@master0:~/ansible$ cat inventory/hosts
[worker]
worker0	ansible_host=10.39.64.20
worker1 ansible_host=10.39.64.21

[controller]
master0 ansible_host=10.39.64.10

[master]
master0 ansible_host=10.39.64.10

[webserver]
webserver0 ansible_host=10.39.64.20

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Install `nginx` on VMs in `webserver` group

```yml
ubuntu@master0:~/ansible$ cat roles/nginx_server/tasks/main.yml
---
# tasks file for nginx_server

- name: Install the latest Nginx
  apt: name=nginx state=latest
- name: start nginx
  service:
    name: nginx
    state: started
```

Delete the installed `nginx` on VM

```sh
sudo apt-get --purge remove nginx-*
```

- LB
  - Get all `worker_node` from Kubernetes
  - Get `nodePort` from `service` in Kubernetes
  - Put the above into `/etc/nginx/nginx.conf` in upfront Nginx webServer(s)
- Customized SSL
- HA. Refer to Full HA of Nginx in Appendix
- Perf tuning, caching


#### Rook Storage Cluster with Ceph

- Pre-requisite

> Reference > https://rook.io/docs/rook/v1.6/ceph-prerequisites.html

Since I have only 3 nodes in Kubernetes cluster, including `master`, I have to utilize `master` as a `worker` for this testcase.

```sh
kubectl describe nodes k8s-master | grep "Taints:"
Taints:             node-role.kubernetes.io/master:NoSchedule
```

```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```

or

```sh
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

```sh
kubectl describe nodes k8s-master | grep "Taints:"
Taints:             <none>
```

- Disk Allocation

vm | device_id | size
-- | -- | ---
k8s_master | `/dev/sdc` | 20G
worker0 | `/dev/sda` | 20G
worker1 | `/dev/sdb` | 20G

- Download and Install

Use the most latest one when writing this document

```sh
git clone --single-branch --branch release-1.6 https://github.com/rook/rook.git
```

- Install by Helm3

> Reference > https://rook.io/docs/rook/v1.6/helm-operator.html

```sh
helm repo add rook-release https://charts.rook.io/release
kubectl create namespace rook-ceph
helm install --namespace rook-ceph rook-ceph rook-release/rook-ceph
```

> Reference > https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md

Or by YAML

```sh
cd ~/rook_release-1.6/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

```sh
cd ~/rook_release-1.6/cluster/examples/kubernetes/ceph
kubectl apply -f cluster.yaml
```

- Check Status

```sh
NAME                                                      READY   STATUS      RESTARTS   AGE
pod/csi-cephfsplugin-bfwps                                3/3     Running     0          7m43s
pod/csi-cephfsplugin-bqpwj                                3/3     Running     0          7m43s
pod/csi-cephfsplugin-cbrtd                                3/3     Running     0          7m43s
pod/csi-cephfsplugin-provisioner-767b98d587-h5b2w         6/6     Running     0          7m42s
pod/csi-cephfsplugin-provisioner-767b98d587-hmqgz         6/6     Running     0          7m42s
pod/csi-rbdplugin-g9fw4                                   3/3     Running     0          7m45s
pod/csi-rbdplugin-gsv4j                                   3/3     Running     0          7m45s
pod/csi-rbdplugin-provisioner-77f75f4469-6zgsq            6/6     Running     0          7m44s
pod/csi-rbdplugin-provisioner-77f75f4469-7vdsn            6/6     Running     0          7m44s
pod/csi-rbdplugin-z9gz6                                   3/3     Running     0          7m45s
pod/rook-ceph-crashcollector-k8s-master-58d59dbdc-gk2cd   1/1     Running     0          6m49s
pod/rook-ceph-crashcollector-worker0-759f7dcc58-p8ch7     1/1     Running     0          6m43s
pod/rook-ceph-crashcollector-worker1-7ff7697d4f-x7hqj     1/1     Running     0          6m48s
pod/rook-ceph-mgr-a-65647dfbf-v88pn                       1/1     Running     0          6m49s
pod/rook-ceph-mon-a-5fbd6986c4-c27mt                      1/1     Running     0          7m30s
pod/rook-ceph-mon-b-545f56fb8-dtrdk                       1/1     Running     0          7m18s
pod/rook-ceph-mon-c-56d5b88b8f-cdj2p                      1/1     Running     0          7m4s
pod/rook-ceph-operator-5d58889f96-zv2np                   1/1     Running     0          9m33s
pod/rook-ceph-osd-prepare-k8s-master-2sh98                0/1     Completed   0          6m7s
pod/rook-ceph-osd-prepare-worker0-qv6p5                   0/1     Completed   0          6m7s
pod/rook-ceph-osd-prepare-worker1-9fl2f                   0/1     Completed   0          6m6s
pod/rook-ceph-tools-599d59ffcc-52nt9                      1/1     Running     0          5m5s

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/csi-cephfsplugin-metrics   ClusterIP   10.100.58.141    <none>        8080/TCP,8081/TCP   7m42s
service/csi-rbdplugin-metrics      ClusterIP   10.105.85.232    <none>        8080/TCP,8081/TCP   7m44s
service/rook-ceph-mgr              ClusterIP   10.107.167.22    <none>        9283/TCP            6m43s
service/rook-ceph-mgr-dashboard    ClusterIP   10.107.27.0      <none>        8443/TCP            6m43s
service/rook-ceph-mon-a            ClusterIP   10.105.170.251   <none>        6789/TCP,3300/TCP   7m31s
service/rook-ceph-mon-b            ClusterIP   10.96.96.219     <none>        6789/TCP,3300/TCP   7m19s
service/rook-ceph-mon-c            ClusterIP   10.96.184.110    <none>        6789/TCP,3300/TCP   7m5s

NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/csi-cephfsplugin   3         3         3       3            3           <none>          7m43s
daemonset.apps/csi-rbdplugin      3         3         3       3            3           <none>          7m45s

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-cephfsplugin-provisioner          2/2     2            2           7m42s
deployment.apps/csi-rbdplugin-provisioner             2/2     2            2           7m44s
deployment.apps/rook-ceph-crashcollector-k8s-master   1/1     1            1           6m49s
deployment.apps/rook-ceph-crashcollector-worker0      1/1     1            1           6m43s
deployment.apps/rook-ceph-crashcollector-worker1      1/1     1            1           6m48s
deployment.apps/rook-ceph-mgr-a                       1/1     1            1           6m49s
deployment.apps/rook-ceph-mon-a                       1/1     1            1           7m30s
deployment.apps/rook-ceph-mon-b                       1/1     1            1           7m18s
deployment.apps/rook-ceph-mon-c                       1/1     1            1           7m4s
deployment.apps/rook-ceph-operator                    1/1     1            1           9m33s
deployment.apps/rook-ceph-tools                       1/1     1            1           5m5s

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-cephfsplugin-provisioner-767b98d587         2         2         2       7m42s
replicaset.apps/csi-rbdplugin-provisioner-77f75f4469            2         2         2       7m44s
replicaset.apps/rook-ceph-crashcollector-k8s-master-58d59dbdc   1         1         1       6m49s
replicaset.apps/rook-ceph-crashcollector-worker0-759f7dcc58     1         1         1       6m43s
replicaset.apps/rook-ceph-crashcollector-worker1-7ff7697d4f     1         1         1       6m48s
replicaset.apps/rook-ceph-mgr-a-65647dfbf                       1         1         1       6m49s
replicaset.apps/rook-ceph-mon-a-5fbd6986c4                      1         1         1       7m30s
replicaset.apps/rook-ceph-mon-b-545f56fb8                       1         1         1       7m18s
replicaset.apps/rook-ceph-mon-c-56d5b88b8f                      1         1         1       7m4s
replicaset.apps/rook-ceph-operator-5d58889f96                   1         1         1       9m33s
replicaset.apps/rook-ceph-tools-599d59ffcc                      1         1         1       5m5s

NAME                                         COMPLETIONS   DURATION   AGE
job.batch/rook-ceph-osd-prepare-k8s-master   1/1           7s         6m7s
job.batch/rook-ceph-osd-prepare-worker0      1/1           7s         6m7s
job.batch/rook-ceph-osd-prepare-worker1      1/1           6s         6m6s
```

> Wait for a while until you can see `rook-ceph-osd-prepare*` pods up

- Install Ceph Toolbox

> Reference > https://rook.io/docs/rook/v1.6/ceph-toolbox.html

```sh
cd ~/rook_release-1.6/cluster/examples/kubernetes/ceph
kubectl create -f toolbox.yaml
```

Check Ceph status
```sh
kubectl -n rook-ceph exec -it rook-ceph-tools-599d59ffcc-hm989 -- /bin/bash

kubectl -n rook-ceph exec -it $(k -n rook-ceph get pod -l "app=rook-ceph-tools" \
  -o jsonpath='{.items[0].metadata.name}') -- /bin/bash

ceph status
ceph osd status
ceph df
rados df
```

The output of running above commands

```sh
kubectl -n rook-ceph exec -it $(k -n rook-ceph get pod -l "app=rook-ceph-tools" \
  -o jsonpath='{.items[0].metadata.name}') -- /bin/bash

[root@rook-ceph-tools-599d59ffcc-xvhbr /]# ceph status
  cluster:
    id:     da1dd047-fc04-44ac-9aa9-0f7af8598378
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim

  services:
    mon: 3 daemons, quorum a,b,c (age 79s)
    mgr: a(active, since 9s)
    osd: 3 osds: 3 up (since 43s), 3 in (since 43s)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 57 GiB / 60 GiB avail
    pgs:     1 active+clean

[root@rook-ceph-tools-599d59ffcc-xvhbr /]# ceph osd status
ID  HOST         USED  AVAIL  WR OPS  WR DATA  RD OPS  RD DATA  STATE
 0  k8s-master  1026M  18.9G      0        0       0        0   exists,up
 1  worker1     1026M  18.9G      0        0       0        0   exists,up
 2  worker0     1026M  18.9G      0        0       0        0   exists,up

[root@rook-ceph-tools-599d59ffcc-xvhbr /]# ceph df
--- RAW STORAGE ---
CLASS  SIZE    AVAIL   USED     RAW USED  %RAW USED
hdd    60 GiB  57 GiB  6.2 MiB   3.0 GiB       5.01
TOTAL  60 GiB  57 GiB  6.2 MiB   3.0 GiB       5.01

--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS  USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0   0 B      0     18 GiB
```

- Teardown the Entire Rook-Ceph Cluster

> Reference >
> https://rook.github.io/docs/rook/v1.6/ceph-teardown.html
> https://github.com/rook/rook/blob/master/Documentation/ceph-common-issues.md

> **Notice**: `/var/lib/rook` must be cleaned-up: Path on each host in the cluster where configuration is cached by the ceph mons and osds (object storage device)

```sh
## This may take a while
kubectl -n rook-ceph delete cephclusters.ceph.rook.io rook-ceph

## Double check
kubectl -n rook-ceph get cephcluster
No resources found in rook-ceph namespace.

cd ~/rook_release-1.6/cluster/examples/kubernetes/ceph
kubectl delete -f operator.yaml
kubectl delete -f common.yaml
kubectl delete -f crds.yaml

## Run this on EACH of OSD node
sudo rm -fr /var/lib/rook/

helm -n rook-ceph delete rook-ceph
kubectl delete namespace rook-ceph
```

- Troubleshooting

Error message shown in `operator` pod log

```sh
kubectl -n rook-ceph logs -f rook-ceph-operator-c4459dbf7-426rk

2021-06-15 14:02:08.901199 I | cephosd: skipping osd.2: "35469c76-ee78-48f8-9141-1b6e6d95f66f" belonging to a different ceph cluster "50602fec-d650-4237-b748-06e040ad21e4"
2021-06-15 14:02:08.901205 I | cephosd: 0 ceph-volume raw osd devices configured on this node
```

Solution is to `dd` the target disk(s) to completely wipe-out previously configured cluster info, on EACH Ceph node

> **Warning**: this command will completely erase data on disk!

```sh
sudo dd if=/dev/zero of=/dev/sdX bs=1M status=progress
```

> Common issues > https://github.com/rook/rook/blob/master/Documentation/ceph-csi-troubleshooting.md

To remove OSD, DaveAck provides this link >
https://documentation.suse.com/ses/7/html/ses-all/storage-salt-cluster.html#osd-management-erasing

- Block Storage

> Reference > https://github.com/rook/rook/blob/master/Documentation/ceph-block.md

```sh
cd ~/rook_release-1.6/cluster/examples/kubernetes/ceph/csi/rbd$
kubectl create -f storageclass.yaml

kubectl -n rook-ceph get storageclasses.storage.k8s.io
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   2d
```
---

<center><font size=5>Beyond this point, VANTIQ deployment can start from now</font></center>

---



## Appendix

#### Full HA of Nginx

If there are 2 virtualIPs, bind them to both Nginx web server to maximize the utilization of 2 Nginx web servers simultaneously on production

```plantuml

cloud freeWorld

rectangle "dnsQuery: pis.malata.com" as dns

rectangle "publicVIP: 159.138.238.56" as pubvip

rectangle "priVIP: 10.0.20.13" as privip
rectangle "priVIP2: 10.0.20.14" as privip2

package nginx-web1 {
  rectangle "pubIP1: 159.138.254.213" as pubip1
  rectangle "internalIP1: 10.0.20.11" as intip1

  pubip1 -[hidden]- intip1
}

package nginx-web2 {
  rectangle "pubIP2: 159.138.235.98" as pubip2
  rectangle "internalIP2: 10.0.20.12" as intip2

  pubip2 -[hidden]- intip2
}

freeWorld --> dns
dns --> pubvip

pubvip --> privip
pubvip --> privip2

privip --> intip1
privip ..> intip2

privip2 ..> intip1
privip2 --> intip2
```

#### Reference

- Copy a file

  ```yml
  cat roles/os_hosts_cp/tasks/main.yml
  ---
  # tasks file for os_mod_hosts
  - name: Insert multiple lines and Backup
    blockinfile:
      path: /etc/hosts
      backup: yes
      block: |
        10.39.64.10	master0
        10.39.64.20	worker0
        10.39.64.21	worker1
  ```


#### Comparison between `terraform` and `ansible`

Clarification: there is no intention to have them compete each other. Actually they're not competitors either. It's not that when one is preferred and another one would be completely wrong. The reason of picking one is that better fits scenario for both technical and business perspectives.

Even the decision made might not be proper for any reason for longer term, such as tool deprecated, community no longer supported, we would be able to manage the switching effort at an acceptable and reasonable cost, to avoid disruptive change.

Whenever picking one is not intentional to replace another. Both Ansible and Terraform tools do a lot of things pretty well.

> And my personal preference is to **use Terraform for orchestration/ provisioning and Ansible for configuration management**.

**There is still some overlapped functions**

> When it comes to orchestration you can use the orchestration tools to not only provision servers, but also databases, caches, load balancers, queues, monitoring, subnet configuration, firewall settings, routing rules, SSL certificates, and almost every other aspect of your infrastructure, mainly public cloud infrastructure.

I can't agree 100%
> Ansible uses procedural style where you write the code that specifies, step-by-step tasks in order to achieve desired end state.

> Ref > https://linuxhandbook.com/terraform-vs-ansible/

Because

> Ansible allows you to write is a declarative code. Even though it executes the tasks in a serial order, which is why a lot of people think its procedural in a way. Here are the reasons why I believe Ansible is declarative,

> - Ansible allows you to write tasks where you focus on WHAT you want rather than HOW to achieve it. These tasks are then mapped to some underlying code (typically python however you could create modules in any language), which is a procedural code and is platform specific.
> - Ansible uses YAML as the language to create the playbooks, its way to define infrastructure as a code. Inherently YAML is a declarative language. All the tools that use YAML are essentially creating a declarative interface for their users e.g. kubernetes, docker compose etc.



> Ref > https://www.quora.com/Is-Ansible-procedural-or-declarative


#### Sample code

```sh
ubuntu@master0:~/ansible$ ansible all -a "df -h" -u root
master0 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           394M  712K  393M   1% /run
/dev/sdb1       124G  2.2G  122G   2% /
tmpfs           2.0G  124K  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/sdb15      105M  6.1M   99M   6% /boot/efi
/dev/sda1        16G   45M   15G   1% /mnt
tmpfs           394M     0  394M   0% /run/user/1000
tmpfs           394M     0  394M   0% /run/user/0
worker0 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           797M  684K  796M   1% /run
/dev/sda1       124G  1.7G  123G   2% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs           797M     0  797M   0% /run/user/0
worker1 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           797M  684K  796M   1% /run
/dev/sda1       124G  1.7G  123G   2% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs           797M     0  797M   0% /run/user/0
```

#### Format Raw Disk
```yml
- name: check data directory
  stat:
    path: /mnt/tidb
  register: data_dir_info

- block:
  - name: create vg
    shell: "vgcreate tidb /dev/{{ block_device }}"
  - name: create lv
    shell: "lvcreate -l 100%VG tidb -n data"
  - name: format lvs
    shell: mkfs.xfs /dev/tidb/data
  - name: create tidb dir
    file:
      path: /mnt/tidb
      state: directory
  - name: mount tidb
    mount:
      src: /dev/mapper/tidb-data
      path: /mnt/tidb
      state: mounted
      fstype: xfs
      boot: True
  - name: create pd dir
    file:
      path: /mnt/tidb/pd
      state: directory
  - name: create tikv dir
    file:
      path: /mnt/tidb/tikv
      state: directory
  when: data_dir_info.stat.exists == False or (data_dir_info.stat.exists == True and data_dir_info.stat.isdir == False)
~                            
```

#### Global Variable

where to set local private docker registry

```yml
task_id: 1

keepalived:
  id: 249
  vip: 3.1.5.249

kubernetes:
  version: v1.17.3
  repository: docker.repo.local:5000

nfs:
  server: ""
  path: ""

tidb:
  root_password: secret
  node_port: 14000

ntp:
  server:
  - ntp.ntsc.ac.cn
  - ntp1.aliyun.com
  - 210.72.145.44

kafka:
  ip: "{{ keepalived.vip }}"
  node_port: 19092

aiops:
  database: aiops_dev
```

#### Template in `j2`

```yml
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "insecure-registries": ["{{ kubernetes.repository }}"]
}
```

#### Task and Sub-Tasks

```yml
- name: install k8s components on debian
  include_tasks: debian.yaml
  when: ansible_distribution_file_variety == "Debian"

- name: install k8s components on redhat
  include_tasks: redhat.yaml
  when: ansible_distribution_file_variety == "RedHat"

- name: config docker
  template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json

- name: enable kubelt
  service:
    name: kubelet
    enabled: True

- name: enable docker
  service:
    name: docker
    state: started
    enabled: True

- name: install helm
  copy:
    src: helm
    dest: /usr/sbin/
    mode: 0777
  when: inventory_hostname in groups["master"]

- name: setup k8s cluster
  include_tasks: setup_k8s.yaml

- name: init k8s
  include_tasks: init_k8s.yaml
  when: inventory_hostname in groups["master"]
```

#### Rook Storage Cluster with Ceph through YAML

> Reference > https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md

- Pre-requisite

Since I have only 3 nodes in Kubernetes cluster, including `master`, I have to utilize `master` as a `worker` for this testcase.

```sh
kubectl describe nodes k8s-master | grep "Taints:"
Taints:             node-role.kubernetes.io/master:NoSchedule
```

```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```

or

```sh
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

```sh
kubectl describe nodes k8s-master | grep "Taints:"
Taints:             <none>
```

- Download and Install

Use the most latest one when writing this document

```sh
git clone --single-branch --branch release-1.6 https://github.com/rook/rook.git

ubuntu@master0:~/rook$ cd cluster/examples/kubernetes/ceph/
ubuntu@master0:~/rook/cluster/examples/kubernetes/ceph$ k create -f crds.yaml -f common.yaml -f operator.yaml
ubuntu@master0:~/rook/cluster/examples/kubernetes/ceph$ k create -f cluster.yaml
```

- Check the status

```sh
ubuntu@master0:~/rook/cluster/examples/kubernetes/ceph$ k get ns
NAME              STATUS   AGE
cert-manager      Active   5m12s
default           Active   6m3s
ingress-nginx     Active   5m20s
kube-node-lease   Active   6m4s
kube-public       Active   6m4s
kube-system       Active   6m4s
rook-ceph         Active   20s

ubuntu@master0:~/rook/cluster/examples/kubernetes/ceph$ k -n rook-ceph get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/rook-ceph-operator-5ddd885974-l85kg   1/1     Running   0          37s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rook-ceph-operator   1/1     1            1           37s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/rook-ceph-operator-5ddd885974   1         1         1       37s
```


## Troubleshooting

#### Terminating namespace gets stuck

After `kubectl delete namespace rook-ceph`, the `STATUS` stays at `Terminating` forever

```sh
ubuntu@master0:~$ k get ns
NAME              STATUS        AGE
cert-manager      Active        24h
default           Active        24h
ingress-nginx     Active        24h
kube-node-lease   Active        24h
kube-public       Active        24h
kube-system       Active        24h
rook-ceph         Terminating   24h
```

- Extract namespace's configuration

```sh
ubuntu@master0:~$ k get ns rook-ceph -o json > rook-ceph.json
```

- Edit the extracted configuration and remove `finalizers` array, from

```json
    "spec": {
        "finalizers": [
          "kubernetes"
        ]
    },
```

to

```json
    "spec": {
        "finalizers": [
        ]
    },
```

- Execute the cleanup command

```sh
ubuntu@master0:~$ k replace --raw "/api/v1/namespaces/rook-ceph/finalize" -f ./rook-ceph.json
```

#### Clean up Resources

- Delete several dozens `crd` (Customized Resource Definition)

```sh
for i in `kubectl -n rook-ceph get crd | grep -v NAME | awk '{print $1}'`; do kubectl -n rook-ceph delete crd $i; done
```

- Delete multiple `secrets`

```sh
for i in `k -n rook-ceph get secrets | grep -v NAME | awk '{print $1}'`; do k -n rook-ceph delete secrets $i; done
```
