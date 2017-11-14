# Create a New User in OpenVPN and OpenLDAP

## Document Objective
- Create new user in both OpenLDAP and OpenVPN
- A new user _must_ have both authentications passed prior to get into the system

## Steps

#### 1\. Create a New User in OpenVPN (on cm01)

- Log into ```ssh ubuntu@cm01```

- Generate keys (\*.crt, \*.csr, \*.key) for ```client1```
```
su - ubuntu
cd ~/openvpn-ca
source vars
./built-key client1
```

- Create \*.ovpn for ```client1```
```
cd ~/client-configs
./make_config.sh client1
```

- Ship \*.ovpn file to the new user

> Note: \*.ovpn file is confidential. Whoever grabs this file will become you in our system.

#### 2\. Create a New User in OpenLDAP through phpLDAPadmin

- Log in https://172.29.167.177/phpldapadmin as ```admin```

<!--
- Create and fill- in information like the following figures

<center><img src="../imgs/20171017_ldap0.png" width="650px"></center>

<center><img src="../imgs/20171017_ldap1.png" width="650px"></center>
-->

> Note: make sure ```Email``` and ```User ID``` are in the same pattern as the samples, or the user might __not__ be able to log into LDAP

- Inform the new user about his/ her ```User ID```, ```Email``` and ```password``` of LDAP.

- You can change your own __LDAP__ password in phpLDAPadmin UI

#### 3\. (optional when necessary) Create a User in Ubuntu
```
sudo adduser client1
sudo usermod -aG sudo client1
```
