---
title: "OSPF の練習"
emoji: "🔗"
type: "tech"
topics: ["cisco", "ccna", "dynamips", "dynagen", "ospf"]
published: true
---

#  本エントリについて

Dynagen、Dynamips を使って、OSPF を練習します。
Dynagen、Dynamips の利用環境はすでに整っているものとします。

# 基本設定

https://zenn.dev/mnod/articles/c273d13817e8a6 の基本設定と同じ構成

# OSPF の有効化

```
area0       area0
[r1] area0 [r2] 
```

OSPFの有効化するには、下記の方法があります。
- `network` コマンド使用
- インタフェースコンフィギュレーションモードで、 `ip ospf` コマンドを使用

## r1 で有効化

ここでは `network` コマンドで有効化してみます。

```
r1(config)#router ospf 1
r1(config-router)#passive-interface fa0/0
r1(config-router)#network 172.16.0.253 0.0.0.255 area 0
r1(config-router)#network 10.2.0.0 0.0.0.255 area 0
```

状態を確認します。
```
r1#show ip protocols
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 172.16.1.254
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    10.2.0.0 0.0.0.255 area 0
    172.16.0.0 0.0.0.255 area 0
 Reference bandwidth unit is 100 mbps
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
  Distance: (default is 110)

r1#show ip ospf
 Routing Process "ospf 1" with ID 172.16.1.254
 Start time: 00:03:20.040, Time elapsed: 00:02:41.868
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 Router is not originating router-LSAs with maximum metric
 Initial SPF schedule delay 5000 msecs
 Minimum hold time between two consecutive SPFs 10000 msecs
 Maximum wait time between two consecutive SPFs 10000 msecs
 Incremental-SPF disabled
 Minimum LSA interval 5 secs
 Minimum LSA arrival 1000 msecs
 LSA group pacing timer 240 secs
 Interface flood pacing timer 33 msecs
 Retransmission pacing timer 66 msecs
 Number of external LSA 0. Checksum Sum 0x000000
 Number of opaque AS LSA 0. Checksum Sum 0x000000
 Number of DCbitless external and opaque AS LSA 0
 Number of DoNotAge external and opaque AS LSA 0
 Number of areas in this router is 1. 1 normal 0 stub 0 nssa
 Number of areas transit capable is 0
 External flood list length 0
 IETF NSF helper support enabled
 Cisco NSF helper support enabled
    Area BACKBONE(0) (Inactive)
        Number of interfaces in this area is 2
        Area has no authentication
        SPF algorithm last executed 00:00:43.688 ago
        SPF algorithm executed 2 times
        Area ranges are
        Number of LSA 1. Checksum Sum 0x00E0B4
        Number of opaque link LSA 0. Checksum Sum 0x000000
        Number of DCbitless LSA 0
        Number of indication LSA 0
        Number of DoNotAge LSA 0
        Flood list length 0

r1#show ip ospf interface
FastEthernet0/0 is up, line protocol is up
  Internet Address 10.2.0.254/24, Area 0
  Process ID 1, Router ID 172.16.1.254, Network Type BROADCAST, Cost: 10
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 172.16.1.254, Interface address 10.2.0.254
  No backup designated router on this network
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    No Hellos (Passive interface)
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 2/2, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 0, maximum is 0
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
Serial1/0 is up, line protocol is up
  Internet Address 172.16.0.253/24, Area 0
  Process ID 1, Router ID 172.16.1.254, Network Type POINT_TO_POINT, Cost: 64
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:07
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/1, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 0, maximum is 0
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
```

ネイバーテーブル、トポロジーテーブルを確認します。
他のルータではOSPFを起動していないので、ネイバーテーブルは空です。
トポロジーテーブル(LSDB、リンクステートデータベース)で `Router Link States` で表されるのは、LSA タイプ1(ルータLSA) です。
全てのOSPFルータで生成され、ルータID、リンク数、リンクの種類、コスト値を含みます。エリア内にフラッディングされます。
```
r1#show ip ospf neigh

r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    164         0x80000002 0x00E0B4 2
```

## r2 で有効化

ここでは `ip ospf` コマンドで有効化してみます。
```
r2(config)#router ospf 1
r2(config-router)#passive-interface f0/0
r2(config-router)#int f0/0
r2(config-if)#ip ospf 1 area 0
r2(config-if)#int s1/1
r2(config-if)#ip ospf 1 area 0
```

