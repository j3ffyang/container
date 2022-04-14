# Configure PEAP-MSCHAPv2 Wireless on Fedora Linux

## Objective
Configure protected-EAP with MSCHAPv2 encryption on Fedora Linux natively installed on MacBookPro16,2 hardware

## Environment Specification

- OS release > `cat /etc/*release*`
```sh
NAME="Fedora Linux"
VERSION="35 (Workstation Edition)"
```

- Kernel > `uname -a`
```sh
Linux mbp 5.16.8-200.mbp.fc33.x86_64 #1 SMP PREEMPT Mon Feb 14 06:11:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

- Hardware > `dmidecode | grep -i "system information" -A3`
```sh
System Information
	Manufacturer: Apple Inc.
	Product Name: MacBookPro16,2
	Version: 1.0
```

## Configuration

1. Download and place the certificate for `MSCHAPv2` encryption. In theory, the certificate must be placed anywhere system-wide readable and avoid being placed under `~/` which has protection tag. Or `wpa_supplicant` daemon won't be able to read

```sh
[jeff@mbp anchors]$ pwd
/etc/pki/ca-trust/source/anchors
[jeff@mbp anchors]$ ls
wireless_rootca.crt
```

2. Create a network through Gnome Settings > Wi-Fi. Here's an example
Note: the company's wireless SSID is `XXXXX_WLAN(5GHz)` specifically, created on Windows :-(

```sh
[jeff@mbp system-connections]$ pwd
/etc/NetworkManager/system-connections

[jeff@mbp system-connections]$ sudo cat XXXXX_WLAN\(5GHz\)-0fb0xxxx-xxxx-xxxx-xxxx-9c34xxxx5547.nmconnection
[sudo] password for jeff:
[connection]
id=XXXXX_WLAN(5GHz)
uuid=0fb0xxxx-xxxx-xxxx-xxxx-9c34xxxx5547
type=wifi
interface-name=wlp229s0
permissions=

[wifi]
mac-address-blacklist=
mode=infrastructure
ssid=XXXXX_WLAN(5GHz)

[wifi-security]
auth-alg=open
key-mgmt=wpa-eap

[802-1x]
ca-cert=/etc/pki/ca-trust/source/anchors/20220411_XX_rootca.crt
eap=peap;
identity=userId
password=passwordSecret
phase2-auth=mschapv2

[ipv4]
dns-search=
method=auto

[ipv6]
addr-gen-mode=stable-privacy
dns-search=
method=auto

[proxy]
[jeff@mbp system-connections]$
```

3. Disable `iwd`
I switched to `iwd` which is recommended to manage wireless. Since my company's networkID contains special character, such as `XXXXX_WLAN(5GHz)`, I guess `iwd` doesn't like `()`. I decided to switch back to `wpa_supplicant`. I'll try to use `iwd` still instead of `wpa_supplicant`, then write another solution

```sh
systemctl stop iwd.service
systemctl disable iwd.service
```

4. Enable `wpa_supplicant`

```sh
systemctl restart wpa_supplicant
```

5. Modify `/etc/NetworkManager/NetworkManager.conf`. Disable `iwd` backend and leave `NetworkManager` being managed by `wpa_supplicant` by default. Then `systemctl restart NetworkManager`

```sh
[device]
# wifi.backend=iwd
# wifi.iwd.autoconnect=yes
```
