# Xray with VLess

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Objective](#objective)
- [Pre-requisites](#pre-requisites)
    - [You have a valid domain name](#you-have-a-valid-domain-name)
    - [CloudFlare (optional)](#cloudflare-optional)
    - [Domain Name Hosting Service Provider](#domain-name-hosting-service-provider)
- [Install `certbot` for `letsEncrypt`](#install-certbot-for-letsencrypt)
    - [(option I) Install plain `certbot`](#option-i-install-plain-certbot)
    - [Generate certificate](#generate-certificate)
    - [(option II) If using domain registration at CloudFlare](#option-ii-if-using-domain-registration-at-cloudflare)
    - [Generate certificate](#generate-certificate-1)
    - [Check certificate expiration](#check-certificate-expiration)
- [`nginx`](#nginx)
    - [`/etc/nginx/nginx.conf`](#etcnginxnginxconf)
    - [Configure `fallback`](#configure-fallback)
- [`xray`](#xray)
    - [Install](#install)
    - [Configure](#configure)
    - [Test Run](#test-run)
- [Client](#client)
    - [`v2ray-core`](#v2ray-core)
    - [A GUI `qv2ray`](#a-gui-qv2ray)
    - [Android `v2rayNG`](#android-v2rayng)
- [Troubleshooting](#troubleshooting)
    - [`xray` doesn't start properly](#xray-doesnt-start-properly)
    - [`nginx` can't start after default install on Debian](#nginx-cant-start-after-default-install-on-debian)

<!-- /code_chunk_output -->

## Objective

Get rid of unexpected firewall


> Reference > https://henrywithu.com/coexistence-of-web-applications-and-vless-tcp-xtls/

## Pre-requisites

#### You have a valid domain name

#### CloudFlare (optional)
- Use CloudFlare proxy to protect (hide/ conseal) your domain name
- In CloudFlare, create a web (with your domain name) and define your public IP address

#### Domain Name Hosting Service Provider

Eg. In Namecheap.com, under your domain name, select Custom DNS, then define DNS (pointing to CloudFlare's DNS)

- izabella.ns.cloudflare.com
- will.ns.cloudflare.com

## Install `certbot` for `letsEncrypt`

#### (option I) Install plain `certbot`

- Debian/ Ubunt
```sh
apt -y install curl git nginx libnginx-mod-stream python3-certbot-nginx
``` 

- CentOS

Install the very latest Nginx > http://nginx.org/en/linux_packages.html#RHEL-CentOS

```sh
sudo yum install epel-release
sudo dnf install curl git nginx nginx-mod-stream python3-certbot-nginx
```

```sh
[root@vultrguest ~]# chown nginx:nginx /usr/local/etc/xray/fullchain.pem
[root@vultrguest ~]# chown nginx:nginx /usr/local/etc/xray/privkey.pem
```

Disable SELinux

#### Generate certificate

```sh
certbot certonly --nginx
```

#### (option II) If using domain registration at CloudFlare

- DNS > Records > add `subdomain` with type `CNAME`
- SSL/TLS > Your SSL/TLS encryption mode is Full

> Reference > https://developers.cloudflare.com/dns/manage-dns-records/reference/proxied-dns-records/

- `certbot` with LetsEncrypt and CloudFlare

- Install `certbot` with cloudFlare plugin with `snapd`

> Reference > https://certbot.eff.org/instructions?ws=nginx&os=debianbuster&tab=wildcard

```sh
apt install snapd
snap install core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot

snap set certbot trust-plugin-with-root=ok

snap install certbot-dns-cloudflare
```

- Create `dns_cloudflare_api_key` for CloudFlare

> Reference > https://certbot-dns-cloudflare.readthedocs.io/en/stable/

```sh
cat ~/cloudflare/.secrets/cloudflare.ini

dns_cloudflare_email 	= "myEmail@Address"
dns_cloudflare_api_key	= "~~37digits~~"
```

#### Generate certificate

```sh
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /home/jeff/cloudflare/.secrets/cloudflare.ini \
  -d yourDomain.org \
  -d www.yourDomain.org
```

#### Check certificate expiration

```sh
openssl x509 -dates -noout < /path/fullchain.pem
notBefore=Nov 14 08:49:01 2021 GMT
notAfter=Feb 12 08:49:00 2022 GMT
```

---

## `nginx`

#### `/etc/nginx/nginx.conf`

```sh
apt -y install curl git nginx libnginx-mod-stream
```

Place the following content outside of `http {}` block

```sh
stream {
        map $ssl_preread_server_name $example_multi {
                webgame.example.com xtls;
                nextcloud.example.com nextcloud;
        }
        upstream xtls {
                server 127.0.0.1:20001; # Xray port
        }
        upstream nextcloud {
                server 127.0.0.1:20002; # Nextcloud port
        }
        server {
                listen 443      reuseport;
                listen [::]:443 reuseport;
                proxy_pass      $example_multi;
                ssl_preread     on;
        }
}
```

Replace `example.com` with your actual domain name

#### Configure `fallback` 

- Pull a cute site

```sh
cd /var/www/html; git clone https://github.com/gd4Ark/2048.git 2048
```

- Edit `/etc/nginx/conf.d/fallback.conf`

```sh
server {
        listen 80;
        server_name webgame.example.com;
        if ($host = webgame.example.com) {
                return 301 https://$host$request_uri;
        }
        server_name nextcloud.example.com;
        if ($host = nextcloud.example.com) {
                return 301 https://$host$request_uri;
        }
        return 404;
}

server {
        listen 127.0.0.1:20009;
        server_name webgame.example.com;
        index index.html;
        root /var/www/html/2048;
}
```

## `xray`

#### Install 

```sh
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

- Uninstall Xray

```sh
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ remove
```

#### Configure

- Generate a UUID for key

```sh
cat /proc/sys/kernel/random/uuid
```

- Edit `config.json` for Xray

```sh
vi /usr/local/etc/xray/config.json
```

Replace the value of `id` with the output

```json
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "listen": "127.0.0.1", # Only listen locally to prevent detection of the following port 20001
            "port": 20001, # The port here corresponds to the upstream port in Nginx
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "1c0022ae-3f18-41ab-93a7-5e85cf883fde", # fill in your UUID
                        "flow": "xtls-rprx-direct",
                        "level": 0
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": "20009" # port of fallback site
                    }
                 ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/usr/local/etc/xray/fullchain.pem", # your domain cert, absolute path
                            "keyFile": "/usr/local/etc/xray/privkey.pem" # your private key, absolute path
                        }
                    ]
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```

- Ignore `nextcloud` Part

#### Test Run

```sh
ubuntu@master0:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

ubuntu@master0:~$ xray --test
Xray 1.4.2 (Xray, Penetrates Everything.) Custom (go1.16.2 linux/amd64)
A unified platform for anti-censorship.
2021/06/08 02:56:03 Using config from STDIN
2021/06/08 02:56:03 [Info] infra/conf/serial: Reading config: stdin:
```

Restart

```sh
systemctl restart nginx xray
```


## Client

#### `v2ray-core`

> Reference > https://github.com/v2ray/v2ray-core

#### A GUI `qv2ray`

> Reference > https://github.com/Qv2ray/Qv2ray

After install, Check V2Ray Core Settings - `Preferences` > `Kernel Settings` > `Check V2Ray Core Settings`.

#### Android `v2rayNG`

> Reference > https://github.com/2dust/v2rayNG

param | value
-- | --
Host | webgame.example.com
Port | 443
Type | VLESS
UUID | the one in `config.json`
Flow | `xtls-rprx-direct`
Transport Protocal | tcp
Security Type | XTLS

## Troubleshooting

#### `xray` doesn't start properly

- `/var/log/nginx/error.log` shows

```sh
2022/05/21 10:36:46 [error] 250480#250480: *8 connect() failed (111: Connection refused) while connecting to upstream, client: 114.253.110.206, server: 0.0.0.0:443, upstream: "127.0.0.1:20001", bytes from/to client:0/0, bytes from/to upstream:0/0
```

- And `systemctl status xray` has not started properly

```sh
● xray.service - Xray Service
     Loaded: loaded (/etc/systemd/system/xray.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/xray.service.d
             └─10-donot_touch_single_conf.conf
     Active: failed (Result: exit-code) since Sat 2022-05-21 10:40:12 CST; 4s ago
       Docs: https://github.com/xtls
    Process: 251119 ExecStart=/usr/local/bin/xray run -config /usr/local/etc/xray/config.json (code=exited, status=23)
   Main PID: 251119 (code=exited, status=23)

May 21 10:40:12 VM-4-15-ubuntu systemd[1]: Started Xray Service.
May 21 10:40:12 VM-4-15-ubuntu xray[251119]: Xray 1.5.5 (Xray, Penetrates Everything.) Custom (go1.18.1 linux/amd64)
May 21 10:40:12 VM-4-15-ubuntu xray[251119]: A unified platform for anti-censorship.
May 21 10:40:12 VM-4-15-ubuntu xray[251119]: 2022/05/21 10:40:12 [Info] infra/conf/serial: Reading config: /usr/local/etc/xray/config.json
May 21 10:40:12 VM-4-15-ubuntu xray[251119]: Failed to start: main: failed to load config files: [/usr/local/etc/xray/config.json] > infra/conf: Failed to build XTLS config. > infra/conf: failed to parse key > open /usr/local/etc/xray/privkey.pem: permission denied
```

- Check `privkey.pem` permission

```sh
jeff@VM-4-15-ubuntu:/usr/local/etc/xray$ ls -la
total 28
drwxr-xr-x 2 root root 4096 May 21 10:39 .
drwxr-xr-x 3 root root 4096 May 20 11:00 ..
-rw-r--r-- 1 root root 1501 May 21 10:39 config.json
-rw-r--r-- 1 root root 5604 May 21 10:16 fullchain.pem
-rw------- 1 root root 1704 May 21 10:16 privkey.pem
```

You may want to grant `nobody`:`www-data` permission to access both `fullchain.pem` and `privkey.pem`

#### `nginx` can't start after default install on Debian

> Reference > https://bugs.launchpad.net/ubuntu/+source/nginx/+bug/1581864

- `fullchain.pem: permission denied` on Debian, even the ownership of `fullchain.pem` and `privkey.pem` is `www-data`, which owns `nginx worker` process

```sh
Feb 08 04:20:47 localhost xray[17384]: Failed to start: main: failed to load config files: [/usr/local/etc/xray/config.json] > infra/conf: Failed to build XTLS config. > infra/conf: failed to parse certificate > open /etc/letsencrypt/live/domainName/fullchain.pem: permission denied
```

You may want to grant `nobody`:`www-data` permission to access both `fullchain.pem` and `privkey.pem`
