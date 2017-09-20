---
layout: post 
title:  "iptables学习"
date:   2017-09-05 17:50:00 +0800
excerpt_separator: <!--abstract-->
tags: iptables
---

公司有一套仿真环境，使用IPTABLES防火墙，隔绝与线上服务器的连接，最近发现环境内部某台机器丢包率很严重，追查过程中，于是有了这篇学习博客。<!--abstract-->

### iptables简介

先简单介绍下iptables吧。iptables包含五张表(tables)：

- raw 优先级最高的一张表，作用是通过NOTRACK给不需要被连接跟踪的包打标记。
- filter 也就是默认表，存放防火墙相关所有操作。
- nat 用于网络地址转换、端口转换等。
- mangle 用于对特定数据包的修改，较少使用。
- security 用于强制访问控制网络规则，如SElinux

每个表包含数量不等的链(chains)：
例如：filter表，包含 INPUT、OUTPUT、FORWARD等链，每条链又由一系列按顺序排列的规则组成。数据包顺序经过不同的链，匹配不同的规则，最后根据规则选择是否投送到目的，或是丢弃。

借用archlinux的一张图，表示下网络包经过iptables的过程：

```
                               XXXXXXXXXXXXXXXXXX
                             XXX     Network    XXX
                               XXXXXXXXXXXXXXXXXX
                                       +
                                       |
                                       v
 +-------------+              +------------------+
 |table: filter| <---+        | table: nat       |
 |chain: INPUT |     |        | chain: PREROUTING|
 +-----+-------+     |        +--------+---------+
       |             |                 |
       v             |                 v
 [local process]     |           ****************          +--------------+
       |             +---------+ Routing decision +------> |table: filter |
       v                         ****************          |chain: FORWARD|
****************                                           +------+-------+
Routing decision                                                  |
****************                                                  |
       |                                                          |
       v                        ****************                  |
+-------------+       +------>  Routing decision  <---------------+
|table: nat   |       |         ****************
|chain: OUTPUT|       |               +
+-----+-------+       |               |
      |               |               v
      v               |      +-------------------+
+--------------+      |      | table: nat        |
|table: filter | +----+      | chain: POSTROUTING|
|chain: OUTPUT |             +--------+----------+
+--------------+                      |
                                      v
                               XXXXXXXXXXXXXXXXXX
                             XXX    Network     XXX
                               XXXXXXXXXXXXXXXXXX
```

一般来说，使用filter表和nat表就能满足防火墙、网络地址转发等简单需求。

### 遇到的问题

最近某台机器出现了丢包率很高的问题，机器上有iptables，刚开始感觉是不是iptables表太长了，过滤包的时候过滤太久了，导致ping超时。后来想了下，毕竟作为linux的防火墙，性能应该没这么差吧，也就七百多条记录。翻了翻dmesg，果然发现了问题。

```
__ratelimit: 4013 callbacks suppressed
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
nf_conntrack: table full, dropping packet.
```

google查了下，发现是和iptables有关。

刚开始设置机器iptables时候，也没多想，就最简单的使用了类似：`iptables -A OUTPUT -d DEST_IP -j ACCEPT`，最后加个：`iptables -A OUTPUT -j DROP` 等简单命令配置的iptables防火墙。

允许访问环境内部机器以及公司的基础监控、部署、DNS等机器，禁止访问其他任何机器，可以满足需求。

