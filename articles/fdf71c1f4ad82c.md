---
title: "Debian12 で LXD を試す"
emoji: "🦖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["debian", "lxd"]
published: true
---

# 環境

Raspberry Pi OS 上の KVM 仮想マシン環境(Debian12 bookworm / CloudImage) に LXC を入れて検証した。

```
$ uname -a
Linux localhost 6.1.0-26-cloud-arm64 #1 SMP Debian 6.1.112-1 (2024-09-30) aarch64 GNU/Linux

$ cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"

$ cat /etc/debian_version
12.7
```

# インストール

apt でインストールできる

```
$ sudo apt update && sudo apt upgrade && sudo apt install lxd
```

# 初期化

全て既定の設定値で初期化する。
- ストレージプール default を作成する
- 仮想ブリッジ lxdbr0 を作成する

```
$ sudo lxd init
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, _auto_ or _none_) [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, _auto_ or _none_) [default=auto]:
Would you like the LXD server to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```


## もろもろ確認

バージョンの確認
```
$ sudo lxc version
Client version: 5.0.2
Server version: 5.0.2
```

デフォルトのプロファイルの確認
```
$ sudo lxc profile list
+---------+---------------------+---------+
|  NAME   |     DESCRIPTION     | USED BY |
+---------+---------------------+---------+
| default | Default LXD profile | 0       |
+---------+---------------------+---------+

$ sudo lxc profile show default
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: default
used_by: []
```

ストレージプールの確認
```
$ sudo lxc storage list
+---------+--------+------------------------------------+-------------+---------+---------+
|  NAME   | DRIVER |               SOURCE               | DESCRIPTION | USED BY |  STATE  |
+---------+--------+------------------------------------+-------------+---------+---------+
| default | dir    | /var/lib/lxd/storage-pools/default |             | 1       | CREATED |
+---------+--------+------------------------------------+-------------+---------+---------+

$ sudo lxc storage info default
info:
  description: ""
  driver: dir
  name: default
  space used: 1.17GiB
  total space: 7.71GiB
used by:
  profiles:
  - default
```

ディレクトリの中身はこんな感じになっている
```
$ sudo ls -lR /var/lib/lxd/storage-pools/default
/var/lib/lxd/storage-pools/default:
total 32
drwx--x--x 2 root root 4096 Jan  9 21:16 buckets
drwx--x--x 2 root root 4096 Jan  9 21:16 containers
drwx--x--x 2 root root 4096 Jan  9 21:16 containers-snapshots
drwx--x--x 2 root root 4096 Jan  9 21:16 custom
drwx--x--x 2 root root 4096 Jan  9 21:16 custom-snapshots
drwx--x--x 2 root root 4096 Jan  9 21:16 images
drwx--x--x 2 root root 4096 Jan  9 21:16 virtual-machines
drwx--x--x 2 root root 4096 Jan  9 21:16 virtual-machines-snapshots

/var/lib/lxd/storage-pools/default/buckets:
total 0

/var/lib/lxd/storage-pools/default/containers:
total 0

/var/lib/lxd/storage-pools/default/containers-snapshots:
total 0

/var/lib/lxd/storage-pools/default/custom:
total 0

/var/lib/lxd/storage-pools/default/custom-snapshots:
total 0

/var/lib/lxd/storage-pools/default/images:
total 0

/var/lib/lxd/storage-pools/default/virtual-machines:
total 0

/var/lib/lxd/storage-pools/default/virtual-machines-snapshots:
total 0
```

ネットワークの確認
```
$ sudo lxc network list
+--------+----------+---------+----------------+---------------------------+-------------+---------+---------+
|  NAME  |   TYPE   | MANAGED |      IPV4      |           IPV6            | DESCRIPTION | USED BY |  STATE  |
+--------+----------+---------+----------------+---------------------------+-------------+---------+---------+
| enp1s0 | physical | NO      |                |                           |             | 0       |         |
+--------+----------+---------+----------------+---------------------------+-------------+---------+---------+
| lxdbr0 | bridge   | YES     | 10.82.242.1/24 | fd42:410c:cec4:c79b::1/64 |             | 1       | CREATED |
+--------+----------+---------+----------------+---------------------------+-------------+---------+---------+

$ sudo lxc network info lxdbr0
Name: lxdbr0
MAC address: 00:16:3e:b7:07:84
MTU: 1500
State: up
Type: broadcast

IP addresses:
  inet  10.82.242.1/24 (global)
  inet6 fd42:410c:cec4:c79b::1/64 (global)

Network usage:
  Bytes received: 0B
  Bytes sent: 0B
  Packets received: 0
  Packets sent: 0

Bridge:
  ID: 8000.00163eb70784
  STP: false
  Forward delay: 1500
  Default VLAN ID: 1
  VLAN filtering: true
  Upper devices:
```

