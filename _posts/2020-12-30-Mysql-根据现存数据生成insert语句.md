---
title: Mysql:根据现存数据生成insert语句
key: AAA-2021-03-08-jsonSerialize
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

## **1. 谈谈需求**
在开发的过程中将两个项目(A与B)公用的几张表，该表中的某些数据是针对项目A的，另外一些数据是用于项目B的。为了逐渐将两个项目独立处理，拆分表和缓存是刻不容缓。

表拆分：通过select语句查找出项目B的数据，将这些数据生成insert语句。

## **2.拆分详解**
### table_user
主要是通过concat方法将其拼接成insert语句，具体脚本如下：

```sql
select concat("insert into table_user(user_id, name, email,
 created_time, modified_time) value(", "'", user_id, "',", quote(name), ",", quote(email), ",'", created_time, "','", modified_time, "');") as result
from (select user_id, name, if(ifnull(email,'')='', '',email) as email, created_time, modified_time  from table_user where user_id > 'fdfdasdfa') as table;
```
将上述的脚本放进一个sql文件（execute.sql），在项目A的数据库中执行上述脚本文件，由于在执行上述脚本的时候会产生一些数据因此需要将其输出的数据指定到某一个文件（data.sql）， 再执行data.sql文件插入数据到项目B数据库中。

<font color = 'red'>获取数据</font>
<font color = 'blue'>mysql --raw -h 项目A数据库链接 -N -u用户名 -p密码 -D表名< 指定文件路径(g:bbb/execute.sql) > 输出数据路径(g:aaa/data.sql)</font>
<font color = 'red'>插入数据</font>
<font color = 'blue'>mysql -h 项目B数据库链接 -N -u用户名 -p密码 -D表名< 指定文件路径(g:/aaa/data.sql)</font>

## **3.踩坑指南**
1. 数据库中的字段不能存在字段为null，一旦有某个字段为null，则拼接的insert语句为null,这样生成的脚本文件是不能执行成功的。
   <font color = 'red'>解决办法：</font>将通过select查找的结果进行if判断处理，加上`if(ifnull(email, "")="","",emial) as email` 该目的是为了将结果为null的默认转成""(空字符串)。
2. insert语句中的value数据来自select查找结果，将该数据两边加上双引号表示他是字符串，不然插入会报错。
3. 由于使用select脚本生成文件的时候会带上column name. 红框中为column name, 这同样导致生成的data.sql脚本文件运行失败。

![image-20201230203434007]({{"/assets/picture/image-20201230203434007.png" | absolute_url}})
<font color = 'red'>解决办法：</font>在运行生成data.sql的脚本文件时，在该命令上加`-N`即可，如上获取数据所示。
4. 对于用户的输入内容需要使用quote函数转义。例如：name, email。 由于用户输入的数据不可控，可能存在特殊字符串 `` ' ``，这样就导致了本来整体的一个字段就分裂成两个了。
5. 使用concat拼接字符串的时候注意，拼接的时候使用双引号，然后value中的数据使用单引号。虽然在做insert插入语句时，value中的数据单引号和双引号都可以，但是quote函数是不对双引号转义，这是如果insert语句中继续使用双引号是包不住带有双引号的数据的，这是插入的时候就会报错。
6. 在生成insert的sql文本时，命令行请带上--raw。由于在转义的过程中会产生 `` \ ``，如果生成文本的命令行中不加--raw就会导致 ``\``被转义成`` \\ ``，其他的反而不转义。
