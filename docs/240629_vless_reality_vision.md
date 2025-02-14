# Xray with Reality+ Vision+ uTLS

## Objective

Get rid of censorship and browse internet with your own proxy in a proper camouflage mode

## Details

Found a detailed description of this solution at https://www.chengxiaobai.com/en/trouble-maker/journey-switching-from-v2ray-to-xray

...
**What are XTLS, Vision, and REALITY? How do they work?**

XTLS was an early solution to the TLS in TLS issue, requiring modifications to the TLS/uTLS official library. Although its maintainability is questioned, the approach is conceptually sound.

Vision can be considered a stable version of XTLS, also designed to address the TLS in TLS problem. It is now referred to as flow control and uses the label xtls-rprx-vision.

The workings of XTLS and Vision were previously discussed 👉 V2Ray, Trojan, XRay.

REALITY aims to further eliminate server TLS fingerprint features and address SNI blocking issues. Its operational principles are scattered across various issues, and by analyzing the project code, I provided a simplified explanation of how REALITY works.

In essence, REALITY relies on disguising itself to deceive inspectors. It obtains a normal Server Hello packet during TLS handshake from reputable websites like Apple or Bing. It then interacts with the client using this normal handshake packet, containing additional verification content. The process involves:

- The server requests a normal TLS handshake Server Hello packet from legitimate websites like Apple or Bing.
- Using this normal handshake packet, the server interacts with the client, incorporating its own constructed authentication content.
- As the packet content is modified, the client cannot directly process this Hello packet. It requires additional validation to proceed with communication.

