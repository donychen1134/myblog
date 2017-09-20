---
layout: post 
title:  "Flow Control and Congestion Control"
date:   2017-09-20 17:50:00 +0800
excerpt_separator: <!--abstract-->
tags: flow_control congestion_control
---

仍然是《Computer Networking: A Top-Down Approach》的一部分，本篇来聊聊流量控制和拥塞控制。<!--abstract-->


## Flow control 流量控制

因为 TCP 是一个全双工的协议，故协议两端都有可能成为发送方或接收方。所以，双方都有缓冲区，可以缓存协议上层未读取的包。

TCP 引入流量控制，是为了防止发送方发送大量数据，导致接收方缓冲区溢出。

发送方维护一个名为 **接收窗口** 的值，来判断接收方的缓存区大小。

假设：Host A 向 Host B 发送一个大文件，Host B 分配大小为 RcvBuffer 的缓冲区。我们定义如下参数：

```
Host B:
RcvBuffer：缓冲区大小
LastByteRead：Host B 的进程最近从缓冲区读取的最后一个字节
LastByteRcvd：Host B 从 A 接收到的最后一个字节
rwnd：Recive Window. Host B 的接收窗口大小

Host A:
LastByteSent：发送的最后一个字节
LastByteAcked：接收到的最后一个ACK

```

对于 Host B:

```
LastByteRcvd - LastByteRead <= RcvBuffer
rwnd = RcvBuffer - (LastByteRcvd - LastByteRead)
```

每次 Host B 给 Host A 回包，会发送 rwnd，初始时，rwnd = RcvBuffer 。

对于 Host A:

```
LastByteSent - LastByteAcked <= rwnd
```

否则缓冲区溢出。


如果 LastByteSent - LastByteAcked = rwnd，则 Host A 不再继续发送数据。达到了流量控制的目的。
但如果不发数据，Host A 也就不知道 Host B 的 rwnd 的情况。

故当 LastByteSent - LastByteAcked = rwnd 时，Host A 继续发送一个包给 Host B。 rwnd = 0 时，B 丢弃这个包。 rwnd > 0 时，B 返回非零的 rwnd，A 继续发送数据。

## Congestion Control 拥塞控制

TCP 的拥塞控制，建立在 **拥塞窗口** 这样一个参数上，记为：cwnd (congestion window)。那么，对于发送方而言：

```
LastByteSent - LastByteAckes <= min{cwnd, rwnd}
```

发送方是如何检测网络拥塞的呢？TCP 定义了两种包丢失的情形：

1. 发生超时
2. 收到三个重复的ACK

当发送方检测到丢包，则认为网络拥塞。

那么，TCP该如何确定自己的发送速率呢？

- 当发生丢包时，发送方应该降低发送速率
- 当收到 ACK 时，发送方可以继续提高发送速率
- 发送方需要探测网络带宽

综上所述，引出我们将要介绍的 **TCP拥塞控制算法**。拥塞控制算法有三个主要的部分：

1. `slow start` 慢启动 （试探网络带宽，迅速提高传输速率）
	- 初始 cwnd 为 1 MSS(Max Segment Size 最大报文长度)
	- 每收到 1 个 ACK，发送方就多发一个报文。发送数量指数级上升，第 i 次，发送 2^(i-1) 个包
	- 发生超时，`ssthresh (slow start threshold) = cwnd / 2 | cwnd = 1` 即：慢启动阈值为拥塞窗口的一半，且拥塞窗口为1。仍然是 `slow start` 状态。
	- 当 cwnd >= ssthresh 时，进入 `congestion avoidance` 状态。
	- 当收到三个 dupACK 时，`ssthresh = cwnd / 2 | cwnd = ssthresh + 3*MSS`，进入 `fast recovery` 状态。
2. `congestion avoidance` 拥塞避免 （达到较高传输速率，线性提升传输速率）
	- 当接收到新 ACK 时，每次增加 1 MSS 的拥塞窗口 cwnd。（线性增长）
	- 发生超时，`ssthresh = cwnd / 2 | cwnd = 1`，且返回 `slow start` 状态。
	- 当收到三个 dupACK 时，`ssthresh = cwnd / 2 | cwnd = ssthresh + 3*MSS `，进入 `fast recovery` 状态。
3. `fast recovery` 快速恢复 （发生拥塞时，根据后续网络状况，决定进入之前两个状态之一）
	- 当接收到新 ACK 时，表明网络从拥塞中恢复，那么应该尽快进入较高发送速率状态。`cwnd = ssthresh | dupACK = 0` 进入 `congestion avoidance` 状态。
	- 发生超时，表明网络仍然处于拥塞状态，此时应调低发送速率。 `cwnd = 1 | ssthresh = cwnd / 2 | dupACK = 0` 进入 `slow start` 状态。

TCP 拥塞控制也经常被称为 sdditive-increase, multiplicative-decrease (AIMD) **和式增加积式减少** 形式的拥塞控制。


![congestion_control](../../../assets/2017-09-20-flow-and-congestion-control/congestion_control.png)


## TCP 三次握手 四次挥手

说完拥塞控制和流量控制，顺便提一下 TCP 建立、断开连接的两个过程。

#### 三次握手

client 选择起始序列号 i ，发送 SEQ=i SYN 包给 server，进入 SYN_SEND 状态。
server 接收 SEQ=i SYN 包，选择起始序列号 j，返回 SEQ=j SYN ACK=i+1 包，给 client，并进入 SYN_RCVD 状态。
client 接收 SEQ=j SYN ACK=i+1 包，返回 SEQ=i+1 ACK=j+1 包给 server，进入 ESTABLISHED 状态。
server 接收 SEQ=i+1 ACK=j+1 包，进入 ESTABLISHED 状态。

![congestion_control](../../../assets/2017-09-20-flow-and-congestion-control/shake.png)

#### 四次挥手

client 发送 FIN 包，进入 FIN_WAIT_1 状态。
server 接收 FIN 包，发送 ACK 包，进入 CLOSE_WAIT 状态。
client 接收 ACK 包，进入 FIN_WAIT_2 状态。
server 发送 FIN 包，进入 LAST_ACK 状态。
client 接收 FIN 包，发送 ACK 包，进入 TIME_WAIT 状态。
server 接收 ACK 包，进入 CLOSED 状态。
client 等待 2 MSL(Max Segment Lifetime) 结束后，进入 CLOSED 状态。

![congestion_control](../../../assets/2017-09-20-flow-and-congestion-control/bye.png)

#### 状态流转图

客户端

![congestion_control](../../../assets/2017-09-20-flow-and-congestion-control/client.png)

服务端

![congestion_control](../../../assets/2017-09-20-flow-and-congestion-control/server.png)


## 总结

TCP 流量控制，是为了防止发送方发送数据过快，占满接收方的缓冲区的一种控制手段；拥塞控制，是为了最大限度的保证数据传输速率的一种控制手段。二者虽然效果相似，但是目的不同。两个指标： `rwnd` 和 `cwnd` 共同作用于发送方，控制流量的同时，也对网络的拥塞起到缓解的作用。




***


参考资料
1. 《Computer Networking: A Top-Down Approach》