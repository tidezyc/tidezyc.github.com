---
layout: post
title: mysql连接嵌套查询
category : database
tags:
- J2EE
- mysql
- 嵌套查询
- 链接查询
published: true
---
在很多时候我们会用到全国城市查询，最简单的实现就是:

``` sql
	CREATE TABLE `city` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  `parent` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

加入我们有如下数据：

1  浙江

2 杭州 1

3 嘉兴 1

4 江干 2

我希望查询来的数据是：

1 浙江

2 杭州 浙江

3 嘉兴 浙江

4 江干 杭州

这时如果用hibernate将直接产品嵌套对象，如果自己用jdbc实现，就可以用连接查询得到：

``` sql
select A.id,A.name,B.name as parent from city A left join city B on A.parent=B.id limit 0,10;
```
