## 【优化】sdk上报业务优化

---

时间：2018年11月-2019年1月

关键字：Nginx，阿里云，成本优化，HTTPS证书，ECC，ECDHE，ECDSA，椭圆曲线，RSA，带宽，方案

重点概括：

* 通过使用阿里云SLB服务，替换ECS+nginx自建代理服务，节省主机费用（约4万人民币/月）
* 通过使用ECC证书替换RSA证书，节省主机和带宽费用（约30万人民币/月）
* 将上报协议从HTTP/HTTPS修改为TCP协议，彻底规避HTTPS协议和SSL证书，节省带宽和计算资源（约50万人民币/月）

---

2019-03-04更新：

2月份的账单出来了，这是全量上线ecc证书后第一个完整周期的账单，实际节省带宽费用超过**30万人民币**。主要原因是阿里云的计费方式是[增强型95计费](https://helpcdn.aliyun.com/document_detail/89729.html)，目前峰值带宽下降到原来的三分之二左右（当然，2月只有28天，比1月少3天，也是一部分原因）。

![](/assets/2019-02-ali-payment.png)

![](/assets/1-bandwidth-payment.png)

![](/assets/2-bandwidth-payment.png)

---

### **业务描述**

25台阿里云ECS（32核+64G内存+300GB磁盘）+nginx+HTTPS做前端接入层，接收sdk上报过来的数据；根据域名、URL等做简单逻辑判断后，通过HTTP协议转发给12台后端ECS+nginx做进一步处理，之后便是redis/Kafka等存储和队列层。

当时看到的**最直接的问题**是：代理层nginx服务器的CPU利用率很高，升级机器配置和增加机器数量成了当时唯一的选择。

但这肯定不是长久之计，我们需要优化它。

---

### **优化第一阶段：优化Nginx代理为阿里云LB服务**

当时碰到的问题是：随着用户量的不断地增长，上报业务量越来越大，而因为使用了HTTPS协议，导致CPU是整个系统的瓶颈资源。

主要问题如下：

1. CPU资源低峰期浪费严重，高峰期是严重的瓶颈
2. 因阿里云CPU（核数）：内存（GB）比例最小为1:2，导致服务器内存浪费严重
3. ECS虚拟机实例扩容、缩容麻烦，需要关机-修改配置-开机，不能自动随负载弹性伸缩
4. 需要单独做监控、报警和日志收集等工具，如Prometheus、ELK等

其实，我们的前端nginx主要就是做两个事情：SSL卸载和流量转发，而这其实就是一个典型的负载均衡器做的事情。

为了解决以上问题，我们想到了利用阿里云提高的SLB（负载均衡器）服务。

不失所望，SLB服务**几乎完美地**解决了我们的问题：

![](/assets/SLB-property.png)

通过对比我们的nginx服务器单机（详细数据暂时无法公开）和SLB服务的最大连接数、CPS、QPS的限制及相关费用，得出至少可以节省三分之二的费用。

具体配置SLB过程在此就不赘述了，为了提高效率和保证质量，我们是通过python API来实现自动部署SLB的。

最终，我们通过16个SLB，替换了目前24个自建的ECS nginx代理服务做转发。

以下是使用了SLB之后，核算出的大致节省的费用：

> 【费用节省情况更新】
>
> 以前是使用25个nginx代理，费用：2338.70 \* 25 = 58467.5；退24个，有一个需要保留。
>
> 目前是16个LB，费用：566.33 \* 30 = 16989.9
>
> 节省9个EIP，费用：14.4 \* 9 = 129.6
>
> 此次预计总共节省费用约：58467.5 - 2338.7 - 16989.9 + 129.6 = 39268.5 rmb/月

是的，大家没有看错，简单做一下优化，每个月就节省了将**近四万人民币**的成本。

不光是成本，还获得了以下的好处：

1. 减轻运维成本，自带了日志搜集和分析工具，能满足日常的分析需求。
2. 随业务负载自动扩容、缩容，自带跨机房级别的高可用。
3. 业务量较优化前上涨10%，分析后认为主要是去除了系统CPU瓶颈导致客户端成功率上升。

以下是自带的日志工具：

![](/assets/slb-sls-search.png)

也可以做固定的图表展示模板：

![](/assets/SLB-sls-dashboard.png)

但是，世界上没有完美的东西，太完美的东西都不太现实，SLB也带来了一个问题：

> SLB比我们自己nginx唯一不好的一点是：延迟会稍微大一点。我们的平均延迟在50ms左右。
>
> 到SLB上，需要80-120ms了。
>
> 目前就这个比我们自建的nginx差一点。

分析：可能的原因是SLB是共享服务，多个SLB实例在阿里云那边其实是共享同一台服务器的，所以可能存在CPU/网络抢占和排队的现象，导致整个处理的延迟增加。

好在我们这个业务的特点，对于增加的这点延迟不敏感。

**优化第一阶段总结：**

1. 业务迁移上云，不是把机器搬上云主机就算上云了，还要考虑充分利用云提供的服务。
2. 了解我们的供应商及其产品特点和最佳实践，非常重要。

---

### **优化第二阶段：优化带宽**

业务当前的峰值总带宽约为12G，每天请求次数约150亿次，而且发现一个奇怪的问题：“流出带宽是流入带宽的2.5倍左右”，而业务逻辑是“数据上报”，正常状态下应该“流入大于流出才对”。

这是一个值得深入研究的问题。

#### 抓包分析

每次请求的量大致如下：

> in：250 + 126 + 627  = 1003 bytes
>
> out：1440 + 1440 + 416 + 258 + 270 + 34 = 3858 bytes

这也解释了为什么从监控上看到“流出带宽大于流入带宽”:

![](/assets/vpn-bw.png)

#### 技术原理分析

熟悉HTTPS：

![](/assets/HTTPS-communication.png)

通过分析，应用的特点是每次上报都需要和服务器建立一次连接，而由于是HTTPS的协议，导致整个流量大部分都被SSL握手和证书传递占用，真实的payload占用的流量比例很低（通过tcpdump抓包分析，不到20%）：![](/assets/tcpdump.png)

优化的方向明确了：如何既满足业务的加密的需求，又降低每次请求的流量！

顺着这个思路，我想到的是：加密肯定有算法和证书，如果把加密算法的复杂度和证书长度降低一点，那么肯定能减少带宽量和CPU使用率，只要能满足业务的最低加密需求即可。

经过一番查找和熟悉，发现目前HTTPS，证书有ECC格式，不旦证书小，而且CPU计算量会下降，还不降低加密强度，简直完美：

![](/assets/ECC-rsa-certificate.png)

关于ECC（椭圆曲线）和RSA，请参考：[https://www.namecheap.com/support/knowledgebase/article.aspx/9503/38/what-is-an-ecc-elliptic-curve-cryptography-certificate](https://www.namecheap.com/support/knowledgebase/article.aspx/9503/38/what-is-an-ecc-elliptic-curve-cryptography-certificate)

目前，很多知名网站已经开始使用ECC证书了，比如google,cloudflare等

![](/assets/google.png)

![](/assets/cloudflare-ecc.png)

#### 

#### 对比验证

马上行动，通过Let's Encrypt申请一个免费的ECC证书，和当前的RSA证书做个对比：  
![](/assets/let-encrypt.png)

看看我们刚刚申请的ECC证书：

![](/assets/ecc-cert-1.png)

对比一下原来的RSA证书：

![](/assets/rsa-certificate.png)

使用两台nginx服务器，一台使用ECC证书，另外一台使用RSA证书。

为了对比能公正，我们首先要确保两台nginx请求量基本一致：

![](/assets/delay.png)

![](/assets/PV.png)

![](/assets/UV.png)

![](/assets/NGX_status.png)

运行16个小时后，可以看到ECC优势明显：

![](/assets/nginx-ecc.png)![](/assets/nginx-default-rsa.png)

运行七天之后统计：

![](/assets/RSA-ECC-7day.png)

CPU使用率从超过50%下降到约25%，带宽下降到原来的88%左右。

整理对比效果如下表：

|  | 证书长度 | 日均请求量 | CPU利用率峰值 | 带宽日峰值 | 日均流量 | 平均请求延迟 | Server Hello阶段最大字节数 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| ECC | 256bits | 52.044 M | 30% | 625.68 Mbps | 8.6618 GB | 207.52 ms | 2668 bytes |
| RSA | 2048bits | 51.772 M | 60% | 720.98 Mbps | 9.9274 GB | 204.29 ms | 3375 bytes |

ECC和RSA证书访问，使用tcpdump抓包对比，可以看到使用ECC证书后，每个连接服务器对外流量少了707bytes：

![](/assets/tcpdump-rsa.png)

![](/assets/tcpdump-ecc.png)

#### 确保兼容性

当然，有个最大的问题就是**服务端需要兼容ECC和RSA两种证书**，万一客户端不支持ECC证书，也需要回落到RSA，避免拒绝客户的连接请求。还好，目前nginx配置可以兼容两种证书，而且可以优先使用ECC：

> ssl\_protocols SSLv2 TLSv1 TLSv1.1 TLSv1.2;
>
> \#优先支持ECC证书加密套件配置
>
> ssl\_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK';
>
> ssl\_prefer\_server\_ciphers on;
>
> ssl\_session\_timeout 60m;

![](/assets/nginx-ecc-rsa-conf.png)

查看access日志可以看到客户端请求，使用ECDHE-ECDSA-开头的ssl\_cipher就是使用了ECC证书的了：

![](/assets/ECDHE-ECDSA.png)

同样，Haproxy也支持配置ECC和RSA双证书的，参考：[https://cbonte.github.io/haproxy-dconv/1.7/configuration.html\#5.1-crt](https://cbonte.github.io/haproxy-dconv/1.7/configuration.html#5.1-crt)

---

### 第三阶段：上报业务从HTTP/HTTPS修改为TCP协议

自己实现加解密，彻底避免SSL证书和HTTPS协议交互负担

---

### 项目总结

* 把公司的钱当成自己的钱，就会想方设法去降低成本。
* 有清晰的思路指导之后，问题就能顺利解决。
* 对使用的技术和工具要多熟悉，比如nginx、SSL等，优化是建立在熟悉的基础上的。
* 熟悉业务特征是优化的必要前提。
* 第一阶段花了三个星期，不能说白做了，但如果能直接想到第二阶段，那么效率更高；行动之前可以思考得更深入和全面一些，有助于整体的工作提高效率。