```
$ ip a show lxdbr0
3: lxdbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 00:16:3e:b7:07:84 brd ff:ff:ff:ff:ff:ff
    inet 10.82.242.1/24 scope global lxdbr0
       valid_lft forever preferred_lft forever
    inet6 fd42:410c:cec4:c79b::1/64 scope global
       valid_lft forever preferred_lft forever

$ sudo apt install bridge-utils

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
lxdbr0          8000.00163eb70784       no
```

## イメージの確認

利用できるリポジトリ
```
$ sudo lxc remote list
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
|      NAME       |                   URL                    |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC | GLOBAL |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| images          | https://images.linuxcontainers.org       | simplestreams | none        | YES    | NO     | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| local (current) | unix://                                  | lxd           | file access | NO     | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu          | https://cloud-images.ubuntu.com/releases | simplestreams | none        | YES    | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu-daily    | https://cloud-images.ubuntu.com/daily    | simplestreams | none        | YES    | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
```

images リポジトリ(images.linuxcontainers.org) はサポートが終了して、空っぽの状態
```
$ sudo lxc image list images:
+-------+-------------+--------+-------------+--------------+------+------+-------------+
| ALIAS | FINGERPRINT | PUBLIC | DESCRIPTION | ARCHITECTURE | TYPE | SIZE | UPLOAD DATE |
+-------+-------------+--------+-------------+--------------+------+------+-------------+
```



代わりに images.lxd.canonical.com のリポジトリを利用する。
ここでは canonical という名前にする。
```
$ sudo lxc remote add canonical https://images.lxd.canonical.com --protocol=simplestreams

$ sudo lxc remote list
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
|      NAME       |                   URL                    |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC | GLOBAL |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| canonical       | https://images.lxd.canonical.com         | simplestreams | none        | YES    | NO     | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| images          | https://images.linuxcontainers.org       | simplestreams | none        | YES    | NO     | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| local (current) | unix://                                  | lxd           | file access | NO     | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu          | https://cloud-images.ubuntu.com/releases | simplestreams | none        | YES    | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu-daily    | https://cloud-images.ubuntu.com/daily    | simplestreams | none        | YES    | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
```

canonical リポジトリが利用できることを確認
```
$ sudo sudo lxc image list canonical:debian/12/arm64
+--------------------+--------------+--------+---------------------------------------+--------------+-----------+---------+------------------------------+
|       ALIAS        | FINGERPRINT  | PUBLIC |              DESCRIPTION              | ARCHITECTURE |   TYPE    |  SIZE   |         UPLOAD DATE          |
+--------------------+--------------+--------+---------------------------------------+--------------+-----------+---------+------------------------------+
| debian/12 (7 more) | ed9172a86d95 | yes    | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 93.10MB | Jan 6, 2025 at 12:00am (UTC) |
+--------------------+--------------+--------+---------------------------------------+--------------+-----------+---------+------------------------------+

$ sudo lxc image show canonical:debian/12/arm64
auto_update: false
properties:
  architecture: arm64
  description: Debian bookworm arm64 (20250106_0004)
  os: Debian
  release: bookworm
  serial: "20250106_0004"
  type: squashfs
  variant: default
public: true
expires_at: 1970-01-01T00:00:00Z
profiles: []

$ sudo lxc image info canonical:debian/12/arm64
Fingerprint: ed9172a86d95ae781244acb1b3f1266bd53db415a2d95d094e86c1e4c66c1d35
Size: 93.10MB
Architecture: aarch64
Type: container
Public: yes
Timestamps:
    Created: 2025/01/06 00:00 UTC
    Uploaded: 2025/01/06 00:00 UTC
    Expires: never
    Last used: never
Properties:
    release: bookworm
    variant: default
    type: squashfs
    os: Debian
    architecture: arm64
    serial: 20250106_0004
    description: Debian bookworm arm64 (20250106_0004)
Aliases:
    - debian/bookworm/default
    - debian/bookworm/default/arm64
    - debian/bookworm
    - debian/bookworm/arm64
    - debian/12/default
    - debian/12/default/arm64
    - debian/12
    - debian/12/arm64
Cached: no
Auto update: disabled
Profiles: []
```

