
## WireGuard

#### Installation

- Ubuntu

```sh
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt install wireguard
```

- Debian

```sh
echo "deb http://deb.debian.org/debian/ unstable main" > /etc/apt/sources.list.d/unstable-wireguard.list
printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' > /etc/apt/preferences.d/limit-unstable
apt update
apt install wireguard
```

#### Generate Keys

```sh
wg genkey | tee wg-private.key | wg pubkey > wg-public.key
```

#### Create Configuration for Server and Client

- On server

```sh
sudo cat /etc/wireguard/wg0.conf

[Interface]
Address = 10.20.30.1/24
ListenPort = 12345  # customizable
PrivateKey = <server_privatekey>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client1_publickey>
AllowedIPs = 10.20.30.2/32
PresharedKey = <preshared_key>

[Peer]
PublicKey = <client2_publickey>
AllowedIPs = 10.20.30.3/32
PresharedKey = <preshared_key>
```

- On client

```sh
[Interface]
PrivateKey = <client_privatekey>
Address = 10.20.30.2/24,fd42:42:42::2/64
DNS = 176.103.130.130,176.103.130.131
[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:12345
AllowedIPs = 0.0.0.0/0,::/0
PresharedKey = <preshared_key>
```

#### Restart WireGuard

```sh
wg-quick down wg0

wg-quick up wg0
```

`wg0` = `/etc/wireguard/wg0.conf`

#### When Adding Client

- On server, add `client_publickey` in peer and assign a unique IP. `server_privatekey` keeps the same
- On client, create a configuration file with `client_privatekey` with the same unique IP. `server_publickey` keeps the same
- Restart `wg-quick down wg0` then `wg-quick up wg0`

> Reference
> https://www.linode.com/docs/networking/vpn/set-up-wireguard-vpn-on-ubuntu/
> https://wiki.debian.org/Wireguard
> https://github.com/angristan/wireguard-install


## An Absolute Solution for macOS

In Apple's AppStore _somewhere_, we're unlocky to not have __wireguard__ GUI, but thank to command line tool. Here's the absolute solution on macOS (of course, it's working on other OS) if you can't find a GUI one or you really love command line which is telling the truth.

0. If you don't have `brew`, go to https://brew.sh and do this

```sh
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

1. Download by `brew`

```sh
brew install wireguard-tools
```

2. Move your configuration file to `/usr/local/etc/wireguard/`

```sh
mkdir -p mkdir /usr/local/etc/wireguard

cp YOUR_WIREGUARD.conf /usr/local/etc/wireguard/
```

3. Start

```sh
wg-quick up YOUR_WIREGUARD
```

> Notice: it's `YOUR_WIREGUARD`, not `YOUR_WIREGUARD.conf` in command

4. Stop

```sh
wg-quick down YOUR_WIREGUARD
```