状態を確認します。
```
r2#show ip protocols
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 172.16.2.253
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
  Routing on Interfaces Configured Explicitly (Area 0):
    FastEthernet0/0
    Serial1/1
 Reference bandwidth unit is 100 mbps
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.1.254         110      00:01:05
  Distance: (default is 110)

r2#show ip ospf
 Routing Process "ospf 1" with ID 172.16.2.253
 Start time: 00:10:44.548, Time elapsed: 00:02:47.008
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 Router is not originating router-LSAs with maximum metric
 Initial SPF schedule delay 5000 msecs
 Minimum hold time between two consecutive SPFs 10000 msecs
 Maximum wait time between two consecutive SPFs 10000 msecs
 Incremental-SPF disabled
 Minimum LSA interval 5 secs
 Minimum LSA arrival 1000 msecs
 LSA group pacing timer 240 secs
 Interface flood pacing timer 33 msecs
 Retransmission pacing timer 66 msecs
 Number of external LSA 0. Checksum Sum 0x000000
 Number of opaque AS LSA 0. Checksum Sum 0x000000
 Number of DCbitless external and opaque AS LSA 0
 Number of DoNotAge external and opaque AS LSA 0
 Number of areas in this router is 1. 1 normal 0 stub 0 nssa
 Number of areas transit capable is 0
 External flood list length 0
 IETF NSF helper support enabled
 Cisco NSF helper support enabled
    Area BACKBONE(0)
        Number of interfaces in this area is 2
        Area has no authentication
        SPF algorithm last executed 00:01:12.656 ago
        SPF algorithm executed 3 times
        Area ranges are
        Number of LSA 2. Checksum Sum 0x012278
        Number of opaque link LSA 0. Checksum Sum 0x000000
        Number of DCbitless LSA 0
        Number of indication LSA 0
        Number of DoNotAge LSA 0
        Flood list length 0

r2#show ip ospf interface
FastEthernet0/0 is up, line protocol is up
  Internet Address 10.2.1.254/24, Area 0
  Process ID 1, Router ID 172.16.2.253, Network Type BROADCAST, Cost: 10
  Enabled by interface config, including secondary ip addresses
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 172.16.2.253, Interface address 10.2.1.254
  No backup designated router on this network
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    No Hellos (Passive interface)
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/1, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 0, maximum is 0
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
Serial1/1 is up, line protocol is up
  Internet Address 172.16.0.254/24, Area 0
  Process ID 1, Router ID 172.16.2.253, Network Type POINT_TO_POINT, Cost: 64
  Enabled by interface config, including secondary ip addresses
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:01
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 2/2, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 172.16.1.254
  Suppress hello for 0 neighbor(s)
```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
r1 をネイバーと認識しました。
```
r2#show ip ospf neigh

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.1.254      0   FULL/  -        00:00:37    172.16.0.253    Serial1/1

r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    171         0x80000003 0x00804E 3
172.16.2.253    172.16.2.253    138         0x80000003 0x00A22A 3

r2#show ip route ospf
     10.0.0.0/24 is subnetted, 2 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:02:21, Serial1/1
```

r1でもネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
こちらも r2 をネイバーと認識しました。
```
r1#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.2.253      0   FULL/  -        00:00:37    172.16.0.254    Serial1/0

r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    207         0x80000003 0x00804E 3
172.16.2.253    172.16.2.253    177         0x80000003 0x00A22A 3

r1#show ip route ospf
     10.0.0.0/24 is subnetted, 2 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:03:30, Serial1/0
```

# 認証

OSPFの認証設定は以下の方法があります。
- インタフェースごとに有効化
- エリアごとに裕逢花

## インタフェースで有効化

```
r1(config)#int s1/0
r1(config-if)#ip ospf message-digest-key 1 md5 cisco
r1(config-if)#ip ospf authentication message-digest
```

状態を確認します。
```
r1#show ip ospf interface s1/0
Serial1/0 is up, line protocol is up
  Internet Address 172.16.0.253/24, Area 0
  Process ID 1, Router ID 172.16.1.254, Network Type POINT_TO_POINT, Cost: 64
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:08
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/1, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
  Message digest authentication enabled
    Youngest key id is 1
```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
認証不一致のため、ネイバー関係が解消しました。
```
r1#show ip ospf neigh

r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    65          0x80000004 0x00DCB6 2
172.16.2.253    172.16.2.253    578         0x80000003 0x00A22A 3

r1#show ip route ospf

```

r2でもネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認しておきます。
```
r2#show ip ospf neigh

r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    670         0x80000003 0x00804E 3
172.16.2.253    172.16.2.253    124         0x80000004 0x00E3AE 2

r2#show ip route ospf

```

## エリアで有効化

```
r2(config)#interface s1/1
r2(config-if)#ip ospf message-digest-key 1 md5 cisco
r2(config-if)#router ospf 1
r2(config-router)#area 0 authentication message-digest
```

状態を確認します。
```
r2#show ip ospf
 Routing Process "ospf 1" with ID 172.16.2.253
 Start time: 00:10:44.548, Time elapsed: 00:14:45.784
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 Router is not originating router-LSAs with maximum metric
 Initial SPF schedule delay 5000 msecs
 Minimum hold time between two consecutive SPFs 10000 msecs
 Maximum wait time between two consecutive SPFs 10000 msecs
 Incremental-SPF disabled
 Minimum LSA interval 5 secs
 Minimum LSA arrival 1000 msecs
 LSA group pacing timer 240 secs
 Interface flood pacing timer 33 msecs
 Retransmission pacing timer 66 msecs
 Number of external LSA 0. Checksum Sum 0x000000
 Number of opaque AS LSA 0. Checksum Sum 0x000000
 Number of DCbitless external and opaque AS LSA 0
 Number of DoNotAge external and opaque AS LSA 0
 Number of areas in this router is 1. 1 normal 0 stub 0 nssa
 Number of areas transit capable is 0
 External flood list length 0
 IETF NSF helper support enabled
 Cisco NSF helper support enabled
    Area BACKBONE(0)
        Number of interfaces in this area is 2
        Area has message digest authentication
        SPF algorithm last executed 00:00:52.424 ago
        SPF algorithm executed 5 times
        Area ranges are
        Number of LSA 2. Checksum Sum 0x011A7C
        Number of opaque link LSA 0. Checksum Sum 0x000000
        Number of DCbitless LSA 0
        Number of indication LSA 0
        Number of DoNotAge LSA 0
        Flood list length 0

r2#show ip ospf int s1/1
Serial1/1 is up, line protocol is up
  Internet Address 172.16.0.254/24, Area 0
  Process ID 1, Router ID 172.16.2.253, Network Type POINT_TO_POINT, Cost: 64
  Enabled by interface config, including secondary ip addresses
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:08
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 2/2, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 172.16.1.254
  Suppress hello for 0 neighbor(s)
  Message digest authentication enabled
    Youngest key id is 1

```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
ネイバー関係が回復し、ルーティングテーブルに経路が乗りました。
```
r2#show ip ospf neigh

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.1.254      0   FULL/  -        00:00:33    172.16.0.253    Serial1/1

r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    192         0x80000005 0x007C50 3
172.16.2.253    172.16.2.253    191         0x80000005 0x009E2C 3

r2#show ip route ospf
     10.0.0.0/24 is subnetted, 2 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:03:14, Serial1/1
```