```
$ sudo sudo lxc image list canonical:rockylinux/9/arm64
+-----------------------+--------------+--------+------------------------------------+--------------+-----------+----------+------------------------------+
|         ALIAS         | FINGERPRINT  | PUBLIC |            DESCRIPTION             | ARCHITECTURE |   TYPE    |   SIZE   |         UPLOAD DATE          |
+-----------------------+--------------+--------+------------------------------------+--------------+-----------+----------+------------------------------+
| rockylinux/9 (3 more) | b630a4a30b52 | yes    | RockyLinux 9 arm64 (20250106_0027) | aarch64      | CONTAINER | 114.18MB | Jan 6, 2025 at 12:00am (UTC) |
+-----------------------+--------------+--------+------------------------------------+--------------+-----------+----------+------------------------------+

$ sudo lxc image show canonical:rockylinux/9/arm64
auto_update: false
properties:
  architecture: arm64
  description: RockyLinux 9 arm64 (20250106_0027)
  os: RockyLinux
  release: "9"
  serial: "20250106_0027"
  type: squashfs
  variant: default
public: true
expires_at: 1970-01-01T00:00:00Z
profiles: []

$ sudo lxc image info canonical:rockylinux/9/arm64
Fingerprint: b630a4a30b52b0ec17f125b3ad8d069dbf074b3cc9b7fdaccb3a8928db5e4b6e
Size: 114.18MB
Architecture: aarch64
Type: container
Public: yes
Timestamps:
    Created: 2025/01/06 00:00 UTC
    Uploaded: 2025/01/06 00:00 UTC
    Expires: never
    Last used: never
Properties:
    serial: 20250106_0027
    os: RockyLinux
    architecture: arm64
    variant: default
    type: squashfs
    release: 9
    description: RockyLinux 9 arm64 (20250106_0027)
Aliases:
    - rockylinux/9/default
    - rockylinux/9/default/arm64
    - rockylinux/9
    - rockylinux/9/arm64
Cached: no
Auto update: disabled
Profiles: []
```

# コンテナ起動

Debian12 を起動してみる。(デフォルトのprofile を利用)

```
$ sudo lxc launch canonical:debian/12 debian-test
Creating debian-test
Retrieving image: metadata: 100% (26.72MB/s)
Starting debian-test
```

## もろもろ確認

```
$ sudo lxc list
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
|    NAME     |  STATE  |         IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
| debian-test | RUNNING | 10.82.242.179 (eth0) | fd42:410c:cec4:c79b:216:3eff:fe4c:483c (eth0) | CONTAINER | 0         |
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+

$ sudo lxc info debian-test
Name: debian-test
Status: RUNNING
Type: container
Architecture: aarch64
PID: 13066
Created: 2025/01/10 08:57 UTC
Last Used: 2025/01/10 08:57 UTC

Resources:
  Processes: 8
  CPU usage:
    CPU usage (in seconds): 2
  Memory usage:
    Memory (current): 47.70MiB
  Network usage:
    lo:
      Type: loopback
      State: UP
      MTU: 65536
      Bytes received: 0B
      Bytes sent: 0B
      Packets received: 0
      Packets sent: 0
      IP addresses:
        inet:  127.0.0.1/8 (local)
        inet6: ::1/128 (local)
    eth0:
      Type: broadcast
      State: UP
      Host interface: vethdda811fd
      MAC address: 00:16:3e:4c:48:3c
      MTU: 1500
      Bytes received: 3.86kB
      Bytes sent: 3.57kB
      Packets received: 30
      Packets sent: 35
      IP addresses:
        inet:  10.82.242.179/24 (global)
        inet6: fd42:410c:cec4:c79b:216:3eff:fe4c:483c/64 (global)
        inet6: fe80::216:3eff:fe4c:483c/64 (link)

$ sudo lxc config show debian-test
architecture: aarch64
config:
  image.architecture: arm64
  image.description: Debian bookworm arm64 (20250106_0004)
  image.os: Debian
  image.release: bookworm
  image.serial: "20250106_0004"
  image.type: squashfs
  image.variant: default
  volatile.base_image: ed9172a86d95ae781244acb1b3f1266bd53db415a2d95d094e86c1e4c66c1d35
  volatile.cloud-init.instance-id: 6bc23886-6973-4c85-9100-275f3bd9e9a3
  volatile.eth0.host_name: vethdda811fd
  volatile.eth0.hwaddr: 00:16:3e:4c:48:3c
  volatile.idmap.base: "0"
  volatile.idmap.current: '[{"Isuid":true,"Isgid":false,"Hostid":165536,"Nsid":0,"Maprange":10000001},{"Isuid":false,"Isgid":true,"Hostid":165536,"Nsid":0,"Maprange":10000001}]'
  volatile.idmap.next: '[{"Isuid":true,"Isgid":false,"Hostid":165536,"Nsid":0,"Maprange":10000001},{"Isuid":false,"Isgid":true,"Hostid":165536,"Nsid":0,"Maprange":10000001}]'
  volatile.last_state.idmap: '[]'
  volatile.last_state.power: RUNNING
  volatile.uuid: 5e40cafb-7c4b-4c85-bd87-0100505c100b
devices: {}
ephemeral: false
profiles:
- default
stateful: false
description: ""
```

