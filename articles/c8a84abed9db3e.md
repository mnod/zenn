---
title: "netflow の練習"
emoji: "🗣️"
type: "tech"
topics: ["cisco", "ccna", "dynamips", "dynagen", "netflow"]
published: true
---


# netflow 

ネットワークを流れるトラフィックの統計情報などを取得する機能です。
統計情報はフローコレクタと呼ばれるサーバへ転送することができます。

https://e-words.jp/w/NetFlow.html

# td-agent

td-agent(fluentd) をフローコレクタとして利用するために、td-agent と netflowプラグインをインストールします。

```
$ curl -fsSL https://toolbelt.treasuredata.com/sh/install-debian-buster-td-agent4.sh | sudo sh 
$ sudo td-agent-gem install fluent-plugin-netflow
```

以下のように設定を追加します。
```
$ sudo cp -pi /etc/td-agent/td-agent.conf{,.000}
$ sudo vi /etc/td-agent/td-agent.conf
$ diff -u /etc/td-agent/td-agent.conf{,.000}
@@ -126,15 +126,3 @@
 #    path /var/log/td-agent/td-%Y-%m-%d/%H.log
 #  </store>
 #</match>
-
-<match netflow.*>
-  @type stdout
-</match>
-
-<source>
-  @type netflow
-  tag netflow.event
-  port 5141
-  versions [5, 9]
-</source>
```

設定を追加したら td-agent を再起動します。
```
$ sudo td-agent --dry-run
$ sudo systemctl restart td-agent.service
```

td-agent(fluentd) で収集したデータを elasticsearch に喰わせて kibana で可視化することができますが、ここではデータ収集のみにとどめます。

# Netflow version 5

Netflow にはいくつかのバージョンがあります。
バージョン5は標準的なもので、所定のキーに基づいてパケットを識別して、所定の統計情報を収集します。

インタフェースに適用します。
必要に応じて、受信方向の統計を取る場合は ingress、送信方向なら egress を指定します。
```
r1(config)#int s1/0
r1(config-if)#ip flow ingress
r1(config-if)#ip flow egress
```

フローエクスポータを指定します。
```
r1(config)#ip flow-export version 5
r1(config)#ip flow-export destination 10.2.0.1 5141 udp
r1(config)#ip flow-export source fa0/0
```

設定内容は、show ip flow interface、show ip flow export で確認することができます。

フローのキャッシュの内容は下記で確認することができます。
```
r1#show ip cache flow
IP packet size distribution (0 total packets):
   1-32   64   96  128  160  192  224  256  288  320  352  384  416  448  480
   .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000

    512  544  576 1024 1536 2048 2560 3072 3584 4096 4608
   .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000

IP Flow Switching Cache, 278544 bytes
  0 active, 4096 inactive, 0 added
  0 ager polls, 0 flow alloc failures
  Active flows timeout in 30 minutes
  Inactive flows timeout in 15 seconds
IP Sub Flow Cache, 25800 bytes
  0 active, 1024 inactive, 0 added, 0 added to flow
  0 alloc failures, 0 force free
  1 chunk, 1 chunk added
  last clearing of statistics never
Protocol         Total    Flows   Packets Bytes  Packets Active(Sec) Idle(Sec)
--------         Flows     /Sec     /Flow  /Pkt     /Sec     /Flow     /Flow

SrcIf         SrcIPaddress    DstIf         DstIPaddress    Pr SrcP DstP  Pkts
```

