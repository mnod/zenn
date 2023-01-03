---
title: "EIGRP の練習"
emoji: "🤴"
type: "tech"
topics: ["cisco", "ccna", "dynamips", "dynagen", "rip"]
published: true
---

#  本エントリについて

Dynagen、Dynamips を使って、EIGRP を練習します。
Dynagen、Dynamips の利用環境はすでに整っているものとします。

## 参考

https://www.infraexpert.com/study/study31.html

# 基本設定

https://zenn.dev/mnod/articles/c273d13817e8a6 の基本設定と同じ構成


# EIGRP の有効化

## r1 で有効化

r1 で EIGRP 有効化します。

- AS 番号は、同じ EIGRP ネットワークで同一である必要があります。
- fa0/0 からの EIGRP パケット送信を停止しています。
- EIGRP でも `クラスフルネットワークの境界で自動集約` することができますが、今回は自動集約機能をオフにしています。経路集約することでルーティングテーブルを小さくすることができ、デバイスの負荷が小さくなります。

```
r1(config)#router eigrp 1
r1(config-router)#passive-interface fa0/0
r1(config-router)#no auto-summary
r1(config-router)#network 10.2.0.0 0.0.0.255
r1(config-router)#network 172.16.0.0 0.0.0.255
r1(config-router)#network 172.16.1.0 0.0.0.255
```

状態を確認します。
```
r1#show ip protocols
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
    10.2.0.0/24
    172.16.0.0/24
    172.16.1.0/24
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
  Distance: internal 90 external 170

r1#show ip eigrp interfaces
IP-EIGRP interfaces for process 1

                        Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Se1/0              0        0/0         0       0/1            0           0
Se1/1              0        0/0         0       0/1            0           0

r1#show ip eigrp traffic
IP-EIGRP Traffic Statistics for AS 1
  Hellos sent/received: 68/0
  Updates sent/received: 0/0
  Queries sent/received: 0/0
  Replies sent/received: 0/0
  Acks sent/received: 0/0
  SIA-Queries sent/received: 0/0
  SIA-Replies sent/received: 0/0
  Hello Process ID: 245
  PDM Process ID: 231
  IP Socket queue:   0/2000/0/0 (current/max/highest/drops)
  Eigrp input queue: 0/2000/0/0 (current/max/highest/drops)
```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
他のルータで EIGRP を起動していないので、ネイバーテーブルは空です。ルーティングテーブルにも EIGRP で学習した経路はありません。
```
r1#show ip eigrp neighbors
IP-EIGRP neighbors for process 1

r1#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.1.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1

r1#show ip route eigrp

```


## r2 で有効化

r2 で EIGRP 有効化します。

```
r2(config)#router eigrp 1
r2(config-router)#passive-interface fa0/0
r2(config-router)#no auto-summary
r2(config-router)#network 10.2.1.0 0.0.0.255
r2(config-router)#network 172.16.2.0 0.0.0.255
r2(config-router)#network 172.16.0.0 0.0.0.255

*Mar  1 00:14:29.503: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 1: Neighbor 172.16.0.253 (Serial1/1) is up: new adjacency
```

状態を確認します。
```
r2#show ip protocols
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
    10.2.1.0/24
    172.16.0.0/24
    172.16.2.0/24
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.0.253          90      00:01:32
  Distance: internal 90 external 170


r2#show ip eigrp interfaces
IP-EIGRP interfaces for process 1

                        Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Se1/1              1        0/0        36       0/15         179           0
Se1/0              0        0/0         0       0/1            0           0
r2#
r2#show ip eigrp traffic
IP-EIGRP Traffic Statistics for AS 1
  Hellos sent/received: 45/23
  Updates sent/received: 5/3
  Queries sent/received: 0/0
  Replies sent/received: 0/0
  Acks sent/received: 0/2
  SIA-Queries sent/received: 0/0
  SIA-Replies sent/received: 0/0
  Hello Process ID: 245
  PDM Process ID: 231
  IP Socket queue:   0/2000/2/0 (current/max/highest/drops)
  Eigrp input queue: 0/2000/2/0 (current/max/highest/drops)
```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
```
r2#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   172.16.0.253            Se1/1             13 00:02:15   36   216  0  3

r2#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.2.253)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2195456
        via 172.16.0.253 (2195456/281600), Serial1/1
P 10.2.1.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.1.0/24, 1 successors, FD is 2681856
        via 172.16.0.253 (2681856/2169856), Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0

r2#show ip route eigrp
     172.16.0.0/24 is subnetted, 3 subnets
D       172.16.1.0 [90/2681856] via 172.16.0.253, 00:02:27, Serial1/1
     10.0.0.0/24 is subnetted, 2 subnets
D       10.2.0.0 [90/2195456] via 172.16.0.253, 00:02:27, Serial1/1
```

