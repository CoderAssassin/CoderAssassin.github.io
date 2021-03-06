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

## 分布式事务解决方案
分布式数据库中，数据保存在不同的数据库中，每个数据库的事务都是独立分开的，彼此互不干扰，因此，在处理分布式数据库事务时，如何保证数据的一致性是个很大的问题。

#### 2PC
2PC(Two-Phase Commit) 两阶段提交，又叫做XA Transaction，主要包括两个阶段，这里我们称事务的发起者为事务协调器，事务的执行者(数据库)为参与者：

* 阶段一：准备阶段。事务协调器向所有的参与者发送事务的内容，所有的参与者执行事务内容，执行成功的参与者反馈YES；否则反馈NO，表示是否可以提交事务。
* 阶段二：提交阶段。若阶段一所有参与者均反馈YES，那么事务协调器向所有的参与者发送提交请求，所有参与者提交事务后返回Ack消息，如果收到所有的参与者的Ack消息说明事务提交成功；如果有一个参与者反馈NO，那么事务协调器发送回滚事务的请求，所有参与者回滚事务后返回Ack消息，事务协调器收到所有参与者的Ack消息说明中断完成。
  ![2pc-success](https://github.com/CoderAssassin/markdownImg/blob/master/JavaConcurrent/2pc.jpg?raw=true)

优点：

* 保证了数据的强一致性

缺点：

* 同步阻塞：指的是事务分两阶段，当其他数据库还没有返回YES或者NO的时候，已经反馈消息的数据库必须继续等待，必须所有的数据库都反馈消息了以后才能继续，这样就造成了部分数据库的阻塞。
* 单点：因为都是依靠协调者发送命令，当协调者崩溃了，那么参与者会一直等待协调者发送消息。
* 脑裂：如果因为网络的原因，在第二阶段只有部分的参与者收到了提交的指令，那么就会造成数据库的数据不一致的问题。

#### 3PC
3PC(Three-Phase Commit) 三阶段提交，是对2PC的改进。3PC有如下改进：

* 3PC引入了超时机制
* 将准备阶段分为CanCommit和PreCommit两个阶段，也就是说在2PC的第二阶段之前会先判断参与者的状态是否都是一致的

主要过程如下：

* 阶段一：CanCommit阶段。事务协调者向所有的参与者发送CanCommit请求，若参与者可以执行事务操作，返回YES，否则NO。
* 阶段二：PreCommit阶段。
	* 如果阶段一所有参与者反馈YES，那么发送PreCommit请求，参与者执行事务操作，成功返回YES，失败返回NO。
	* 如果阶段一有任意一个参与者返回NO，**或者在等待超时时间后协调者没有收到参与者的反馈**，那么发送abort请求，如果参与者再等待时间超时后还没有收到进一步的指令，那么也会自动中断事务。
* 阶段三：doCommit阶段。
	* 如果阶段二的所有参与者均反馈Yes，那么协调者向所有参与者发送doCommit请求，参与者执行事务完毕后返回Ack。
	* 如果阶段二有任意一个参与者反馈No或者协调者在一定时间后无法收到所有参与者的反馈，那么协调者向所有的参与者发起abort请求，参与者回滚事务并返回Ack。

![3pc](https://github.com/CoderAssassin/markdownImg/blob/master/JavaConcurrent/3pc.png?raw=true)

优点：

* 3PC引入了超时机制，所以参与者等待请求超时之后会自动中断，协调者等待反馈超时之后会自动发送中断请求，这样就避免了上边的单点问题。

缺点：

* 依旧存在脑裂问题，在阶段三的最后若协调者发送abort请求，但是有部分参与者无法收到abort请求，会导致数据库不一致。

#### TCC
相对于2PC，TCC通过对业务逻辑的调度实现分布式事务。分布式中，对一个特定业务逻辑S的调用只是一个临时性操作。也就是说，调用的一方M如果认为全局事务应该rollback，那么会取消之前的临时性操作，也就是对业务逻辑S的一个取消操作；如果M认为全局事务Commit时，就是对应S的一个确认操作。
TCC一共分为如下三个操作阶段：

* 初步操作(Try)：这里比较难理解，这个Try只是一个初步操作，虽然表面上看起来是执行了业务逻辑，感觉已经执行完毕，但是其实不是这样，还需要后边的Commit或者Cancel才能说明一次业务逻辑处理结束。有点类似事务的提交需要先处理然后提交，这里类似先处理。
* 确认操作(Confirm)：确认操作和初步操作加起来才算一次成功的业务逻辑，TCC要Commit全局事务的时候，需要对所有的Try的业务逻辑进行Confirm，这样才算是Try的完成。
* 取消操作(Cancel)：取消操作加上初步操作算是一次失败的业务逻辑的结束，当TCC要rollback全局事务时，会逐个对Try执行取消操作，表示对Try的操作的取消。

#### Raft
Raft是一种分布式协议标准，Raft的诞生是为了解决一致性问题。在一个分布式系统中，很难要求所有的个体都保持100%一致，因为网络或服务器崩溃等原因，会导致一部分个体不能和其他服务器达成一致，而Raft确保容错性，也就是说即使系统中有少量服务器宕机，也不会影响整个系统的服务。Raft认为，只要有超过半数的个体达成一致就可以。
Raft中有以下三种角色：

* Leader：处理所有客户端的交互，日志复制等，一般只能有一个Leader。类似redis集群中的master。
* Follower：类似redis集群中的slave。
* Candidator：可以被选为Leader。

具体的过程类似redis的集群。

#### Paxos
Paxos算法是分布式一致性算法，解决分布式如何就某个值达成达成一致。在Paxos分布式系统中采用的也是投票机制，只要系统中超过半数的节点在线且通信正常即可正常对外服务。
Paxos分布式系统的概念：

* Proposer：提议发起者。发起提议到集群中。
* Acceptor：提议批准者。接受提议。
* Replica：分布式系统中的节点，可以是Proposer或者Acceptor。
* ProposalId：提议的编号，编号越高优先级越高。
* Paxos Instance：Paxos中用来在多个节点之间对同一个值达成一致的过程
* acceptedProposal：在一个Paxos Instance内，已经接收过的提议
* acceptedValue：在一个Paxos Instance内，已经接收过的提议对应的值
* minProposal：在一个Paxos Instance内，当前接收的最小提议值

#### 原始的Paxos
原始的Paxos分布式系统主要包括两个阶段：准备阶段和提议阶段。
大致过程如下：

* 阶段一，获取递增的ProposalId
* Proposer向所有的Acceptor发送Prepare(n)请求，n为ProposalId号
* Acceptor比较n和minProposal，如果n>minProposal，那么minProposal=n；否则返回acceptedProposal和acceptedValue
* 如果Proposer收到半数以上的反馈后发现有acceptedValue返回，那么保存acceptedValue并生成更高的提议
* 当前步骤表示Paxos Instance中没有优先级更高的提议，进入第二阶段，发送请求accept(n,value)到所有Acceptor
* Acceptor比较n和minProposal，若n>=minProposal，那么acceptedProposal=minProposal=n，acceptedValue=value，保存本地；否则返回minProposal
* Proposer收到半数的反馈后，若发现返回的值>n，那么跳转步骤1，否则value达成一致

缺陷：可能会出现活锁，比如如下的过程：

![paxos-lock](https://github.com/CoderAssassin/markdownImg/blob/master/JavaConcurrent/paxos.png?raw=true)
* S1作为提议者，发起prepare(3.1),并在S1,S2和S3达成多数派；
* 随后S5作为提议者 ，发起了prepare(3.5)，并在S3,S4和S5达成多数派；
* S1发起accept(3.1,value1)，由于S3上提议 3.5>3.1,导致accept请求无法达成多数派，S1尝试重新生成提议
* S1发起prepare(4.1),并在S1，S2和S3达成多数派
* S5发起accpet(3.5,value5)，由于S3上提议4.1>3.5，导致accept请求无法达成多数派，S5尝试重新生成提议
* S5发起prepare(5.5),并在S3,S4和S5达成多数派，导致后续的S1发起的accept(4.1,value1)失败


## Reference
* [https://blog.csdn.net/QuinnNorris/article/details/80995871](https://blog.csdn.net/QuinnNorris/article/details/80995871)
* [http://www.cnblogs.com/hxsyl/p/4381980.html](http://www.cnblogs.com/hxsyl/p/4381980.html)
* [https://blog.csdn.net/lizhen1114/article/details/80110317](https://blog.csdn.net/lizhen1114/article/details/80110317)
* [https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html](https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html)
* [https://blog.csdn.net/liaomin416100569/article/details/78908196](https://blog.csdn.net/liaomin416100569/article/details/78908196)
* [https://www.jdon.com/artichect/raft.html](https://www.jdon.com/artichect/raft.html)
* [https://www.cnblogs.com/cchust/p/5617989.html](https://www.cnblogs.com/cchust/p/5617989.html)
