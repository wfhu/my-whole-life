---
description: 关键字：CORS，simple request
---

# AWS下CloudFront+S3组合出现的CORS问题详解

时间：2023年11月至2024年5月

现象：偶尔（几天或者几个星期才有一个）有用户反馈，页面会出现CORS报错，无法正常访问，把文件修改链接后在CloudFront上重新发布或者失效对应的文件后，报错现象消失，恢复正常。同一时期内，对应使用其他CDN厂商的区域（如中国大陆使用了腾讯云CDN+COS对象存储），未出现类似的情况。

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption><p>CloudFront上“行为”相关配置</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>CloudFront上“行为”相关配置-详情</p></figcaption></figure>