这里还要说要REJECT和DROP，到底用哪个的问题。可以参考 [Drop versus Reject](http://www.chiark.greenend.org.uk/~peterb/network/drop-vs-reject)。

简而言之，REJECT相对于DROP，能让请求方显示更多信息。例如：

```
# 目标IP 192.168.40.10，配置iptables规则如下：
192.168.40.10:~$ iptables -A INPUT -s 192.168.40.20 -j REJECT

# 此时，如果源IP来ping目标IP，会显示：
192.168.40.20:~$ ping 192.168.40.10
PING 192.168.40.10 (192.168.40.10) 56(84) bytes of data.
From 192.168.40.10 icmp_seq=1 Destination Port Unreachable
From 192.168.40.10 icmp_seq=2 Destination Port Unreachable
From 192.168.40.10 icmp_seq=3 Destination Port Unreachable
From 192.168.40.10 icmp_seq=4 Destination Port Unreachable

# 修改目标IP的iptables配置为DROP：
192.168.40.10:~$ iptables -R INPUT 1 -s 192.168.40.20 -j DROP

# 此时，再在源IP上ping目标IP，则不会显示任何提示信息，Ctrl + C 后，会显示丢包。
192.168.40.20:~$ ping 192.168.40.10
PING 192.168.40.10 (192.168.40.10) 56(84) bytes of data.
^C
--- 192.168.40.10 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2840ms

```

但是iptables不加 `-t` 参数的情况下，默认使用filter表，每个连接都会使用nf_conntrack记录连接状态，以供iptables的 `nat` 和 `state` 模块使用。

使用场景：

- `nat` 根据转发规规则修改源、目标地址，依靠 `nf_conntrack` 的连接记录，让返回的包能路由到发请求的机器。
- `state` 直接使用 `nf_conntrack` 来记录连接状态（`NEW`/`ESTABLISHED`/`RELATED`/`INVALID`）来匹配过滤规则。

各连接状态解释：

```
NEW -- meaning that the packet has started a new connection, or otherwise associated with a connection which has not seen packets in both directions, and

ESTABLISHED -- meaning that the packet is associated with a connection which has seen packets in both directions,

RELATED -- meaning that the packet is starting a new connection, but is associated with an existing connection, such as an FTP data transfer, or an ICMP error.

INVALID -- meaning that the packet can not be recognize which connection or state it blongs to.
```

dmesg中显示这个错误的原因，是 `nf_conntrack` 模块记录的连接条数过多，连接进来比释放的块，把哈希表写满，导致新来的包会被丢弃。

那么，怎么排查、解决这个问题呢？

### 解决问题

`/proc/net/nf_conntrack` 文件中，记录了iptables跟踪的每个连接的详情，记录格式为：

```
网络层协议名 网络层协议编号 传输层协议名 传输层协议编号 记录失效前剩余秒数 连接状态 [...key=value...] flag格式
ipv4        2           tcp         6           57             TIME_WAIT src=127.0.0.1 dst=127.0.0.1 sport=30675 dport=9002 src=127.0.0.1 dst=127.0.0.1 sport=9002 dport=30675 [ASSURED] mark=0 secmark=0 use=2

flag:
[ASSURED] 请求和响应都有流量
[UNREPLIED] 没收到响应，哈希表满时首先丢弃这部分连接
```

和 `nf_conntrack` 相关的一些常用系统参数：

```
# 哈希表大小，默认 16384.
# netfilter使用不可交换的内和空间内存，哈希表过大可能影响内存使用
net.netfilter.nf_conntrack_buckets

# 连接跟踪表大小，默认 nf_conntrack_buckets * 4 (65536)。
# 默认数值过小，可以改到和服务器较忙时连接相配的数量，同时，按比例修改哈希表大小
net.netfilter.nf_conntrack_max

# 哈希表实时连接跟踪数（只读）
net.netfilter.nf_conntrack_count

# time_wait超时时间，默认120秒
# nf_conntrack文件里大多都是time_wait，可以调小些以减小nf_conntrack数量。
net.netfilter.nf_conntrack_tcp_timeout_time_wait

# established的连接的超时时间，默认五天，432000秒。
net.netfilter.nf_conntrack_timeout_established

# 通用超时设置，默认600秒
net.netfilter.nf_conntrack_generic_timeout

```

#### 方案一

通过调大 `nf_conntrack` 哈希表和跟踪表大小。但是缺点是，会增加内存消耗。

调整示例：

```
net.netfilter.nf_conntrack_max  =   1048576
net.netfilter.nf_conntrack_buckets =   262144
net.netfilter.nf_conntrack_tcp_timeout_established  =   3600
net.netfilter.nf_conntrack_tcp_timeout_close_wait  =   60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait  =   60
net.netfilter.nf_conntrack_tcp_timeout_time_wait  =   60
```

#### 方案二

禁用相关模块。缺点：不能使用 `state`、`nat` 相关模块了。

在我们的场景中，iptables仅仅作为封禁ip访问，未使用 `state`、`nat`等模块。

```
# 查看相关模块
lsmod | egrep "ip_table|iptable|nat|conntrack"

# 停掉iptables后，移除相关模块
service iptables stop 

rmmod iptable_nat
rmmod ip6table_nat
rmmod nf_defrag_ipv4
rmmod nf_defrag_ipv6
rmmod nf_nat
rmmod nf_nat_ipv4
rmmod nf_nat_ipv6
rmmod nf_conntrack
rmmod nf_conntrack_ipv4
rmmod nf_conntrack_ipv6
rmmod xt_conntrack

# 禁用 nf_conntrack 模块 
blacklist nf_conntrack 
blacklist nf_conntrack_ipv6 
blacklist xt_conntrack 
blacklist nf_conntrack_ftp 
blacklist xt_state 
blacklist iptable_nat 
blacklist ipt_REDIRECT 
blacklist nf_nat 
blacklist nf_conntrack_ipv4
```

注：nf、nat、conntrack等模块，在机器达到某种连接负载后才会出现。直接找台空机器，写入iptables规则，除非机器设定启动加载，否则netfilter各模块不会加载。

需要使用 `modprobe` 命令加载对应模块，使用 `rmmod [mod_name]` 或 `modprobe -r [mod_name]` 命令卸载模块，`modinfo` 查看模块信息。

#### 方案三

之前我们也提到了，raw表的功能是通过NOTRACK给不需要被连接跟踪的包打标记。所以，我们可以使用raw表，将特定包取消打标签，可以减少nf_conntrack文件中，记录的条目。

例如：在本例中，所有iptables规则都未使用 `--state` 来跟踪链接状态。

故可以使用：`iptables -t raw -A OUTPUT -s LOCAL_IP -j NOTRACK` 来对本机出口的包，都不记录状态。

***


参考资料
1. [nf_conntrack踩坑总结](https://testerhome.com/topics/7509)
2. [iptables(简体中文)](https://wiki.archlinux.org/index.php/Iptables_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
3. [iptables状态机制](http://os.51cto.com/art/201108/285209.htm)
4. [解决nf_conntrack表满的几种思路](http://jaseywang.me/2012/08/16/%E8%A7%A3%E5%86%B3-nf_conntrack-table-full-dropping-packet-%E7%9A%84%E5%87%A0%E7%A7%8D%E6%80%9D%E8%B7%AF/)