r1でもネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
```
r1#show ip ospf neigh

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.2.253      0   FULL/  -        00:00:34    172.16.0.254    Serial1/0

r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    225         0x80000005 0x007C50 3
172.16.2.253    172.16.2.253    226         0x80000005 0x009E2C 3

r1#show ip route ospf
     10.0.0.0/24 is subnetted, 2 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:03:46, Serial1/0

```

エリアごとの認証だとこの後の練習で面倒なので、r2もインタフェースごとの認証にしておきます。
```
r2(config-if)#router ospf 1
r2(config-router)#no area 0 authentication message-digest
r2(config-router)#int s1/1
r2(config-if)#ip ospf authentication message-digest
```

# マルチアエリア

エリア0だけの構成でしたが、エリア1を追加してみます。
```
area0      area0      area1
[r1] area0 [r2] area1 [r4]
```

## r2 で area 1 の追加

```
r2(config)#router ospf 1
r2(config-router)#network 172.16.2.0 0.0.0.255 area 1
```

状態を確認します。
```
r2#show ip protocols
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 172.16.2.253
  It is an area border router
  Number of areas in this router is 2. 2 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    172.16.2.0 0.0.0.255 area 1
  Routing on Interfaces Configured Explicitly (Area 0):
    FastEthernet0/0
    Serial1/1
 Reference bandwidth unit is 100 mbps
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.1.254         110      00:00:09
  Distance: (default is 110)

r2#show ip ospf
 Routing Process "ospf 1" with ID 172.16.2.253
 Start time: 00:10:44.548, Time elapsed: 00:34:33.860
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 It is an area border router
 Router is not originating router-LSAs with maximum metric
 Initial SPF schedule delay 5000 msecs
 Minimum hold time between two consecutive SPFs 10000 msecs
 Maximum wait time between two consecutive SPFs 10000 msecs
 Incremental-SPF disabled
 Minimum LSA interval 5 secs
 Minimum LSA arrival 1000 msecs
 LSA group pacing timer 240 secs
 Interface flood pacing timer 33 msecs
 Retransmission pacing timer 66 msecs
 Number of external LSA 0. Checksum Sum 0x000000
 Number of opaque AS LSA 0. Checksum Sum 0x000000
 Number of DCbitless external and opaque AS LSA 0
 Number of DoNotAge external and opaque AS LSA 0
 Number of areas in this router is 2. 2 normal 0 stub 0 nssa
 Number of areas transit capable is 0
 External flood list length 0
 IETF NSF helper support enabled
 Cisco NSF helper support enabled
    Area BACKBONE(0)
        Number of interfaces in this area is 2
        Area has no authentication
        SPF algorithm last executed 00:00:52.440 ago
        SPF algorithm executed 7 times
        Area ranges are
        Number of LSA 3. Checksum Sum 0x01F71E
        Number of opaque link LSA 0. Checksum Sum 0x000000
        Number of DCbitless LSA 0
        Number of indication LSA 0
        Number of DoNotAge LSA 0
        Flood list length 0
    Area 1
        Number of interfaces in this area is 1
        Area has no authentication
        SPF algorithm last executed 00:00:52.448 ago
        SPF algorithm executed 2 times
        Area ranges are
        Number of LSA 4. Checksum Sum 0x028648
        Number of opaque link LSA 0. Checksum Sum 0x000000
        Number of DCbitless LSA 0
        Number of indication LSA 0
        Number of DoNotAge LSA 0
        Flood list length 0

r2#show ip ospf interface
FastEthernet0/0 is up, line protocol is up
  Internet Address 10.2.1.254/24, Area 0
  Process ID 1, Router ID 172.16.2.253, Network Type BROADCAST, Cost: 10
  Enabled by interface config, including secondary ip addresses
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 172.16.2.253, Interface address 10.2.1.254
  No backup designated router on this network
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    No Hellos (Passive interface)
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/1, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 0, maximum is 0
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
Serial1/1 is up, line protocol is up
  Internet Address 172.16.0.254/24, Area 0
  Process ID 1, Router ID 172.16.2.253, Network Type POINT_TO_POINT, Cost: 64
  Enabled by interface config, including secondary ip addresses
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:05
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 2/2, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 172.16.1.254
  Suppress hello for 0 neighbor(s)
  Message digest authentication enabled
    Youngest key id is 1
Serial1/0 is up, line protocol is up
  Internet Address 172.16.2.253/24, Area 1
  Process ID 1, Router ID 172.16.2.253, Network Type POINT_TO_POINT, Cost: 64
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:06
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/3, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 0, maximum is 0
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
```

トポロジーテーブルを確認します。r2 はエリア0とエリア1をまたがる ABR なので、それぞれのエリアの LSA をもっています。
`Summary Net Link States` は LSA タイプ3(ネットワーク集約LSA) を表します。ABR が生成し、各エリアごとの経路情報、コスト値を含みます。各エリア内にフラッディングします。
(経路の集約は自分で設定する必要があります。)

