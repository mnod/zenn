---
title: "OSPF スタブエリアの練習"
emoji: "🎫"
type: "tech"
topics: ["cisco", "ccna", "dynamips", "dynagen", "ospf"]
published: true
---

# 本エントリについて

Dynagen、Dynamips を使って、OSPF を練習します。
Dynagen、Dynamips の利用環境はすでに整っているものとします。

## 参考

https://www.infraexpert.com/study/ospfz18.html

# 基本設定

https://zenn.dev/mnod/articles/ba267c7eecabaf の続編です。

# スタブエリア

下記図で area 1 をスタブエリアとします。
```
        area0      area0      area1
eigrp1 [r1] area0 [r2] area1 [r4]
```

## 事前設定

r2、r4 でEIGRP を停止して、OSPF area 1 を追加します。
```
r2(config)#no router eigrp 1

r2(config)#router ospf 1
r2(config-router)#network 172.16.2.0 0.0.0.255 area 1
```
```
r4(config)#no router eigrp 1

r4(config)#router ospf 1
r4(config-router)#network 10.2.3.0 0.0.0.255 area 1
r4(config-router)#network 172.16.2.0 0.0.0.255 area 1
```

r4 でトポロジーテーブルとルーティングテーブルを確認します。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    32          0x80000009 0x0039FD 2
192.168.3.1     192.168.3.1     32          0x80000005 0x00F969 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    59          0x80000003 0x003DEA
10.2.1.0        172.16.2.253    59          0x80000003 0x00AFB7
172.16.0.0      172.16.2.253    59          0x80000003 0x00ED93

r4#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.0.0 [110/128] via 172.16.2.253, 00:00:33, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O IA    10.2.0.0 [110/138] via 172.16.2.253, 00:00:33, Serial1/1
O IA    10.2.1.0 [110/74] via 172.16.2.253, 00:00:33, Serial1/1
```

r2 でトポロジーテーブルとルーティングテーブルを確認します。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    644         0x8000000B 0x007056 3
172.16.2.253    172.16.2.253    1382        0x8000000B 0x005968 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    81          0x80000001 0x002007
172.16.2.0      172.16.2.253    1378        0x80000001 0x00DBA5

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    85          0x80000009 0x0039FD 2
192.168.3.1     192.168.3.1     86          0x80000005 0x00F969 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    112         0x80000003 0x003DEA
10.2.1.0        172.16.2.253    114         0x80000003 0x00AFB7
172.16.0.0      172.16.2.253    114         0x80000003 0x00ED93

r2#show ip route ospf
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:02:00, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:01:29, Serial1/0
```

r1 でトポロジーテーブルとルーティングテーブルを確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    681         0x8000000B 0x007056 3
172.16.2.253    172.16.2.253    1422        0x8000000B 0x005968 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    120         0x80000001 0x002007
172.16.2.0      172.16.2.253    1417        0x80000001 0x00DBA5

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:11:23, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:11:23, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:02:06, Serial1/0
```

### r1 で EIGRP 経路を再配布

r1 で EIGRP を動作させます。
EIGRP の経路をOSPFへ再配布します。
```
r1(config)#router eigrp 1
r1(config-router)#network 172.16.1.0 0.0.0.255
r1(config-router)#router ospf 1
r1(config-router)#redistribute eigrp 1 metric-type 1 subnets
```

r1 の状態を確認します。
```
r1#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.1.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1

r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    21          0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1           0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    505         0x80000001 0x002007
172.16.2.0      172.16.2.253    1802        0x80000001 0x00DBA5

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    20          0x80000001 0x004066 0

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:17:47, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:17:47, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:08:30, Serial1/0
```

r2 の状態を確認します。
`Summary ASB Link States` は LSA タイプ4(ASBR集約LSA) を表します。
ASBRのあるエリアのABRで生成し、ABRが知っているASBRのルータID、コスト値を含みます。スタブエリア、NSSAを除くAS全体にフラッディングします。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    81          0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    59          0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    563         0x80000001 0x002007
172.16.2.0      172.16.2.253    1861        0x80000001 0x00DBA5

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    568         0x80000009 0x0039FD 2
192.168.3.1     192.168.3.1     569         0x80000005 0x00F969 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    595         0x80000003 0x003DEA
10.2.1.0        172.16.2.253    596         0x80000003 0x00AFB7
172.16.0.0      172.16.2.253    596         0x80000003 0x00ED93

                Summary ASB Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.1.254    172.16.2.253    77          0x80000001 0x00E29F

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    82          0x80000001 0x004066 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:01:21, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:10:01, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:09:29, Serial1/0
```

