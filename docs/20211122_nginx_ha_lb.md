
# Nginx High Availability and Load Balancing with KeepAlived

#### Objective

Install and configure 2 Nginx web services, that do high availability, load balancing and proxy (forward) in-bound request to internal application (running inside of Kubernetes cluster)

#### Nginx High Availability

- Environment

. | internal IP | external IP
-- | -- | --
web01 | 10.30.10.19 | 1.2.3.4
web02 | 10.30.10.11 | 1.2.3.5
floatingIP | 10.30.10.100 | 1.2.3.6

Legend: __external IP__ = public internet IP

- Architecture Diagram (active-passive. Active-active is described later in this doc)

```plantuml
cloud freeWorld

rectangle "dnsQuery: app.domain.com" as dns
rectangle "publicIP: 1.2.3.6" as pubip
rectangle "floatingIP: 10.30.10.100" as fip
rectangle "web01: 10.30.10.19" as m01
rectangle "web02: 10.30.10.11" as m02

freeWorld --> dns
dns --> pubip
pubip --> fip:nat
fip --> m01
fip --> m02
```

#### Upfront Nginx web service

- Install packages on both nodes (web01 and web02) on Ubuntu

```sh
sudo apt install nginx keepalived
```

The following is part of `/etc/nginx/nginx.conf`

```sh
 include /etc/nginx/conf.d/*.conf;
 include /etc/nginx/sites-enabled/*;

 merge_slashes off;

 # set client body size to 2M #
 client_max_body_size 2M;

 upstream app.domain.com {
         server 10.30.10.9:31399;
         server 10.30.10.5:31399;
         server 10.30.10.6:31399;
         server 10.30.10.24:31399;
         server 10.30.10.22:31399;
         server 10.30.10.17:31399;
         server 10.30.10.25:31399;
         server 10.30.10.32:31399;
         server 10.30.10.8:31399;
         server 10.30.10.23:31399;
         server 10.30.10.15:31399;
         server 10.30.10.42:31399;
 }

 server {
     ssl_certificate /home/ubuntu/certificates/cert.txt;
     ssl_certificate_key /home/ubuntu/certificates/key.txt;
     # include /etc/letsencrypt/options-ssl-nginx.conf;
     # ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

     listen       443 ssl;
     server_name  app.domain.com;

     location / {
         # proxy_pass https://iot.hostname.net:31712/;
         proxy_pass https://app.domain.com/;
     }

     location /api/v1/wsock/websocket {
         proxy_pass https://app.domain.com;
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection "upgrade";
         proxy_read_timeout 86400;
    }
 }
```

> - Port `31399`: `nodePort` exposed on Kubernetes cluster
> - `client_max_body_size 2M;` is to set client body size to 2M


- `/etc/keepalived/keepalived.conf`

On web01

```sh
! Configuration File for keepalived

global_defs {
...
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 50
}

vrrp_instance VI_1 {
    state MASTER
    interface ens3
    virtual_router_id 50
    priority 110
    advert_int 1
    virtual_ipaddress {
	10.30.10.100
    }
    track_script {
        check_nginx
    }
}
```

On web02

```sh
! Configuration File for keepalived

global_defs {
...
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 50
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 50
    priority 100
    advert_int 1
    virtual_ipaddress {
	10.30.10.100
    }
    track_script {
	check_nginx
    }
}
```

- `/etc/keepalived/check_nginx.sh`

```sh
#!/bin/sh

if [ -z "`/bin/pidof nginx`" ]; then
	systemctl stop keepalived.service
	exit 1
fi
```

```sh
sudo chmod +x /etc/keepalived/check_nginx.sh
```

> Nginx **should** start before keepalived starts!

- Check which node the floating IP is being attached to

```sh
$ curl http://10.30.10.100 # or curl http://1.2.3.6
```

```sh
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx web01!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

#### Full HA of Nginx (active-active)

If there are 2 virtualIPs, bind them to both Nginx web server to maximize the utilization of 2 Nginx web servers simultaneously on production

```plantuml

cloud freeWorld

rectangle "dnsQuery: app.domain.com" as dns

rectangle "publicVIP: 1.2.3.6" as pubvip

rectangle "priVIP: 10.30.10.13" as privip
rectangle "priVIP2: 10.30.10.14" as privip2

package nginx-web1 {
  rectangle "pubIP1: 1.2.3.4" as pubip1
  rectangle "internalIP1: 10.30.10.11" as intip1

  pubip1 -[hidden]- intip1
}

package nginx-web2 {
  rectangle "pubIP2: 1.2.3.5" as pubip2
  rectangle "internalIP2: 10.30.10.12" as intip2

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