r1 でもネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
```
r1#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   172.16.0.254            Se1/0             12 00:02:57   51   306  0  5

r1#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.1.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 10.2.1.0/24, 1 successors, FD is 2195456
        via 172.16.0.254 (2195456/281600), Serial1/0
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2681856
        via 172.16.0.254 (2681856/2169856), Serial1/0

r1#show ip route eigrp
     172.16.0.0/24 is subnetted, 3 subnets
D       172.16.2.0 [90/2681856] via 172.16.0.254, 00:03:04, Serial1/0
     10.0.0.0/24 is subnetted, 2 subnets
D       10.2.1.0 [90/2195456] via 172.16.0.254, 00:03:09, Serial1/0
```

## r3 で有効化

r3 で EIGRP 有効化します。
```
r3(config)#router eigrp 1
r3(config-router)#passive-interface fa0/0
r3(config-router)#no auto-summary
r3(config-router)#network 10.2.2.0 0.0.0.255
r3(config-router)#network 172.16.3.0 0.0.0.255
r3(config-router)#network 172.16.1.0 0.0.0.255

*Mar  1 00:19:12.195: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 1: Neighbor 172.16.1.254 (Serial1/0) is up: new adjacency
```

状態を確認します。
```
r3#show ip protocols
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
    10.2.2.0/24
    172.16.1.0/24
    172.16.3.0/24
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.1.254          90      00:00:27
  Distance: internal 90 external 170


r3#show ip eigrp interfaces
IP-EIGRP interfaces for process 1

                        Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Se1/1              0        0/0         0       0/1            0           0
Se1/0              1        0/0        54       0/15         267           0

r3#show ip eigrp traffic
IP-EIGRP Traffic Statistics for AS 1
  Hellos sent/received: 20/9
  Updates sent/received: 3/5
  Queries sent/received: 0/0
  Replies sent/received: 0/0
  Acks sent/received: 1/0
  SIA-Queries sent/received: 0/0
  SIA-Replies sent/received: 0/0
  Hello Process ID: 245
  PDM Process ID: 231
  IP Socket queue:   0/2000/1/0 (current/max/highest/drops)
  Eigrp input queue: 0/2000/1/0 (current/max/highest/drops)

```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
```
r3#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   172.16.1.254            Se1/0             11 00:01:03   54   324  0  9

r3#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2195456
        via 172.16.1.254 (2195456/281600), Serial1/0
P 10.2.1.0/24, 1 successors, FD is 2707456
        via 172.16.1.254 (2707456/2195456), Serial1/0
P 10.2.2.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2681856
        via 172.16.1.254 (2681856/2169856), Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.2.0/24, 1 successors, FD is 3193856
        via 172.16.1.254 (3193856/2681856), Serial1/0
P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1

r3#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.0.0 [90/2681856] via 172.16.1.254, 00:01:18, Serial1/0
D       172.16.2.0 [90/3193856] via 172.16.1.254, 00:01:18, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
D       10.2.0.0 [90/2195456] via 172.16.1.254, 00:01:18, Serial1/0
D       10.2.1.0 [90/2707456] via 172.16.1.254, 00:01:18, Serial1/0
```

r1 で確認します。
```
r1#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
1   172.16.1.253            Se1/1             14 00:02:01   47   423  0  3
0   172.16.0.254            Se1/0             10 00:07:16   45   270  0  5

r1#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.1.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 10.2.1.0/24, 1 successors, FD is 2195456
        via 172.16.0.254 (2195456/281600), Serial1/0
P 10.2.2.0/24, 1 successors, FD is 2195456
        via 172.16.1.253 (2195456/281600), Serial1/1
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2681856
        via 172.16.0.254 (2681856/2169856), Serial1/0
P 172.16.3.0/24, 1 successors, FD is 2681856
        via 172.16.1.253 (2681856/2169856), Serial1/1

r1#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.2.0 [90/2681856] via 172.16.0.254, 00:07:21, Serial1/0
D       172.16.3.0 [90/2681856] via 172.16.1.253, 00:02:12, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
D       10.2.1.0 [90/2195456] via 172.16.0.254, 00:07:27, Serial1/0
D       10.2.2.0 [90/2195456] via 172.16.1.253, 00:02:12, Serial1/1
```

r2 で確認します。
```
r2#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   172.16.0.253            Se1/1             11 00:08:21   36   216  0  10

r2#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.2.253)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2195456
        via 172.16.0.253 (2195456/281600), Serial1/1
P 10.2.1.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 10.2.2.0/24, 1 successors, FD is 2707456
        via 172.16.0.253 (2707456/2195456), Serial1/1
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.1.0/24, 1 successors, FD is 2681856
        via 172.16.0.253 (2681856/2169856), Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.3.0/24, 1 successors, FD is 3193856
        via 172.16.0.253 (3193856/2681856), Serial1/1

r2#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.1.0 [90/2681856] via 172.16.0.253, 00:08:32, Serial1/1
D       172.16.3.0 [90/3193856] via 172.16.0.253, 00:03:17, Serial1/1
     10.0.0.0/24 is subnetted, 3 subnets
D       10.2.0.0 [90/2195456] via 172.16.0.253, 00:08:32, Serial1/1
D       10.2.2.0 [90/2707456] via 172.16.0.253, 00:03:17, Serial1/1
```