ネットワークの確認
```
$ sudo lxc network show lxdbr0
config:
  ipv4.address: 10.82.242.1/24
  ipv4.nat: "true"
  ipv6.address: fd42:410c:cec4:c79b::1/64
  ipv6.nat: "true"
description: ""
name: lxdbr0
type: bridge
used_by:
- /1.0/instances/debian-test
- /1.0/profiles/default
managed: true
status: Created
locations:
- none

$ sudo lxc network info lxdbr0
Name: lxdbr0
MAC address: 00:16:3e:b7:07:84
MTU: 1500
State: up
Type: broadcast

IP addresses:
  inet  10.82.242.1/24 (global)
  inet6 fd42:410c:cec4:c79b::1/64 (global)
  inet6 fe80::216:3eff:feb7:784/64 (link)

Network usage:
  Bytes received: 3.65kB
  Bytes sent: 4.29kB
  Packets received: 41
  Packets sent: 33

Bridge:
  ID: 8000.00163eb70784
  STP: false
  Forward delay: 1500
  Default VLAN ID: 1
  VLAN filtering: true
  Upper devices: vethdda811fd

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
lxdbr0          8000.00163eb70784       no              vethdda811fd
```

ストレージプールの確認
```
$ sudo lxc storage show default
config:
  source: /var/lib/lxd/storage-pools/default
description: ""
name: default
driver: dir
used_by:
- /1.0/instances/debian-test
- /1.0/profiles/default
status: Created
locations:
- none

$ sudo lxc storage info default
info:
  description: ""
  driver: dir
  name: default
  space used: 1.70GiB
  total space: 7.71GiB
used by:
  instances:
  - debian-test
  profiles:
  - default

$ sudo ls -l /var/lib/lxd/storage-pools/default/containers
total 4
d--x------ 4 165536 root 4096 Jan 10 08:57 debian-test

$ sudo du -sh /var/lib/lxd/storage-pools/default/containers
457M    /var/lib/lxd/storage-pools/default/containers
```


ローカルイメージの確認。(イメージはキャッシュされる)
```
$ sudo lxc image list
+-------+--------------+--------+---------------------------------------+--------------+-----------+---------+------------------------------+
| ALIAS | FINGERPRINT  | PUBLIC |              DESCRIPTION              | ARCHITECTURE |   TYPE    |  SIZE   |         UPLOAD DATE          |
+-------+--------------+--------+---------------------------------------+--------------+-----------+---------+------------------------------+
|       | ed9172a86d95 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 93.10MB | Jan 10, 2025 at 8:57am (UTC) |
+-------+--------------+--------+---------------------------------------+--------------+-----------+---------+------------------------------+

$ sudo lxc image show ed9172a86d95
auto_update: true
properties:
  architecture: arm64
  description: Debian bookworm arm64 (20250106_0004)
  os: Debian
  release: bookworm
  serial: "20250106_0004"
  type: squashfs
  variant: default
public: false
expires_at: 1970-01-01T00:00:00Z
profiles:
- default

$ sudo lxc image info ed9172a86d95
Fingerprint: ed9172a86d95ae781244acb1b3f1266bd53db415a2d95d094e86c1e4c66c1d35
Size: 93.10MB
Architecture: aarch64
Type: container
Public: no
Timestamps:
    Created: 2025/01/06 00:00 UTC
    Uploaded: 2025/01/10 08:57 UTC
    Expires: never
    Last used: 2025/01/10 08:57 UTC
Properties:
    os: Debian
    release: bookworm
    serial: 20250106_0004
    type: squashfs
    variant: default
    architecture: arm64
    description: Debian bookworm arm64 (20250106_0004)
Aliases:
Cached: yes
Auto update: enabled
Source:
    Server: https://images.lxd.canonical.com
    Protocol: simplestreams
    Alias: debian/12
Profiles:
    - default
```

