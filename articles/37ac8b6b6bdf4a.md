---
title: "Libreswan で IPsec を試す"
emoji: "🚇"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["libreswan", "ipsec"]
published: true
---

# 準備

必要なパッケージを導入する。(bridge-utilsは疎通確認を目的として仮想ブリッジを作成するため)
```
# apt install bridge-utils libreswan
```

## サーバ側

サーバ、クライアントというのもおかしいが。
疎通確認のために、仮想ブリッジを作成してアドレスをふっておく。
```
# brctl addbr br1
# ip a add 192.168.101.254/24 dev br1

# brctl show
bridge name     bridge id               STP enabled     interfaces
br1             8000.fee96947bbf0       no

# ip a show br1
3: br1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether fe:e9:69:47:bb:f0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.101.254/24 scope global br1
       valid_lft forever preferred_lft forever
```

## クライアント側

同様に。

```
# brctl addbr br1
# ip a add 192.168.102.254/24 dev br1

# brctl show
# ip a show br1
```

# Host to host VPN with PSK

参考 https://libreswan.org/wiki/Host_to_host_VPN_with_PSK


## サーバ側 

設定ファイルを作成する。left を自サーバ(192.168.11.213)、right は対向サーバ((192.168.11.214)とする。

```: /etc/ipsec.d/test3.conf
config setup
    protostack=netkey

conn vpn-to-test4
    authby=secret
    type=tunnel
    left=192.168.11.213
    right=192.168.11.214
    auto=start
```

シークレットを作成
```: /etc/ipsec.d/test3.secrets
192.168.11.213 192.168.11.214 : PSK "YourSecretPassword!"
```

カーネルパラメータ変更

```: /etc/sysctl.conf
net.ipv4.conf.default.accept_redirects = 0 
net.ipv4.conf.default.send_redirects = 0
```

```
# sysctl -p
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.send_redirects = 0
```

```
# ipsec verify
# systemctl restart ipsec
```

```
$ systemctl status ipsec
$ journalctl -xeu ipsec.service
```

## クライアント側

同様に。
```: /etc/ipsec.d/test4.conf
config setup
    protostack=netkey

conn vpn-to-test3
    authby=secret
    type=tunnel
    left=192.168.11.214
    right=192.168.11.213
    auto=start
```

```: /etc/ipsec.d/test4.secrets
192.168.11.214 192.168.11.213 : PSK "YourSecretPassword!"
```

```
# ipsec verify
# systemctl restart ipsec
```


サーバ側から ping してみる
```
$ ping -c 1 192.168.11.214
```

パケットキャプチャしてみると、icmp ではなくて esp となっている。
```
$ sudo tcpdump -i vnet0 esp or icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on vnet0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:02:03.510598 IP 192.168.11.201 > 192.168.11.202: ESP(spi=0xd84efcf4,seq=0x1), length 120
18:02:03.512038 IP 192.168.11.202 > 192.168.11.201: ESP(spi=0xf6489297,seq=0x1), length 120
```

# ルーティングを追加してみる

結論をいうと、暗号化されない。

サーバ側にルーティングを追加
```
# ip route add 192.168.102.0/24 via 192.168.11.214
# ip route show 192.168.102.0/24
```

クライアント側にルーティングを追加
```
# ip route add 192.168.101.0/24 via 192.168.11.213
# ip route show 192.168.102.0/24
```

サーバ側からpingしてみる
```
$ ping -c 1 -I 192.168.101.254 192.168.102.254
```

```
18:32:34.665162 IP 192.168.101.254 > 192.168.102.254: ICMP echo request, id 15658, seq 1, length 64
18:32:34.665908 IP 192.168.102.254 > 192.168.101.254: ICMP echo reply, id 15658, seq 1, length 64
```

```
$ ping -c 1 -I 192.168.101.254 192.168.11.214
```

```
18:33:40.194012 IP 192.168.101.254 > 192.168.11.202: ICMP echo request, id 15660, seq 1, length 64
18:33:40.194745 IP 192.168.11.202 > 192.168.101.254: ICMP echo reply, id 15660, seq 1, length 64
```

追加したルーティングはそれぞれでいったん削除する。

```
# ip route del 192.168.102.0/24
```

```
# ip route del 192.168.101.0/24
```

# 静的ルート自動追加

## サーバ側

```: /etc/ipsec.d/test3.conf 
config setup
    protostack=netkey
    virtual_private=%v4:192.168.101.0/24,%v4:192.168.102.0/24

conn vpn-to-test4
    authby=secret
    type=tunnel
    left=192.168.11.213
    leftsubnet=192.168.101.0/24
    leftsourceip=192.168.101.254
    right=192.168.11.214
    rightsubnet=192.168.102.0/24
    rightsourceip=192.168.101.254
    auto=start
```

```
# systemctl restart ipsec
# ip route show 192.168.102.0/24
192.168.102.0/24 dev enp1s0 scope link src 192.168.101.254 
```

## クライアント側

```: /etc/ipsec.d/test4.conf
config setup
    protostack=netkey
    virtual_private=%v4:192.168.101.0/24,%v4:192.168.102.0/24

conn vpn-to-test3
    authby=secret
    type=tunnel
    left=192.168.11.214
    leftsubnet=192.168.102.0/24
    leftsourceip=192.168.102.254
    right=192.168.11.213
    rightsubnet=192.168.101.0/24
    rightsourceip=192.168.101.254
    auto=start
```

```
# systemctl restart ipsec
# ip route show 192.168.101.0/24
192.168.101.0/24 dev enp1s0 scope link src 192.168.102.254 
```

疎通確認

```
$ ping -c 1 192.168.11.214
```

```
17:34:44.859000 IP 192.168.11.213 > 192.168.11.214: ESP(spi=0x9220d271,seq=0x1), length 120
17:34:44.859296 IP 192.168.11.214 > 192.168.11.213: ESP(spi=0x86e37eff,seq=0x1), length 120
```

```
$ ping -c 1 192.168.102.254
```
ここはやはり暗号化されない。ここをなんとかしたい。
```
$ sudo tcpdump -i vnet0 esp or icmp

17:55:42.908638 IP 192.168.11.201 > 192.168.11.202: ICMP echo request, id 15040, seq 1, length 64
17:55:42.909520 IP 192.168.11.202 > 192.168.11.201: ICMP echo reply, id 15040, seq 1, length 64
```



# Subnet to subnet VPN with PSK

https://libreswan.org/wiki/Subnet_to_subnet_VPN_with_PSK

## サーバ側

```: /etc/ipsec.d/test3.conf 
config setup
    protostack=netkey
    virtual_private=%v4:192.168.101.0/24,%v4:192.168.102.0/24

conn mysubnet
    also=vpn-to-test4
    leftsubnet=192.168.101.0/24
    leftsourceip=192.168.101.254
    rightsubnet=192.168.102.0/24
    rightsourceip=192.168.102.254
    auto=start

conn vpn-to-test4
    authby=secret
    type=tunnel
    left=192.168.11.213
    right=192.168.11.214
    auto=start
```


```
# systemctl restart ipsec
# ip route show 192.168.102.0/24
192.168.102.0/24 dev enp1s0 scope link src 192.168.101.254 
```

## クライアント側

```: /etc/ipsec.d/test4.conf 
config setup
    protostack=netkey
    virtual_private=%v4:192.168.101.0/24,%v4:192.168.102.0/24

conn mysubnet
    also=vpn-to-test3
    leftsubnet=192.168.102.0/24
    leftsourceip=192.168.102.254
    rightsubnet=192.168.101.0/24
    rightsourceip=192.168.101.254

conn vpn-to-test3
    authby=secret
    type=tunnel
    left=192.168.11.202
    right=192.168.11.201
    auto=start
```

```
# systemctl restart ipsec
# ip route show 192.168.101.0/24
192.168.101.0/24 dev enp1s0 scope link src 192.168.102.254 
```



疎通確認
```
# ping -c 1 192.168.102.254
```
暗号化された。
```
17:41:32.190068 IP 192.168.11.213 > 192.168.11.214: ESP(spi=0x2996d234,seq=0x1), length 120
17:41:32.190320 IP 192.168.11.214 > 192.168.11.213: ESP(spi=0x9f822498,seq=0x1), length 120
```

```
# ping -c 1 192.168.11.214
```

```
17:42:20.415115 IP 192.168.11.213 > 192.168.11.214: ESP(spi=0x393e2d64,seq=0x1), length 120
17:42:20.415380 IP 192.168.11.214 > 192.168.11.213: ESP(spi=0x163c1330,seq=0x1), length 120
```



念のため逆方向も。
```
# ping -c 1 192.168.101.254
```

```
17:43:09.432723 IP 192.168.11.214 > 192.168.11.213: ESP(spi=0x9f822498,seq=0x2), length 120
17:43:09.433010 IP 192.168.11.213 > 192.168.11.214: ESP(spi=0x2996d234,seq=0x2), length 120
```

```
# ping -c 1 192.168.11.213
```

```
17:43:28.821432 IP 192.168.11.214 > 192.168.11.213: ESP(spi=0x163c1330,seq=0x2), length 120
17:43:28.821674 IP 192.168.11.213 > 192.168.11.214: ESP(spi=0x393e2d64,seq=0x2), length 120
```
