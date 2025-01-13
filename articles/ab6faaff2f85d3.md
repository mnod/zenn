---
title: "Debian12 で LXC をもう少し試す"
emoji: "🦒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["debian", "lxc"]
published: true
---

# 概要

LXCでは、デフォルトは veth を作成して lxcbr0 に割り当てて利用する。
この場合、ホストOS以外から通信するとき、ルーティング等の設定が必要となる。
KVMとかでするように、自分で仮想スイッチを作成して、lxcbr0 の代わりにそちらを指定して利用するのが、もっとも融通がききそう。
ということで試してみたい。
ネットワーク設定には、veth を利用する以外にもいくつか選択肢があるので、それについてもいくつか試す。

# 環境

https://zenn.dev/mnod/articles/d1ee99c2e42749 の続き。

#  ホストのNWを共有

ホストのIPアドレスを共有して利用する。
(試していないが、ホストOSとポートの衝突とかあると怒られるのではないか)

設定変更
```
$ sudo cp -pi /var/lib/lxc/debian-test/config{,.000}

$ sudo vi /var/lib/lxc/debian-test/config

$ sudo diff -u /var/lib/lxc/debian-test/config{,.000}
--- /var/lib/lxc/debian-test/config     2025-01-05 23:14:22.102515948 +0000
+++ /var/lib/lxc/debian-test/config.000 2025-01-05 08:05:41.882975107 +0000
@@ -4,7 +4,7 @@
 # Uncomment the following line to support nesting containers:
 #lxc.include = /usr/share/lxc/config/nesting.conf
 # (Be aware this has security implications)
-lxc.net.0.type = none
+lxc.net.0.type = veth
 lxc.net.0.hwaddr = 00:16:3e:db:9c:ac
 lxc.net.0.link = lxcbr0
 lxc.net.0.flags = up
```

変更した設定でコンテナ起動
```
$ sudo lxc-start --name debian-test -d

$ sudo lxc-ls -f --filter debian-test
NAME             STATE   AUTOSTART GROUPS IPV4                    IPV6                                  UNPRIVILEGED
debian-test      RUNNING 0         -      10.0.3.1, 192.168.11.41 2405:6580:d400:2900:5054:ff:fe28:3c42 false
debian-test-copy STOPPED 0         -      -                       -                                     false
```

# macvlan

macvlan インタフェースを作成して、ホストのインタフェースに割り当てる。

- ホストと同じネットワークに属することができる。
- ホストOS外部とコンテナとの間では通信ができるが、ホストOSとコンテナ間の通信は不可。
- `lxc.net.0.macvlan.mode = bridge` を指定すると、同様のNW設定のコンテナとの間で通信ができる。


設定変更

```
$ sudo diff -u /var/lib/lxc/debian-test/config{,.000}
--- /var/lib/lxc/debian-test/config     2025-01-05 23:34:22.659506458 +0000
+++ /var/lib/lxc/debian-test/config.000 2025-01-05 08:05:41.882975107 +0000
@@ -4,11 +4,10 @@
 # Uncomment the following line to support nesting containers:
 #lxc.include = /usr/share/lxc/config/nesting.conf
 # (Be aware this has security implications)
-lxc.net.0.type = macvlan
+lxc.net.0.type = veth
 lxc.net.0.hwaddr = 00:16:3e:db:9c:ac
-lxc.net.0.link = enp1s0
+lxc.net.0.link = lxcbr0
 lxc.net.0.flags = up
 lxc.apparmor.profile = generated
 lxc.apparmor.allow_nesting = 1
```

`lxc-start` で起動すると、外部のDHCPサーバからアドレスが割り当てられている。

```
$ sudo lxc-ls -f --filter debian-test
NAME             STATE   AUTOSTART GROUPS IPV4          IPV6                                   UNPRIVILEGED
debian-test      RUNNING 0         -      192.168.11.42 2405:6580:d400:2900:216:3eff:fedb:9cac false
debian-test-copy STOPPED 0         -      -             -                                      false
```


# Linuxプリッジを利用する

ホストOSからも、外部OSからも通信可能となる。

## Linuxブリッジの作成

設定ファイル作成

:::message

Cloudイメージで起動したとき、/etc/netplan/50-cloud-init.yaml にNW設定があるのだが、
/etc/netplan/50-cloud-init.yaml の冒頭部のメッセージの通り、
ここに設定を記載すると、仮想マシンを再起動すると消える
```
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
```
設定変更は ASCIIコード順で後にくるファイルを作って記載する
:::


```: /etc/netplan/99-linux-bridge.yaml
network:
    ethernets:
        enp1s0:
            dhcp4: false
            match:
                macaddress: 52:54:00:28:3c:42
            set-name: enp1s0
    bridges:
        br0:
            interfaces:
              - enp1s0
            dhcp4: false
            addresses:
              - 192.168.11.41/24
            routes:
              - to: default
                via: 192.168.11.254
            nameservers:
                addresses:
                  - 192.168.11.249
                  - 192.168.11.254
    version: 2
```