From a feature perspective, this entire process appears like interacting with major websites. The design cleverly utilizes the handshake characteristics of well-known websites while adding its authentication elements. The TLS handshake process was previously explained 👉 [Trojan Shared 443 Port Scheme](https://www.chengxiaobai.com/en/trouble-maker/trojan-shared-443-port-scheme).

In summary, REALITY’s design approach is indeed quite innovative!

Relevant GitHub Issues:

[Obtaining Disguised Packets](https://github.com/XTLS/Xray-core/issues/1697#issuecomment-1441246622)

[Communication Verification](https://github.com/XTLS/Xray-core/issues/1697#issuecomment-1441215569)

[Client MITM](https://github.com/XTLS/Xray-core/issues/1588#issuecomment-1415861613)


## Pre-requisites

Everything here is open-source and you can build your own from scratch.

- A Linux server with public in a free world

## Install and configure

#### Installing

Stop any existing `xray.service` and `nginx.service` if they are running.

```sh
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

The default configuration is located at `/usr/local/etc/xray/config.json`

Reference > https://github.com/XTLS/Xray-install . It also contains the uninstall step

#### Generating `uuid` and keys

> The `uuid` and `keys` generated here randomly for demo only. Create your own for security

```sh
(base) ➜  ~ xray uuid
0e5980f3-9046-48c9-ae5b-9ef6ba9358ca

(base) ➜  ~ xray x25519
Private key: 8OEi4FLCDVDQ-eHyMpEEjwkwDss0UrAfms8OpVkc9U8
Public key: V5hCc-56ECTchzNGF0MS64ADeKAnD5T_nZsLSD2SVwE
```

#### (On server) Configuring `config.json`

```json
{
    "log": {
        "loglevel": "warning"
    },
    "routing": {
        "domainStrategy": "AsIs",
        "rules": [
            {
                "type": "field",
                "ip": [
                    "geoip:private"
                ],
                "outboundTag": "block"
            }
        ]
    },
    "inbounds": [
        {
            "listen": "0.0.0.0",
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "0e5980f3-9046-48c9-ae5b-9ef6ba9358ca", // your UUID
                        "flow": "xtls-rprx-vision"
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                    "show": false, 
                    "dest": "www.amazon.com:443", 
                    "xver": 0,
                    "serverNames": [ 
                        "www.amazon.com"
                    ],
                    "privateKey": "8OEi4FLCDVDQ-eHyMpEEjwkwDss0UrAfms8OpVkc9U8", 
                    "shortIds": [ 
                        "0123456789abcdef" // any string from 0 to F up to 16 digits
                    ]
                }
            },
            "sniffing": {
                "enabled": true,
                "destOverride": [
                    "http",
                    "tls",
                    "quic"
                ]
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom",
            "tag": "direct"
        },
        {
            "protocol": "blackhole",
            "tag": "block"
        }
    ],
    "policy": {
        "levels": {
            "0": {
                "handshake": 2,
                "connIdle": 120
            }
        }
    }
} 
```

#### (On client) Configuring `config.json` for both Linux and Android

```json
{
  "dns": {
    "hosts": {
      "domain:googleapis.cn": "googleapis.com"
    },
    "servers": [
      "1.1.1.1",
      {
        "address": "223.5.5.5",
        "domains": [
          "geosite:cn",
          "geosite:geolocation-cn"
        ],
        "expectIPs": [
          "geoip:cn"
        ],
        "port": 53
      }
    ]
  },
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true,
        "userLevel": 8
      },
      "sniffing": {
        "destOverride": [
          "http",
          "tls"
        ],
        "enabled": true,
        "routeOnly": false
      },
      "tag": "socks"
    },
    {
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {
        "userLevel": 8
      },
      "tag": "http"
    }
  ],
  "log": {
    "loglevel": "warning"
  },
  "outbounds": [
    {
      "mux": {
        "concurrency": -1,
        "enabled": false,
        "xudpConcurrency": 8,
        "xudpProxyUDP443": ""
      },
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_IP_ADDRESS",
            "port": 443,
            "users": [
              {
                "encryption": "none",
                "flow": "xtls-rprx-vision",
                "id": "0e5980f3-9046-48c9-ae5b-9ef6ba9358ca", // must be identical with server's
                "level": 8,
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "quicSettings": {
          "header": {
            "type": "none"
          },
          "key": "",
          "security": ""
        },
        "realitySettings": {
          "allowInsecure": false,
          "fingerprint": "chrome",
          "publicKey": "V5hCc-56ECTchzNGF0MS64ADeKAnD5T_nZsLSD2SVwE", // pair to server's
          "serverName": "www.amazon.com",
          "shortId": "0123456789abcdef",
          "show": false,
          "spiderX": "/"
        },
        "security": "reality",
        "tcpSettings": {
          "header": {
            "type": "none"
          }
        }
      },
      "tag": "proxy"
    },
    {
      "protocol": "freedom",
      "settings": {},
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      },
      "tag": "block"
    }
  ],
  "remarks": "joey_hk",
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "ip": [
          "1.1.1.1"
        ],
        "outboundTag": "proxy",
        "port": "53"
      },
      {
        "ip": [
          "223.5.5.5"
        ],
        "outboundTag": "direct",
        "port": "53"
      },
      {
        "domain": [
          "domain:googleapis.cn"
        ],
        "outboundTag": "proxy"
      },
      {
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "direct"
      },
      {
        "ip": [
          "geoip:cn"
        ],
        "outboundTag": "direct"
      },
      {
        "domain": [
          "geosite:cn"
        ],
        "outboundTag": "direct"
      },
      {
        "domain": [
          "geosite:geolocation-cn"
        ],
        "outboundTag": "direct"
      },
      {
        "outboundTag": "proxy",
        "port": "0-65535"
      }
    ]
  }
}
```

You would have to modify 3 values in your configuration JSON file:
- Your IP address
- `uuid`
- Public key

And under in-bound settings in client configuration

## Starting the service

On *both* of server and client, you can run

```sh
/usr/local/bin/xray run -config /usr/local/etc/xray/config.json
```

If your Linux support `systemd`, you can manage the service

```sh
sudo systemctl start  xray.service
sudo systemctl enable xray.service
```

On Android, I've tested to load `config.json` within `v2rayNG`

## Proxy settings with browser

And under in-bound settings in client configuration, we can see there are 2 proxies setup and available on localhost. You can use any of them

- http over `10809`
- socks over `10808` with `udp` protocol

These are _customize-able_ up to your personal preference

In my case, using Chrome browser can detect the proxy automatically. Sometimes it doesn't. You can execute

```sh
google-chrome-stable --proxy-server=http://localhost:10809
```