# 認証設定

## r1 で認証設定

r1 の s1/1 で認証設定します。
```
r1(config)#key chain ccna
r1(config-keychain)#key 10
r1(config-keychain-key)#key-string cisco
r1(config-keychain-key)#int s1/1
r1(config-if)#ip authentication mode eigrp 1 md5
r1(config-if)#ip authentication key-chain eigrp 1 ccna
```

r3 とのネイバー関係が解消され、r3 から学習した経路が無くなりました。
```
r1# show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   172.16.0.254            Se1/0             12 00:14:07   41   246  0  7

r1#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.1.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 10.2.1.0/24, 1 successors, FD is 2195456
        via 172.16.0.254 (2195456/281600), Serial1/0
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2681856
        via 172.16.0.254 (2681856/2169856), Serial1/0

r1#show ip route eigrp
     172.16.0.0/24 is subnetted, 3 subnets
D       172.16.2.0 [90/2681856] via 172.16.0.254, 00:14:12, Serial1/0
     10.0.0.0/24 is subnetted, 2 subnets
D       10.2.1.0 [90/2195456] via 172.16.0.254, 00:14:17, Serial1/0
```

r3 でも確認します。
r1 とのネイバー関係が解消され、EIGRP で学習した経路が無くなりました。
```
r3#show ip eigrp neighbors
IP-EIGRP neighbors for process 1

r3#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.2.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1

r3#show ip route eigrp

```

## r3 で認証設定

r3 の s1/0 で認証設定します。
```
r3(config)#key chain ccna
r3(config-keychain)#key 10
r3(config-keychain-key)#key-string cisco
r3(config-keychain-key)#int s1/0
r3(config-if)#ip authentication mode eigrp 1 md5
r3(config-if)#
r3(config-if)#ip authentication key-chain eigrp 1 ccna
```

状態を確認します。
r1 とのネイバー関係が復活し、EIGRP で学習した経路が見られるようになりました。
```
r3#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   172.16.1.254            Se1/0             13 00:00:33   35   210  0  15

r3#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2195456
        via 172.16.1.254 (2195456/281600), Serial1/0
P 10.2.1.0/24, 1 successors, FD is 2707456
        via 172.16.1.254 (2707456/2195456), Serial1/0
P 10.2.2.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2681856
        via 172.16.1.254 (2681856/2169856), Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.2.0/24, 1 successors, FD is 3193856
        via 172.16.1.254 (3193856/2681856), Serial1/0
P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1

r3#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.0.0 [90/2681856] via 172.16.1.254, 00:00:43, Serial1/0
D       172.16.2.0 [90/3193856] via 172.16.1.254, 00:00:43, Serial1/0
     10.0.0.0/24 is subnetted, 3 subnets
D       10.2.0.0 [90/2195456] via 172.16.1.254, 00:00:43, Serial1/0
D       10.2.1.0 [90/2707456] via 172.16.1.254, 00:00:43, Serial1/0
```


# ロードバランス

等しいコストのパスによる **等コストロードバランス** の他に、コストが異なるパスを利用した **不等コストロードバランス** にも対応しています。
まずは等コストロードバランスについて見ていきます。

## r4 で EIGRP 有効化

r4で EIGRP を有効化します。
```
r4(config)#router eigrp 1
r4(config-router)#passive-interface fa0/0
r4(config-router)#no auto-summary
r4(config-router)#network 10.2.3.0 0.0.0.255
r4(config-router)#network 172.16.2.0 0.0.0.255
r4(config-router)#network 172.16.3.0 0.0.0.255
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
    172.16.3.0/24
  Passive Interface(s):
    FastEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    172.16.3.254          90      00:00:36
    172.16.2.253          90      00:00:36
  Distance: internal 90 external 170


r4#show ip eigrp interfaces
IP-EIGRP interfaces for process 1

                        Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Se1/1              1        0/0        43       0/15         219           0
Se1/0              1        0/0        29       0/15         159           0

r4#show ip eigrp traffic
IP-EIGRP Traffic Statistics for AS 1
  Hellos sent/received: 26/23
  Updates sent/received: 8/10
  Queries sent/received: 1/0
  Replies sent/received: 0/1
  Acks sent/received: 4/4
  SIA-Queries sent/received: 0/0
  SIA-Replies sent/received: 0/0
  Hello Process ID: 245
  PDM Process ID: 231
  IP Socket queue:   0/2000/2/0 (current/max/highest/drops)
  Eigrp input queue: 0/2000/2/0 (current/max/highest/drops)

```

