# Ubuntu Server behind Proxy / Firewall

Reference > http://www.iasptk.com/ubuntu-server-behind-proxy-firewall/

#### Make your environment aware of the local proxy

```sh
sudo nano /etc/environment
```

Add the following three lines to the environment file.

```sh
http_proxy=http://127.0.0.1:3128
https_proxy=http://127.0.0.1:3128
ftp_proxy=http://127.0.0.1:3128
```

#### ```apt-get```

```sh
sudo nano /etc/apt/apt.conf.d/proxy
```

Add the following three lines to the proxy file.

```sh
Acquire::http::Proxy “http://127.0.0.1:3128/”;
Acquire::https::Proxy “http://127.0.0.1:3128/”;
Acquire::ftp::Proxy “http://127.0.0.1:3128/”;
```

Force using IPv4, touch ```/etc/apt/apt.conf.d/99force-ipv4``` and add

```sh
Acquire::ForceIPv4 "true";
```

Or just run

```sh
apt-get -o Acquire::ForceIPv4=true update
```

#### ```wget```

Configuring the ```wgetrc``` file

Like most of the applications wget has a configuration file too – wgetrc:

```sh
/etc/wgetrc
~/.wgetrc
```

```sh
sudo nano /etc/wgetrc

# You can set the default proxies for Wget to use for http, https, and ftp.
# They will override the value in the environment.
#https_proxy = http://proxy.yoyodyne.com:18023/
#http_proxy = http://proxy.yoyodyne.com:18023/
#ftp_proxy = http://proxy.yoyodyne.com:18023/

# If you do not want to use proxy at all, set this to off.
#use_proxy = on
```

#### ```curl```

create the file (if it does not exist) :

```sh
~/.curlrc
```

add the line:

```sh
proxy = <proxy_host>:<proxy_port>
```

```sh
curl -I -k http://ip  # to ignore SSL
```

#### ```apt-key add```

```sh
sudo apt-key adv –keyserver hkp://keyserver.ubuntu.com:80 –recv 7F0CEB10
```

OR

Save the key to a text file some.key and import it with

```sh
cat some.key | sudo apt-key add -
```

#### ```add-apt-repository```

Set the variable https_proxy to your proxy

Edit ```/etc/sudoers``` or the correct file in ```/etc/sudoers.d/``` so it contains:


```sh
Defaults env_keep = https_proxy
```

#### PHP PEAR

```sh
sudo pear config-set http_proxy http://127.0.0.1:3128
```

#### RUBY GEM INSTALL

```sh
sudo gem install –http-proxy http://127.0.0.1:3128 bundler
```

(http://proxy-ip:port) (bundler = the name of the gem)

#### BUNDLER

```sh
sudo -E bundle install
```

Reference > https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/installing/install_proxy.html