# コマンド実行

コンテナ内のコマンドを実行
```
$ sudo lxc exec debian-test -- uname -a
Linux debian-test 6.1.0-26-cloud-arm64 #1 SMP Debian 6.1.112-1 (2024-09-30) aarch64 GNU/Linux

$ sudo lxc exec debian-test -- cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"

$ sudo lxc exec debian-test -- cat /etc/debian_version
12.8

$ sudo lxc exec debian-test -- ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 08:57 ?        00:00:00 /sbin/init
root         120       1  0 08:57 ?        00:00:00 /lib/systemd/systemd-journald
root         128       1  0 08:57 ?        00:00:00 /lib/systemd/systemd-udevd
systemd+     131       1  0 08:57 ?        00:00:00 /lib/systemd/systemd-resolved
systemd+     136       1  0 08:57 ?        00:00:00 /lib/systemd/systemd-networkd
message+     137       1  0 08:57 ?        00:00:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only
root         139       1  0 08:57 ?        00:00:00 /lib/systemd/systemd-logind
root         144       1  0 08:57 pts/0    00:00:00 /sbin/agetty -o -p -- \u --noclear --keep-baud - 115200,38400,9600 vt220
root         156       0  0 09:26 pts/1    00:00:00 ps -ef
```

シェルを起動。外部への疎通確認をかねて nginx をインストールしてみる
```
$ sudo lxc exec debian-test -- /bin/bash

# apt install --no-install-recommends nginx
```

ホストOSから疎通確認
```
$ curl -I 10.82.242.179
HTTP/1.1 200 OK
Server: nginx/1.22.1
Date: Fri, 10 Jan 2025 09:33:49 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Fri, 10 Jan 2025 09:31:13 GMT
Connection: keep-alive
ETag: "6780e8e1-267"
Accept-Ranges: bytes
```


# コンテナ停止
```
$ sudo lxc stop debian-test


$ sudo lxc list
+-------------+---------+------+------+-----------+-----------+
|    NAME     |  STATE  | IPV4 | IPV6 |   TYPE    | SNAPSHOTS |
+-------------+---------+------+------+-----------+-----------+
| debian-test | STOPPED |      |      | CONTAINER | 0         |
+-------------+---------+------+------+-----------+-----------+
```

ネットワーク確認
```
$ sudo lxc network info lxdbr0
Name: lxdbr0
MAC address: 00:16:3e:b7:07:84
MTU: 1500
State: up
Type: broadcast

IP addresses:
  inet  10.82.242.1/24 (global)
  inet6 fd42:410c:cec4:c79b::1/64 (global)
  inet6 fe80::216:3eff:feb7:784/64 (link)

Network usage:
  Bytes received: 20.46kB
  Bytes sent: 634.62kB
  Packets received: 252
  Packets sent: 316

Bridge:
  ID: 8000.00163eb70784
  STP: false
  Forward delay: 1500
  Default VLAN ID: 1
  VLAN filtering: true
  Upper devices:

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
lxdbr0          8000.00163eb70784       no
```


## 停止したコンテナの起動

```
$ sudo lxc start debian-test

$ sudo lxc list
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
|    NAME     |  STATE  |         IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
| debian-test | RUNNING | 10.82.242.179 (eth0) | fd42:410c:cec4:c79b:216:3eff:fe4c:483c (eth0) | CONTAINER | 1         |
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
```

# コンテナの削除

コンテナを停止させてから削除する

```
$ sudo lxc stop debian-test

$ sudo lxc list
+-------------+---------+------+------+-----------+-----------+
|    NAME     |  STATE  | IPV4 | IPV6 |   TYPE    | SNAPSHOTS |
+-------------+---------+------+------+-----------+-----------+
| debian-test | STOPPED |      |      | CONTAINER | 1         |
+-------------+---------+------+------+-----------+-----------+

$ sudo lxc delete debian-test

$ sudo lxc list
+------+-------+------+------+------+-----------+
| NAME | STATE | IPV4 | IPV6 | TYPE | SNAPSHOTS |
+------+-------+------+------+------+-----------+
```


# イメージの削除

