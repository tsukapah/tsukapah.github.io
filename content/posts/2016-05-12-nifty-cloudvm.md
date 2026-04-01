---
title: NIFTY CloudにVMインポートするのに手こずった件
date: '2016-05-12T18:49:54+09:00'
lastmod: '2016-05-16T11:14:32+09:00'
draft: false
tags:
- NiftyCloud
- CentOS6
- VMインポート
description: VMware上で動いている仮想マシンをNIFTY CloudにVMインポートした際、ドキュメントがあまりないこともありかなり手こずったのでやったことを書いておきます。
params:
  qiita_url: https://qiita.com/tsukapah/items/124057c9475d5a035ca1
  qiita_id: 124057c9475d5a035ca1
---

VMware上で動いている仮想マシンをNIFTY CloudにVMインポートした際、ドキュメントがあまりないこともありかなり手こずったのでやったことを書いておきます。
長い上にチラ裏感がすごいので間違いとかあったら優しく指摘してください。
あと、やったことをメモってるだけなので何かを解決してくれるものではないと思うのでご了承ください。

# 事前情報
## インポート元ホスト
VMware ESXi 5.0.0
## インポートするサーバOS

```bash
# cat /etc/redhat-release
CentOS release 6.2 (Final)
```

## CPU
```bash
# cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 44
model name      : Intel(R) Xeon(R) CPU           E5606  @ 2.13GHz
stepping        : 2
cpu MHz         : 2133.409
cache size      : 8192 KB
fpu             : yes
fpu_exception   : yes
cpuid level     : 11
wp              : yes
flags           : fpu vme de pse tsc msr pae mce cx8 apic mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss syscall nx rdtscp lm constant_tsc up arch_perfmon pebs bts xtopology tsc_reliable nonstop_tsc aperfmperf unfair_spinlock pni pclmulqdq ssse3 cx16 sse4_1 sse4_2 popcnt aes hypervisor lahf_lm arat dts
bogomips        : 4266.81
clflush size    : 64
cache_alignment : 64
address sizes   : 40 bits physical, 48 bits virtual
power management:
```

## メモリ
```bash
# cat /proc/meminfo
MemTotal:        2054984 kB
MemFree:         1617976 kB
Buffers:           30416 kB
Cached:           212312 kB
SwapCached:            0 kB
Active:           213756 kB
Inactive:         120932 kB
Active(anon):      92120 kB
Inactive(anon):     1408 kB
Active(file):     121636 kB
Inactive(file):   119524 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       2097144 kB
SwapFree:        2097144 kB
Dirty:                 8 kB
Writeback:             0 kB
AnonPages:         91980 kB
Mapped:            38580 kB
Shmem:              1568 kB
Slab:              48072 kB
SReclaimable:      21732 kB
SUnreclaim:        26340 kB
KernelStack:        1736 kB
PageTables:        17068 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     3124636 kB
Committed_AS:     458524 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      273876 kB
VmallocChunk:   34359459668 kB
HardwareCorrupted:     0 kB
AnonHugePages:     24576 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:        8192 kB
DirectMap2M:     2088960 kB
```

## インターフェース
```bash
# ifconfig -a
eth0      Link encap:Ethernet  HWaddr 00:0C:29:C0:79:43
          inet addr:192.168.1.138  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:fec0:7943/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:10023 errors:0 dropped:0 overruns:0 frame:0
          TX packets:305 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:883791 (863.0 KiB)  TX bytes:39306 (38.3 KiB)
          
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:64 errors:0 dropped:0 overruns:0 frame:0
          TX packets:64 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
```

## インターフェース定義
```bash
# find /etc/sysconfig/network-scripts -name ifcfg-* -ls -exec cat {} \;
2228763    4 -rw-r--r--   1 root     root          254 10月  7  2011 /etc/sysconfig/network-scripts/ifcfg-lo
DEVICE=lo
IPADDR=127.0.0.1
NETMASK=255.0.0.0
NETWORK=127.0.0.0
# If you're having problems with gated making 127.0.0.0/8 a martian,
# you can change this to something else (255.255.255.255, for example)
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback
2228236    4 -rw-r--r--   3 root     root           38  5月  7 15:49 /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
BOOTPROTO=dhcp
ONBOOT=yes
2231590    4 -rw-r--r--   1 root     root           38  5月  7 15:49 /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
BOOTPROTO=dhcp
ONBOOT=yes
```

このサーバはVMware上ではNICはひとつしか持たせていませんでしたが、
NIFTY CloudではGlobalとPrivateにNICを持たせるつもりなのでこのようにeth1も設定しています。

## DHCPのプロセス確認
```bash
# ps -ef | grep dhclient
root      1340     1  0 11:41 ?        00:00:00 /sbin/dhclient -1 -q -lf /var/lib/dhclient/dhclient-eth0.leases -pf /var/run/dhclient-eth0.pid eth0
root      2815  2763  0 11:55 pts/0    00:00:00 grep dhclient
```

## VMwareToolsのプロセス確認
```bash
# ps -ef | grep vmtoolsd
root      1461     1  0 11:41 ?        00:00:00 /usr/sbin/vmtoolsd
root      2817  2763  0 11:55 pts/0    00:00:00 grep vmtoolsd
```

