# 【原理】MySQL增加varchar类型字段的长度之性能问题分析

时间：2023年10月

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