ping を打って、キャッシュの内容を確認します。
```
r1#show ip cache flow
IP packet size distribution (24 total packets):
   1-32   64   96  128  160  192  224  256  288  320  352  384  416  448  480
   .000 .000 .000 1.00 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000

    512  544  576 1024 1536 2048 2560 3072 3584 4096 4608
   .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000

IP Flow Switching Cache, 278544 bytes
  5 active, 4091 inactive, 5 added
  43 ager polls, 0 flow alloc failures
  Active flows timeout in 30 minutes
  Inactive flows timeout in 15 seconds
IP Sub Flow Cache, 25800 bytes
  5 active, 1019 inactive, 5 added, 5 added to flow
  0 alloc failures, 0 force free
  1 chunk, 1 chunk added
  last clearing of statistics never
Protocol         Total    Flows   Packets Bytes  Packets Active(Sec) Idle(Sec)
--------         Flows     /Sec     /Flow  /Pkt     /Sec     /Flow     /Flow

SrcIf         SrcIPaddress    DstIf         DstIPaddress    Pr SrcP DstP  Pkts
Se1/0         172.16.0.254    Se1/0         172.16.0.254    01 0000 0000     5
Se1/0         172.16.0.254    Se1/0*        172.16.0.254    01 0000 0000     5

SrcIf         SrcIPaddress    DstIf         DstIPaddress    Pr SrcP DstP  Pkts
Se1/0         172.16.0.254    Local         172.16.0.253    01 0000 0800     5
```

```
r1#show ip cache flow
IP packet size distribution (29 total packets):
   1-32   64   96  128  160  192  224  256  288  320  352  384  416  448  480
   .000 .000 .000 1.00 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000

    512  544  576 1024 1536 2048 2560 3072 3584 4096 4608
   .000 .000 .000 .000 .000 .000 .000 .000 .000 .000 .000

IP Flow Switching Cache, 278544 bytes
  1 active, 4095 inactive, 6 added
  87 ager polls, 0 flow alloc failures
  Active flows timeout in 30 minutes
  Inactive flows timeout in 15 seconds
IP Sub Flow Cache, 25800 bytes
  1 active, 1023 inactive, 6 added, 6 added to flow
  0 alloc failures, 0 force free
  1 chunk, 1 chunk added
  last clearing of statistics never
Protocol         Total    Flows   Packets Bytes  Packets Active(Sec) Idle(Sec)
--------         Flows     /Sec     /Flow  /Pkt     /Sec     /Flow     /Flow
ICMP                 5      0.0         4   100      0.0       0.1      15.1
Total:               5      0.0         4   100      0.0       0.1      15.1

SrcIf         SrcIPaddress    DstIf         DstIPaddress    Pr SrcP DstP  Pkts

SrcIf         SrcIPaddress    DstIf         DstIPaddress    Pr SrcP DstP  Pkts
Se1/0         172.16.0.254    Local         172.16.0.253    01 0000 0800     5
```

fluentd に送信されたログを確認します。
```
admin@ip-10-0-0-73:~/zenn$ sudo tail -f /var/log/td-agent/td-agent.log

2002-03-01 09:18:31.000000000 +0900 netflow.event: {"version":5,"uptime":1111508,"flow_records":1,"flow_seq_num":6,"engine_type":0,"engine_id":0,"sampling_algorithm":0,"sampling_interval":0,"ipv4_src_addr":"172.16.0.254","ipv4_dst_addr":"172.16.0.253","ipv4_next_hop":"0.0.0.0","input_snmp":3,"output_snmp":0,"in_pkts":5,"in_bytes":500,"first_switched":"2002-03-01T00:18:04.967Z","last_switched":"2002-03-01T00:18:05.059Z","l4_src_port":0,"l4_dst_port":2048,"tcp_flags":16,"protocol":1,"src_tos":0,"src_as":0,"dst_as":0,"src_mask":24,"dst_mask":24,"host":"10.2.0.254"}
```


# Flexible Netflow

Flexible Netflow では、フローを識別するキーと、統計情報を収集するフィールドを柔軟に指定することができます。
これは **フローレコード** で設定します。
```
r1(config)#flow record FLOWRECORD
r1(config-flow-record)#match ipv4 source address
r1(config-flow-record)#match ipv4 destination address
r1(config-flow-record)#match transport source-port
r1(config-flow-record)#match transport destination-port
r1(config-flow-record)#collect counter bytes
r1(config-flow-record)#collect counter packets
```