ネイバーテーブル、トポロジーテーブル、ルーティングテーブルを確認します。
トポロジーテーブル、ルーティングテーブルに 10.2.0.0/24 の経路が2つ現れています。
ルーティングテーブルに乗っている、コストが等しい2つの経路でロードバランスが行われます。(等コストロードバランス)
```
r4#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
1   172.16.3.254            Se1/0             12 00:01:16   29   200  0  17
0   172.16.2.253            Se1/1             12 00:01:22   43   258  0  17

r4#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.253)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 2 successors, FD is 2707456
        via 172.16.2.253 (2707456/2195456), Serial1/1
        via 172.16.3.254 (2707456/2195456), Serial1/0
P 10.2.1.0/24, 1 successors, FD is 2195456
        via 172.16.2.253 (2195456/281600), Serial1/1
P 10.2.2.0/24, 1 successors, FD is 2195456
        via 172.16.3.254 (2195456/281600), Serial1/0
P 10.2.3.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2681856
        via 172.16.2.253 (2681856/2169856), Serial1/1
P 172.16.1.0/24, 1 successors, FD is 2681856
        via 172.16.3.254 (2681856/2169856), Serial1/0
P 172.16.2.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0

r4#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.0.0 [90/2681856] via 172.16.2.253, 00:01:27, Serial1/1
D       172.16.1.0 [90/2681856] via 172.16.3.254, 00:01:27, Serial1/0
     10.0.0.0/24 is subnetted, 4 subnets
D       10.2.0.0 [90/2707456] via 172.16.3.254, 00:01:27, Serial1/0
                 [90/2707456] via 172.16.2.253, 00:01:27, Serial1/1
D       10.2.1.0 [90/2195456] via 172.16.2.253, 00:01:27, Serial1/1
D       10.2.2.0 [90/2195456] via 172.16.3.254, 00:01:27, Serial1/0
```

r2 でネイバーテーブルを確認してみます。r4 をネイバーとして認識しています。
```
r2#show ip eigrp neigh
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
1   172.16.2.254            Se1/0             11 00:04:40   48   288  0  8
0   172.16.0.253            Se1/1             12 00:27:18   28   200  0  25
```

r3 でもネイバーテーブルを確認してみます。こちらも r4 をネイバーとして認識しています。
```
r3#show ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
1   172.16.3.253            Se1/1             11 00:04:53   30   200  0  10
0   172.16.1.254            Se1/0             13 00:11:13   35   210  0  24
```

r1 でトポロジーテーブル、ルーティングテーブルを確認します。
トポロジーテーブル、ルーティングテーブルに 10.2.3.0/24 の経路が2つ現れています。
```
r1#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.1.254)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 10.2.1.0/24, 1 successors, FD is 2195456
        via 172.16.0.254 (2195456/281600), Serial1/0
P 10.2.2.0/24, 1 successors, FD is 2195456
        via 172.16.1.253 (2195456/281600), Serial1/1
P 10.2.3.0/24, 2 successors, FD is 2707456
        via 172.16.0.254 (2707456/2195456), Serial1/0
        via 172.16.1.253 (2707456/2195456), Serial1/1
P 172.16.0.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/1
P 172.16.2.0/24, 1 successors, FD is 2681856
        via 172.16.0.254 (2681856/2169856), Serial1/0
P 172.16.3.0/24, 1 successors, FD is 2681856
        via 172.16.1.253 (2681856/2169856), Serial1/1

r1#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.2.0 [90/2681856] via 172.16.0.254, 00:03:45, Serial1/0
D       172.16.3.0 [90/2681856] via 172.16.1.253, 00:03:45, Serial1/1
     10.0.0.0/24 is subnetted, 4 subnets
D       10.2.1.0 [90/2195456] via 172.16.0.254, 00:26:30, Serial1/0
D       10.2.2.0 [90/2195456] via 172.16.1.253, 00:06:20, Serial1/1
D       10.2.3.0 [90/2707456] via 172.16.1.253, 00:03:45, Serial1/1
                 [90/2707456] via 172.16.0.254, 00:03:45, Serial1/0
                 
```

## ロードバランスの確認

r4 から 10.2.0.1 へ向けて traceroute を実行してみると、2つの経路でロードバランスされているのが分かります。
```
r4#traceroute
Protocol [ip]:
Target IP address: 10.2.0.1
Source address: 10.2.3.254
Numeric display [n]:
Timeout in seconds [3]:
Probe count [3]:
Minimum Time to Live [1]:
Maximum Time to Live [30]:
Port Number [33434]:
Loose, Strict, Record, Timestamp, Verbose[none]:
Type escape sequence to abort.
Tracing the route to 10.2.0.1

  1 172.16.3.254 24 msec
    172.16.2.253 32 msec
    172.16.3.254 16 msec
  2 172.16.0.253 44 msec
    172.16.1.254 40 msec
    172.16.0.253 32 msec
  3 10.2.0.1 40 msec 60 msec 52 msec
```

