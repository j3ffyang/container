# HAProxy and keepAlived for Multiple Kubernetes Master Nodes

#### Background

Several months ago, in a project, I was trying to deploy a Kubernetes cluster with `kubespray`. Considering to build an high available cluster with multiple master nodes and separate `etcd` (not in container but directly on file system), an `haproxy` can be a good option. I built my own `haproxy` to enable a floating IP for __multiple Kubernetes APIs__ to which worker nodes connect.

#### Architecture

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

> The above diagram is drawn in `plantuml` format. It can be displayed in Atom or Sublime with `plantuml` plugin enabled.

Description:

- `haproxy` and `keepalived` are installed on `10.0.10.14` and `10.0.10.15` VMs respectively
- `10.0.10.200` is the floating virtual IP failover between `10.0.10.14` and `10.0.10.15`
- Load Balancing (LB) service, bound to `10.0.10.200` over port `8383` (this will be also defined and used in Ansible deployment code as well). Request hitting `10.0.10.200` over port `8383` will be forwarded to those 3 Kubernetes master nodes (`10.0.10.14/15/16`) over port `6443`

> Note: `etcd` cluster is created by Ansible later

> Reference >
> https://kubesphere.io/docs/installing-on-linux/high-availability-configurations/set-up-ha-cluster-using-keepalived-haproxy/
> https://kubespray.io/#/docs/ha-mode

#### Install

- Install `haproxy` and `keepalived` on `10.0.10.14` and `10.0.10.15`

```sh
for i in {14..15}; do ssh root@node$i "apt install keepalived haproxy -y"; done
```

#### Configure `haproxy`

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

#### Configure `keepalived`

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
    10.0.10.200/24              # The virtualIP address
  }

  track_script {
    chk_haproxy
  }
}
```

Change `unicase_arc_ip` and `unicast_peer` if adding additional peer(s)

- Restart `haproxy` and `keepalived`

```sh
systemctl restart haproxy keepalived
systemctl enable haproxy keepalived
```

#### Test

```sh
systemctl stop haproxy
```

This will force the floatIP floating to the peer of `haproxy`

> Reference > https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md