Flexible Netflow では、出力フォーマットを指定することができます。
これは **フローエクスポータ** で設定します。(ここではデフォルトのままとしています。)
```
r1(config)#flow exporter FLOWEXPORTER
r1(config-flow-exporter)#destination 10.2.0.1
r1(config-flow-exporter)#source fa0/0
r1(config-flow-exporter)#transport udp 5141
```


**フローモニター** で、設定したフローレコードとフローエクスポータを指定します。
```
r1(config)#flow monitor FLOWMONITOR
r1(config-flow-monitor)#record FLOWRECORD
r1(config-flow-monitor)#exporter FLOWEXPORTER
```

フローモニターをインタフェースに、適用します。
```
r1(config)#int s1/0
r1(config-if)#ip flow monitor FLOWMONITOR input
```

設定内容を確認します。
```
r1#show flow record
flow record FLOWRECORD:
  Description:        User defined
  No. of users:       1
  Total field space:  20 bytes
  Fields:
    match ipv4 source address
    match ipv4 destination address
    match transport source-port
    match transport destination-port
    collect counter packets
    collect counter bytes


r1#show flow monitor
Flow Monitor FLOWMONITOR:
  Description:       User defined
  Flow Record:       FLOWRECORD
  Flow Exporter:     FLOWEXPORTER
  Cache:
    Type:              normal
    Status:            allocated
    Size:              4096 entries / 196620 bytes
    Inactive Timeout:  15 secs
    Active Timeout:    1800 secs
    Update Timeout:    1800 secs



r1#show flow exporter
Flow Exporter FLOWEXPORTER:
  Description:              User defined
  Tranport Configuration:
    Destination IP address: 10.2.0.1
    Source IP address:      10.2.0.254
    Source Interface:       FastEthernet0/0
    Transport Protocol:     UDP
    Destination Port:       5141
    Source Port:            56556
    DSCP:                   0x0
    TTL:                    255


r1#show flow interface
Interface Serial1/0
  FNF:  monitor:         FLOWMONITOR
        direction:       Input
        traffic(ip):     on

```


ping を実行して、fluentd に出力されたログを確認します。
Flexible Netflow ではバージョンは9になります。
```
admin@ip-10-0-0-73:~/zenn$ sudo tail -f /var/log/td-agent/td-agent.log

2002-03-01 09:41:50.000000000 +0900 netflow.event: {"version":"9","flow_seq_num":"0","flowset_id":"256","ipv4_src_addr":"172.16.0.254","ipv4_dst_addr":"172.16.0.253","l4_src_port":0,"l4_dst_port":2048,"in_pkts":5,"in_bytes":500,"host":"10.2.0.254"}
2002-03-01 09:42:12.000000000 +0900 netflow.event: {"version":"9","flow_seq_num":"1","flowset_id":"256","ipv4_src_addr":"172.16.0.254","ipv4_dst_addr":"172.16.0.253","l4_src_port":0,"l4_dst_port":2048,"in_pkts":5,"in_bytes":500,"host":"10.2.0.254"}
```

大量のパケットで統計情報を取ると、CPU負荷が高まります。
**フローサンプラー** を指定することで、統計対象となるパケットを間引きすることができます。
```
r1(config)#sample FLOWSAMPLER
r1(config-sampler)#mode random 1 out-of 2
```

フローモニターをインタフェースに適用するときに、同時に、フローサンプラーも指定します。
```
r1(config)#int s1/0
r1(config-if)#no ip flow monitor FLOWMONITOR input
r1(config-if)#ip flow monitor FLOWMONITOR sampler FLOWSAMPLER input
```

設定内容を確認します。
```
r1#show flow interface
Interface Serial1/0
  FNF:  monitor:         FLOWMONITOR
        direction:       Input
        traffic(ip):     sampler FLOWSAMPLER

r1#show flow monitor
Flow Monitor FLOWMONITOR:
  Description:       User defined
  Flow Record:       FLOWRECORD
  Flow Exporter:     FLOWEXPORTER
  Cache:
    Type:              normal
    Status:            allocated
    Size:              4096 entries / 196620 bytes
    Inactive Timeout:  15 secs
    Active Timeout:    1800 secs
    Update Timeout:    1800 secs

```