10.2.0.1 から r4 の 10.2.3.254 へ ping しても、同様にロードバランスされています。
```
admin@ip-10-0-0-73:~/zenn$ ping -R 10.2.3.254 -c 2
PING 10.2.3.254 (10.2.3.254) 56(124) bytes of data.
64 bytes from 10.2.3.254: icmp_seq=1 ttl=253 time=68.3 ms
NOP
RR:     10.2.0.1
        172.16.0.253
        172.16.2.253
        10.2.3.254
        172.16.2.254
        172.16.0.254
        10.2.0.254
        10.2.0.1

64 bytes from 10.2.3.254: icmp_seq=2 ttl=253 time=45.5 ms
NOP
RR:     10.2.0.1
        172.16.1.254
        172.16.3.254
        10.2.3.254
        172.16.3.253
        172.16.1.253
        10.2.0.254
        10.2.0.1


--- 10.2.3.254 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 45.531/56.932/68.333/11.401 ms
```

# メトリック

EIGRP のメトリックは下記の要素を利用した複合メトリックです。

- **帯域幅** 10^7/経路上の最小の帯域幅(kbps)。
- 負荷
- **遅延** 経路上の遅延(usec)の合計/10。
- 信頼性
- MTU

これらに、K値と呼ばれるウェイトをかけ合わせることでメトリックを求めます。
計算式は以下のようになります。
(K値は変更することができます。K値が異なるとネイバーになれないので注意が必要です。)

- K == 0 のとき
$[ K_1 \cdot \text{帯域幅} + K_2 \cdot \text{帯域幅} / ( 256 - \text{負荷} ) + K_3 \cdot \text{遅延} ] \cdot 256$

- K5 <> 0 のとき
$[ K_1 \cdot \text{帯域幅} + K_2 \cdot \text{帯域幅} / ( 256 - \text{負荷} ) + K_3 \cdot \text{遅延} ] \cdot [ K_5 / (K_4 + \text{信頼性}) ] \cdot 256$

:::message
K値のデフォルトは、`K1, K3 == 1`、`K2, K4, K5 == 0` ですので、 **デフォルトのメトリック** は下記のようになります。
$(\text{帯域幅} + \text{遅延}) \cdot 256$
:::

## FD、AD

r4 のトポロジーテーブルの 10.2.0.0/24 の経路に着目します。
```
r4#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.253)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 2 successors, FD is 2707456
        via 172.16.2.253 (2707456/2195456), Serial1/1
        via 172.16.3.254 (2707456/2195456), Serial1/0
以下略
```

```
r4#show ip eigrp topology 10.2.0.0/24
IP-EIGRP (AS 1): Topology entry for 10.2.0.0/24
  State is Passive, Query origin flag is 1, 2 Successor(s), FD is 2707456
  Routing Descriptor Blocks:
  172.16.2.253 (Serial1/1), from 172.16.2.253, Send flag is 0x0
      Composite metric is (2707456/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1544 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
  172.16.3.254 (Serial1/0), from 172.16.3.254, Send flag is 0x0
      Composite metric is (2707456/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1544 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
```


r4 の s1/1 経由した 10.2.0.0/24 への経路上の各インタフェースの状態は下記のようになっています。

```
r4#show int s1/1
Serial1/1 is up, line protocol is up
  Hardware is M4T
  Internet address is 172.16.2.254/24
  MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation HDLC, crc 16, loopback not set
  Keepalive set (10 sec)
  Restart-Delay is 0 secs
  CRC checking enabled
  Last input 00:00:01, output 00:00:00, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: weighted fair
  Output queue: 0/1000/64/0 (size/max total/threshold/drops)
     Conversations  0/1/256 (active/max active/max total)
     Reserved Conversations 0/0 (allocated/max allocated)
     Available Bandwidth 1158 kilobits/sec
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     1524 packets input, 100235 bytes, 0 no buffer
     Received 580 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
     1263 packets output, 83630 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 output buffer failures, 0 output buffers swapped out
     2 carrier transitions     DCD=up  DSR=up  DTR=up  RTS=up  CTS=up
```

```
r2#show int s1/1
Serial1/1 is up, line protocol is up
  Hardware is M4T
  Internet address is 172.16.0.254/24
  MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation HDLC, crc 16, loopback not set
  Keepalive set (10 sec)
  Restart-Delay is 0 secs
  CRC checking enabled
  Last input 00:00:01, output 00:00:03, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: weighted fair
  Output queue: 0/1000/64/0 (size/max total/threshold/drops)
     Conversations  0/1/256 (active/max active/max total)
     Reserved Conversations 0/0 (allocated/max allocated)
     Available Bandwidth 1158 kilobits/sec
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     1670 packets input, 109189 bytes, 0 no buffer
     Received 602 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
     1567 packets output, 102541 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 output buffer failures, 0 output buffers swapped out
     2 carrier transitions     DCD=up  DSR=up  DTR=up  RTS=up  CTS=up
```

