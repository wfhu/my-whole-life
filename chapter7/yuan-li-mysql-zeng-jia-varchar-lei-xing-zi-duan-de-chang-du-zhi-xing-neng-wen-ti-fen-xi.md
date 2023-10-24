# 【原理】MySQL增加varchar类型字段的长度之性能问题分析

## 问题说明：

时间：2023年10月

数据库版本类型：MySQL 5.7.32

背景：某业务表有超过15.8亿行数据，表大小约400GB，索引大小约111GB，某一个字段需要从varchar(15)类型修改成varchar(64)类型，我们查找资料并简单分析，认为理论上不需要修改已有的数据字段，只需要修改表结构即可，只会影响后续新插入的数据。但实际运行时出乎意料，修改表结构花了7个多小时才完成。

修改表结构的SQL语句如下：

```sql
ALTER TABLE my_database.t_my_table MODIFY phone varchar(64),ALGORITHM=INPLACE, LOCK=NONE;
```

旧表的表结构如下：

```sql
CREATE TABLE `t_my_table` (
  `id` bigint(20) NOT NULL COMMENT '主键',
  `phone` varchar(15) DEFAULT NULL COMMENT '手机号',
  `app_key` varchar(32) DEFAULT NULL COMMENT '应用key',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `create_time_index` (`create_time`) USING BTREE,
  KEY `app_key_time_index` (`app_key`,`create_time`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 原理分析：

因为_varchar_字段的存储，是由1个字节或者2个字节的长度前缀（length prefix）+真实数据格式进行存储的，当真实数据小于或等于255个字节时，长度前缀为1个字节，当真实数据超过255字节时，长度前缀变成2个字节。

因为我们的字符集是utf8mb4，意味着varchar(64)需要占用的真实数据长度为64\*4=256，超过了255，所以长度前缀需要从1个字节变成2个字节，所以上述ALTER TABLE的语句需要修改表中每一行的历史数据，所以时间会很长。

参考链接一：[https://dev.mysql.com/doc/refman/5.7/en/char.html](https://dev.mysql.com/doc/refman/5.7/en/char.html)

> In contrast to `CHAR`, `VARCHAR` values are stored as a 1-byte or 2-byte length prefix plus data. The length prefix indicates the number of bytes in the value. A column uses one length byte if values require no more than 255 bytes, two length bytes if values may require more than 255 bytes.

参考链接二：[https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html#data-types-storage-reqs-innodb](https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html#data-types-storage-reqs-innodb)

> To calculate the number of bytes used to store a particular [`CHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html), [`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html), or [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html) column value, you must take into account the character set used for that column and whether the value contains multibyte characters. In particular, when using a `utf8` Unicode character set, you must keep in mind that not all characters use the same number of bytes. `utf8mb3` and `utf8mb4` character sets can require up to three and four bytes per character, respectively. For a breakdown of the storage used for different categories of `utf8mb3` or `utf8mb4` characters, see [Section 10.9, “Unicode Support”](https://dev.mysql.com/doc/refman/5.7/en/charset-unicode.html).

## 进一步验证我们的想法

基于以上分析，如果我只是把phone字段的类别从varchar(15)修改到varchar(63)，因63 \* 4 = 254，所占用空间小于255个字节，理论上不需要动已有的数据，所以应该很快结束。

SQL如下：

```sql
ALTER TABLE my_database.t_my_table MODIFY phone varchar(63),ALGORITHM=INPLACE, LOCK=NONE;
```

运行截图如下，显示仅用了10ms就完成了表结构的修改：

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption><p>alter column length to 63</p></figcaption></figure>

可以看到，与我们预想的一致：不需要修改已有数据的情况下，速度非常快。

### 其他验证

如果我们再次将phone字段从varchar(63)修改成varchar(20)，理论上也应该会非常快

SQL如下：

```sql
ALTER TABLE my_database.t_my_table MODIFY phone varchar(20),ALGORITHM=INPLACE, LOCK=NONE;
```

运行截图如下，显示以下报错：

```sql
【执行SQL：(1)】
ALTER TABLE my_database.t_my_table MODIFY phone varchar(20),ALGORITHM=INPLACE, LOCK=NONE
执行失败，失败原因：(conn=547542) ALGORITHM=INPLACE is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

## 反思

第一：对于varchar的存储方式其实存在一知半解的情况，对细节的把握不是很清晰。

第二：对于varchar和字符集（CHARSET）的关系没有理解到位，一开始存在认为varchar(64)就是只占64个字节的错误。

第三：对于online DDL和修改表结构时间长短这二者的关系，存在概念上的混淆；实际上online DDL只是表明不会阻塞其他的读/写操作，并不代表online DDL就可以很快速的完成。
