#### 【阅读笔记】网络之TCP拥塞控制算法BBR

---

**原文链接**：[https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/](https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/)

**标题**：BBR, the new kid on the TCP block

**阅读时间**：2019年9月

**重点内容摘要**：

一、IP层只管发送网络包，但是把管理数据流（包括检测和修复丢包、管理数据流量）交给传输层的协议（也即TCP协议）。

二、TCP并不是一个固定的协议，TCP怎么管理数据流，怎么检测丢包并做出反应，是有很多的实现的变种的。

三、Reno流量控制算法的缺点在于：

* 对丢包反应过于敏感（丢包就把发送速率降低到原来的一半）
* 在带宽比较大的情况下，在丢包之后再恢复回原来的带宽利用率耗时较长
* Congestion Avoidance会一直增加（每个RTT增加一个segment），最终会导致丢包的发生。



