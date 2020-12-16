---
title: 网络编程007：QUIC协议
index_img: /cover/22.jpg
banner_img: /cover/top.jpg
date: 2020-6-22
categories: 网络优化
---

TCP协议在创建连接之前需要进行三次握手：

![网络编程懒人入门(十)：一泡尿的时间，快速读懂QUIC协议_11111.png](http://www.52im.net/data/attachment/forum/201910/31/222033u1b3zwxj1m05cr3g.png)

如果是传输加密数据的话，还需要建立 TLS 连接，握手次数更多：

![网络编程懒人入门(十)：一泡尿的时间，快速读懂QUIC协议_2.png](http://www.52im.net/data/attachment/forum/201910/31/201028q2e00d2zsas6ac23.png)

前面我们说过，TCP在网络稳定的情况下性能会比较好，但是在弱网情况下就会暴露出许多问题。



### QUIC协议登场

QUIC 是 Quick UDP Internet Connections 的缩写，谷歌发明的新传输协议。

UDP 协议是无连接协议。它无需在传输层对数据包进行确认，为了确保数据传输的可靠性，应用层协议需要自己完成包传输情况的确认。

这其实是一种很常见的优化思路：由于 TCP 是在操作系统内核和中间件固件中实现的，因此对 TCP 进行重大更改几乎是不可能的。所以采用UDP，然后将TCP的一些特性自己接管过来实现，这样就可以自由选择，甚至根据应用场景自由调整优化。



### QUIC 核心特性

#### 建立连接需要的 RTT 少

> RTT(Round-Trip Time)：往返时延。 是客户端发送数据，到收到回应的时间。

![技术扫盲：新一代基于UDP的低延时网络传输层协议——QUIC详解_1.jpeg](http://www.52im.net/data/attachment/forum/201801/04/113826ga5tuc1hjfcchfj1.jpeg)

TCP协议的RTT主要是多在 TCP的三次握手上。TLS1.3的握手只需要一个 RTT。所以QUIC的握手应该是和1.3差不多的，或者它会直接使用TLS1.3，囧，我也没去细看这些。

#### 改进的拥塞控制

从拥塞算法本身来看，QUIC 只是按照 TCP 协议重新实现了一遍，那么 QUIC 协议到底改进在哪些方面呢？主要有如下几点。

##### 单调递增的 Packet Number

TCP 为了保证可靠性，使用了基于字节序号的 Sequence Number 及 Ack 来确认消息的有序到达。

但是这样会导致一个问题：

![技术扫盲：新一代基于UDP的低延时网络传输层协议——QUIC详解_2.jpeg](http://www.52im.net/data/attachment/forum/201801/04/115848rp2apnj3a3msh3n2.jpeg)

如上图所示，超时事件 RTO 发生后，客户端发起重传，然后接收到了 Ack 数据。由于序列号一样，**这个 Ack 数据到底是原始请求的响应还是重传请求的响应呢**？不好判断！！！

> RTT 采样的结果会影响RTO，而 RTO 设长了，重发就慢，丢了老半天才重发，没有效率，性能差；设短了，会导致可能并没有丢就重发。于是重发的就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发。

由于 Quic 重传的 Packet 和原始 Packet 的 Pakcet Number 是严格递增的，所以很容易就解决了这个问题。

![技术扫盲：新一代基于UDP的低延时网络传输层协议——QUIC详解_3.jpeg](http://www.52im.net/data/attachment/forum/201801/04/115928zw25wvzeneezjinz.jpeg)

如上图所示，RTO 发生后，根据重传的 Packet Number 就能确定精确的 RTT 计算。如果 Ack 的 Packet Number 是 N+M，就根据重传请求计算采样 RTT。如果 Ack 的 Pakcet Number 是 N，就根据原始请求的时间计算采样 RTT，没有歧义性。

##### 没有队头阻塞的多路复用

![技术扫盲：新一代基于UDP的低延时网络传输层协议——QUIC详解_12.jpeg](http://www.52im.net/data/attachment/forum/201801/04/120441vx7hgqphme3qpcqy.jpeg)

HTTP2 在一个 TCP 连接上同时发送 4 个 Stream。其中 Stream1 已经正确到达，并被应用层读取。但是 Stream2 的第三个 tcp segment 丢失了，TCP 为了保证数据的可靠性，需要发送端重传第 3 个 segment 才能通知应用层读取接下去的数据，虽然这个时候 Stream3 和 Stream4 的全部数据已经到达了接收端，但都被阻塞住了。

不仅如此，由于 HTTP2 强制使用 TLS，还存在一个 TLS 协议层面的队头阻塞。

> Record 是 TLS 协议处理的最小单位，最大不能超过 16K，一些服务器比如 Nginx 默认的大小就是 16K。由于一个 record 必须经过数据一致性校验才能进行加解密，所以一个 16K 的 record，就算丢了一个字节，也会导致已经接收到的 15.99K 数据无法处理，因为它不完整。
>
> 当 Record 大小超过了 MTU 的时候，就会分多个数据包发送。

**那 QUIC 多路复用为什么能避免上述问题呢？**

- QUIC 最基本的传输单元是 Packet，不会超过 MTU 的大小，整个加密和认证过程都是基于 Packet 的，不会跨越多个 Packet。这样就能避免 TLS 协议存在的队头阻塞；
- Stream 之间相互独立，比如 Stream2 丢了一个 Pakcet，不会影响 Stream3 和 Stream4。不存在 TCP 队头阻塞。



![技术扫盲：新一代基于UDP的低延时网络传输层协议——QUIC详解_14.jpeg](http://www.52im.net/data/attachment/forum/201801/04/120556muuvqdaq3aya1d0w.jpeg)

##### 连接迁移

一条 TCP 连接是由四元组标识的（源 IP，源端口，目的 IP，目的端口）。

什么叫连接迁移呢？就是当其中任何一个元素发生变化时，这条连接依然维持着，能够保持业务逻辑不中断。当然这里面主要关注的是客户端的变化，因为客户端不可控并且网络环境经常发生变化，而服务端的 IP 和端口一般都是固定的。

比如大家使用手机在 WIFI 和 4G 移动网络切换时，客户端的 IP 肯定会发生变化，需要重新建立和服务端的 TCP 连接。

那 QUIC 是如何做到连接迁移呢？很简单，任何一条 QUIC 连接不再以 IP 及端口四元组标识，而是以一个 64 位的随机数作为 ID 来标识，这样就算 IP 或者端口发生变化时，只要 ID 不变，这条连接依然维持着，上层业务逻辑感知不到变化，不会中断，也就不需要重连。

由于这个 ID 是客户端随机产生的，并且长度有 64 位，所以冲突概率非常低。

还有其他的新特性，暂时就不介绍了。

最新的HTTP3也是基于QUIC的，所以，是一种趋势。

### 参考文档

http://www.52im.net/thread-1309-1-1.html

http://www.52im.net/thread-2816-1-1.html

[https://liudanking.com/network/tls1-3-quic-%E6%98%AF%E6%80%8E%E6%A0%B7%E5%81%9A%E5%88%B0-0-rtt-%E7%9A%84/](https://liudanking.com/network/tls1-3-quic-是怎样做到-0-rtt-的/)

http://www.52im.net/thread-515-1-1.html
