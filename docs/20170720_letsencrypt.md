## Document Objective
- Enable HTTPS & SSL/TLS
- Redirect http to https
- Create www sub- domain
- Test and Improve SSL/TLS

## Reference
[Step by Step Configuration @ https://serverfault.com/.../lets-encrypt-with-an-nginx-reverse-proxy](https://serverfault.com/questions/768509/lets-encrypt-with-an-nginx-reverse-proxy)

[Improve Nginx SSL @ https://scaron.info/](https://scaron.info/blog/improve-your-nginx-ssl-configuration.html)

[Test and Analyze Your SSL @ https://www.ssllabs.com/...](https://www.ssllabs.com/ssltest/analyze.html?d=daotech.io)

## Install LetsEncrypt
There are multiple methods to install and generate LetsEncrypt Certificates. In our case, we use __Certbot__ in command line, by following [https://certbot.eff.org](https://certbot.eff.org/docs/intro.html#installation)

## Install Certbot

Reference [https://certbot.eff.org/#ubuntuxenial-nginx](https://certbot.eff.org/#ubuntuxenial-nginx)

```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

## Request and Generate Certificates
```shell
sudo certbot certonly --webroot \
  --webroot-path=/data/wordpress/html \
  -d daotech.io -d www.daotech.io
```

where ```/data/wordpress/html``` is ```DocumentRoot``` of Wordpress

After the command running finishes, there are certificate files generated

```shell
root@ubuntu:/etc/letsencrypt/live/daotech.io# pwd
/etc/letsencrypt/live/daotech.io
root@ubuntu:/etc/letsencrypt/live/daotech.io# ls
cert.pem  chain.pem  fullchain.pem  privkey.pem  README
```

## Configure Nginx

- Configure Nginx to Use the Certificates

```shell
server {
    # where 2 pem files go
    ssl_certificate /etc/letsencrypt/live/daotech.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/daotech.io/privkey.pem;

    # Turn on OCSP stapling as recommended at
    # https://community.letsencrypt.org/t/integration-guide/13123
    # requires nginx version >= 1.3.7
    ssl_stapling on;
    ssl_stapling_verify on;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  # drop SSLv3 (POODLE vulnerability)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK';

    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    # All LetsEncrypt verify and renew the keys
    location /.well-known {
	allow all;
	alias /data/wordpress/html/.well-known;
    }
}
```

- Redirect HTTP to HTTPS/ SSL

```shell
server {
    listen  80;
    server_name daotech.io www.daotech.io;
    rewrite ^ https://$host$request_uri? permanent;
}

server {
    listen  443 ssl;
    server_name daotech.io www.daotech.io;
#   rewrite     ^   https://$host$request_uri? permanent;
}
```

- Nginx Reverse Proxy

```shell
server {
    location / {
    proxy_pass  http://127.0.0.1:8080;
	proxy_redirect 		off;
	proxy_set_header    Host                $host;
	proxy_set_header    X-Real-IP           $remote_addr;
	proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
	proxy_set_header    X-Forwarded-Proto   $scheme;
	proxy_set_header    Accept-Encoding     "";
	proxy_set_header    Proxy               "";
    }
}
```

## Enhance Nginx SSL/TLS
- First, generate your DH parameters with OpenSSL:

```shell
cd /etc/ssl/certs
openssl dhparam -out dhparam.pem 4096
```
- Then, add the following to your Nginx configuration:

```shell
ssl_dhparam /etc/ssl/certs/dhparam.pem;
```

## Auto Renew Certificates

Run by __root__
```shell
ubuntu@ubuntu:~$ su -
Password:
root@ubuntu:~# certbot renew --renew-hook "systemctl reload nginx"
Saving debug log to /var/log/letsencrypt/letsencrypt.log

-------------------------------------------------------------------------------
Processing /etc/letsencrypt/renewal/daotech.io.conf
-------------------------------------------------------------------------------
Cert not yet due for renewal

The following certs are not due for renewal yet:
  /etc/letsencrypt/live/daotech.io/fullchain.pem (skipped)
No renewals were attempted.
No hooks were run.
```

You can test renewals with either a dry-run, which will contact Let's Encrypt staging servers to do a real test of contacting your domain, but won't store the resulting certificates:

```shell
certbot --dry-run renew
```
Or you can force an early __renewal__ with: (tested)

```
certbot renew --force-renew --renew-hook "service nginx reload"
```

Or just run like creating new (tested)

```shell
sudo certbot certonly --webroot \
  --webroot-path=/data/wordpress/html \
  -d daotech.io -d www.daotech.io
```

Then ```systemctl restart nginx.service"

Configure a cronjob to automate the renew execution
```shell
# at 4:47am/pm, renew all Let's Encrypt certificates over 60 days old
47 4,16   * * *   root   certbot renew --quiet --renew-hook "service nginx reload"
```

#### Test HTTPS/ SSL

Fo example, hit
[https://www.ssllabs.com/ssltest/analyze.html?d=daotech.io](https://www.ssllabs.com/ssltest/analyze.html?d=daotech.io)