```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1300        0x80000005 0x007C50 3
172.16.2.253    172.16.2.253    110         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.2.0      172.16.2.253    106         0x80000001 0x00DBA5

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    110         0x80000001 0x009F1A 1

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    111         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    111         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    114         0x80000001 0x00F191
```

r1でトポロジーテーブル、ルーティングテーブルを確認します。
r1が属しているのは Area 0 のみなので、Area 0 の LSA のみを持っています。
ルーティングテーブルにおいて、他のエリアの経路には `O IA` のコードがつきます。ここでは、エリア1 の経路に `O IA` のコードがついています。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1368        0x80000005 0x007C50 3
172.16.2.253    172.16.2.253    180         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.2.0      172.16.2.253    176         0x80000001 0x00DBA5

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:03:25, Serial1/0
     10.0.0.0/24 is subnetted, 2 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:23:14, Serial1/0
```

## r4 で有効化

```
r4(config)#router ospf 1
r4(config-router)#network 172.16.2.0 0.0.0.255 area 1
r4(config-router)#network 10.2.3.0 0.0.0.255 area 1
r4(config-router)#passive-interface f0/0
```

状態を確認します。
```
r4#show ip protocols
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 192.168.3.1
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    10.2.3.0 0.0.0.255 area 1
    172.16.2.0 0.0.0.255 area 1
 Reference bandwidth unit is 100 mbps
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.2.253         110      00:00:19
  Distance: (default is 110)

r4#show ip ospf
 Routing Process "ospf 1" with ID 192.168.3.1
 Start time: 00:47:30.748, Time elapsed: 00:02:16.436
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 Router is not originating router-LSAs with maximum metric
 Initial SPF schedule delay 5000 msecs
 Minimum hold time between two consecutive SPFs 10000 msecs
 Maximum wait time between two consecutive SPFs 10000 msecs
 Incremental-SPF disabled
 Minimum LSA interval 5 secs
 Minimum LSA arrival 1000 msecs
 LSA group pacing timer 240 secs
 Interface flood pacing timer 33 msecs
 Retransmission pacing timer 66 msecs
 Number of external LSA 0. Checksum Sum 0x000000
 Number of opaque AS LSA 0. Checksum Sum 0x000000
 Number of DCbitless external and opaque AS LSA 0
 Number of DoNotAge external and opaque AS LSA 0
 Number of areas in this router is 1. 1 normal 0 stub 0 nssa
 Number of areas transit capable is 0
 External flood list length 0
 IETF NSF helper support enabled
 Cisco NSF helper support enabled
    Area 1
        Number of interfaces in this area is 2
        Area has no authentication
        SPF algorithm last executed 00:00:30.740 ago
        SPF algorithm executed 3 times
        Area ranges are
        Number of LSA 5. Checksum Sum 0x026B50
        Number of opaque link LSA 0. Checksum Sum 0x000000
        Number of DCbitless LSA 0
        Number of indication LSA 0
        Number of DoNotAge LSA 0
        Flood list length 0

r4#show ip ospf interface
FastEthernet0/0 is up, line protocol is up
  Internet Address 10.2.3.254/24, Area 1
  Process ID 1, Router ID 192.168.3.1, Network Type BROADCAST, Cost: 10
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 192.168.3.1, Interface address 10.2.3.254
  No backup designated router on this network
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    No Hellos (Passive interface)
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 2/2, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 0, maximum is 0
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 0, Adjacent neighbor count is 0
  Suppress hello for 0 neighbor(s)
Serial1/1 is up, line protocol is up
  Internet Address 172.16.2.254/24, Area 1
  Process ID 1, Router ID 192.168.3.1, Network Type POINT_TO_POINT, Cost: 64
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:07
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/1, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 172.16.2.253
  Suppress hello for 0 neighbor(s)
```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
属しているのは Area 1 のみなので、Area 1 の LSA のみを持っています。
ルーティングテーブルで `O IA` のコードがついているのは、エリア0 の経路です。
```
r4#show ip ospf neigh

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.2.253      0   FULL/  -        00:00:31    172.16.2.253    Serial1/1

r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    154         0x80000002 0x0041FE 2
192.168.3.1     192.168.3.1     131         0x80000002 0x004224 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    451         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    451         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    451         0x80000001 0x00F191

r4#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.0.0 [110/128] via 172.16.2.253, 00:01:30, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
O IA    10.2.0.0 [110/138] via 172.16.2.253, 00:01:30, Serial1/1
O IA    10.2.1.0 [110/74] via 172.16.2.253, 00:01:30, Serial1/1
```

r2でもネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
ABR では属している両方のエリアのタイプ1 LSAを持っているので、ルーティングテーブルでは `O IA` のコードは現れません。
```
r2#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.1.254      0   FULL/  -        00:00:31    172.16.0.253    Serial1/1
192.168.3.1       0   FULL/  -        00:00:33    172.16.2.254    Serial1/0

r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1678        0x80000005 0x007C50 3
172.16.2.253    172.16.2.253    488         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    163         0x80000001 0x002007
172.16.2.0      172.16.2.253    484         0x80000001 0x00DBA5

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    190         0x80000002 0x0041FE 2
192.168.3.1     192.168.3.1     169         0x80000002 0x004224 3

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    489         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    491         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    491         0x80000001 0x00F191

r2#show ip route ospf
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:08:34, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:03:08, Serial1/0
```

r1でトポロジーテーブル、ルーティングテーブルを確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1718        0x80000005 0x007C50 3
172.16.2.253    172.16.2.253    531         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    206         0x80000001 0x002007
172.16.2.0      172.16.2.253    527         0x80000001 0x00DBA5

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:08:53, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:28:42, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:03:33, Serial1/0
```

