#### 【阅读笔记】计算机网络之RFC7323

---

**原文链接**：[https://tools.ietf.org/html/rfc7323](https://tools.ietf.org/html/rfc7323)

**标题**：TCP Extensions for High Performance

**阅读时间**：2019年8月-2019年9月

**说明**：对于TCP网络的性能瓶颈及调优问题，需要对协议本身有深入理解才有可能解决。

**重点内容摘要**：

#### 一、TCP性能：

bandwidth\*delay product：网络速率与网络延迟的乘积，代表填满整个管道需要的数据量；尽量让整个管道都填满数据（Keep The Pipe Full），是提高TCP性能的基本方法。

long, fat pipe（以及LFN）：网络速率很高、延迟很大的网络。

##### 1、Window Size

TCP包头内表示Window Size的只有16位长，所有接收端传递给发送端的Window Size最大也只能是2^16=65KB，从而可能导致线路的利用率较低。

通过增加Window Scale选项，使得Window Size最多增加14位，所以最大的Window Size可以达到2^\(16+14\)=1GB。

##### 2、网络丢包

由于TCP协议的Slow Start等机制，少量的丢包会导致发送端发送速率大幅度回撤，恢复到正常水平需要一段比较长的时间，从而导致性能问题。TCP协议自身的解决方案有：Fast Retransmit，Fast Recovery，Selective Acknowledgement等。

Fast Retransmit：普通情况下，TCP发送端需要等待timeout才能重新发送一个包；但是有了Fast Retransmit，发送端如果收到多个duplicated ack包（一般是三个重复包，总共四个），就会马上重传，放弃继续等待timeout。

Fast Recovery：在发生Fast Retransmit的时候，不会把CWND（拥塞窗口大小，用于控制发送端发送的速率）降到1，而是降到原来的一半。

有一张图很好的解释了Fast Retransmit和Fast Recovery：  
![](/assets/Fast-Retransmit-Fast-Recovery-Reno.png)

```
注意：TCP Congestion Control有很多算法，上面这个是Reno算法，而Tahoe算法则没有Fast Recovery。可以认为Reno是Tahoe的改进版。
具体参考：https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_Tahoe_and_Reno
```

Slow Start、Congestion Avoidance、Fast Retransmit、Fast Recovery，详细内容可以参考RFC5681：[TCP Congestion Control](https://tools.ietf.org/html/rfc5681)

> **注意**：Congestion Control主要是从发送端来进行控制，避免导致整个链路拥塞，重点在于感知整个链路的健康状况并作出相应的调整。而TCP的Flow Control（主要是通过[滑动窗口-sliding window](https://en.wikipedia.org/wiki/Sliding_window_protocol)），则主要是接收端主动控制，避免发送端发送过多数据给自己。

由于各个厂商对于TCP的Congestion Control算法存在诸多不同的诉求和改进需要，所有有很多新的算法出来，比如 [BBR](https://cloud.google.com/blog/products/gcp/tcp-bbr-congestion-control-comes-to-gcp-your-internet-just-got-faster)

关于Reno/Cubic/Vegas/BBR等拥塞算法，可以参考这篇文章：[https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/](https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/)

---

#### 二、TCP的可靠性

重复的sequence number主要来自两方面

1、序列号溢出了、用完了被覆盖，可以通过PAWS彻底解决，利用了TCP Timestamps选项

2、上一个session链接的包被delay了，可以通过MSL以及临时端口的随机化来解决

#### 三、TCP选项-TCP Options

Window Scale Option只会在SYN包和对应的SYN-ACK包中设置

Timestamps Option会在所有的数据包和ACK包中设置，不仅仅是SYN相关的包

