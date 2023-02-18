---
title: "Debian11 で Libvirt(KVM)"
emoji: "🧛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["debian","kvm","libvirt"]
published: true
---


# 環境について

Debian11の環境で、Libvirt を通じて、KVM の操作をしてみます。

# インストール

関連するパッケージをインストールします。
(NAT を使わない場合、dnsmasq は不要かもしれません。)

```
# apt install --no-install-recommends libvirt-clients libvirt-daemon-system virtinst dnsmasq
```


# 確認

バージョンを確認します。
```
# virsh version
Compiled against library: libvirt 7.0.0
Using library: libvirt 7.0.0
Using API: QEMU 7.0.0
Running hypervisor: QEMU 5.2.0
```

デフォルトの接続先を確認します。
下記の時は、以後 `--connect qemu:///system` オプションを明示的に指定する必要はありません。

```
# virsh uri
qemu:///system
```

# ストレージプール

ストレージプールを確認します。
初期状態ではストレージプールはありません。
```
# virsh pool-list
 Name   State   Autostart
---------------------------

```

Debian で Libvirt を利用する場合、 `/var/lib/libvirt/images` ディレクトリが自動で作成されるので、
こちらをデフォルトのストレージプールとして設定してみます。
ストレージプールを作成後、利用するためには、スタートさせる必要があります。
```
# virsh pool-define-as default dir --target /var/lib/libvirt/images/
Pool default defined


# virsh pool-start default
Pool default started


# virsh pool-list
 Name      State    Autostart
-------------------------------
 default   active   no

```

次回、自動でスタートさせるためには、自動スタートの設定が必要です。
```
# virsh pool-autostart default
Pool default marked as autostarted


# virsh pool-list
 Name      State    Autostart
-------------------------------
 default   active   yes

```

xml の内容を確認します。
```
# virsh pool-dumpxml default
<pool type='dir'>
  <name>default</name>
  <uuid>9d4a9d2c-ed01-425c-9b8b-ec1c6b744a6b</uuid>
  <capacity unit='bytes'>15791431680</capacity>
  <allocation unit='bytes'>4877070336</allocation>
  <available unit='bytes'>10914361344</available>
  <source>
  </source>
  <target>
    <path>/var/lib/libvirt/images</path>
    <permissions>
      <mode>0711</mode>
      <owner>0</owner>
      <group>0</group>
    </permissions>
  </target>
</pool>
```


# 仮想ネットワークの管理

仮想ネットワークを確認します。
初期状態では default がありますが、起動していません。

```
# virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes

# virsh net-dumpxml default
<network>
  <name>default</name>
  <uuid>b00d5f3b-e209-4ff3-aaeb-08a9f6314255</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:c6:88:fe'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>

```

https://metonymical.hatenablog.com/entry/2018/12/25/210811

Open vSwitch の仮想ブリッジ `ovsbr0` を使用するために以下の内容の xml を作成します。

```:ovs.xml
<network>
  <name>ovs</name>
  <forward mode='bridge'/>
  <bridge name='ovsbr0'/>
  <virtualport type='openvswitch'/>
  <portgroup name='untag' default='yes'/>
  <portgroup name='vlan10'>
    <vlan>
      <tag id='10'/>
    </vlan>
  </portgroup>
</network>
```

作成した xml を利用して、仮想ネットワークを定義します。
```
# virsh net-define ovs.xml
Network ovs defined from ovs.xml

# virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes
 ovs       inactive   no          yes

# virsh net-dumpxml ovs
<network>
  <name>ovs</name>
  <uuid>95a00202-557e-4780-82e8-3f489ddd1da4</uuid>
  <forward mode='bridge'/>
  <bridge name='ovsbr0'/>
  <virtualport type='openvswitch'/>
  <portgroup name='untag' default='yes'>
  </portgroup>
  <portgroup name='vlan10'>
    <vlan>
      <tag id='10'/>
    </vlan>
  </portgroup>
</network>
```