```
r1#show int f0/0
FastEthernet0/0 is up, line protocol is up
  Hardware is Gt96k FE, address is c200.034f.0000 (bia c200.034f.0000)
  Internet address is 10.2.0.254/24
  MTU 1500 bytes, BW 10000 Kbit/sec, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Half-duplex, 10Mb/s, 100BaseTX/FX
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input 00:41:03, output 00:00:01, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue: 0/40 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     7 packets input, 570 bytes
     Received 1 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 watchdog
     0 input packets with dribble condition detected
     630 packets output, 64127 bytes, 0 underruns
     0 output errors, 0 collisions, 0 interface resets
     0 unknown protocol drops
     0 babbles, 0 late collision, 0 deferred
     0 lost carrier, 0 no carrier
     0 output buffer failures, 0 output buffers swapped out
```

各インタフェースの帯域幅と遅延をまとめます。
| インタフェース | 帯域幅(Kbit/sec) | 遅延(usec) |
| --- | --- | --- |
| r4 s1/1 | 1544 | 20000 |
| r2 s1/1 | 1544 | 20000 |
| r1 f0/0 | 10000 | 1000 |

- AD(Advertised Distance。ネイバールータから宛先ネットワークまでのメトリック)
経路上の最小帯域幅は `1544 Kbit/sec` なので、帯域幅 1000000 / 1544 = 6476.68394
遅延の合計値 `DLY 20000 usec`  `DLY 1000 usec` なので 21000 / 10 = 2100
(6476+2100) * 256 = 2195456 が AD となります。

- FD(Feasible Distance。ローカルルータから宛先ネットワークまでのメトリック)
経路上の最小帯域幅は `1544 Kbit/sec` なので、帯域幅 1000000 / 1544 = 6476.68394
遅延の合計値 `DLY 20000 usec`  `DLY 20000 usec`  `DLY 1000 usec` なので 41000 / 10 = 4100
(6476+4100) * 256 = 2707456 が FD となります。

- サクセサ(プライマリールート)
最小のFD値をもつルートがサクセサになります。
ルーティングテーブルに格納されます。

- フィージブルサクセサ(バックアップルート)
サクセサの FD より小さいAD を持つルートがフィージブルサクセサになります。
サクセサのダウン時に、経路の再計算不要で、即サクセサとなります。


## 帯域幅の変更

メトリクス値の変更を目的として、インタフェースの帯域幅の値を、任意の値に設定することができます。
(インタフェースの実際の転送速度は影響を受けません)
```
r4(config)#int s1/1
r4(config-if)#bandwidth 1500

r4#show int s1/1
Serial1/1 is up, line protocol is up
  Hardware is M4T
  Internet address is 172.16.2.254/24
  MTU 1500 bytes, BW 1500 Kbit/sec, DLY 20000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation HDLC, crc 16, loopback not set
  Keepalive set (10 sec)
  Restart-Delay is 0 secs
  CRC checking enabled
  Last input 00:00:03, output 00:00:01, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: weighted fair
  Output queue: 0/1000/64/0 (size/max total/threshold/drops)
     Conversations  0/1/256 (active/max active/max total)
     Reserved Conversations 0/0 (allocated/max allocated)
     Available Bandwidth 1125 kilobits/sec
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     2008 packets input, 131583 bytes, 0 no buffer
     Received 747 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
     1746 packets output, 115138 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 output buffer failures, 0 output buffers swapped out
     2 carrier transitions     DCD=up  DSR=up  DTR=up  RTS=up  CTS=up

```

トポロジーテーブル、ルーティングテーブルを確認します。
```
r4#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.253)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2707456
        via 172.16.3.254 (2707456/2195456), Serial1/0
        via 172.16.2.253 (2756096/2195456), Serial1/1
P 10.2.1.0/24, 1 successors, FD is 2244096
        via 172.16.2.253 (2244096/281600), Serial1/1
P 10.2.2.0/24, 1 successors, FD is 2195456
        via 172.16.3.254 (2195456/281600), Serial1/0
P 10.2.3.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2730496
        via 172.16.2.253 (2730496/2169856), Serial1/1
        via 172.16.3.254 (3193856/2681856), Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2681856
        via 172.16.3.254 (2681856/2169856), Serial1/0
P 172.16.2.0/24, 1 successors, FD is 2218496
        via Connected, Serial1/1
P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0

r4#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.0.0 [90/2730496] via 172.16.2.253, 00:00:56, Serial1/1
D       172.16.1.0 [90/2681856] via 172.16.3.254, 00:00:56, Serial1/0
     10.0.0.0/24 is subnetted, 4 subnets
D       10.2.0.0 [90/2707456] via 172.16.3.254, 00:00:56, Serial1/0
D       10.2.1.0 [90/2244096] via 172.16.2.253, 00:00:56, Serial1/1
D       10.2.2.0 [90/2195456] via 172.16.3.254, 00:58:46, Serial1/0

r4#show ip eigrp topology 10.2.0.0/24
IP-EIGRP (AS 1): Topology entry for 10.2.0.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 2707456
  Routing Descriptor Blocks:
  172.16.3.254 (Serial1/0), from 172.16.3.254, Send flag is 0x0
      Composite metric is (2707456/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1544 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
  172.16.2.253 (Serial1/1), from 172.16.2.253, Send flag is 0x0
      Composite metric is (2756096/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1500 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2

```

