### 【优化】nginx启用reuseport

---

时间：2018年12月-2019年1月

关键字：SO\_REUSEPORT，port sharding，nginx，性能优化，Linux内核，socket

重点概括：

* nginx开启SO\_REUSEPORT参数，优化多线程、高并发场景下的性能表现

---

通过查看[nginx性能优化的文档](https://www.nginx.com/blog/performance-tuning-tips-tricks/)，看到一个点：启用reuseport，nginx在[1.9.1版本](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/)的时候已经加入了该支持，当然，也需要有合适的[Linux内核版本才能支持](https://lwn.net/Articles/542629/)。

尝试在一个nginx上启用了这一配置参数，前后对比，发现以下几个方面优化效果明显：

* CPU负载从30下降到8
* context switch从60K下降到40k
* cpu usage下降
* nginx服务平均延迟下降\(\)
* nginx慢请求数量下降

---

修改参数如下，只需要在listen 端口后面加上 reuseport即可 ：

```
server {
        listen 80 reuseport;
        listen 443 ssl http2 reuseport;
        server_name   xxx.xxx.com;
        charset utf-8;
```

_注意：同一个nginx实例下针对同一个IP+端口，只需要其中一个设置了reuseport即可全部生效。_

修改后可以查看listen的socket数量的变化，未使用reuseport时，一个port有一个socket的：

```bash
[root@xxx-xxx228 vhosts]# ss -lnt | grep ":443"
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     511    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     469    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     510    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     506    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
LISTEN     479    511          *:443                      *:*
LISTEN     512    511          *:443                      *:*
[root@xxx-xxx228 vhosts]# ss -lnt | grep ":443" | wc -l
31
```

---

修改该参数前后对比如下：

![](/assets/cpu-reuseport.png)

![](/assets/load-reuseport-import.png)

![](/assets/nginx-3s-trend-import.png)

![](/assets/nginx-time-reuseport-import.png)

![](/assets/sock-mem-reuseport-import.png)

![](/assets/context-reuseport-import.png)

关于SO\_REUSERPORT的内容，详细的可以参考 [https://lwn.net/Articles/542629/](https://lwn.net/Articles/542629/) 和 [https://n0p.me/portsharding/](https://n0p.me/portsharding/)

---

\*注意\*，启用REUSEPORT也可能导致一些意想不到的问题，比如单个请求处理的最大延迟可能会增加，可以参考 [https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/](https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/)