作成した仮想ネットワークを利用するためには、起動する必要があります。
```
# virsh net-start ovs
Network ovs started

# virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes
 ovs       active     no          yes

```

次回以降、自動で起動させるためには、自動起動設定をします。
```
# virsh net-autostart ovs
Network ovs marked as autostarted

# virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes
 ovs       active     yes         yes

```

# 仮想マシンの起動

インストールメディアを用意します。
```
# mkdir /var/lib/libvirt/images/isos
# ln -s /home/user/Rocky-9.1-x86_64-minimal.iso /var/lib/libvirt/images/isos/Rocky-9.1-x86_64-minimal.iso
# ls -lR /var/lib/libvirt/images
/var/lib/libvirt/images:
total 4
drwxr-xr-x 2 root root 4096 Feb 13 18:29 isos

/var/lib/libvirt/images/isos:
total 0
lrwxrwxrwx 1 root root 39 Feb 13 18:29 Rocky-9.1-x86_64-minimal.iso -> /home/user/Rocky-9.1-x86_64-minimal.iso
```

およそ以下のようなコマンドで、仮想マシンのインストールを開始します。
```
# virt-install --virt-type kvm --name rockylinux --cdrom Rocky-9.1-x86_64-minimal.iso --os-variant rhel9.0 --disk size=10 --network network=ovs --memory 1024 --graphics vnc,listen=0.0.0.0,port=5900
WARNING  Requested memory 1024 MiB is less than the recommended 1536 MiB for OS rhel9.0
WARNING  Unable to connect to graphical console: virt-viewer not installed. Please install the 'virt-viewer' package.
WARNING  No console to launch for the guest, defaulting to --wait -1

Starting install...
Allocating 'rockylinux.qcow2'                                                                                                                                                                                         |  10 GB  00:00:01

Domain is still running. Installation may be in progress.
Waiting for the installation to complete.
```

仮想マシンを起動すると、tap デバイスが作成されて、仮想ブリッジに割り当てられています。
```
# ovs-vsctl show
537967ba-2d48-46f4-9ab8-bb3116153289
    Bridge ovsbr0
        Port ens4
            Interface ens4
        Port ovsbr0
            Interface ovsbr0
                type: internal
        Port vnet0
            Interface vnet0
    ovs_version: "2.15.0"

# ip link show vnet0
10: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether fe:54:00:80:60:76 brd ff:ff:ff:ff:ff:ff
```

ストレージプール default のボリュームを確認します。
```
# virsh vol-list default
 Name               Path
--------------------------------------------------------------
 isos               /var/lib/libvirt/images/isos
 rockylinux.qcow2   /var/lib/libvirt/images/rockylinux.qcow2
```

virt-install 実行時に作成されたボリューム rockylinux.qcow2 を確認します。
```
# virsh vol-info /var/lib/libvirt/images/rockylinux.qcow2
Name:           rockylinux.qcow2
Type:           file
Capacity:       10.00 GiB
Allocation:     1.34 GiB

# virsh vol-dumpxml /var/lib/libvirt/images/rockylinux.qcow2
<volume type='file'>
  <name>rockylinux.qcow2</name>
  <key>/var/lib/libvirt/images/rockylinux.qcow2</key>
  <source>
  </source>
  <capacity unit='bytes'>10737418240</capacity>
  <allocation unit='bytes'>1440272384</allocation>
  <physical unit='bytes'>10739318784</physical>
  <target>
    <path>/var/lib/libvirt/images/rockylinux.qcow2</path>
    <format type='qcow2'/>
    <permissions>
      <mode>0600</mode>
      <owner>64055</owner>
      <group>64055</group>
    </permissions>
    <timestamps>
      <atime>1676364544.476000000</atime>
      <mtime>1676364327.712000000</mtime>
      <ctime>1676364327.712000000</ctime>
      <btime>0</btime>
    </timestamps>
    <compat>1.1</compat>
    <features>
      <lazy_refcounts/>
    </features>
  </target>
</volume>
```