## 遅延の変更

同様に、インタフェースの遅延の値を変更することができます。
```
r4(config)#int s1/1
r4(config-if)#delay 2500

r4#show int s1/1
Serial1/1 is up, line protocol is up
  Hardware is M4T
  Internet address is 172.16.2.254/24
  MTU 1500 bytes, BW 1500 Kbit/sec, DLY 25000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation HDLC, crc 16, loopback not set
  Keepalive set (10 sec)
  Restart-Delay is 0 secs
  CRC checking enabled
  Last input 00:00:02, output 00:00:01, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: weighted fair
  Output queue: 0/1000/64/0 (size/max total/threshold/drops)
     Conversations  0/1/256 (active/max active/max total)
     Reserved Conversations 0/0 (allocated/max allocated)
     Available Bandwidth 1125 kilobits/sec
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     2061 packets input, 134853 bytes, 0 no buffer
     Received 765 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
     1799 packets output, 118408 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 output buffer failures, 0 output buffers swapped out
     2 carrier transitions     DCD=up  DSR=up  DTR=up  RTS=up  CTS=up

```

トポロジーテーブルとルーティングテーブルを確認します。
```
r4#show ip eigrp topology
IP-EIGRP Topology Table for AS(1)/ID(172.16.3.253)

Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.2.0.0/24, 1 successors, FD is 2707456
        via 172.16.3.254 (2707456/2195456), Serial1/0
        via 172.16.2.253 (2884096/2195456), Serial1/1
P 10.2.1.0/24, 1 successors, FD is 2244096
        via 172.16.2.253 (2372096/281600), Serial1/1
P 10.2.2.0/24, 1 successors, FD is 2195456
        via 172.16.3.254 (2195456/281600), Serial1/0
P 10.2.3.0/24, 1 successors, FD is 281600
        via Connected, FastEthernet0/0
P 172.16.0.0/24, 1 successors, FD is 2730496
        via 172.16.2.253 (2858496/2169856), Serial1/1
        via 172.16.3.254 (3193856/2681856), Serial1/0
P 172.16.1.0/24, 1 successors, FD is 2681856
        via 172.16.3.254 (2681856/2169856), Serial1/0
P 172.16.2.0/24, 1 successors, FD is 2346496
        via Connected, Serial1/1
P 172.16.3.0/24, 1 successors, FD is 2169856
        via Connected, Serial1/0

r4#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.0.0 [90/2858496] via 172.16.2.253, 00:00:41, Serial1/1
D       172.16.1.0 [90/2681856] via 172.16.3.254, 00:00:41, Serial1/0
     10.0.0.0/24 is subnetted, 4 subnets
D       10.2.0.0 [90/2707456] via 172.16.3.254, 00:00:41, Serial1/0
D       10.2.1.0 [90/2372096] via 172.16.2.253, 00:00:41, Serial1/1
D       10.2.2.0 [90/2195456] via 172.16.3.254, 01:01:12, Serial1/0

r4#show ip eigrp topology 10.2.0.0/24
IP-EIGRP (AS 1): Topology entry for 10.2.0.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 2707456
  Routing Descriptor Blocks:
  172.16.3.254 (Serial1/0), from 172.16.3.254, Send flag is 0x0
      Composite metric is (2707456/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1544 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
  172.16.2.253 (Serial1/1), from 172.16.2.253, Send flag is 0x0
      Composite metric is (2884096/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1500 Kbit
        Total delay is 46000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2

```


通常は、最もよいメトリクス値がルーティングテーブルに乗り、ルーティングテーブルにしたがってルーティングされます。
ここでは経路は1つの経路しかありませんので、ロードバランスはおきません。
```
r4#traceroute
Protocol [ip]:
Target IP address: 10.2.0.1
Source address: 10.2.3.254
Numeric display [n]:
Timeout in seconds [3]:
Probe count [3]:
Minimum Time to Live [1]:
Maximum Time to Live [30]:
Port Number [33434]:
Loose, Strict, Record, Timestamp, Verbose[none]:
Type escape sequence to abort.
Tracing the route to 10.2.0.1

  1 172.16.3.254 12 msec 4 msec 20 msec
  2 172.16.1.254 28 msec 44 msec 24 msec
  3 10.2.0.1 68 msec 48 msec 40 msec
r4#
```