コンテナ削除してもイメージは残る。また。リモートからダウンロードしたイメージはローカルにキャッシュされている。
後から再利用することもできるが、不要なら削除する。

```
$ sudo lxc image list
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
|    ALIAS     | FINGERPRINT  | PUBLIC |              DESCRIPTION              | ARCHITECTURE |   TYPE    |   SIZE   |         UPLOAD DATE          |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
| debian-test1 | 9ec484eba6b6 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 157.23MB | Jan 10, 2025 at 8:19pm (UTC) |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
|              | ed9172a86d95 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 93.10MB  | Jan 10, 2025 at 8:57am (UTC) |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+


$ sudo lxc image delete ed9172a86d95


$ sudo lxc image list
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
|    ALIAS     | FINGERPRINT  | PUBLIC |              DESCRIPTION              | ARCHITECTURE |   TYPE    |   SIZE   |         UPLOAD DATE          |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
| debian-test1 | 9ec484eba6b6 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 157.23MB | Jan 10, 2025 at 8:19pm (UTC) |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
```

## ストレージプール確認
```
$ sudo lxc storage list
+---------+--------+------------------------------------+-------------+---------+---------+
|  NAME   | DRIVER |               SOURCE               | DESCRIPTION | USED BY |  STATE  |
+---------+--------+------------------------------------+-------------+---------+---------+
| default | dir    | /var/lib/lxd/storage-pools/default |             | 1       | CREATED |
+---------+--------+------------------------------------+-------------+---------+---------+


$ sudo lxc storage show default
config:
  source: /var/lib/lxd/storage-pools/default
description: ""
name: default
driver: dir
used_by:
- /1.0/profiles/default
status: Created
locations:
- none

$ sudo lxc storage info default
info:
  description: ""
  driver: dir
  name: default
  space used: 1.63GiB
  total space: 7.71GiB
used by:
  profiles:
  - default
```

# スナップショット

## スナップショット作成

```
$ sudo lxc snapshot debian-test

$ sudo lxc info debian-test
Name: debian-test
Status: RUNNING
Type: container
Architecture: aarch64
PID: 14519
Created: 2025/01/10 08:57 UTC
Last Used: 2025/01/10 19:50 UTC

Resources:
  Processes: 11
  CPU usage:
    CPU usage (in seconds): 2
  Memory usage:
    Memory (current): 36.79MiB
  Network usage:
    eth0:
      Type: broadcast
      State: UP
      Host interface: veth7e862121
      MAC address: 00:16:3e:4c:48:3c
      MTU: 1500
      Bytes received: 3.88kB
      Bytes sent: 4.43kB
      Packets received: 31
      Packets sent: 43
      IP addresses:
        inet:  10.82.242.179/24 (global)
        inet6: fd42:410c:cec4:c79b:216:3eff:fe4c:483c/64 (global)
        inet6: fe80::216:3eff:fe4c:483c/64 (link)
    lo:
      Type: loopback
      State: UP
      MTU: 65536
      Bytes received: 0B
      Bytes sent: 0B
      Packets received: 0
      Packets sent: 0
      IP addresses:
        inet:  127.0.0.1/8 (local)
        inet6: ::1/128 (local)

Snapshots:
+-------+----------------------+------------+----------+
| NAME  |       TAKEN AT       | EXPIRES AT | STATEFUL |
+-------+----------------------+------------+----------+
| snap0 | 2025/01/10 19:56 UTC |            | NO       |
+-------+----------------------+------------+----------+
```

## スナップショット戻し


確認用のファイルを作成
```
$ sudo lxc exec debian-test -- touch /root/test

$ sudo lxc exec debian-test -- ls -l /root/test
-rw-r--r-- 1 root root 0 Jan 10 19:59 /root/test
```

スナップショットを戻す。
- いったん停止して再開したようだ。
- 確認用に作成したファイルは消えている。

```
$ sudo lxc restore debian-test snap0

$ sudo lxc exec debian-test -- uptime
 20:01:46 up 0 min,  0 user,  load average: 0.24, 0.21, 0.10

$ sudo lxc exec debian-test -- ls -l /root/test
```