設定の反映。
- 設定ファイル(yml)に所有者(root)以外に読み取り権があると警告が表示されるが実害はなさそう
- `netplan try` で、問題がなければ Enter 押下で設定がコミットされる。Enter 押下しないと、タイムアウト(デフォルトは120秒)後にロールバックされる。

```
$ sudo chmod 600 /etc/netplan/99-linux-bridge.yaml
$ sudo netplan try --timeout 30
または
$ sudo netplan apply
```

確認
```
$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.a61bf4e0cc92       no              enp1s0
lxcbr0          8000.00163e000000       no

$ ip a show br0
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether a6:1b:f4:e0:cc:92 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.41/24 brd 192.168.11.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 2405:6580:d400:2900:a41b:f4ff:fee0:cc92/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86381sec preferred_lft 14381sec
    inet6 fe80::a41b:f4ff:fee0:cc92/64 scope link
       valid_lft forever preferred_lft forever
```

## 設定変更

```
$ sudo diff -u /var/lib/lxc/debian-test/config{,.000}
--- /var/lib/lxc/debian-test/config     2025-01-07 21:48:21.963096715 +0000
+++ /var/lib/lxc/debian-test/config.000 2025-01-05 08:05:41.882975107 +0000
@@ -6,7 +6,7 @@
 # (Be aware this has security implications)
 lxc.net.0.type = veth
 lxc.net.0.hwaddr = 00:16:3e:db:9c:ac
-lxc.net.0.link = br0
+lxc.net.0.link = lxcbr0
 lxc.net.0.flags = up
 lxc.apparmor.profile = generated
 lxc.apparmor.allow_nesting = 1
```


## 確認

```
$ sudo lxc-start -n debian-test -d

$ sudo lxc-ls -f --filter debian-test
NAME             STATE   AUTOSTART GROUPS IPV4          IPV6                                   UNPRIVILEGED
debian-test      RUNNING 0         -      192.168.11.42 2405:6580:d400:2900:216:3eff:fedb:9cac false
debian-test-copy STOPPED 0         -      -             -                                      false

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.a61bf4e0cc92       no              enp1s0
                                                        vethUH5xhr
lxcbr0          8000.00163e000000       no
```


# Open vSwitch を利用する

## OVSブリッジの作成

まずは必要なパッケージをインストール
```
$ sudo apt install --no-install-recommends openvswitch-switch 
```

OVSが動いていることを確認
```
$ sudo systemctl status ovsdb-server.service

$ sudo ovs-vsctl show
d950b269-d296-4406-a92e-d758c29af0f9
    ovs_version: "3.1.0"
```

設定ファイル作成

```/etc/netplan/99-bridge.yaml
network:
    version: 2
    ethernets:
        enp1s0:
            dhcp4: false
            match:
                macaddress: 52:54:00:28:3c:42
            set-name: enp1s0
    bridges:
        ovsbr0:
            openvswitch: {}
            interfaces:
              - enp1s0
            dhcp4: false
            addresses:
              - 192.168.11.41/24
            routes:
              - to: default
                via: 192.168.11.254
            nameservers:
                addresses:
                  - 192.168.11.249
                  - 192.168.11.254
```

`netplan try` または `netplan apply` などで設定を反映した後、状態を確認する。

```
$ sudo ovs-vsctl show
d950b269-d296-4406-a92e-d758c29af0f9
    Bridge ovsbr0
        fail_mode: standalone
        Port enp1s0
            Interface enp1s0
        Port ovsbr0
            Interface ovsbr0
                type: internal
    ovs_version: "3.1.0"

$ ip a show ovsbr0
8: ovsbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 66:ba:80:e9:de:45 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.41/24 brd 192.168.11.255 scope global ovsbr0
       valid_lft forever preferred_lft forever
    inet6 2405:6580:d400:2900:64ba:80ff:fee9:de45/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86370sec preferred_lft 14370sec
    inet6 fe80::64ba:80ff:fee9:de45/64 scope link
       valid_lft forever preferred_lft forever
```


## 設定変更

```
$ sudo diff -u /var/lib/lxc/debian-test/config{,.000}
--- /var/lib/lxc/debian-test/config     2025-01-07 22:59:30.440664757 +0000
+++ /var/lib/lxc/debian-test/config.000 2025-01-05 08:05:41.882975107 +0000
@@ -6,9 +6,7 @@
 # (Be aware this has security implications)
 lxc.net.0.type = veth
 lxc.net.0.hwaddr = 00:16:3e:db:9c:ac
-#lxc.net.0.link = br0
-lxc.net.0.script.up = /etc/lxc/ifup
-lxc.net.0.script.down = /etc/lxc/ifdown
+lxc.net.0.link = lxcbr0
 lxc.net.0.flags = up
 lxc.apparmor.profile = generated
 lxc.apparmor.allow_nesting = 1
```