r4 の状態を確認します。
タイプ4、タイプ5のLSAが見られます。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    616         0x80000009 0x0039FD 2
192.168.3.1     192.168.3.1     615         0x80000005 0x00F969 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    643         0x80000003 0x003DEA
10.2.1.0        172.16.2.253    643         0x80000003 0x00AFB7
172.16.0.0      172.16.2.253    643         0x80000003 0x00ED93

                Summary ASB Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.1.254    172.16.2.253    124         0x80000001 0x00E29F

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    129         0x80000001 0x004066 0

r4#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O IA    172.16.0.0 [110/128] via 172.16.2.253, 00:10:11, Serial1/1
O E1    172.16.1.0 [110/148] via 172.16.2.253, 00:02:03, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O IA    10.2.0.0 [110/138] via 172.16.2.253, 00:10:11, Serial1/1
O IA    10.2.1.0 [110/74] via 172.16.2.253, 00:10:11, Serial1/1
```

## スタブエリア設定

ABRである r2 でスタブエリアの設定をします。
```
r2(config)#router ospf 1
r2(config-router)#area 1 stub
```

スタブエリアのルータであるr4でもスタブエリアの設定をします。
```
r4(config)#router ospf 1
r4(config-router)#area 1 stub
```

r4 で状態を確認します。
タイプ4、タイプ5 のLSAが見られなくなりました。
また、デフォルトルートのタイプ3 LSAが見られるようになりました。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    50          0x8000000B 0x004DEB 2
192.168.3.1     192.168.3.1     44          0x80000007 0x00144F 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    65          0x80000001 0x001D64
10.2.0.0        172.16.2.253    65          0x80000004 0x0059CF
10.2.1.0        172.16.2.253    65          0x80000004 0x00CB9C
172.16.0.0      172.16.2.253    65          0x80000004 0x000A78

r4#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.0.0 [110/128] via 172.16.2.253, 00:00:41, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O IA    10.2.0.0 [110/138] via 172.16.2.253, 00:00:41, Serial1/1
O IA    10.2.1.0 [110/74] via 172.16.2.253, 00:00:41, Serial1/1
O*IA 0.0.0.0/0 [110/65] via 172.16.2.253, 00:00:41, Serial1/1
```

r2 で状態を確認します。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    284         0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    262         0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    78          0x80000001 0x002007
172.16.2.0      172.16.2.253    16          0x80000002 0x00D9A6

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    83          0x8000000B 0x004DEB 2
192.168.3.1     192.168.3.1     79          0x80000007 0x00144F 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    99          0x80000001 0x001D64
10.2.0.0        172.16.2.253    100         0x80000004 0x0059CF
10.2.1.0        172.16.2.253    100         0x80000004 0x00CB9C
172.16.0.0      172.16.2.253    100         0x80000004 0x000A78

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    285         0x80000001 0x004066 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:01:44, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:01:44, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:01:23, Serial1/0
```

## トータリースタブエリア

エリア1 をトータリースタブエリアに設定します。
ABR である r2 の設定を変更します。
スタブエリア内のルータである r4 は設定を変更する必要はありません。
```
r2(config)#router ospf 1
r2(config-router)#area 1 stub no-summary
```

r2 で状態を確認します。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    481         0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    458         0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    275         0x80000001 0x002007
172.16.2.0      172.16.2.253    213         0x80000002 0x00D9A6

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    279         0x8000000B 0x004DEB 2
192.168.3.1     192.168.3.1     276         0x80000007 0x00144F 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    23          0x80000002 0x001B65

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    482         0x80000001 0x004066 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:00:31, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:00:31, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:00:31, Serial1/0
```

r4 でトポロジーテーブルとルーティングテーブルを確認します。
タイプ3 LSAが、デフォルトルートのもののみになりました。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    324         0x8000000B 0x004DEB 2
192.168.3.1     192.168.3.1     319         0x80000007 0x00144F 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    67          0x80000002 0x001B65

r4#show ip route ospf
O*IA 0.0.0.0/0 [110/65] via 172.16.2.253, 00:01:08, Serial1/1
```

# Not-So-Stubby Area (NSSA)

下記図で area 1 を NSSA とします。
```
        area0      area0      area1
eigrp1 [r1] area0 [r2] area1 [r4] eigrp1
```

## 事前設定

前項のシナリオに続いて、r4 で eigrp を動作させます。

```
r4(config)#router eigrp 1
r4(config-router)#network 172.16.3.0 0.0.0.255
r4(config-router)#router ospf 1
r4(config-router)#redistribute eigrp 1 metric-type 1 subnets
```

状態を確認します。
```
r4#show ip eigrp topolo
IP-EIGRP Topology Table for AS(1)/ID(192.168.3.1)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0

r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    915         0x8000000B 0x004DEB 2
192.168.3.1     192.168.3.1     910         0x80000007 0x00144F 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    658         0x80000002 0x001B65