## abr での経路集約

経路集約によって、LSA の数を減らすことができます。計算の負荷が減り、ルーティングテーブルも小さくなります。

```
area0      area0      area1
[r1] area0 [r2] area1 [r4]
　　　　　　　　　　　subinterface
                      192.168.0.0/24
                      192.168.1.0/24
                      192.168.2.0/24
                      192.168.3.0/24
```

経路集約を試すために r4 で下記のようにアドレスを追加します。
```
r4(config)#int f0/1.0
r4(config-if)#ip addr 192.168.0.1 255.255.255.0
r4(config-if)#no shut
r4(config-if)#int f0/1.1
r4(config-subif)#encapsulation dot1Q 1
r4(config-subif)#ip addr 192.168.1.1 255.255.255.0
r4(config-subif)#no shut
r4(config-subif)#int f0/1.2
r4(config-subif)#encapsulation dot1Q 2
r4(config-subif)#ip addr 192.168.2.1 255.255.255.0
r4(config-subif)#no shut
r4(config-subif)#int f0/1.3
r4(config-subif)#encapsulation dot1Q 3
r4(config-subif)#ip addr 192.168.3.1 255.255.255.0
r4(config-subif)#no shut

r4#show ip int bri
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            10.2.3.254      YES NVRAM  up                    up
FastEthernet0/1            192.168.0.1     YES manual up                    up
FastEthernet0/1.1          192.168.1.1     YES manual up                    up
FastEthernet0/1.2          192.168.2.1     YES manual up                    up
FastEthernet0/1.3          192.168.3.1     YES manual up                    up
Serial1/0                  172.16.3.253    YES NVRAM  up                    up
Serial1/1                  172.16.2.254    YES NVRAM  up                    up
Serial1/2                  unassigned      YES NVRAM  administratively down down
Serial1/3                  unassigned      YES NVRAM  administratively down down
```

追加したインタフェースで ospf を有効にします。
```
r4(config)#router ospf 1
r4(config-router)#network 192.168.0.0 0.0.3.255 area 1
```

状態を確認します。
ルータIDが、設定したIPアドレスのうちの最大のものである `192.168.3.1` に変化しています。
```
r4#show ip protocols
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 192.168.3.1
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    10.2.3.0 0.0.0.255 area 1
    172.16.2.0 0.0.0.255 area 1
    192.168.0.0 0.0.3.255 area 1
 Reference bandwidth unit is 100 mbps
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.2.253         110      00:05:05
  Distance: (default is 110)
```

トポロジーテーブルを確認します。
```
r4#show ip ospf database

            OSPF Router with ID (192.168.3.1) (Process ID 1)

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    454         0x80000002 0x0041FE 2
192.168.3.1     192.168.3.1     105         0x80000003 0x00FA56 7

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    752         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    752         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    752         0x80000001 0x00F191
```

r2でもトポロジーテーブル、ルーティングテーブルを確認します。
追加した経路に対して、現段階では `192.168.0.0/24` から `192.168.3.0/24` の4つのタイプ3LSAが見られます。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    9           0x80000006 0x007A51 3
172.16.2.253    172.16.2.253    795         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    470         0x80000001 0x002007
172.16.2.0      172.16.2.253    790         0x80000001 0x00DBA5
192.168.0.0     172.16.2.253    144         0x80000001 0x002AA2
192.168.1.0     172.16.2.253    144         0x80000001 0x001FAC
192.168.2.0     172.16.2.253    144         0x80000001 0x0014B6
192.168.3.0     172.16.2.253    144         0x80000001 0x0009C0

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    497         0x80000002 0x0041FE 2
192.168.3.1     192.168.3.1     150         0x80000003 0x00FA56 7

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    798         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    798         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    798         0x80000001 0x00F191

r2#show ip route ospf
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:13:28, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:08:02, Serial1/0
O    192.168.0.0/24 [110/74] via 172.16.2.254, 00:02:37, Serial1/0
O    192.168.1.0/24 [110/74] via 172.16.2.254, 00:02:37, Serial1/0
O    192.168.2.0/24 [110/74] via 172.16.2.254, 00:02:37, Serial1/0
O    192.168.3.0/24 [110/74] via 172.16.2.254, 00:02:37, Serial1/0
```

r1でもトポロジーテーブル、ルーティングテーブルを確認します。
r4 で追加したアドレスが `O IA` のコードで、 `192.168.0.0/24` から `192.168.3.0/24` の4つのルートが乗っています。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    34          0x80000006 0x007A51 3
172.16.2.253    172.16.2.253    822         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    497         0x80000001 0x002007
172.16.2.0      172.16.2.253    818         0x80000001 0x00DBA5
192.168.0.0     172.16.2.253    171         0x80000001 0x002AA2
192.168.1.0     172.16.2.253    171         0x80000001 0x001FAC
192.168.2.0     172.16.2.253    171         0x80000001 0x0014B6
192.168.3.0     172.16.2.253    171         0x80000001 0x0009C0

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:13:40, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:33:29, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:08:20, Serial1/0
O IA 192.168.0.0/24 [110/138] via 172.16.0.254, 00:02:54, Serial1/0
O IA 192.168.1.0/24 [110/138] via 172.16.0.254, 00:02:54, Serial1/0
O IA 192.168.2.0/24 [110/138] via 172.16.0.254, 00:02:54, Serial1/0
O IA 192.168.3.0/24 [110/138] via 172.16.0.254, 00:02:54, Serial1/0
```