# ボリュームの作成

`default` ストレージプールに qcow2 フォーマットの、容量10GBのボリュームを作成します。

```
# virsh vol-create-as default rockylinux_vdb.qcow2 10GB --format qcow2 --allocation 0
Vol rockylinux_vdb.qcow2 created

# virsh vol-list default
 Name                   Path
----------------------------------------------------------------------
 isos                   /var/lib/libvirt/images/isos
 rockylinux.qcow2       /var/lib/libvirt/images/rockylinux.qcow2
 rockylinux_vdb.qcow2   /var/lib/libvirt/images/rockylinux_vdb.qcow2

# virsh vol-info /var/lib/libvirt/images/rockylinux_vdb.qcow2
Name:           rockylinux_vdb.qcow2
Type:           file
Capacity:       9.31 GiB
Allocation:     196.00 KiB

# virsh vol-dumpxml /var/lib/libvirt/images/rockylinux_vdb.qcow2
<volume type='file'>
  <name>rockylinux_vdb.qcow2</name>
  <key>/var/lib/libvirt/images/rockylinux_vdb.qcow2</key>
  <source>
  </source>
  <capacity unit='bytes'>10000000000</capacity>
  <allocation unit='bytes'>0</allocation>
  <physical unit='bytes'>10000000000</physical>
  <target>
    <path>/var/lib/libvirt/images/rockylinux_vdb.qcow2</path>
    <format type='raw'/>
    <permissions>
      <mode>0600</mode>
      <owner>0</owner>
      <group>0</group>
    </permissions>
    <timestamps>
      <atime>1676364968.448000000</atime>
      <mtime>1676364968.432000000</mtime>
      <ctime>1676364968.432000000</ctime>
      <btime>0</btime>
    </timestamps>
  </target>
</volume>
```

## アタッチ

作成したボリュームを仮想マシンにアタッチします。
アタッチ前後の状態を比較するために、いったん現状の状態の xml を取得します。
```
# virsh dumpxml rockylinux > before.xml
```

ボリュームを仮想マシンにアタッチします。
```
# virsh attach-disk rockylinux /var/lib/libvirt/images/rockylinux_vdb.qcow2 vdb --subdriver qcow2
Disk attached successfully

# virsh dumpxml rockylinux | diff -u - before.xml
--- -   2023-02-14 04:09:52.813796808 -0500
+++ before.xml  2023-02-14 04:04:11.268000000 -0500
@@ -67,14 +67,6 @@
       <alias name='virtio-disk0'/>
       <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
     </disk>
-    <disk type='file' device='disk'>
-      <driver name='qemu' type='qcow2'/>
-      <source file='/var/lib/libvirt/images/rockylinux_vdb.qcow2' index='4'/>
-      <backingStore/>
-      <target dev='vdb' bus='virtio'/>
-      <alias name='virtio-disk1'/>
-      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
-    </disk>
     <disk type='file' device='cdrom'>
       <driver name='qemu'/>
       <target dev='sda' bus='sata'/>
```

