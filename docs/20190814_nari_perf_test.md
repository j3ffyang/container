
# 4.2 系统配置，性能测试和健康检查

## 4.2.1 框架和部署设计审核

## 4.2.2 Linux 操作系统

#### Linux 版本和内核

目标 | 命令
-- | --
检查Linux版本（按Ubuntu为例）| ```cat /etc/lsb-release```
检查内核参数 | ```sysctl -a```
CPU | ```cat /proc/cpuinfo```
内存 | ```cat /proc/meminfo```
内存使用状况 | ```vmstat```

#### 文件系统

目标 | 命令
-- | --
当下文件打开状况 | ```lsof | wc```
文件系统类型 | ```df -Th``` or ```lsblk -f```
文件系统空间使用状态 | ```df```

#### 进程

目标 | 命令
-- | --
所有进程 | ```ps -ef```
进程运行 | ```top``` and ```htop```

#### 网络

目标 | 命令
-- | --
路由表 | ```ip route```
IP地址 | ```ip addr```
MTU | ```ip a | grep mtu```
网络PING性能和稳定性 | 如 ```ping 208.67.222.222```
DNS配置（按Ubuntu 18.04 LTS为例）| ```cat /etc/resolv.conf``` and ```cat /run/systemd/resolve/stub-resolv.conf```
网络当下连接状况 | ```netstat -pant```
网络分析工具 | ```tcpdump -i eth0```
软件防火墙状态 | ```sudo ufw status``` （以Ubuntu为例）


#### 磁盘

目标 | 命令
-- | --
块存储配置 | ```lsblk```
分区 | ```sudo fdisk -l```
性能 | ```sudo hdparm -Tt /dev/sda```
Mount 表 | ```cat /etc/fstab```
当下mount状态 | ```mount```
i/o状态 | ```iostat```

## 4.2.3 Kubernetes 和 Docker

#### Docker

目标 | 命令
-- | --
Docker 版本 | ```docker --version```
查看 docker 基本配置 | ```ls /etc/docker/```
列出所有镜像 | ```docker image ls```
列出所有运行的容器 | ```docker ps -a```

#### Kubernetes

防火墙端口配置参照 https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

目标 | 命令
-- | --
版本 | ```kubectl version``` and ```kubeadm version```
所有节点 | ```kubectl get node```
检查镜像列表 | ```kubeadm config images list```
所有namespace的容器 | ```kubectl get pods --all-namespaces```
所有namespace的容器（按镜像分类） | ```kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq -c```
所有运行的容器（按POD分类） | ```kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' | sort```
所有service列表 | ```kubectl get svc --all-namespaces```
检查kube-system 命名空间 | ```kubectl -n kube-system get all```
检查default 命名空间 | ```kubectl -n default get all```
检查 coreDNS 配置 | ```kubectl -n kube-system get deployment.apps/coredns -o yaml```
检查 persistent volume 和 persistent volume claim 的配置 | ```kubectl get pv``` and ```kubectl get pvc```

参照
- https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images/
- https://github.com/coredns/deployment/blob/master/kubernetes/Scaling_CoreDNS.md
- https://coredns.io/plugins/kubernetes/

#### Helm

目标 | 命令
-- | --
镜像库列表 | ```./helm repo list```

#### 网络插件配置（以 Flannel为例）

目标 | 命令
-- | --
Flannel配置 | ```cat /run/flannel/subnet.env```


- https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c
- https://rancher.com/blog/2019/2019-03-21-comparing-kubernetes-cni-providers-flannel-calico-canal-and-weave/

#### 性能 (以100～ 1000个POD为例)

- 集群数量和 Master数量
- API 请求 (API-responsiveness)
- Pod 起停时间 (startup)
- Scheduler (etcd) throughput 性能 (CPU, memory, IO)

参照
- https://kubernetes.io/blog/2015/09/kubernetes-performance-measurements-and/
- https://www.mirantis.com/blog/scale-performance-testing-kubernetes-answers-specific-applications/
- https://coreos.com/blog/improving-kubernetes-scheduler-performance.html
- https://solinea.com/blog/kubernetes-performance-testing
- https://github.com/kubernetes/perf-tests