ABR である r2 で経路集約を設定します。
```
r2(config)#router ospf 1
r2(config-router)#area 1 range 192.168.0.0 255.255.252.0
```

トポロジーテーブル、ルーティングテーブルを確認します。
r4で追加したアドレスについてのタイプ3 SLA が 192.168.0.0 の一つだけになりました。
ルーティングテーブルに Null0 デバイスの経路ができました。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    138         0x80000006 0x007A51 3
172.16.2.253    172.16.2.253    924         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    599         0x80000001 0x002007
172.16.2.0      172.16.2.253    920         0x80000001 0x00DBA5
192.168.0.0     172.16.2.253    26          0x80000002 0x0019B5

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    626         0x80000002 0x0041FE 2
192.168.3.1     192.168.3.1     280         0x80000003 0x00FA56 7

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.0.0        172.16.2.253    925         0x80000001 0x0041E8
10.2.1.0        172.16.2.253    927         0x80000001 0x00B3B5
172.16.0.0      172.16.2.253    927         0x80000001 0x00F191

r2#show ip route ospf
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:00:43, Serial1/1
O       10.2.3.0 [110/74] via 172.16.2.254, 00:00:43, Serial1/0
O    192.168.0.0/24 [110/74] via 172.16.2.254, 00:00:43, Serial1/0
O    192.168.1.0/24 [110/74] via 172.16.2.254, 00:00:43, Serial1/0
O    192.168.2.0/24 [110/74] via 172.16.2.254, 00:00:43, Serial1/0
O    192.168.3.0/24 [110/74] via 172.16.2.254, 00:00:43, Serial1/0
O    192.168.0.0/22 is a summary, 00:00:43, Null0
```

r1でもトポロジーテーブル、ルーティングテーブルを確認します。
r4で追加したアドレスについてのタイプ3 SLA が 192.168.0.0 の一つだけになりました。
ルーティングテーブルに乗る経路が 192.168.0.0/22 の一つだけになりました。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    198         0x80000006 0x007A51 3
172.16.2.253    172.16.2.253    986         0x80000006 0x009F29 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.3.0        172.16.2.253    661         0x80000001 0x002007
172.16.2.0      172.16.2.253    981         0x80000001 0x00DBA5
192.168.0.0     172.16.2.253    88          0x80000002 0x0019B5

r1#show ip route ospf
     172.16.0.0/24 is subnetted, 3 subnets
O IA    172.16.2.0 [110/128] via 172.16.0.254, 00:16:28, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.1.0 [110/74] via 172.16.0.254, 00:36:17, Serial1/0
O IA    10.2.3.0 [110/138] via 172.16.0.254, 00:11:08, Serial1/0
O IA 192.168.0.0/22 [110/138] via 172.16.0.254, 00:01:35, Serial1/0
```

# 再配送

```
area0      area0      
[r1] area0 [r2] eigrp1 [r4]
```

静的ルート、外部ルートを再配送することができます。
ここでは EIGRP の AS1で学習したルートの再配送を練習します。

## EIGRP を有効化
```
r2(config)#router ospf 1
r2(config-router)#no network 172.16.2.0 0.0.0.255 area 1

r2(config)#router eigrp 1
r2(config-router)#no auto-summary
r2(config-router)#network 172.16.2.0 0.0.0.255
```

状態を確認します。
```
r2#show ip protocols
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 172.16.2.253
  Number of areas in this router is 2. 2 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
  Routing on Interfaces Configured Explicitly (Area 0):
    FastEthernet0/0
    Serial1/1
 Reference bandwidth unit is 100 mbps
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.2.253         110      00:12:00
    172.16.1.254         110      00:12:00
    192.168.3.1          110      00:12:00
  Distance: (default is 110)

Routing Protocol is "eigrp 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Default networks flagged in outgoing updates
  Default networks accepted from incoming updates
  EIGRP metric weight K1=1, K2=0, K3=1, K4=0, K5=0
  EIGRP maximum hopcount 100
  EIGRP maximum metric variance 1
  Redistributing: eigrp 1
  EIGRP NSF-aware route hold timer is 240s
  Automatic network summarization is not in effect
  Maximum path: 4
  Routing for Networks:
    172.16.2.0/24
  Routing Information Sources:
    Gateway         Distance      Last Update
  Distance: internal 90 external 170
```


r4 でEIGPR を有効化します。
```
r4(config)#no router ospf 1

r4(config)#router eigrp 1
r4(config-router)#passive-interface f0/0
r4(config-router)#no auto-summary
r4(config-router)#network 172.16.2.0 0.0.0.255
r4(config-router)#network 10.2.3.0 0.0.0.255
```

状態を確認します。
```
r4#show ip protocols
Routing Protocol is "eigrp 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Default networks flagged in outgoing updates
  Default networks accepted from incoming updates
  EIGRP metric weight K1=1, K2=0, K3=1, K4=0, K5=0
  EIGRP maximum hopcount 100
  EIGRP maximum metric variance 1
  Redistributing: eigrp 1
  EIGRP NSF-aware route hold timer is 240s
  Automatic network summarization is not in effect
  Maximum path: 4
  Routing for Networks:
    10.2.3.0/24
    172.16.2.0/24
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    (this router)         90      00:01:15
    172.16.2.253          90      00:01:04
  Distance: internal 90 external 170
```

