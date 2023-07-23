---
title: '数据库字段的not null意义何在'
key: AAA-2021-6-2-not null
tags: [MySQL]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
lightbox: true
---
## **1. 官网解释存在的问题**
[点此进入官网](https://dev.mysql.com/doc/refman/5.7/en/problems-with-null.html)
总结一下官网所给出的解释：
1. null和空字符串是两回事，查找为null的要使用 is null的方式；
2. 在使用一些聚合函数(count(), min(), sum() 等)时会忽略null的统计;
    Note: count(*)是计算行数，count(某个字段)是统计不为null的个数；
3. 将null插入自增的字段，该字段会显示null;
4. 对于MyISAM，InnoDB和MEMORY 这类存储引擎是支持插入null数据的；

## **2.的踩坑经验**
### 2.1 数据库字段为null有意义的情况
例如：使用number统计某店铺今天的出售情况。如果卖出一件，number加一，如果退款一件，number减一；那么如果number为0说明该店铺今天产生了交易，只是卖出的全部退款了；那么如果number为null，说明今天根本就没有产生交易；因此number在为null和0时是两种不同的意义，此时将数据库的字段设为null是有必要的。

### 2.2 数据库字段可为0却设置成null
1. 一般情况下，Po的对象都是包装类去接受数据类型的，如果此时在数据库中取出的数据为null，在赋值为基本类型时会报NPE, 因此每个字段要判空。如果设置成not null默认为0时从源头解决了此问题，大胆赋值无需判空；
2. 使用not null提升读取性能并且减小存储空间，因为null的空间大小是大于0的。
