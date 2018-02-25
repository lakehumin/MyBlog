---
title: ZooKeeper学习笔记
date: 2018-02-25 10:07:12
tags:
- ZooKeeper
categories:
- 学习笔记
---
{% asset_img default.jpg ZooKeeper %}

## FastLeaderElection选举算法

### 核心思想
通过不断更新自己的选举结果，广播消息，让其他人知道自己的推荐leader，来让所有人尽快达成一致

### 一些概念
- **electionEpoch**
	它是选举算法FastLeaderElection的一个属性（也被叫做logicalclock），在zookeeper的server启动的时候就会创建选举算法，该值初始是0，之后该server每执行一次选举，electionEpoch 就会自增，同时在选举的时候，一旦别人的投票中的electionEpoch 比自己的大，也会更新自己的electionEpoch来和别人保持一致
- **proposedLeader**
	推荐的leader id。为被推荐leader的myid的值
- **proposedZxid**
	推荐leader的最大事务zxid，由本地文件恢复得到，为被推荐leader机器的lastProcessedZxid。表示该被推荐的leader所保留的最新的事务，用于选举的比较
- **proposedEpoch**
	peerEpoch值，由上述的lastProcessedZxid的高32得到

### 一些理念
- **若是选出leader，再收到投票消息会直接返回选举结果**
- **判断规则**
	先比较peerEpoch，再zxid，最后是myid，胜者被推荐为leader
- **系统稳定运行的关键**
	有一个Leader，并且多数Follower认可这个Leader
- **大多数认定即可**

### 一些疑问
1. **为什么收到消息的electionEpoch大于当前logicallock后，会更新logicallock为收到的electionEpoch，并清空票箱recvset**
	是为了保证这是最新的同一次选举
2. **为什么收到的logicalclock比自己小时不予理睬**
	此算法的核心思想是接收消息，更新自己，再通知大家，直到选出leader。比自己小时不用更新自己，没必要回复，减少不必要的信息传输。后面自然会有更新消息通知到他，让其更新logicalclock。
3. **当统计选票确定leader后，如何确保其他大多数机器也认为该leader合法**
	由于目的是实现系统稳定运行，选出leader。不管过程怎样最后是系统稳定运行的状态即可。当统计选票确定leader后会等待一段时间，看是否有更优的提议，没有就确认当前推荐leader为正式leader，确定对应状态是leader还是follower。其实这一步也只能基本保证该leader合法。不过没关系，由于leader会不断检测是否依然有多个follower认定自己是leader，follower也会根据leader是否有不断给自己心跳来确定leader是否有效。因此，若出现最初统计的leader不合法后，会再次进行选举，最后一定会形成系统稳定运行的状态。

### 选举流程
1. 发起选举logicalclock++，推荐自己为leader，设置当前投票为本机myid, lastProcessedZxid, peerEpoch。并通知所有服务器。
2. 只要当前服务器状态为LOOKING，进入循环，不断地读取其它Server发来的通知、进行比较、更新自己的投票、发送自己的投票、统计投票结果，直到leader选出或出错退出。收到通知后，根据对方状态进行处理
	- LOOKING状态
		1. 如果发送过来的逻辑时钟大于目前的逻辑时钟，那么说明这是更新的一次选举投票，此时更新本机的逻辑时钟logicalclock，清空投票箱（因为已经过期没有用了），调用totalOrderPredicate函数判断对方的投票是否优于当前的投票（判断规则上面提过了），是的话用对方推荐的leader更新下一次的投票，否则使用初始的投票（投自己），最后都将调用sendNotifications() 通知所有服务器我的选择，并将投票信息放入投票箱recvset。
		2. 如果对方处于上轮投票，即electionEpoch比我小，则不予理睬
		3. 如果对方也处于本轮投票，调用totalOrderPredicate函数判断对方的投票是否优于当前的投票，是的话更新当前的投票并新生成notification消息放入发送队列，否则使用初始的投票（投自己）不发送通知。投票信息放入投票箱。
		4. 统计投票结果，判断所推荐的leader是否得到集群多数人的同意（根据计票器的实现不同，可以是单纯看数量是否超过n/2，也可以是按权重来判断，我们这里假设单纯看数量），如果得到多数人同意，那么还需等待一段时间，看是否有比当前更优的提议，如果没有，则认为投票结束。根据投票结果修改自己的状态。以上任何一条不满足，则继续循环。
	- OBSERVING状态
		不做任何事
	- FOLLOWING或LEADING状态
		1. 如果选举周期相同（选票是同一轮选举产生），将该数据保存到投票箱，根据当前投票箱的投票判断对方推荐的leader是否得到多数人的同意，如果是则设置状态退出选举过程，否则执行下面第2步。
		2. 这是一条与当前逻辑时钟不符合的消息，或者对方推荐的leader没有得到多数人的同意（有可能是收集到的投票数不够），那么说明可能在另一个选举过程中已经有了选举结果，于是将该选举结果加入到outofelection集合中，再根据outofelection来判断是否可以结束选举，如果可以也是保存逻辑时钟，设置状态，退出选举过程。否则继续循环。outofelection用于保存那些状态为FOLLOWING或者LEADING的ZooKeeper节点发送的选票，由于对方的状态为FOLLOWING或者LEADING，所以它们当前不参与选举过程（可能人家已经选完了），因此称为“out of election”。

### 参考链接
[ZooKeeper的一致性算法赏析](https://my.oschina.net/pingpangkuangmo/blog/778927)
[Zookeeper架构及FastLeaderElection机制](http://www.jasongj.com/zookeeper/fastleaderelection/)
[Zookeeper的FastLeaderElection算法分析](https://www.jianshu.com/p/ccaecde36dd3)

## zookeeper一致性保证
1. 所有机器按照事务zxid顺序执行便能实现最终一致性
2. 先发起proposal，获得过半ACK后发起commit。两段提交保证响应成功即事务操作成功。集群最终将全部实现该操作。

### 参考
[ZooKeeper应用场景及解析](http://blog.csdn.net/gs80140/article/details/51496925)