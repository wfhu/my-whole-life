#### 【阅读笔记】网络之TCP拥塞控制算法BBR

---

**原文链接**：[https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/](https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/)

**标题**：BBR, the new kid on the TCP block

**阅读时间**：2019年9月

**重点内容摘要**：

一、IP层只管发送网络包，但是把管理数据流（包括检测和修复丢包、管理数据流量）交给传输层的协议（也即TCP协议）。

二、TCP并不是一个固定的协议，TCP怎么管理数据流，怎么检测丢包并做出反应，是有很多的实现的变种的。



