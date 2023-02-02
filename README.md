# Use Debian as a  route and configure a global transparent proxy for mainland China

## This guide will help you to config global transparent proxy on Debian 11.

## Other distributions are NOT tested and have NO guarantees.

## To begin with, you should ensure that you have got root access in order to do the following steps.

***This tutorial is not for beginners, you should have some basic operating knowledge of Linux and v2ray,like how to write a correct v2ray config file, how to set up a static ip address for Linux using any tools you like(even this essay recommend to using `Network-Manager` But any other tool that can accomplish this is also possible), beginners please consider using `openwrt` and other more easy to use, community support more powerful software***

***This configuration applies to your pppoe dial-up using Debian as the primary route and configure ip/domain based routing, if you only use it as a proxy gateway please modify it at your discretion***

***And please,this wont tell you how to setup the static ip address,Please refer to other articles to complete the configuration yourself***

***Please do this in The environment of the Internet***

***And this wont proxy any non-tcp traffic,if you need proxy udp,please use other tools like tun2socks etc.***

### 1.Setup software mirror and install necessary  software

1.If you are using Debian 11,just run this command 

``` bash
cat > /etc/apt/sources.list<<EOF
deb https://mirrors.bfsu.edu.cn/debian/ bullseye main contrib non-free
# deb-src https://mirrors.bfsu.edu.cn/debian/ bullseye main contrib non-free
deb https://mirrors.bfsu.edu.cn/debian/ bullseye-updates main contrib non-free
# deb-src https://mirrors.bfsu.edu.cn/debian/ bullseye-updates main contrib non-free

deb https://mirrors.bfsu.edu.cn/debian/ bullseye-backports main contrib non-free
# deb-src https://mirrors.bfsu.edu.cn/debian/ bullseye-backports main contrib non-free

deb https://mirrors.bfsu.edu.cn/debian-security bullseye-security main contrib non-free
# deb-src https://mirrors.bfsu.edu.cn/debian-security bullseye-security main contrib non-free
EOF
```

If not, please stand on your own two feet

2.install necessary software and enable ip forward 

``` bash
apt update -y&&apt install wget curl nftables -y
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf &&sysctl -p
```

3.install v2ray via official script and enable via systemctl 

``` bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

4.Refer to the following nftables configuration file to write your own configuration that applies to you

``` perl
#!/usr/sbin/nft -f
flush ruleset
include "/etc/iprules/bypass.nft"
define RESERVED_IP = {
    10.0.0.0/8,
    100.64.0.0/10,
    127.0.0.0/8,
    169.254.0.0/16,
    172.16.0.0/12, 
    192.0.0.0/24,
    224.0.0.0/4,
    240.0.0.0/4,
    255.255.255.255/32
}
table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                ct state { established, related } accept
                iifname { "lo" } accept
                iifname { "eno1" } accept
                icmp type echo-request accept
                tcp dport {22} accept
                udp dport {53, 67, 68} accept
        }

        chain forward {
                type filter hook forward priority filter; policy drop;
                ip daddr !=$bypass udp dport {443} reject #Drop some quic 
                ct state { established, related } accept
                iifname { "eno1" } accept
        }

        chain output {
                type filter hook output priority filter; policy accept;
        }
}
table ip nat {
    chain prerouting_proxy {
            type nat hook prerouting priority filter; policy accept;
            ip daddr $RESERVED_IP return
            ip daddr $bypass return
            meta l4proto tcp  redirect to :1234
    }
    chain postrouting_snat {
            type nat hook postrouting priority srcnat; policy accept;
            oifname { "enp3s0" } masquerade
    }
}
```

***Don't delete RESERVED_IP! it's extremally important to avoid you lost access to your router!***

Please replace the `enp3s0` to your WAN interface name(in pppoe,it usually be `ppp0`) and the `eno1` to you LAN interface name

The file `/etc/iprules/bypass.nft` is the IP list you want to bypass (IP list of China).

An example of the bypass list:

```
define bypassd = {
10.0.0.0/8,
172.16.0.0/12,
192.168.0.0/16,
}
```

In most cases, this list should contain CHN IP list 

you may get list from [ipip.net](https://github.com/17mon/china_ip_list) or from [clang.cn](https://ispip.clang.cn/all_cn_cidr.txt)

If this device don't offer NAT services , delete

``` perl
    chain postrouting_snat {
            type nat hook postrouting priority srcnat; policy accept;
            oifname { "enp3s0" } masquerade
    }
```

**Enable the nftables but don't apply the rules now `systemctl enable nftables`**

5.Install the `AdGuardHome` as the DNS server and the DHCP server via official install script 

``` bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

An example of DNS upstream config guide: https://github.com/fernvenue/adguardhome-upstream

Enable DHCP server in `AdGuardHome` panel if needed.

***If you need use pppoe to access the Internet or the Local Area Network,please follow this,In other cases, the configuration is complete,and please refer the config file of v2ray in the end***

6.install `pppoeconf`

``` bash
apt install pppoeconf -y
```

***Please note that you now need to connect one of the interfaces of this device to the cable used for pppoe dial-up,And you may lose internet access until the configuration is complete***

please follow the guide of `pppoeconf`,it will guide you to configure the pppoe dial-up

**Congratulations! you finished the basic config of the router!**

7.Seteup the static ip address for the LAN interface 

*Please use your favorite tool to configure yourself, only need to configure IP address and mask can be, no need to configure gateway, DNS, etc.*





#### Refer

1.Sample config file for the v2ray which use Vless + Tls + Websocket

```json
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "tag": "transparent",
            "port": 1234,
            "protocol": "dokodemo-door",
            "settings": {
                "network": "tcp,udp",
                "followRedirect": true
            },
            "sniffing": {
                "enabled": true,
                "destOverride": [
                    "http",
                    "tls"
                ]
            }
        }
    ],
    "outbounds": [
        {
            "tag": "proxy",
            "protocol": "vless",
            "settings": {
                "vnext": [
                    {
                        "address": "example.com", // 换成你的域名或服务器 IP（发起请求时无需解析域名了）
                        "port": 443,
                        "users": [
                            {
                                "id": "", // 填写你的 UUID
                                "encryption": "none",
                                "level": 0
                            }
                        ]
                    }
                ]
            },
            "streamSettings": {
                "network": "ws",
                "security": "tls",
                "tlsSettings": {
                    "serverName": "example.com" // 换成你的域名
                },
                "wsSettings": {
                    "path": "/websocket" // 必须换成自定义的 PATH，需要和服务端的一致
                }
            }
        },
        {
            "protocol": "freedom",
            "tag": "direct"
        }
    ],
    "routing": {
        "domainMatcher": "mph",
        "domainStrategy": "IPOnDemand",
        "rules": [
            {
                "type": "field",
                "network": "udp",
                "port": "123",
                "outboundTag": "direct"
            },
            {
                "outboundTag": "direct",
                "protocol": [
                    "bittorrent"
                ],
                "type": "field"
            },
            {
                "type": "field",
                "domain": [
                    "geosite:cn"
                ],
                "outboundTag": "direct"
            },
            {
                "ip": [
                    "geoip:private",
                    "geoip:cn"
                ],
                "outboundTag": "direct",
                "type": "field"
            }
        ]
    }
}
```

Other protocol please replace by your self

