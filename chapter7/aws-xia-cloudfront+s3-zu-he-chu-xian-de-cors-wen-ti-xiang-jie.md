---
description: 关键字：CORS，simple request
---

# AWS下CloudFront+S3组合出现的CORS问题详解

时间：2023年11月至2024年5月

现象：偶尔（几天或者几个星期才有一个）有用户反馈，页面会出现CORS报错，无法正常访问，把文件修改链接后在CloudFront上重新发布（会更新URL路径）或者失效对应的旧文件URL后，报错现象消失，恢复正常。同一时期内，对应使用其他CDN厂商的区域（如中国大陆使用了腾讯云CDN+COS对象存储），未出现类似的情况。

CloudFront相关配置（有问题的配置）：

<figure><img src="../.gitbook/assets/image (2) (1).png" alt=""><figcaption><p>CloudFront上“行为”相关配置</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption><p>CloudFront上“行为”相关配置-详情</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption><p>缓存策略</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>源请求策略</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>响应标头策略</p></figcaption></figure>

源站（S3）CORS相关配置：

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption><p>S3上CORS相关配置</p></figcaption></figure>

客户端出现的CORS报错信息：

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption><p>Chrome客户端出现的CORS错误</p></figcaption></figure>

排查与优化：经过很长一段时间的搜索、查看文档、测试、咨询国内AWS销售代表介绍的技术人员等方式，始终没有解决问题。

在一次与AWS代理商及技术人员沟通中得知，国内AWS的技术人员实际上是没有权限查看后台日志的，一定要开通技术支持服务，通过ticket，这样远在美国的技术团队可以查看到具体的日志。最终决定：开通AWS的Business级别的技术支持，提ticket进行咨询。

果然，AWS的技术支持服务还是很棒的，经过ticket回复、电话详细沟通、开通CloudFront日志等操作之后，我们按照要求提供了非常详细的请求和对应返回的Header信息，在第二个工作日，对方找到了根本原因并给出了对应的优化方案。

{% code overflow="wrap" lineNumbers="true" fullWidth="true" %}
````
• 为什么您复现时，有带上跨域(CORS)请求还是没有在回覆里看到对应的标头？

由于您当前 CloudFront 配置的是 Simple Request[1]，因此该 SimpleCORS 策略并不会处理非 Simple Request 的请求，您可以参考[2]我在自己环境模拟的结果，您可以观察到差别主要是您复现的过程中加入了以下两个标头，让请求变成了非 Simple Request。
```
Pragma: no-cache
Cache-Control: no-cache
```

• 但为什么在 SimpleCORS 策略没有运作的情况下，有些用户还是看到的正常跨域的回覆呢？

因为该跨域的回覆是来自您的源站 S3 储存桶，而不是 SimpleCORS 策略添加的。在您 CloudFront 的缓存策略中，并没有将 CORS 相关的标头加入成为缓存键值，这个行为衍生的影响是当任一边缘站点在没有缓存的情况下，处理到第一次请求时如果该请求有带上 origin 标头，边缘站点在回源到 S3 时，将会取得来自您 S3 回覆的 `access-control-allow-origin: *` 标头，且后续命中相同缓存的请求就算没有带上 origin 标头，还是能取得相同的缓存版本。

相对地，如果边缘站点处理到的第一个请求没有  origin 标头，S3 就不会返回  `access-control-allow-origin: *` 标头，后续命中相同缓存的请求就算带上了 origin 标头也不会接收到预期的跨域回覆。

• 如何改善？

有两个方案提供给您参考：
1. 根据官方博客[3]的建议，将缓存策略调整为自定义的版本，并将 Access-Control-Request-Headers, Access-Control-Request-Method 与 Origin 这三的标头加入策略中，此外您就不用特定配置 Origin Request Policy。
2. 使用非 SimpleCORS 的回覆标头策略(Response headers policy)，例如您可以创建一个符合您使用情境的自定义策略，或者是使用像是其他默认且没包含 Simple request 场井的策略(例如：CORS-With-Preflight)。

希望以上的信息能帮助到您，若有任何我说明不清楚，或是您需要进一步讨论的地方，请您务必要让我知道，谢谢您。

[1] https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests 
[2] curl 测试
# Simple Request
% curl -D - -sSo /dev/null https://xxxx.cloudfront.net/yyyyy  --resolve "*:443:xxx.xxx.xxx.xxx" -H 'origin: *'
HTTP/2 200 
content-type: application/x-mpegurl
content-length: 69637
date: xxxxxxxxx 10:54:27 GMT
last-modified: xxxxxxxx 20:11:56 GMT
etag: "exxxxxxxxxxxxxxxx5"
x-amz-server-side-encryption: AES256
accept-ranges: bytes
server: AmazonS3
vary: Accept-Encoding
x-cache: Hit from cloudfront
via: 1.1 fxxxxxxxxxxxxxxxxxxc.cloudfront.net (CloudFront)
x-amz-cf-pop: HKG62-C2
x-amz-cf-id: Ixxxxxxxxxxqxxxxw==
age: 2921
access-control-allow-origin: *