r1 のルーティングテーブル、トポロジーテーブルを確認します。
こちら側では 10.2.3.0/24 への最適経路が2つルーティングテーブルに乗っていますので、ロードバランスされます。
```
r1#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.2.0 [90/2681856] via 172.16.0.254, 00:04:27, Serial1/0
D       172.16.3.0 [90/2681856] via 172.16.1.253, 01:04:49, Serial1/1
     10.0.0.0/24 is subnetted, 4 subnets
D       10.2.1.0 [90/2195456] via 172.16.0.254, 01:40:19, Serial1/0
D       10.2.2.0 [90/2195456] via 172.16.1.253, 01:20:09, Serial1/1
D       10.2.3.0 [90/2707456] via 172.16.1.253, 01:04:48, Serial1/1
                 [90/2707456] via 172.16.0.254, 01:04:48, Serial1/0

r1#show ip eigrp topology 10.2.3.0/24
IP-EIGRP (AS 1): Topology entry for 10.2.3.0/24
  State is Passive, Query origin flag is 1, 2 Successor(s), FD is 2707456
  Routing Descriptor Blocks:
  172.16.0.254 (Serial1/0), from 172.16.0.254, Send flag is 0x0
      Composite metric is (2707456/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1544 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
  172.16.1.253 (Serial1/1), from 172.16.1.253, Send flag is 0x0
      Composite metric is (2707456/2195456), Route is Internal
      Vector metric:
        Minimum bandwidth is 1544 Kbit
        Total delay is 41000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
```

ping を実施してみます。
```
admin@ip-10-0-0-73:~/zenn$ ping -R 10.2.3.254 -c 2
PING 10.2.3.254 (10.2.3.254) 56(124) bytes of data.
64 bytes from 10.2.3.254: icmp_seq=1 ttl=253 time=47.6 ms
NOP
RR:     10.2.0.1
        172.16.0.253
        172.16.2.253
        10.2.3.254
        172.16.3.253
        172.16.1.253
        10.2.0.254
        10.2.0.1

64 bytes from 10.2.3.254: icmp_seq=2 ttl=253 time=44.6 ms
NOP
RR:     10.2.0.1
        172.16.1.254
        172.16.3.254
        10.2.3.254
        172.16.3.253
        172.16.1.253
        10.2.0.254
        10.2.0.1


--- 10.2.3.254 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 44.605/46.126/47.648/1.536 ms
```

## 不等コストロードバランス

不等コストロードバランスを有効にするには、varaisnce で定数(n)を指定します。
トポロジーテーブルに掲載されている経路のうち、フィージブルサクセサのFD値のn倍した値が、サクセサのAD値より大きい場合、ルーティングテーブルに登録され、ロードバランスに使用されるようになります。

```
r4(config)#router eigrp 1
r4(config-router)#variance 2
```

```
r4#show ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.0.0 [90/3193856] via 172.16.3.254, 00:00:28, Serial1/0
                   [90/2858496] via 172.16.2.253, 00:00:28, Serial1/1
D       172.16.1.0 [90/2681856] via 172.16.3.254, 00:00:28, Serial1/0
     10.0.0.0/24 is subnetted, 4 subnets
D       10.2.0.0 [90/2707456] via 172.16.3.254, 00:00:28, Serial1/0
                 [90/2884096] via 172.16.2.253, 00:00:28, Serial1/1
D       10.2.1.0 [90/2372096] via 172.16.2.253, 00:00:28, Serial1/1
D       10.2.2.0 [90/2195456] via 172.16.3.254, 00:00:28, Serial1/0
```

traceroute で確認します。
```
r4#traceroute
Protocol [ip]:
Target IP address: 10.2.0.1
Source address: 10.2.3.254
Numeric display [n]:
Timeout in seconds [3]:
Probe count [3]:
Minimum Time to Live [1]:
Maximum Time to Live [30]:
Port Number [33434]:
Loose, Strict, Record, Timestamp, Verbose[none]:
Type escape sequence to abort.
Tracing the route to 10.2.0.1

  1 172.16.3.254 16 msec
    172.16.2.253 24 msec
    172.16.3.254 32 msec
  2 172.16.0.253 44 msec
    172.16.1.254 24 msec
    172.16.0.253 40 msec
  3 10.2.0.1 52 msec 40 msec 32 msec
```



# まとめ

Dynagen、Dynamips を使って、EIGRP を練習しました。
EIGRP は FD、AD、サクセサ、フィージブルサクセサがややこしいです。電卓を叩きながら練習しました。
今回経路集約の練習はしませんでしたが、進め方によってはこのトポロジーでも練習ができたと思います。今後経路集約の項目を追加するかもしれません。