r4#show ip route ospf
O*IA 0.0.0.0/0 [110/65] via 172.16.2.253, 00:11:22, Serial1/1
```

r2 で状態を確認します。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1217        0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1195        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    1012        0x80000001 0x002007
172.16.2.0      172.16.2.253    949         0x80000002 0x00D9A6

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    1016        0x8000000B 0x004DEB 2
192.168.3.1     192.168.3.1     1013        0x80000007 0x00144F 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    760         0x80000002 0x001B65

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1221        0x80000001 0x004066 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:12:52, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:12:52, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:12:52, Serial1/0
```

r1 で状態を確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1266        0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1246        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    1062        0x80000001 0x002007
172.16.2.0      172.16.2.253    1000        0x80000002 0x00D9A6

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1265        0x80000001 0x004066 0

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:38:35, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:38:35, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:17:50, Serial1/0
```

r2、r4で、スタブエリアの設定を解除します。
```
r2(config)#router ospf 1
r2(config-router)#no area 1 stub
```
```
r4(config)#router ospf 1
r4(config-router)#no area 1 stub
```

状態を確認します。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    60          0x8000000D 0x003102 2
192.168.3.1     192.168.3.1     56          0x80000009 0x00F765 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    82          0x80000001 0x0041E8
10.2.1.0        172.16.2.253    82          0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    82          0x80000001 0x00F191

                Summary ASB Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.1.254    172.16.2.253    82          0x80000001 0x00E29F

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1414        0x80000001 0x004066 0
172.16.3.0      192.168.3.1     62          0x80000001 0x00965D 0

r4#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O IA    172.16.0.0 [110/128] via 172.16.2.253, 00:00:55, Serial1/1
O E1    172.16.1.0 [110/148] via 172.16.2.253, 00:00:55, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O IA    10.2.0.0 [110/138] via 172.16.2.253, 00:00:55, Serial1/1
O IA    10.2.1.0 [110/74] via 172.16.2.253, 00:00:55, Serial1/1
```

r2 で状態を確認します。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1444        0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1422        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    84          0x80000001 0x002007
172.16.2.0      172.16.2.253    1176        0x80000002 0x00D9A6

                Summary ASB Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
192.168.3.1     172.16.2.253    84          0x80000001 0x008C45

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    89          0x8000000D 0x003102 2
192.168.3.1     192.168.3.1     88          0x80000009 0x00F765 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    114         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    114         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    114         0x80000001 0x00F191

                Summary ASB Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.1.254    172.16.2.253    114         0x80000001 0x00E29F

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1446        0x80000001 0x004066 0
172.16.3.0      192.168.3.1     92          0x80000001 0x00965D 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:02:00, Serial1/1
O E1    172.16.3.0 [110/84] via 172.16.2.254, 00:01:32, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:02:00, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:01:32, Serial1/0
```

r1 で状態を確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1474        0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1453        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    116         0x80000001 0x002007
172.16.2.0      172.16.2.253    1208        0x80000002 0x00D9A6

                Summary ASB Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
192.168.3.1     172.16.2.253    116         0x80000001 0x008C45

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1473        0x80000001 0x004066 0
172.16.3.0      192.168.3.1     124         0x80000001 0x00965D 0

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:41:59, Serial1/0
O E1    172.16.3.0 [110/148] via 172.16.0.254, 00:01:55, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:41:59, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:02:00, Serial1/0
```

## NSSA 設定

ABR である r2 で NSSA の設定をします。
```
r2(config)#router ospf 1
r2(config-router)#area 1 nssa default-information-originate
```

NSSA のルータである r4 で NSSA の設定をします。
```
r4(config)#router ospf 1
r4(config-router)#area 1 nssa
```

r4 でトポロジーテーブルとルーティングテーブルを確認します。
`Type-7 AS External Link States` は LSA タイプ7(NSSA外部LSA) を表します。
NSSA の ASBR が生成し、　再配布した経路情報、コスト、ネクストホップを含みます。NSSA内にフラッディングします。
タイプ7 LSAは NSSA 内のみに存在し、ABR はタイプ5 LSAに変換して他のエリアへフラッディングします。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    39          0x8000000F 0x00D258 2
192.168.3.1     192.168.3.1     214         0x80000009 0x00F765 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    55          0x80000002 0x00E43E
10.2.1.0        172.16.2.253    55          0x80000002 0x00570B
172.16.0.0      172.16.2.253    55          0x80000002 0x0095E6

                Type-7 AS External Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Tag
0.0.0.0         172.16.2.253    55          0x80000001 0x00787C 0
172.16.3.0      192.168.3.1     37          0x80000001 0x003AF1 0