## スナップショット削除
```
$ sudo lxc delete debian-test/snap0

$ sudo lxc info debian-test
Name: debian-test
Status: RUNNING
Type: container
Architecture: aarch64
PID: 14908
Created: 2025/01/10 08:57 UTC
Last Used: 2025/01/10 20:01 UTC

Resources:
  Processes: 11
  CPU usage:
    CPU usage (in seconds): 2
  Memory usage:
    Memory (current): 34.61MiB
  Network usage:
    lo:
      Type: loopback
      State: UP
      MTU: 65536
      Bytes received: 0B
      Bytes sent: 0B
      Packets received: 0
      Packets sent: 0
      IP addresses:
        inet:  127.0.0.1/8 (local)
        inet6: ::1/128 (local)
    eth0:
      Type: broadcast
      State: UP
      Host interface: veth844ce0f7
      MAC address: 00:16:3e:4c:48:3c
      MTU: 1500
      Bytes received: 2.86kB
      Bytes sent: 2.95kB
      Packets received: 24
      Packets sent: 30
      IP addresses:
        inet:  10.82.242.179/24 (global)
        inet6: fd42:410c:cec4:c79b:216:3eff:fe4c:483c/64 (global)
        inet6: fe80::216:3eff:fe4c:483c/64 (link)
```


# リモートへのマイグレーション

スナップショットを作成してリモートでインポートするやり方と、
lxc copy コマンドを実行してするやり方がある。(ライブマイグレートには対応していない。マイグレーション実施するときにいったんコンテナを停止する必要がある。)

## 手動でのマイグレーション

スナップショットの取得
```
$ sudo lxc snapshot debian-test testsnapshot


$ sudo lxc info debian-test
Name: debian-test
Status: RUNNING
Type: container
Architecture: aarch64
PID: 14908
Created: 2025/01/10 08:57 UTC
Last Used: 2025/01/10 20:01 UTC
:
:
Snapshots:
+--------------+----------------------+------------+----------+
|     NAME     |       TAKEN AT       | EXPIRES AT | STATEFUL |
+--------------+----------------------+------------+----------+
| testsnapshot | 2025/01/10 20:13 UTC |            | NO       |
+--------------+----------------------+------------+----------+
```

取得したスナップショットを、イメージとしてローカルリポジトリに登録
```
$ sudo lxc publish debian-test/testsnapshot --alias debian-test1
Instance published with fingerprint: 9ec484eba6b637d006364d2d2eb7c3a372f2934299c2a17837ca65619492fb4e

$ sudo lxc image list
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
|    ALIAS     | FINGERPRINT  | PUBLIC |              DESCRIPTION              | ARCHITECTURE |   TYPE    |   SIZE   |         UPLOAD DATE          |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
| debian-test1 | 9ec484eba6b6 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 157.23MB | Jan 10, 2025 at 8:19pm (UTC) |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
|              | ed9172a86d95 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 93.10MB  | Jan 10, 2025 at 8:57am (UTC) |
+--------------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
```

イメージをエクスポートする。
```
$ sudo lxc image export debian-test1 /tmp/debian-test1_archive
Image exported successfully!

$ ls -lh /tmp/debian-test1_archive.tar.gz
-rw-r--r-- 1 root root 158M Jan 10 20:27 /tmp/debian-test1_archive.tar.gz
```

リモートサイト(サイトBとする)の初期化
```
$ sudo lxd init
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, _auto_ or _none_) [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, _auto_ or _none_) [default=auto]:
Would you like the LXD server to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:

$ sudo lxc image list
+-------+-------------+--------+-------------+--------------+------+------+-------------+
| ALIAS | FINGERPRINT | PUBLIC | DESCRIPTION | ARCHITECTURE | TYPE | SIZE | UPLOAD DATE |
+-------+-------------+--------+-------------+--------------+------+------+-------------+
```

エクスポートしたイメージフィルをインポートする
```
$ ls -lh /tmp/debian-test1_archive.tar.gz
-rw-r--r-- 1 debian debian 158M Jan 10 21:00 /tmp/debian-test1_archive.tar.gz

$ sudo lxc image import /tmp/debian-test1_archive.tar.gz
Image imported with fingerprint: 9ec484eba6b637d006364d2d2eb7c3a372f2934299c2a17837ca65619492fb4e

$ sudo lxc image list
+-------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
| ALIAS | FINGERPRINT  | PUBLIC |              DESCRIPTION              | ARCHITECTURE |   TYPE    |   SIZE   |         UPLOAD DATE          |
+-------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
|       | 9ec484eba6b6 | no     | Debian bookworm arm64 (20250106_0004) | aarch64      | CONTAINER | 157.23MB | Jan 10, 2025 at 9:04pm (UTC) |
+-------+--------------+--------+---------------------------------------+--------------+-----------+----------+------------------------------+
```

