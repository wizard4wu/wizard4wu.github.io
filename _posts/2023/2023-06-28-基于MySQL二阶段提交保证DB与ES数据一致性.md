---
title: '方案设计|基于MySQL二阶段提交保证DB与ES数据一致性'
key: key-2023-6-28-DBandESDataConsistent
tags: ["MySQL","方案设计"]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
---

### 1. MySQL二阶段提交

二阶段的初衷是为了保证MySQL在发生宕机时对数据一致性的保证，主要是对redo log和bin log的两阶段的提交。

![image-20230628070858575]({{"/assets/picture/2023-06/image-20230628070858575.png" | absolute_url}})

(图片来源：小林coding)

二阶段: 在开启bin log的前提下，为了保证bin log和redo log的一致性，MySQL内部会维护XA事务。该事务主要由bin log作为协调者，redo log是参与者。 XA事务内部会维护一个事务ID（XID）

1. prepare阶段：将XID写入redo  log并且，将该log标记为prepare状态，同时将redo log写入磁盘（前提设置每次事务执行持久化）
2. commit阶段：将XID写入bin log，然后将bin log写入磁盘，接着调用存储引擎将redo log状态设置为commit。

整个过程只要bin log中写入了XID即可，是否将redo log为commit的状态写入磁盘显得没那么重要了，在恢复数据时先根据redo状态判断，再根据XID在bin log中是否存在来判断恢复对应的数据。接下来看看异常的情况：

1. 写入redo log，但是没有没有写入bin log：这时redo log中有XID，状态为prepare，bin log中没有XID。在重启恢复数据时发现状态为prepare，检查XID在bin log中是否存在，不存在就会回滚该事务；
2. 写入redo log和bin log但是修改redo log的状态失败：这时redo log和bin log都是有XID的，这时在重启时发现redo log为prepare状态，再去bin log中找到XID，认为该事务有效；

实际上事务执行的成功与否是以bin log是否成功写入到磁盘为标志，commit状态只是为了更方便判断。



如果不适用二阶段提交会发生什么？

redo log用于宕机恢复数据，bin log用于主从复制。

1.先提交redo log，再提交bin log：

redo log提交成功，bin log提交失败。主库重新启动适用redo log 恢复事务成功，但是对于从库没有bin log压根不存在该事务，所以存在数据偏差。

2.先提交bin log在提交redo log：

bin log提交成功，redo log 提交失败。主库重启无法使用redo log恢复事务，但是对于从库可以使用bin log执行该事务，主从数据产生偏差。

### 2.DB与ES数据一致性

此处主要是基于一主多从的DB架构，对于分库分表的架构需要更深入的讨论。

在项目中保持数据一致性是很多开发者所面临的一个问题，接下来我会由浅入深和大家讲讲如何将MySQL二阶段提交应用到DB与ES数据一致性。

#### 2.1 基本方案

主要分成两部：1.执行sql语句写入数据；2.写入数据到ES侧，要保证写入失败抛出异常即可(可以考虑通过OpenFeign调用)；

使用事务注解：执行sql语句再调用API写入数据到ES侧

1. 执行sql语句失败，直接回滚；

2. 执行sql语句成功，调用API失败抛异常，回滚数据；

上述整个方案很简单，基本上能够保证两侧数据的一致性，但是对于网络抖动产生的超时问题就不能保证。因为当在调用API超时，该事务不确定该数据有没有成功送到ES侧，因此事务根据超时异常不确定要不要回滚数据。

#### 2.2基于MySQL二阶段提交方案

分成三步去完成整个提交流程：



<font color="purple">步骤1.执行sql语句并将该条数据标记成prepare状态；</font>


<font color="purple">步骤2.调用API写入数据到ES侧；</font>


<font color="purple">步骤3.将prepare状态修改成commit状态；</font>
对于获取该数据：

1.可以直接获取ES数据；

2.获取DB时要判断是否为commit状态，如果是直接返回；如果为prepare状态则说明上次调用存在异常，则去获取ES侧数据，同时将ES侧数据更新到DB，并将状态设置成commit状态，并将数据返回。

所以上游调用完全不能感知数据的不一致性。

二阶段提交方案主要用于解决网络抖动的超时问题，所以对于其他case事务注解基本能够完全覆盖。注意一点：对于步骤3的话要对其异常进行捕获，不要因为步骤3产生的异常影响到步骤1。

#### 2.3案例分析
+ 步骤1出错直接回滚

+ 步骤2出错(非超时)直接回滚;

+ 步骤2出错(超时错误)不回滚，并且根据是超时错误选择不提交修改状态，此时还是处于prepare状态;

+ 步骤3出错不回滚，对步骤3进行异常捕获不要抛出任何异常。目的就是保证它处于prepare状态下次从ES侧取数据;

### 3总结
本文主要介绍了通过现有的二阶段提交方案应用到ES和DB两端的数据一致性。