```
r4#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 2 subnets
C       172.16.2.0 is directly connected, Serial1/1
C       172.16.3.0 is directly connected, Serial1/0
     10.0.0.0/24 is subnetted, 1 subnets
C       10.2.3.0 is directly connected, FastEthernet0/0
C    192.168.0.0/24 is directly connected, FastEthernet0/1
C    192.168.1.0/24 is directly connected, FastEthernet0/1.1
C    192.168.2.0/24 is directly connected, FastEthernet0/1.2
C    192.168.3.0/24 is directly connected, FastEthernet0/1.3
```

r2 でルーティングテーブルを確認します。
```
r2#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 2 subnets
C       172.16.0.0 is directly connected, Serial1/1
C       172.16.2.0 is directly connected, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 01:37:25, Serial1/1
C       10.2.1.0 is directly connected, FastEthernet0/0
D       10.2.3.0 [90/2195456] via 172.16.2.254, 01:23:07, Serial1/0
```

## 再配送設定

r2で 再配送の設定をします。
EIGRP 側に OSPF のルートを再配送してみます。
```
r2(config)#router eigrp 1
r2(config-router)#redistribute ospf 1
r2(config-router)#default-metric 1533 2000 255 1 1500
```

r4のルーティングテーブルを確認します。
```
r4#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(192.168.3.1)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2693888
        via 172.16.2.253 (2693888/2181888), Serial1/1
P 10.2.1.0/24, 1 successors, FD is 2693888
        via 172.16.2.253 (2693888/2181888), Serial1/1
P 10.2.3.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2693888
        via 172.16.2.253 (2693888/2181888), Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1

r4#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 3 subnets
D EX    172.16.0.0 [170/2693888] via 172.16.2.253, 00:00:18, Serial1/1
C       172.16.2.0 is directly connected, Serial1/1
C       172.16.3.0 is directly connected, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
D EX    10.2.0.0 [170/2693888] via 172.16.2.253, 00:00:18, Serial1/1
D EX    10.2.1.0 [170/2693888] via 172.16.2.253, 00:00:18, Serial1/1
C       10.2.3.0 is directly connected, FastEthernet0/0
C    192.168.0.0/24 is directly connected, FastEthernet0/1
C    192.168.1.0/24 is directly connected, FastEthernet0/1.1
C    192.168.2.0/24 is directly connected, FastEthernet0/1.2
C    192.168.3.0/24 is directly connected, FastEthernet0/1.3
```

今度は OSPF 側に EIGRP のルートを再配送してみます。
```
r2(config)#router ospf 1
r2(config-router)#redistribute eigrp 1 metric-type 1 subnets
```

トポロジーテーブルの `Type-5 AS External Link States` は LSA タイプ5(AS外部LSA) を表します。
ASBR で生成され、外部ルートから再配布した経路情報、コスト、ネクストホップを含みます。スタブエリア、NSSAエリアを除くOSPFのAS全体にフラッディングされます。
```
r2# show ip ospf datab

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1170        0x80000007 0x007852 3
172.16.2.253    172.16.2.253    16          0x80000009 0x005A6A 3

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    16          0x80000004 0x008143 0
172.16.3.253    172.16.3.253    618         0x80000003 0x00D0DF 7

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.2.3.0        172.16.2.253    15          0x80000001 0x001441 0
172.16.2.0      172.16.2.253    15          0x80000001 0x003471 0
```

r1でルーティングテーブルを確認します。
```
r1# show ip ospf datab

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1274        0x80000007 0x007852 3
172.16.2.253    172.16.2.253    121         0x80000009 0x005A6A 3

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.2.3.0        172.16.2.253    121         0x80000001 0x001441 0
172.16.2.0      172.16.2.253    121         0x80000001 0x003471 0

r1#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 3 subnets
C       172.16.0.0 is directly connected, Serial1/0
C       172.16.1.0 is directly connected, Serial1/1
O E1    172.16.2.0 [110/84] via 172.16.0.254, 00:00:46, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
C       10.2.0.0 is directly connected, FastEthernet0/0
O       10.2.1.0 [110/74] via 172.16.0.254, 02:26:41, Serial1/0
O E1    10.2.3.0 [110/84] via 172.16.0.254, 00:00:46, Serial1/0
```

## asbr での経路集約

```
area0      area0      
[r1] area0 [r2] eigrp1 [r4]
　　　　　　　　　　　subinterface
                      192.168.0.0/24
                      192.168.1.0/24
                      192.168.2.0/24
                      192.168.3.0/24
```

経路集約の練習のために追加したアドレスを EIGRP にアドバタイズします。
```
r4(config)#router eigrp 1
r4(config-router)#network 192.168.0.0 0.0.3.255
```

r2 でトポロジーテーブルとルーティングテーブルを確認します。
r4 で追加したアドレスは`192.168.0.0/24` から `192.168.3.0/24` の4つのLSAでトポロジーテーブルに登録されています。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1408        0x80000009 0x007454 3
172.16.2.253    172.16.2.253    658         0x8000000B 0x00982A 3

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    658         0x80000008 0x007947 0

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.2.3.0        172.16.2.253    636         0x80000001 0x001441 0
172.16.2.0      172.16.2.253    636         0x80000001 0x003471 0
192.168.0.0     172.16.2.253    108         0x80000001 0x001EDC 0
192.168.1.0     172.16.2.253    108         0x80000001 0x0013E6 0
192.168.2.0     172.16.2.253    108         0x80000001 0x0008F0 0
192.168.3.0     172.16.2.253    108         0x80000001 0x00FCFA 0