## OSのFWでDHCP通信が許可されているか確認
```bash
# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
NIFTY F/Wでコントロールしたほうが楽なので、iptablesのサービスは止めてしまいました。


## VMwareToolsバージョン確認
```bash
# vmware-toolbox-cmd -v
8.6.0.8802 (build-515842)
```

# エクスポートの事前作業
## VMwareToolsのバージョンアップ
もし最新でないのであれば最新にしておきましょう。

## NICの設定変更
NIFTY CloudのサーバではNICは2つ(Global×1、Private×1)まで持てるようです。
インポート時にはNICの定義は全てDHCPになっている必要があります。
Private側を最終的に固定にする場合も、インポートする際にはDHCPで設定します。

事前情報のところにも書いてありますが、私の環境は以下のように設定しました。

```bash
# find /etc/sysconfig/network-scripts -name "ifcfg-*" -ls -exec cat {} \;
2228763    4 -rw-r--r--   1 root     root          254 10月  7  2011 /etc/sysconfig/network-scripts/ifcfg-lo
DEVICE=lo
IPADDR=127.0.0.1
NETMASK=255.0.0.0
NETWORK=127.0.0.0
# If you're having problems with gated making 127.0.0.0/8 a martian,
# you can change this to something else (255.255.255.255, for example)
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback
2228236    4 -rw-r--r--   3 root     root           38  5月  7 15:49 /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
BOOTPROTO=dhcp
ONBOOT=yes
2231590    4 -rw-r--r--   1 root     root           38  5月  7 15:49 /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
BOOTPROTO=dhcp
ONBOOT=yes
```

## デフォルトゲートウェイの設定
Globalからsshで入れなくなる原因になったりするので、デフォルトゲートウェイの指定は消しておきましょう。
`/etc/sysconfig/network`に`GATEWAY`の指定がある場合はその行を削除するかコメントアウトします。

```bash
# cat /etc/sysconfig/network
NETWORKING=yes
```

## インターフェースのマッピング削除
これに気づくのに時間がかかりました。
MACアドレスとインターフェースのマッピングがこのファイルで固定化されてしまうため、
インポート後にNICの認識をすると、MACアドレスが違うため意図しないインターフェース名で登録されてしまいます。
なので、以下のファイルを削除します。

+ /etc/udev/rules.d/70-persistent-net.rules  
※ 70のところは環境によって違うかも

```bash
# rm -f /etc/udev/rules.d/70-persistent-net.rules 
```

# VMwareエクスポート
VMware vSphere Clientを使用してvCenter Serverに接続します。
エクスポートしたいサーバを停止した状態で以下のように作業します。

1. [ファイル] - [エクスポート] - [OVFテンプレートのエクスポート] を選びます。
2.  名前と出力先のディレクトリを入力し、[OK]を押します。  
なお、フォーマットは`ファイルのフォルダ(OVF)`にします。
3. 30〜1時間くらいでエクスポートが終わると思います。

# VMインポート
無事エクスポートできたら、NIFTY Cloudにインポートします。

1. コントロールパネルにログインしメニューの[サーバー]の中にある`[VMインポート]`を押します。
2. OVFファイルには先程エクスポートしてできたフォルダの中から拡張子が`.ovf`のものを選択し、[サーバータイプ選択]を押し、先の進みます。
3. ゾーン、タイプは適宜選択してください。
4. [サーバー設定]では必要項目を全て入力します。  
この時、ネットワークの設定でグローバルは好きなものを選びますが、プライベート側は`共通プライベート`で`自動割り当て`を選択します。  
他のネットワークに接続させたくてもとりあえずこれにします。  
あとで変更できるので大丈夫です。  
こうしないとなぜかインポートエラーになります。
原因はサポートに問い合わせ中です。  
[確認]を押して確認画面に進みます。
5. 内容に問題なければ[インポート]ボタンを押して時がすぎるのを待ちましょう。  
私は2時間くらい待たされました。

※ちなみに、どうでもいいですが、学生の頃学校の先生から **Server** は サーバー じゃなくて **サーバ** だよと強く教わりました。
同様に **Router** は **ルータ** です。
なのでカタカナで **サーバー** と書いてあるのが気になってしょうがないです。

# ネットワーク
## ネットワーク設定変更
1. NIFTY Cloudのコンパネで対象サーバを選択し、[選択したサーバーの操作]プルダウンから[ネットワーク設定変更]を選びます。
2. プライベートネットワークを個別に作成したものに変更し、確認を押して先に進みます。

## eth1の設定変更
この時点ではまだプライベートネットワークには繋がっていないので、いったんグローバルIP経由、もしくはWebコンソールを使用して設定を変更します。
以下の感じに設定しました。

```bash
$ cat /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
TYPE=Ethernet
IPADDR=192.168.10.23
NETMASK=255.255.255.0
```

## ルーティング追加
デフォルトではeth0側のDHCPで取得したGWアドレスがデフォルトになっているので、
必要であればプライベートLAN向けの静的ルーティングを追加します。

```bash
$ cat /etc/sysconfig/network-scripts/route-eth1
192.168.1.0/24	via 192.168.10.1
```

これでとりあえずPrivate側で通信できるようになったと思います。

