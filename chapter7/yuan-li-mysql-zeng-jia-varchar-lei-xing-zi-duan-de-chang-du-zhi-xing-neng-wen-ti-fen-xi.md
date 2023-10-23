# 【原理】MySQL增加varchar类型字段的长度之性能问题分析

## 问题说明：

时间：2023年10月

数据库版本类型：MySQL 5.7.32

背景：某业务表有超过16亿行数据，某一个字段需要从varchar(15)类型修改成varchar(64)类型，我们查找资料并简单分析，认为理论上不需要修改已有的数据字段，只需要修改表结构即可，只会影响后续新插入的数据。但实际运行时出乎意料，修改表结构花了7个多小时才完成。

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

## 进一步验证