イメージより、コンテナを起動する
```
$ sudo lxc launch 9ec484eba6b6 debian-test-remote
Creating debian-test-remote
Starting debian-test-remote


$ sudo lxc list
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
|        NAME        |  STATE  |        IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
| debian-test-remote | RUNNING | 10.34.31.193 (eth0) | fd42:11a1:a94e:39e9:216:3eff:fe74:369c (eth0) | CONTAINER | 0         |
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+


$ sudo lxc info debian-test-remote
Name: debian-test-remote
Status: RUNNING
Type: container
Architecture: aarch64
PID: 12251
Created: 2025/01/10 21:07 UTC
Last Used: 2025/01/10 21:08 UTC

Resources:
  Processes: 11
  CPU usage:
    CPU usage (in seconds): 2
  Memory usage:
    Memory (current): 45.27MiB
  Network usage:
    eth0:
      Type: broadcast
      State: UP
      Host interface: vethad470fd2
      MAC address: 00:16:3e:74:36:9c
      MTU: 1500
      Bytes received: 4.37kB
      Bytes sent: 4.24kB
      Packets received: 33
      Packets sent: 41
      IP addresses:
        inet:  10.34.31.193/24 (global)
        inet6: fd42:11a1:a94e:39e9:216:3eff:fe74:369c/64 (global)
        inet6: fe80::216:3eff:fe74:369c/64 (link)
    lo:
      Type: loopback
      State: UP
      MTU: 65536
      Bytes received: 0B
      Bytes sent: 0B
      Packets received: 0
      Packets sent: 0
      IP addresses:
        inet:  127.0.0.1/8 (local)
        inet6: ::1/128 (local)
```

## lxc copy コマンドによるマイグレート

マイグレート元・先どちらかのサイトでリモートアクセスを有効化
(ここではマイグレート元(以下、ホストA)で有効化している)
```
$ sudo lxc config set core.https_address "[::]"
$ sudo lxc config set core.trust_password password
```

マイグレート先サイト(先ほどのホストBと同じ)で、適当な名前をつけてリモートホストを登録する
```
$ sudo lxc remote add lxd1 192.168.11.44
Generating a client certificate. This may take a minute...
Certificate fingerprint: 51777670d0d2ecec19bc08a3243f0229d74b942bd956b39210e76e369f473bfd
ok (y/n/[fingerprint])? y
Admin password (or token) for lxd1:
Client certificate now trusted by server: lxd1

$ sudo lxc list lxd1:
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
|    NAME     |  STATE  |         IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
| debian-test | RUNNING | 10.82.242.179 (eth0) | fd42:410c:cec4:c79b:216:3eff:fe4c:483c (eth0) | CONTAINER | 1         |
+-------------+---------+----------------------+-----------------------------------------------+-----------+-----------+
```

ホストAの、移行したいコンテナをいったん停止する。
```
$ sudo lxc stop lxd1:debian-test

$ sudo lxc list lxd1:
+-------------+---------+------+------+-----------+-----------+
|    NAME     |  STATE  | IPV4 | IPV6 |   TYPE    | SNAPSHOTS |
+-------------+---------+------+------+-----------+-----------+
| debian-test | STOPPED |      |      | CONTAINER | 1         |
+-------------+---------+------+------+-----------+-----------+
```

lxc copy コマンドでホストAのコンテナをホストBへコピーする
```
$ sudo lxc copy lxd1:debian-test debian-test-copy
Transferring instance: debian-test-copy/testsnapshot: 179.85MB (2.02MB/s)

$ sudo lxc list
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
|        NAME        |  STATE  |        IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
| debian-test-copy   | STOPPED |                     |                                               | CONTAINER | 1         |
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
| debian-test-remote | RUNNING | 10.34.31.193 (eth0) | fd42:11a1:a94e:39e9:216:3eff:fe74:369c (eth0) | CONTAINER | 0         |
+--------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+

$ sudo lxc info debian-test-copy
Name: debian-test-copy
Status: STOPPED
Type: container
Architecture: aarch64
Created: 2025/01/10 21:25 UTC

Snapshots:
+--------------+----------------------+------------+----------+
|     NAME     |       TAKEN AT       | EXPIRES AT | STATEFUL |
+--------------+----------------------+------------+----------+
| testsnapshot | 2025/01/10 20:13 UTC |            | NO       |
+--------------+----------------------+------------+----------+
```