r2#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 2 subnets
C       172.16.0.0 is directly connected, Serial1/1
C       172.16.2.0 is directly connected, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 02:00:37, Serial1/1
C       10.2.1.0 is directly connected, FastEthernet0/0
D       10.2.3.0 [90/2195456] via 172.16.2.254, 01:46:19, Serial1/0
D    192.168.0.0/24 [90/2195456] via 172.16.2.254, 00:00:35, Serial1/0
D    192.168.1.0/24 [90/2195456] via 172.16.2.254, 00:00:35, Serial1/0
D    192.168.2.0/24 [90/2195456] via 172.16.2.254, 00:00:37, Serial1/0
D    192.168.3.0/24 [90/2195456] via 172.16.2.254, 00:00:37, Serial1/0
```

r1 でもトポロジーテーブルとルーティングテーブルを確認します。
LSA タイプ5の経路は `O E1` `O E2` でルーティングテーブルに乗ります。
経路集約されていないので `192.168.0.0/24` から `192.168.3.0/24` の4つのLSA、経路が見られます。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1393        0x80000009 0x007454 3
172.16.2.253    172.16.2.253    645         0x8000000B 0x00982A 3

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.2.3.0        172.16.2.253    623         0x80000001 0x001441 0
172.16.2.0      172.16.2.253    623         0x80000001 0x003471 0
192.168.0.0     172.16.2.253    95          0x80000001 0x001EDC 0
192.168.1.0     172.16.2.253    95          0x80000001 0x0013E6 0
192.168.2.0     172.16.2.253    95          0x80000001 0x0008F0 0
192.168.3.0     172.16.2.253    95          0x80000001 0x00FCFA 0

r1#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 3 subnets
C       172.16.0.0 is directly connected, Serial1/0
C       172.16.1.0 is directly connected, Serial1/1
O E1    172.16.2.0 [110/84] via 172.16.0.254, 00:11:01, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
C       10.2.0.0 is directly connected, FastEthernet0/0
O       10.2.1.0 [110/74] via 172.16.0.254, 02:36:56, Serial1/0
O E1    10.2.3.0 [110/84] via 172.16.0.254, 00:11:01, Serial1/0
O E1 192.168.0.0/24 [110/84] via 172.16.0.254, 00:02:13, Serial1/0
O E1 192.168.1.0/24 [110/84] via 172.16.0.254, 00:02:14, Serial1/0
O E1 192.168.2.0/24 [110/84] via 172.16.0.254, 00:02:14, Serial1/0
O E1 192.168.3.0/24 [110/84] via 172.16.0.254, 00:02:14, Serial1/0
```

## asbr で経路集約

r2 で経路集約の設定をします。
```
r2(config)#router ospf 1
r2(config-router)#summary-address 192.168.0.0 255.255.252.0
```

トポロジーテーブルを確認します。 `192.168.0.0` の一つに集約されました。
ルーティングテーブルに Null0 の集約ルートが乗りました。
```
r2#show ip ospf database

            OSPF Router with ID (172.16.2.253) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1651        0x80000009 0x007454 3
172.16.2.253    172.16.2.253    901         0x8000000B 0x00982A 3

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.2.253    172.16.2.253    901         0x80000008 0x007947 0

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.2.3.0        172.16.2.253    879         0x80000001 0x001441 0
172.16.2.0      172.16.2.253    879         0x80000001 0x003471 0
192.168.0.0     172.16.2.253    50          0x80000002 0x000DEF 0

r2# show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 2 subnets
C       172.16.0.0 is directly connected, Serial1/1
C       172.16.2.0 is directly connected, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
O       10.2.0.0 [110/74] via 172.16.0.253, 00:34:25, Serial1/1
C       10.2.1.0 is directly connected, FastEthernet0/0
D       10.2.3.0 [90/2195456] via 172.16.2.254, 00:30:46, Serial1/0
D    192.168.0.0/24 [90/2195456] via 172.16.2.254, 00:20:37, Serial1/0
D    192.168.1.0/24 [90/2195456] via 172.16.2.254, 00:20:37, Serial1/0
D    192.168.2.0/24 [90/2195456] via 172.16.2.254, 00:20:38, Serial1/0
D    192.168.3.0/24 [90/2195456] via 172.16.2.254, 00:20:38, Serial1/0
O    192.168.0.0/22 is a summary, 00:18:19, Null0
```

r1 でトポロジーテーブルとルーティングテーブルを確認します。
```
r1#show ip ospf database

            OSPF Router with ID (172.16.1.254) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
172.16.1.254    172.16.1.254    1741        0x80000009 0x007454 3
172.16.2.253    172.16.2.253    993         0x8000000B 0x00982A 3

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.2.3.0        172.16.2.253    971         0x80000001 0x001441 0
172.16.2.0      172.16.2.253    971         0x80000001 0x003471 0
192.168.0.0     172.16.2.253    142         0x80000002 0x000DEF 0

r1#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/24 is subnetted, 3 subnets
C       172.16.0.0 is directly connected, Serial1/0
C       172.16.1.0 is directly connected, Serial1/1
O E1    172.16.2.0 [110/84] via 172.16.0.254, 00:16:15, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
C       10.2.0.0 is directly connected, FastEthernet0/0
O       10.2.1.0 [110/74] via 172.16.0.254, 02:42:10, Serial1/0
O E1    10.2.3.0 [110/84] via 172.16.0.254, 00:16:15, Serial1/0
O E1 192.168.0.0/22 [110/84] via 172.16.0.254, 00:02:26, Serial1/0
```

# まとめ

Dynagen、Dynamips を使って、OSPF を練習しました。
Type4 LSA、スタブエリア、NSSAエリアについては次回に回します。
DR選定、Type2 LSA については、折りをみて。
