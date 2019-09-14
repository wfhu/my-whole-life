#### 【阅读笔记】网络之TCP拥塞控制算法BBR

---

**原文链接**：[https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/](https://blog.apnic.net/2017/05/09/bbr-new-kid-tcp-block/)

**标题**：BBR, the new kid on the TCP block

**阅读时间**：2019年9月

**核心理解：**

整个传输线路有三种状态：空闲（包来了马上得到处理）、排队（处理不过来了，先放到队列里面等着）、丢包（队列都塞满了，只能丢弃）；

Reno和CUBIC实际上是一类（AIMD），Reno使得传输线路在上面三种状态间来回切换；CUBIC也类似，但是相比Reno，CUBIC让传输线路较少处于空闲状态，也因为如此，CUBIC会导致延迟的增加，因为充分利用了排队功能。

Vegas和BBR是另外一种尝试，它们努力让传输线路一直维持在**即将开始排队但是还没排队的状态**。它们重点依赖对RTT的计算，而不像Reno和CUBIC主要依赖对ack包的追踪。

Vegas传输速度也是线性恢复的，这一点和Reno类似。另外一个问题在于，Vegas在开始排队时就开始回撤，如果同一个线路上有类似Reno的session也在跑，因为Reno需要等到丢包了才会回撤，所以整个线路都会被Reno挤占掉。

**重点内容摘要**：

一、IP层只管发送网络包，但是把管理数据流（包括检测和修复丢包、管理数据流量）交给传输层的协议（也即TCP协议）。

二、TCP并不是一个固定的协议，TCP怎么管理数据流，怎么检测丢包并做出反应，是有很多的实现的变种的。

三、Reno流控制算法，基于ACK反馈，它的特点在于：

* 对out-of-order的ACK反馈包的反应比较敏感（会把发送速率降低到原来的一半）。

* 在带宽比较大的情况下，在降速之后再恢复回原来的带宽利用率耗时较长（10G带宽+30msRTT，需要3.5小时才能让带宽从5G提升到10G）。

* Congestion Avoidance理论上会一直增加（每个RTT增加一个segment），最终会导致out-of-order ACK反馈的发生，所以最终会导致类似锯齿状（sawtooth）的带宽使用情况。

* 如果发生丢包导致的timeout的情况，Reno就丢失了整个流控制的ACK反馈信号，只能重新开始整个session的管理（slow-start）。

三、BIC及CUBIC流控制算法：

* BIC基于二分查找算法（binary chop）来从out-of-order的ACK反馈中恢复发送速率。
* BIC有一个最大的常量限制，而且一般比Reno算法固定的一个segment要大一些。
* BIC在低RTT网络环境下，表现得非常的激进（很容易在短时间内又超过上次碰到的最大值），BIC使用指数函数（exponential function）来管理这个流的发送数据大小。
* CUBIC是BIC的一种改进，它使用三阶多项式函数（third-order polynomial function）来管理流数据大小。
* CUBIC对out-of-order的ACK反馈，回撤程度比Reno小，而且能更快地恢复到原来的flow rate



