#### 【阅读笔记】Redis Cluster Specification

---

**原文地址**：[https://redis.io/topics/cluster-spec](https://redis.io/topics/cluster-spec)

**阅读时间**：2019年8月

**重点内容摘要**：

1000个节点的情况下性能可以线性增长。

整个集群中间是没有任何代理的，而且数据的复制是异步的。

只能使用database 0，不支持SELECT指令。

节点之间通过Redis Cluster Bus来通讯。

Redis Cluster使用last failover wins，所以可能会丢失写入数据。

整体来讲Redis Cluster选择了CAP里面的CP，它不会允许持续的、大量的丢失数据的情况发生；当集群分裂时，少数节点部分（minority side）超过NODE\_TIMEOUT的时长后会拒绝接受写入请求。

通过使用replicas migration特性，可以提升整个集群的可用性，因为它可以调度replica给没有replicas的master节点，从而提升整个集群的抗FAIL的能力。

整个Cluster的key space被分割成16384个slot。

通过使用hash tags来确保把某些key放到同一个slot上（进而放到同一个实例上），主要是方便进行multi-key的操作。

hash tags通过{...}的内容来做为计算slot的内容。

每个node有一个全局且永久唯一的node id，即使机器的IP和端口变了也不会有影响。

Cluster Node之间通讯的端口：在普通客户端口上增加10000；是二进制协议，主要是为了提高性能。

通过gossip协议以及一个可配置的update方式，避免节点之间的消息过多，确保内部通讯消息数量不会随着节点数增加而呈指数级增长。

管理员通过MEET指令，可以将某一个node加入集群中，详细可参考：[https://redis.io/commands/cluster-meet](https://redis.io/commands/cluster-meet)

通过CLUSTER SETSLOT指令（针对某个slot里面的全部key，会下发MIGRATE指令），可以将某个slot和node关联起来。

MOVED指令：客户端一般建议更新slot-&gt;node的map信息；但是ASK指令，客户端只是本次查询去查询新的node，接下来针对这个slot的请求还是往老的节点上去请求，主要目的就是满足slot迁移过程中某一个key可能在旧的也可能在新的node上的情况。

聪明和高效的客户端，一般都需要在内存中记录slot-&gt;node的信息；在启动以及受到MOVED指令时，需要全局更新（有一个专门的、高效的、不需要解析的指令：CLUSTER SLOTS）。

multi-key的操作，如果key不存在或者在resharding过程中在不同节点上，会收到-TRYAGAIN的错误。

默认情况下，Redis Cluster下的slave节点是不允许读（当然也不允许写），默认请求会被redirect到它的master节点上去；通过READONLY指令，可以允许replica节点处理读请求（因为数据是异步同步的，所以可能会拿到过期的数据）。

每个节点在NODE\_TIMEOUT的一半时间内，需要确保和所有的其他节点都PING/PONG通讯一次。

失效检测机制：

* PFAIL表示本机对另外节点的检测
* FAIL标识会最终扩散到集群其他节点

slave在master是FAIL状态时，会启动elect和promotion机制，但是如果NODE\_TIMEOUT \* 2（最少2秒）时间内没收到大多数（majority）节点的响应，会重新发起一次，但是后面的超时时间会翻倍。

使用FAIL机制，而不是直接让slave做promotion来代替master，主要是防止slave认为master有问题但是实际上其他节点连接master没有问题。

如果Cluster集群的所有节点（包括主从）都互相断开了，理论上是不会产生选举的，因为slave无法得到其他master的响应。

currentEpoch和configEpoch就是一个64位的无符号整形数字，节点创建的时候是0；节点接收到的Epoch大于本地的就会更新；它们用来解决不同节点之间的配置冲突（比如网络分裂以及节点失效的情况下，通过Epoch的数据来确认哪个节点的配置是最新的）。

redis Cluster内如果使用PUB/SUB，也是可以用的，但是目前是会把消息广播给整个集群所有节点，所以效率肯定不高。