# 非 Simple Request
% curl -D - -sSo /dev/null https://xxxxxx.cloudfront.net/yyyyyy8  --resolve "*:443:xxx.xxx.xxx.xxx" -H 'origin: *' -H 'cache-control: no-cache'
HTTP/2 200 
content-type: application/x-mpegurl
content-length: 69637
date: xxxxxxxx 10:54:27 GMT
last-modified: xxxxxxxxx 20:11:56 GMT
etag: "exxxxxxxxx5"
x-amz-server-side-encryption: AES256
accept-ranges: bytes
server: AmazonS3
vary: Accept-Encoding
x-cache: Hit from cloudfront
via: 1.1 0xxxxxxxxxx0.cloudfront.net (CloudFront)
x-amz-cf-pop: HKG62-C2
x-amz-cf-id: -rxxxxxxxxxg==
age: 2926

[3] https://repost.aws/knowledge-center/no-access-control-allow-origin-error 
````
{% endcode %}

简单概括：

首先，确保CloudFront的边缘节点目前都没有缓存相关文件。

场景一：边缘节点接受到的第一个请求**带了**“origin”请求头，所以S3会返回“Access-Control-Allow-Origin: \*”头，那么一切都没问题，因为CloudFront会将相关的“Access-Control-Allow-Origin: \*”头都缓存起来；后续所有的请求，即便是没有带上"origin"请求头，也都会由CloudFront从缓存中发送带有“Access-Control-Allow-Origin: \*”头的相关Response给客户端，客户端自然不会出现CORS报错。

场景二：边缘节点接受到的第一个请求**没有带**“origin”请求头，则S3不会返回“Access-Control-Allow-Origin: \*”头，那么CloudFront边缘节点内就不会缓存有“Access-Control-Allow-Origin: \*”头，这时又有两种情况：

&#x20;   2.1、对于Simple Request，CloudFront配置的回覆标头策略(Response headers policy)（此处是Managed-SimpleCORS策略）一定会添加对应的Header返回给客户端，所以不会出现CORS错误。

&#x20;   2.2、对于非Simple Request，CloudFront配置的回覆标头策略不起作用，同时，由于CloudFront边缘节点缓存的信息中没有“Access-Control-Allow-Origin: \*”头，客户端接收到的Response里面自然不会有“Access-Control-Allow-Origin: \*”头，也就会报CORS错误了。

理解了上面的内容，也就很好地解释了为什么这个CORS只是偶尔出现，而且只能在某一区域复现。根据建议修改后的CloudFront下“行为”配置如下：

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption><p>CloudFront修改后的“行为”配置</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1) (1).png" alt=""><figcaption><p>新增的缓存策略</p></figcaption></figure>

经过这样配置之后，INVALIDATE相关资源URL后，至今再未发生过类似的情况。

参考资料一：[https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html)

参考资料二：[https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple\_requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple\_requests)

#### 学习到的东西

1、HTTP的东西还挺复杂的，以前都没仔细了解过，Simple request，Preflighted request，这些概念实际上是不知道的。所以：专业的事情还是需要有专业的人来做，看似简单的事情，内行能看到更多。

2、使用海外AWS服务，一定要购买“企业支持服务”，否则生产环境出问题会很难受。AWS在国内的销售和技术人员，实际上都是偏产品、功能、特性等层面的，可以给建议，可以提供咨询，但是真正能解决具体问题的还是提ticket。

这一点与国内云厂商的服务差别很大，国内云厂商，主打一个人际关系，通过找到具体的研发人员，可以直接帮忙定位和排查问题，只要是大客户，问问认识的销售啥的，帮忙找到研发人员帮忙，基本都不是事。而AWS在国内的技术人员甚至连后台的日志都无法查看到（从某AWS国内代理商的技术人员那里得到的信息，不一定完全准确），底层逻辑是按商业规则办事，交钱买服务，有专业的人在后面支撑。

3、AWS的产品设计，似乎有故意复杂化的可能性，不能像国内CDN厂商一样It Just Works。CloudFront自带的Managed Policies，特别是配置项中推荐的配置（那一句：Recommended for S3很容易迷惑人），并不能默认覆盖我的用户场景。而我们使用国内另外一家云厂商的CDN服务，几乎不需要做特别的配置，相关的Header默认就有，完全没出过问题。

一方面，把这些底层的技术能力暴露给用户，确实能够对那些技术能力很强的用户有帮助。另一方面，不排除故意把产品设计得过度复杂、难以入手，强迫用户购买“企业支持服务”。
