#### 【阅读笔记】网络之RFC1323

---

**原文链接**：[https://tools.ietf.org/html/rfc1323](https://tools.ietf.org/html/rfc1323)

**标题**：TCP Extensions for High Performance

**阅读时间**：2019年8月

**说明**：对于TCP网络的性能瓶颈及调优问题，需要对协议本身有深入理解才有可能解决。

**重点内容摘要**：

TCP性能：

bandwidth\*delay product：网络速率与网络延迟的乘积，代表填满整个管道需要的数据量；尽量让整个管道都填满数据，是提高TCP性能的基本方法。

long, fat pipe（以及LFN）：网络速率很高、延迟很大的网络。

1、Window Size

TCP包头内表示Window Size的只有16位长，所有接收端传递给发送端的Window Size最大也只能是2^16=65KB，从而可能导致线路的利用率较低。通过增加Window Scale选项，使得Window Size最多增加14位，所以最大的Window Size可以达到2^30=1GB。

2、网络丢包

由于TCP协议的Slow Start等机制，少量的丢包会导致发送端发送速率大幅度回撤，恢复到正常水平需要一段比较长的时间，从而导致性能问题。解决方案有：Fast Retransmit，Fast Recovery，Selective Acknowledgement等。