OVSブリッジに veth を割り当て/解除するためのスクリプトを作成
```: /etc/lxc/ifup
#!/bin/bash

BRIDGE=ovsbr0
ovs-vsctl --may-exist add-br $BRIDGE
ovs-vsctl --if-exists del-port $BRIDGE $5
ovs-vsctl --may-exist add-port $BRIDGE $5
```

```: /etc/lxc/ifdown
#!/bin/bash
BRIDGE=ovsbr0
ovs-vsctl --if-exists del-port ${BRIDGE} $5
```

```
$ sudo chmod +x /etc/lxc/if*

$ ls -l /etc/lxc/if*
-rwxr-xr-x 1 root root  70 Jan  7 22:57 /etc/lxc/ifdown
-rwxr-xr-x 1 root root 148 Jan  7 22:56 /etc/lxc/ifup
```

## 確認

```
$ sudo lxc-start -n debian-test -d

$ sudo lxc-ls -f --filter debian-test
NAME             STATE   AUTOSTART GROUPS IPV4          IPV6                                   UNPRIVILEGED
debian-test      RUNNING 0         -      192.168.11.42 2405:6580:d400:2900:216:3eff:fedb:9cac false
debian-test-copy STOPPED 0         -      -             -                                      false

$ sudo ovs-vsctl show
d950b269-d296-4406-a92e-d758c29af0f9
    Bridge ovsbr0
        fail_mode: standalone
        Port vethjzLaIk
            Interface vethjzLaIk
        Port enp1s0
            Interface enp1s0
        Port ovsbr0
            Interface ovsbr0
                type: internal
    ovs_version: "3.1.0"
```

# 他のホストへの移行

## バックアップ

STOPPED の状態でバックアップを取得する。

```
$ sudo lxc-ls -f --filter debian-test
NAME             STATE   AUTOSTART GROUPS IPV4 IPV6 UNPRIVILEGED
debian-test      STOPPED 0         -      -    -    false
debian-test-copy STOPPED 0         -      -    -    false
```

`/var/lib/lxc` ディレクトリ配下のコンテナ名ディレクトリをバックアップする。

```
$ sudo du -sh /var/lib/lxc/debian-test
374M    /var/lib/lxc/debian-test
```

バックアップとして tar で固める。
```
$ sudo tar -czf /tmp/debian-test.tar.gz -C /var/lib/lxc debian-test

$ ls -lh /tmp/debian-test.tar.gz
-rw-r--r-- 1 root root 162M Jan  8 21:56 /tmp/debian-test.tar.gz
```

## リストア

移行先ホストは LXC パッケージのインストールしただけのまっさらな状態。
```
$ sudo lxc-ls -f

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
lxcbr0          8000.00163e000000       no
```

取得した tar アーカイブを、移行先ホストの `/var/lib/lxc` 配下に展開する
```
$ sudo tar -xzf /tmp/debian-test.tar.gz -C /var/lib/lxc
```

確認すると以下のようになる
```
$ sudo lxc-ls -f
NAME        STATE   AUTOSTART GROUPS IPV4 IPV6 UNPRIVILEGED
debian-test STOPPED 0         -      -    -    false
```

config の内容
```
$ sudo grep -v '#'  /var/lib/lxc/debian-test/config
lxc.net.0.type = veth
lxc.net.0.hwaddr = 00:16:3e:db:9c:ac
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1
lxc.include = /usr/share/lxc/config/debian.common.conf
lxc.tty.max = 4
lxc.arch = arm64
lxc.pty.max = 1024
lxc.rootfs.path = dir:/var/lib/lxc/debian-test/rootfs
lxc.uts.name = debian-test
```

## 確認


起動してみる

```
$ sudo lxc-start -n debian-test -d

$ sudo lxc-ls -f
NAME        STATE   AUTOSTART GROUPS IPV4      IPV6 UNPRIVILEGED
debian-test RUNNING 0         -      10.0.3.91 -    false
```

コンテナ内容の確認

```
$ sudo lxc-attach -n debian-test -- uname -a

$ sudo lxc-attach -n debian-test -- cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"

$ sudo lxc-attach -n debian-test -- cat /etc/debian_version
12.8
```

ホストの状態を確認
```
$ sudo lxc-info -n debian-test
Name:           debian-test
State:          RUNNING
PID:            16631
IP:             10.0.3.91
Link:           vethwdaWTU
 TX bytes:      1.63 KiB
 RX bytes:      3.84 KiB
 Total bytes:   5.47 KiB

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
lxcbr0          8000.00163e000000       no              vethwdaWTU
```