アタッチしたとき、仮想マシンのシスログには下記のようなメッセージが出力されます。
```
[23234.383960] pcieport 0000:00:02.6: pciehp: Slot(0-6): Attention button pressed
[23234.390474] pcieport 0000:00:02.6: pciehp: Slot(0-6) Powering on due to button press
[23234.397926] pcieport 0000:00:02.6: pciehp: Slot(0-6): Card present
[23234.398460] pcieport 0000:00:02.6: pciehp: Slot(0-6): Link Up
[23234.624674] pci 0000:07:00.0: [1af4:1042] type 00 class 0x010000
[23234.633946] pci 0000:07:00.0: reg 0x14: [mem 0x00000000-0x00000fff]
[23234.635238] pci 0000:07:00.0: reg 0x20: [mem 0x00000000-0x00003fff 64bit pref]
[23234.645689] pci 0000:07:00.0: BAR 4: assigned [mem 0xfc000000-0xfc003fff 64bit pref]
[23234.646621] pci 0000:07:00.0: BAR 1: assigned [mem 0xfdc00000-0xfdc00fff]
[23234.647211] pcieport 0000:00:02.6: PCI bridge to [bus 07]
[23234.647816] pcieport 0000:00:02.6:   bridge window [io  0x7000-0x7fff]
[23234.658811] pcieport 0000:00:02.6:   bridge window [mem 0xfdc00000-0xfddfffff]
[23234.660599] pcieport 0000:00:02.6:   bridge window [mem 0xfc000000-0xfc1fffff 64bit pref]
[23234.663712] virtio-pci 0000:07:00.0: enabling device (0000 -> 0002)
[23234.689168] virtio_blk virtio5: [vdb] 19531250 512-byte logical blocks (10.0 GB/9.31 GiB)
```


### アタッチしたボリュームの利用

仮想マシン側では、vdb として認識されています。
通常のLinuxでの操作と同じように、パーティション作成、ファイルシステム作成、マウントして利用します。
```
# ls -l /dev/vd*
brw-rw----. 1 root disk 252,  0 Feb 13 21:42 /dev/vda
brw-rw----. 1 root disk 252,  1 Feb 13 21:42 /dev/vda1
brw-rw----. 1 root disk 252,  2 Feb 13 21:42 /dev/vda2
brw-rw----. 1 root disk 252, 16 Feb 14 04:08 /dev/vdb

# parted /dev/vdb print
Error: /dev/vdb: unrecognised disk label
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 10.0GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags:

# parted /dev/vdb mklabel msdos
Information: You may need to update /etc/fstab.


# parted /dev/vdb print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 10.0GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start  End  Size  Type  File system  Flags


# parted /dev/vdb mkpart primary 0% 100%
Information: You may need to update /etc/fstab.


# parted /dev/vdb print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 10.0GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  9999MB  9998MB  primary


# mkfs -t ext4 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done
Creating filesystem with 2440960 4k blocks and 610800 inodes
Filesystem UUID: 942322b6-ca7d-4e8a-abe4-ebbbeca69cec
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done


# mount /dev/vdb1 /mnt

# df /mnt
Filesystem     1K-blocks  Used Available Use% Mounted on
/dev/vdb1        9508032    24   9003432   1% /mnt
```

## デタッチ

アタッチしたボリュームを仮想マシンから取り外す際は、まず最初に仮想マシン側でアンマウントします。
```
# umount /mnt
```

アンマウントしたら取り外します。
```
# virsh detach-disk rockylinux vdb --print-xml
<disk type="file" device="disk">
      <driver name="qemu" type="qcow2"/>
      <source file="/var/lib/libvirt/images/rockylinux_vdb.qcow2" index="4"/>

      <target dev="vdb" bus="virtio"/>
      <alias name="virtio-disk1"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </disk>

# virsh detach-disk rockylinux vdb
Disk detached successfully
```

仮想マシンのシスログには下記のようなメッセージが出力されます。
```
[23863.541774] pcieport 0000:00:02.6: pciehp: Slot(0-6): Attention button pressed
[23863.542365] pcieport 0000:00:02.6: pciehp: Slot(0-6): Powering off due to button press
```

## ボリュームの削除

不要なボリュームは下記で削除ができます。
```
# virsh vol-delete /var/lib/libvirt/images/rockylinux_vdb.qcow2
Vol /var/lib/libvirt/images/rockylinux_vdb.qcow2 deleted

# virsh vol-list default
 Name               Path
--------------------------------------------------------------
 isos               /var/lib/libvirt/images/isos
 rockylinux.qcow2   /var/lib/libvirt/images/rockylinux.qcow2

```
