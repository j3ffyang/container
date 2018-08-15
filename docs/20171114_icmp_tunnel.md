# ICMP Tunneling

## Document Objective
- Establish tunneling between VPN server and ICMP tunnel server, to allow all VPN clients to access ICMP tunnel server

## Architecture Overview

```
/------------\    /--------------------\    /-----------------\
| VPN client |    | VPN server         |    |                 |
| 10.8.0.22  |<-->|  10.8.0.1/24       |    |                 |
|            |    | ICMP tunnel client |<-->| ICMP tunnel srv |
|            |    |  10.10.10.101      |    | 10.10.10.1/24   |
\------------/    \--------------------/    \-----------------/
      |                                              ^
      |                                              |
      v---------------------------------------------->            
```

## Deployment

#### 1\. Install ICMP tunneling between VPN server and ICMP tunnel server

- Install ```hans``` on both tunnel server and tunnel client (which is OpenVPN server as well)

  Reference document> http://code.gerade.org/hans/

  ```hans``` [source code](https://sourceforge.net/projects/hanstunnel/)
- Choose a tunneling network, such as ```10.10.10.0/24```. Then tunnel server would be ```10.10.10.1``` and the client ```10.10.10.101``` by default
- Ping each other

#### 2\. Configure ```push route``` on OpenVPN server

In ```/etc/openvpn/server.conf``` of OpenVPN,

```
push "route 172.29.167.128 255.255.255.128"
push "route 10.10.10.0 255.255.255.0"
```

where ```172.29.167.128/25``` is the network behind OpenVPN and ```10.10.10.0/24``` is tunneling network.

Then, on OpenVPN server

```
systemctl restart openvpn@server.service
```

You should re- connect your OpenVPN client after OpenVPN server configuration changed.

#### 3\. Add ```POSTROUTING``` chain in OpenVPN

This enables forwarding packet from VPN client to ICMP tunnel server

- Enable ```ip_forward```

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Add ```iptables POSTROUTING``` rule

```
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o tun1 \
  -j SNAT --to-source 10.10.10.101
```

or

```
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o tun1 \
  -j MASQUERADING
```

Now you should be able to ping from VPN client to ICMP tunnel server

#### 4\. Sample routing tables

- On VPN client

```
[jeff@fedora ~]$ ip r
default via 192.168.3.1 dev wlp0s20f0u9u1 proto static metric 600
10.8.0.1 via 10.8.0.5 dev tun0
10.8.0.5 dev tun0 proto kernel scope link src 10.8.0.6
10.10.10.0/24 via 10.8.0.5 dev tun0
172.29.167.128/25 via 10.8.0.5 dev tun0
192.168.3.0/24 dev wlp0s20f0u9u1 proto kernel scope link src 192.168.3.114 metric 600
```

- On VPN server (and ICMP tunnel client)

```
default via 172.29.167.131 dev ens32 onlink
10.8.0.0/24 via 10.8.0.2 dev tun0
10.8.0.2 dev tun0  proto kernel  scope link  src 10.8.0.1
10.10.10.0/24 dev tun1  proto kernel  scope link  src 10.10.10.101
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 linkdown
172.19.0.0/16 dev docker_gwbridge  proto kernel  scope link  src 172.19.0.1
172.29.167.128/25 dev ens32  scope link
```

- On ICMP tunnel server

```
default via 10.10.0.1 dev eth0 onlink
10.10.0.0/24 dev eth0  proto kernel  scope link  src 10.10.0.2
10.10.10.0/24 dev tun0  proto kernel  scope link  src 10.10.10.1
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1
```

#### 5\. Troubleshooting

In case you want to troubleshoot by using ```tcpdump``` to view ```icmp``` packet back and forth

```
tcpdump -nn -i tun0 icmp
```
