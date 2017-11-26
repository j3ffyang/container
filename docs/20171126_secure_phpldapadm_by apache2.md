# Secure phpLDAPadmin by Apache2 Passwd Authentication and SSL

#### Document Objective
- As subject

#### Pre- requisite
- Apache2 and phpLDAPadmin have been installed and configured properly. phpLDAPadmin can be accessed via web
- htpasswd and apache2-utils have been installed

Reference Doc:
- [https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-apache-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-apache-on-ubuntu-14-04)
- [https://fxdata.cloud/tutorials/install-and-configure-openldap-and-phpldapadmin-on-an-ubuntu-14-04-server](https://fxdata.cloud/tutorials/install-and-configure-openldap-and-phpldapadmin-on-an-ubuntu-14-04-server)

## Deployment

#### Enable htpasswd

- Create passwd file

```
sudo htpasswd -c /etc/apache2/.htpasswd some_user
```

- Check the created passwd file by ```cat /etc/apache2/.htpasswd```

```
some_user:$apr1$lzxsIfXG$tmCvCfb49vpPFwKGVsuYz.
```

- Configure Apache Password Authentication

Look for ```Apache2``` configuration file. The default is located at ```/etc/apache2/sites-enabled/000-default.conf```. Add ```AuthType```, ```AuthName```, ```AuthUserFile``` and ``` Require``` 4 lines

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory "/var/www/html">
        AuthType Basic
        AuthName "Restricted Content"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>
</VirtualHost>
```

For __```phpLDAPadmin```__, ```/etc/phpldapadmin/apache.conf```, change like

```
<Directory /usr/share/phpldapadmin/htdocs/>

    DirectoryIndex index.php
    Options +FollowSymLinks
    AllowOverride None

    Order allow,deny
    Allow from all

        AuthType Basic
        AuthName "Restricted Content"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user

    <IfModule mod_mime.c>
```

- Then restart ```apache2```

```
systemctl restart apache2.service
```

#### Enable SSL/ HTTPS
- Create an SSL cert

```
sudo mkdir /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt
```

- Enable SSL module

```
sudo a2enmod ssl
```

- Configure the HTTP Virtual Host

```
sudo vi /etc/apache2/sites-enabled/000-default.conf
```

```
<VirtualHost *:80>
    ServerAdmin webmaster@server_domain_or_IP
    DocumentRoot /var/www/html
    ServerName server_domain_or_IP
    Redirect permanent /phpldapadmin https://server_domain_or_IP/phpldapadmin
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Configure the __HTTPS__ Virtual Host File

Enable SSL and it will create ```/etc/apache2/site-enabled/default-ssl.conf```

```
sudo a2ensite default-ssl.conf
```

- Edit ```sudo vi /etc/apache2/sites-enabled/default-ssl.conf```

Modify

```
ServerAdmin webmaster@server_domain_or_IP
ServerName server_domain_or_IP

SSLCertificateFile /etc/apache2/ssl/apache.crt
SSLCertificateKeyFile /etc/apache2/ssl/apache.key
```

- Restart

```
sudo systemctl restart apache2.service
```

Then we should have http://IP/phpldapadmin redirected to https://IP/phpldapadmin with ```htpasswd``` protected
