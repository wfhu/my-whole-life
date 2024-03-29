# ∞【列表】非技术重点工作日志

## 物理机房与阿里云机房费用对比

时间：2019年7月

拿物理机房当前的价格和阿里云对比，发现阿里云能节省20%以上的费用（数十万绝对金额）；拿到这个费用去和各个物理机房谈判，分分钟都愿意降价，最终下降的幅度比迁移到阿里云还能更节省。最终决定将业务留在物理机房了。

另外，从计算价格的过程来看，技术和财务的思考方式确实不一样。技术更多的是直来直去，首先考虑的是技术上替换的可行性和运维管理的方便性；而财务则从费用角度出发，主要关注整体费用和单价、短期费用和长期趋势产生的费用等等。

技术和财务两边掌握的信息充分同步之后，最终往往能得出比原来单方面想法更优的解决方案。

## 某云厂商的故障沟通

时间：2019年7月

直接表达清楚自己的意思，比遮遮掩掩要强，既能体现自己的诚实可信，也可以避免大家误判。比如这次未告知的情况下直接封IP的事件，如果能提前直接告诉我们扛不住这么大的压力，那么我们可以提前做好准备工作，把相关业务模块迁移走或者降低权重，就可以避免对厂商的其他客户造成影响，同时我们自己的业务也不会突然被封锁掉。就是因为销售人员的遮遮掩掩，导致两方面都得到了最不好的结果。

故障之后的沟通，按照常理来讲应该是道歉、原因说明、后续的补偿方案以及避免再次发生类似事件的办法，而不是直接想着挽留用户继续留在自己的平台。用户为什么要留在你这个平台呢？如果出发点都是从自己的角度，想着要维持当前的消费额度，那么用户自然会流失的。

大致的内容：

> 我为了你们公司继续使用我们的产品付出了很多，和公司领导说了很多好话，承受了很多压力。
>
> 我希望你们能继续使用我们的产品，因为你们为我们团队带来了不错的销售额度。
>
> 你到时候能和我们的领导一起聊聊么？可以把你们的意见直接反馈给我们领导，这样我也能减少一点压力。

## 因AWS封锁API事件而引发的思考

时间：2019年9月

从2019年7月开始，我们把一部分海外的业务迁移到了AWS上，稳定运行了将近50天。自己非常的得意和喜悦，内心觉得还是AWS这种知名大厂很可靠，自己的选择没有错。

然而这几天的经历狠狠地给了自己一个耳光。

* 阿里云被攻击了，会把IP封掉，一般也就4小时，最长也就一天；实在不行找他们客服或者销售，总能帮助解决掉。
* 青云被攻击了，会把IP封掉，24小时候之后提工单处理即可重新开通；最后青云实在是扛不住攻击带来的影响，销售叫我们撤下了相关服务。
* AWS最厉害，不但封闭了受攻击的IP的访问，还把账号的操作权限给LOCK住了，导致整个账号完全无法使用。更令人感觉麻烦和无助的是：他们的销售不会帮助我们处理，唯一的途径是在后台提工单，响应时长按天计。当然，其中一个重要原因也是我们没有买他们的support plan，费用大概是消费额的10%。

带来的思考：

一、歧视其他云厂商。曾经一直鄙视阿里云、腾讯云等厂商，认为云厂商只有AWS才正宗，对AWS的技术十分崇拜，在使用上对AWS几乎是压倒性的推荐。这个思想从2014年起基本上就根深蒂固了。现在想想，这就是一种偏见，一种先入为主的偏见。实际上，适合自己的、能解决问题的就是对的。

二、文化的差异真的很大。阿里云、腾讯云的服务，不说很完美，但是都有；而AWS几乎没有任何服务，除非你给钱买服务。这一点体现出中西文化的巨大差异。



## 数据服务迁移上云的坑

时间：2022年年底-2023年上半年

* 招投标的流程要非常严谨，对公司的整体能力是一个比较大的考验；这个影响很大，没做好的话容易被人坑，特别是一些带有个人目的的，容易导致项目中途烂尾、项目拖延、浪费时间和人力等
* 各个云厂商提交的方案可能本身就有坑，需要专业能力排除掉，切不可盲目相信云厂商（如计算节点需要本地磁盘做shuffle的问题，方案中没考虑这部分磁盘，实施过程中就导致额外的成本上升），这块越在早期发现越有利，所以招投标的时候，前期准备阶段的投入要加大，避免把问题引入到实施阶段
* 迁移过程和方案，很可能会有较大的阻力，特别是某些容易被忽视或名字容易产生歧义的部门或团队，需要真正细化方案，模拟整个迁移过程（如“基础数据“”这个描述确实非常模糊，很容易双方团队的理解不一致）
* 预算的提交不应该过于乐观，切不可直接把最乐观的情况当成预算提交上去（默认认为云厂商的优化+我们自己的优化一定能按时、按量地达成，实际根本达不成），没有管理好老板们的预期，导致项目效果打折扣；最好是自己心里有一个最优的数据，但是提交一个相对保守的预算数字，为自己和团队留一些buffer空间









