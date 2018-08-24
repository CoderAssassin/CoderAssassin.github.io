---
layout:     post
title:      "分布式数据库原则"
subtitle:   " concurrent "
date:       2018-08-24 14:00:00
author:     "Aliyang"
header-img: "img/post-bg-concurrent-cap.jpg"
tags:
    - java并发
---
## 关系数据库事务
关系数据库需要满足ACID这4个特性：

* Atomicity 原子性：整个数据库事务是不可分割的工作单位。一个事务中的所有的操作必须要一次性完成，要么都完成要么都不完成。
* Consistency 一致性：指事务将数据库从一种状态转变为下一种一致的状态。事务的开始和结束的时候数据库的数据保持一致。
* Isolation 隔离性：或者叫并发控制、可串行化、锁等。事务的隔离性要求每个读写事务的对象对其他事务的操作对象能相互分离。
* Durability 持久性：事务一旦提交，其结果就是永久性的。

## 分布式数据库CAP原则
CAP定理是NOSQL数据库的基石，指的是在一个分布式系统中，Consistency(一致性)、Availability(可用性)、Partition tolerance(分区容错性)这三者只能满足其中两个，不能全部满足，这个结论是实践的总结，但是理论上，如果网络很好的情况下，三个特性可以同时满足。现实中的分布式系统因为网络的原因，很有可能会出现延迟或丢包等问题，因此必须要实现分区容忍性，一致性和可用性之间只能二选一。
对于传统的数据库来说，需要强一致性；而NoSQL系统一般注重性能和扩展性，而非强一致性。

* Consistency(一致性)：分布式数据系统中的所有的数据备份，在同一时刻值都相同。也就是同一时刻所有机器上的数据保持一致。
* Availability(可用性)：当集群中的一部分节点故障后，集群整体还能响应客户端的请求。也就是说，每个用户的请求都能收到正常的响应，并且响应时间在用户可接受范围内。
* Partition tolerance(分区容错性)：尽管节点之间网络通信丢失或延迟了任意数量的消息，但是整个系统仍然在正常运行。

可以有三种组合：

* CA：指的是单点集群。
* CP：舍弃了可用性，可用性指的是高性能，所以性能不是很高。
* AP：是弱一致性，一般的NoSQL数据库对一致性的要求不是很高。

## BASE
为了解决想要保证强一致性而导致可用性降低的问题而提出的，BASE是反ACID的，主要由如下三个术语组成：

* 基本可用（Basically Available）
* 软状态（Soft state）
* 最终一致（Eventually consistent）


## Reference
* [https://blog.csdn.net/QuinnNorris/article/details/80995871](https://blog.csdn.net/QuinnNorris/article/details/80995871)
* [http://www.cnblogs.com/hxsyl/p/4381980.html](http://www.cnblogs.com/hxsyl/p/4381980.html)
