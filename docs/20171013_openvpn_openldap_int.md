# Integrate OpenVPN with OpenLDAP

## Document Objective
- As subject

#### Reference Document
- [DigitalOcean: Encrypt OpenLDAP using StartTLS](https://www.digitalocean.com/community/tutorials/how-to-encrypt-openldap-connections-using-starttls)

> Note: Pay attention to __Configuring Remote Clients__ part and read the Q&A at the bottom of above tutorial

#### Environment and Pre- requisite
- OpenVPN is installed on cm01, as OpenLDAP client
- OpenLDAP is running on cm02 (in our environment, StartTLS is enabled)
- From OpenVPN (as OpenLDAP client), connection can be established

```
ubuntu@cm01:/etc/ssl/certs$ ldapsearch -H ldap://cm02.devops.org -x -b "dc=devops,dc=org" -LLL -Z dn
dn: dc=devops,dc=org

dn: cn=admin,dc=devops,dc=org

dn: cn=jeff yang,dc=devops,dc=org
```
- Install ```openvpn-auth-ldap``` on __OpenVPN box__

## Steps
#### Make sure OpenLDAP ```ca_server.pem``` is on OpenVPN_box

```
scp root@cm02.devops.org:/etc/ssl/certs/ca_server.pem ~/
cat ~/ca_server.pem | sudo tee -a /etc/ldap/ca_certs.pem
```

#### Specify ```ca_server.pem``` in ```/etc/ldap/ldap.conf```

```
TLS_CACERT /etc/ldap/ca_certs.pem
```

#### Add ```auth-ldap``` plugin in ```/etc/openvpn/server.conf```

```
plugin /usr/lib/openvpn/openvpn-auth-ldap.so \
     /etc/openvpn/auth/auth-ldap.conf
client-cert-not-required
```

###### Sample of ```/etc/openvpn/auth/auth-ldap.conf```

```
<LDAP>
	# LDAP server URL
	URL		ldap://cm02.devops.org

	# Bind DN (If your LDAP server doesn't support anonymous binds)
	# BindDN		uid=Manager,ou=People,dc=example,dc=com

	# Bind Password
	# Password	SecretPassword

	# Network timeout (in seconds)
	Timeout		15

	# Enable Start TLS
	TLSEnable	yes

	# Follow LDAP Referrals (anonymously)
	FollowReferrals yes

	# TLS CA Certificate File
	TLSCACertFile	/etc/ldap/ca_certs.pem

	# TLS CA Certificate Directory
	TLSCACertDir	/etc/ssl/certs

	# Client Certificate and key
	# If TLS client authentication is required
	# TLSCertFile	/usr/local/etc/ssl/client-cert.pem
	# TLSKeyFile	/usr/local/etc/ssl/client-key.pem

	# Cipher Suite
	# The defaults are usually fine here
	# TLSCipherSuite	ALL:!ADH:@STRENGTH
</LDAP>

<Authorization>
	# Base DN
	BaseDN		"dc=devops,dc=org"

	# User Search Filter
	# SearchFilter	"(&(uid=%u)(accountStatus=active))"
	# SearchFilter	"(&(cn=*)(objectclass=user))"
	# SearchFilter	"(&(cn=%u)(accountStatus=active))"
	SearchFilter "(&(uid=%u))"

	# Require Group Membership
	RequireGroup	false

	# Add non-group members to a PF table (disabled)
	#PFTable	ips_vpn_users

	<Group>
		BaseDN		"ou=Groups,dc=example,dc=com"
		SearchFilter	"(|(cn=developers)(cn=artists))"
		MemberAttribute	uniqueMember
		# Add group members to a PF table (disabled)
		#PFTable	ips_vpn_eng
	</Group>
</Authorization>
```

Restart ```systemctl restart openvpn@server```

#### Edit ```~/client-configs/base.conf```

```
...
;mute 20

auth-user-pass
```

Then you might want to re- generate client key(s)