r4#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.0.0 [110/128] via 172.16.2.253, 00:00:43, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O IA    10.2.0.0 [110/138] via 172.16.2.253, 00:00:43, Serial1/1
O IA    10.2.1.0 [110/74] via 172.16.2.253, 00:00:43, Serial1/1
O*N2 0.0.0.0/0 [110/1] via 172.16.2.253, 00:00:43, Serial1/1
```

r2 で状態を確認します。
172.16.3.0 のタイプ5 LSAは、NSSA 内のタイプ7 LSAが、ABR(r2)でタイプ5に変換されたものです。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1609        0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1587        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    69          0x80000001 0x002007
172.16.2.0      172.16.2.253    1341        0x80000002 0x00D9A6

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    74          0x8000000F 0x00D258 2
192.168.3.1     192.168.3.1     251         0x80000009 0x00F765 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    91          0x80000002 0x00E43E
10.2.1.0        172.16.2.253    92          0x80000002 0x00570B
172.16.0.0      172.16.2.253    92          0x80000002 0x0095E6

                Type-7 AS External Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Tag
0.0.0.0         172.16.2.253    92          0x80000001 0x00787C 0
172.16.3.0      192.168.3.1     76          0x80000001 0x003AF1 0

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1610        0x80000001 0x004066 0
172.16.3.0      172.16.2.253    69          0x80000001 0x006185 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:01:36, Serial1/1
O N1    172.16.3.0 [110/84] via 172.16.2.254, 00:01:15, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:01:36, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:01:15, Serial1/0
```

r1 で状態を確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1638        0x8000000C 0x00744F 3
172.16.2.253    172.16.2.253    1618        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    100         0x80000001 0x002007
172.16.2.0      172.16.2.253    1372        0x80000002 0x00D9A6

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    1637        0x80000001 0x004066 0
172.16.3.0      172.16.2.253    99          0x80000001 0x006185 0

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:44:43, Serial1/0
O E1    172.16.3.0 [110/148] via 172.16.0.254, 00:01:39, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:44:43, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:01:44, Serial1/0
```


## トータリーNSSA

ABR である r2 でトータリーNSSA の設定をします。
NSSA のルータである r4 では特に追加設定は必要ありません。
```
r2(config)#router ospf 1
r2(config-router)#no area 1 nssa default-information-originate
r2(config-router)#area 1 nssa no-summary
```

r4 で状態を確認します。
タイプ3 LSAが、デフォルトルートのものだけになりました。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    359         0x8000000F 0x00D258 2
192.168.3.1     192.168.3.1     535         0x80000009 0x00F765 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    41          0x80000001 0x00A4D4

                Type-7 AS External Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.3.0      192.168.3.1     358         0x80000001 0x003AF1 0

r4#show ip route ospf
O*IA 0.0.0.0/0 [110/65] via 172.16.2.253, 00:00:48, Serial1/1
```

r2 で状態を確認します。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    47          0x8000000D 0x007250 3
172.16.2.253    172.16.2.253    1977        0x8000000C 0x005769 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    459         0x80000001 0x002007
172.16.2.0      172.16.2.253    1731        0x80000002 0x00D9A6

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    464         0x8000000F 0x00D258 2
192.168.3.1     192.168.3.1     642         0x80000009 0x00F765 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         172.16.2.253    146         0x80000001 0x00A4D4

                Type-7 AS External Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.3.0      192.168.3.1     467         0x80000001 0x003AF1 0

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    49          0x80000002 0x003E67 0
172.16.3.0      172.16.2.253    460         0x80000001 0x006185 0

r2#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O E1    172.16.1.0 [110/84] via 172.16.0.253, 00:02:33, Serial1/1
O N1    172.16.3.0 [110/84] via 172.16.2.254, 00:02:33, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:02:33, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:02:33, Serial1/0
```

r1 で状態を確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    71          0x8000000D 0x007250 3
172.16.2.253    172.16.2.253    2           0x8000000D 0x00556A 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    486         0x80000001 0x002007
172.16.2.0      172.16.2.253    1758        0x80000002 0x00D9A6

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
172.16.1.0      172.16.1.254    71          0x80000002 0x003E67 0
172.16.3.0      172.16.2.253    485         0x80000001 0x006185 0

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 4 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:51:07, Serial1/0
O E1    172.16.3.0 [110/148] via 172.16.0.254, 00:08:03, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:51:07, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:08:08, Serial1/0
```

# まとめ

Dynagen、Dynamips を使って、OSPF を練習しました。
スタブエリア、NSSA の設定によって、エリア内に配布されるLSAが少なくなり、ルーティングテーブルを小さくすることができるようになりました。
ABRは複数のエリアのLSAを処理する必要があります。ASBRのルータは複数のルーティングプロトコルを動かす必要があります。
