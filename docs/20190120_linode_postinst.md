## Post-Install after You Get a Linode

#### Assumption
The base OS chosen is Ubuntu 18.04

#### Turn on ```vi``` mode in ```/etc/bash.bashrc```
This is optional but important. Add the following into ```/etc/bash.bashrc```

```
set -o vi
```

Then execute ```source /etc/bash.bashrc```

#### Turn on firewall

```
ufw allow 1194/udp
ufw allow ssh/tcp

ufw enable
ufw
```

#### Add non-root user with ```sudo``` allowed
The objective of this is to disable ```root``` login through SSH even only key authentication is allowed.

- Create non-root user
```
adduser nonroot
usermod -aG sudo nonroot
```

- Switch to ```nonroot``` user and place its ssh pub_key
```
su - nonroot
ssh localhost
cd ~/.ssh/
cat id_rsa.pub >> ~/.ssh/authorized_keys
```

#### Disable ```root``` login in ```/etc/ssh/sshd_config``` and ```PasswordAuthentication```

```
PermitRootLogin no
PasswordAuthentication no
```

Then ```sudo systemctl restart sshd.service```
