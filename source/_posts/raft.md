---
title: Raft学习笔记
date: 2018-04-05 18:18:17
tags:
- raft
- 一致性
categories:
- 学习笔记
---
{% asset_img default.jpg raft %}

## 一些概念
1. broadcastTime<<electionTimeout<<MTBF
	- broadcastTime。leader发送心跳的时间间隔
	- electionTimeout。选举超时时长（值随机），若这段时间内没有leader心跳触发选举。
	- MTBF。服务器发生故障的时间间隔。

	MTBF一般为几个月以上。electionTimeout为机器故障时的不可用时长一般为10ms-500ms。broadcastTime应该比electionTimeout小一个数量级，为0.5ms-20ms。
2. server有三种状态follower、candidate、leader。系统初始为follower状态，超过electionTimeout开始选举。没选出结果超出electionTimeout，继续开始下一轮选举。
![](state.jpg)
3. 三种角色分工
	- leader：负责日志的同步管理，处理来自客户端的请求，与follower保持这heartBeat的联系
	- follower：刚启动时所有节点为follower状态，响应leader的日志同步请求，响应candidate的请求，把请求到follower的事务转发给leader
	- candidate：负责选举投票，Raft刚启动时由一个节点从follower转为candidate发起选举，选举出leader后从candidate转为leader状态
4. 服务器状态
	在所有服务器上持久存在的：（在响应远程过程调用 RPC 之前稳定存储的）

    | 名称        | 描述                                                    |
    | :---------: |:-------------------------------------------------------:|
    | currentTerm | 服务器最后知道的任期号（从0开始递增）                         |
    | votedFor    | 在当前任期内收到选票的候选人 id（如果没有就为 null）           |
    | log[]       | 日志条目；每个条目包含状态机的要执行命令和从领导人处收到时的任期号 |

	在所有服务器上不稳定存在的：  

	| 名称         | 描述                                                    |
	| :---------: |:-------------------------------------------------------:|
	| commitIndex | 已知的被提交的最大日志条目的索引值（从0开始递增）               |
	| lastApplied | 被状态机执行的最大日志条目的索引值（从0开始递增）               |
	在领导人服务器上不稳定存在的：（在选举之后初始化的）

	| 名称         | 描述                                                    |
	| :---------: |:-------------------------------------------------------:|
	| nextIndex[] | 对于每一个服务器，记录需要发给它的下一个日志条目的索引（初始化为领导人上一条日志的索引值+1）|
	| matchIndex[]| 对于每一个服务器，记录已经复制到该服务器的日志的最高索引值（从0开始递增）                     |


5. RequestVote RPC数据格式

	| 参数          | 描述                      |
	| :----------: |:------------------------:|
	| term         | 候选人的任期号              |
	| candidateId  | 请求投票的候选人 id         |
	| lastLogIndex | 候选人最新日志条目的索引值    |
	| lastLogTerm  | 候选人最新日志条目对应的任期号 |

	| 返回值       | 描述                        |
	| :---------: |:--------------------------:|
	| term        | 目前的任期号，用于候选人更新自己 |
	| voteGranted | 如果候选人收到选票为 true      |

6. AppendEntries RPC数据格式

	| 参数          | 描述                                 |
	| :----------: |:------------------------------------:|
	| term         | leader的任期号                         |
	| leaderId     | leader的 id，为了其他服务器能重定向到leader |
	| prevLogIndex | 最新日志之前的日志的索引值                |
	| prevLogTerm  | 最新日志之前的日志的领导人任期号           |
	| entries[]    | 将要存储的日志条目（表示 heartbeat 时为空，有时会为了效率发送超过一条）|
	| leaderCommit | 领导人提交的日志条目索引值                |

	| 返回值   | 描述                                                           |
	| :-----: |:-------------------------------------------------------------:|
	| term    | 当前的任期号，用于领导人更新自己的任期号                             |
	| success | 如果其它服务器包含能够匹配上 prevLogIndex 和 prevLogTerm 的日志时为真|

## 一些疑惑
1. **leader发送心跳后没有收到大多数（大多数指过半，下同）的回复会进入重新选举的状态吗**
	不会，也没必要。因为这样的leader由于没有大多数follower不会成功响应。另一分区则会进行选举选出term更高的leader。
2. **客户端请求了follower之后如何定位到leader**
	每次AppendEntries RPC时会带上leaderId，因此follower接到客户端的请求后就能将请求转到leader了。
3. **为什么follower收到candidate的RPC请求后为什么也会重置定时器继续follower的状态**
	为了减少leader的竞争，尽快选出leader
4. **follower的votedFor会不会因为新leader被其他大多数选出来了，而跟着设置为这个leader**
	不会，设置为任何值都没意义。新leader已经选出，再不会有第二个能获取过半投票。所以可能出现leader选出后，他投票给另一个candidate。
5. **若是某个follower故障导致进入选举会引起集群选举吗**
	会的，因为此时的term++后比任何服务器都新，会造成所以其他收到RequestVote RPC的服务器更新term变成follower，重新选举。
6. **如何确保快速选出leader**
	进入选举后很有可能出现选票瓜分的情况，没有一个candidate获得超过半数。发生electionTimeout后重新选举，由于electionTimeout随机，所以必有某服务器先进入选举，有更大可能成为leader，从而能很快的结束选举，选出leader。
7. **candidate收到AppendEntries RPC后，若对方term与自己相等为什么也会变为follower**
	此时自己没有对方更新的数据，且对方已经被大多数承认为leader，为了快速选出leader没必要继续选举。
8. **如何保证同任期同index的日志内容一样**
	每台机器都会保存某一任期的投票情况，且一旦投票就不会发生改变。选leader的时候，是收集指定任期的选票情况，过半数才能选为leader。这就保证了具体某一任期的leader独一无二。不会出现两个leader任期相同的情况。然后日志index又是leader给的，所以同任期同index的日志必定是一样的。
9. **是否存在不同任期的日志同index。若存在，那解决日志冲突时会不会出现日志index不连续的情况**
	存在的。每个日志的追加leader都会按日志index递增，但leader挂了后，会选出新的leader，此时新leader可能没同步到上一个leader的日志，于是就可能出现同index不同任期的日志。但不管怎样对每一个leader来说他们都维护了index的递增，所以最好无论是谁的日志覆盖了谁，最好的index应该都是连续递增的。
10. **为何同任期同index的日志，他们之前的日志也一定相同**
	因为日志append的时候会判断之前的任期和index，只有相同才能append成功，再加上同任期同index日志内容相同的结论。类似数据归纳法，逐个类推，之前所有日志全部相同。
11. **Raft从来不会通过计算复制的数目来提交之前任期的日志条目**
	只有领导人当前任期的日志条目才能通过计算数目来进行提交。一旦当前任期的日志条目以这种方式被提交，那么由于日志匹配原则，之前的日志条目也都会被间接的提交。原因考虑下面的情况：
	![](example.jpg)
	a阶段s1为leader，任期为2。b阶段s5变为leader，任期3。c阶段leader又变成了s1，此时term2index2的日志复制给了大多数，但该日志并不能变成已提交状态。因为此时s1的任期为4，这是在复制以前任期的日志。由于可能出现d阶段s5重新变为leader把term2index2的日志覆盖的情况，所以只能等到e阶段s1将term4index3的日志（属于当前任期的日志）复制给了大多数后，确定term4index3为已提交状态，同时间接将term2index2日志设为已提交。
12. **为何只要leader的AppendEntries RPC收到大多数回复就可以认为该日志为已提交**
	因为若此时leader挂掉，那么由于大多数都有这条最新的日志，选出来的leader必定也有。那么新leader给其他follower同步日志的时候一定会带上这条日志，若新leader有新的日志提交，那么这条日志必定间接提交。若没有新提交那么，该leader再挂，再次选出来的leader也一定包含这条日志（因为一定是因为有最新的日志才会成为新leader，而新的日志必定是上一个leader加上的，考虑prevLogIndex，所以新leader也会带着那条日志），那么问题又回来了。
	若最初始leader没挂，而是新加了几条日志再挂，新加的日志若成功必定含有该日志，问题回到上面。若不成功（准确说是不确定），则依然是上面的两个问题之一。
13. **若leader的AppendEntries RPC没收到大多数回复该怎么返回**
	若是没有大多数，则可能换leader后，被不带有该日志的leader把其截掉。也可能被带有该日志的leader复制给所有机器。所以都有可能，不确定，返回给客户端不确定的答案，如超时，让他重试。
14. **客户端发起重试的时候怎么确保已经执行过的操作不被重复执行**
	当客户端没收到leader的回复时，可能是leader不确定答案，可能是leader挂了，也可能是leader回复网络丢包等等。此时客户端对同一log发起重试，若之前已经执行过那么这次不应该让leader再次执行。怎么判断这是同一次操作呢？解决办法是，客户端对每条指令都会有一个唯一的序列号。
15. **新leader选出来的时候它是不知道以往日志的提交情况的，往往需要在自己任期提交一次才能确定之前的所有日志为已提交，怎么优化这点**
	之前的leader可能已经设置某log为已提交，但是新leader可能还没有收到commitIndex的更新。那么之后为了确认之前的日志的提交情况，它必须在自己任期成功提交一次。当然你不能等客户端在你任期发起请求，然后你才有机会提交一次。所以最后解决办法是，选为leader的初始发送一个带有空操作log的AppendEntries RPC，一旦成功便可以确认之前所有的提交。

## 一些流程
### 选举过程
1. 转变为candidate开始选举
	- candidate的currentTerm++
	- 给自己投票
	- 重置选举计时器
	- 向其他服务器发送RequestVote RPC
2. 如果收到了来自大多数服务器的投票，则成为leader，并立即发送AppendEntries RPC，以保持leader正常运转。
3. 如果收到了来自其他leader的AppendEntries RPC（heartbeat），则看term是否大于等于currentTerm。是则转换状态为追随者，否则拒绝该请求。
4. 如果选举超时，开始新一轮的选举

### log复制过程
1. 客户端请求leader,leader将该请求追加到本地log
2. 向follower发起AppendEntries RPC
3. 当大多数follower回复成功后，设置log为已提交，回复客户端并apply到状态机（操作顺序无所谓，只要大多数回复，后面都是成立的）。

## 一些规则
- **所以服务器遵守的规则**
	- 如果commitIndex > lastApplied，lastApplied自增，将log[lastApplied]应用到状态机
	- 如果RPC的请求或者响应中包含一个term T大于currentTerm，则currentTerm赋值为T，并切换状态为follower
	- 一个term只能投票给一个服务器
- **follower遵守的规则**
	- 响应来自候选人和领导人的 RPC
	- 如果在超过选取领导人时间之前没有收到来自当前领导人的AppendEntries RPC或者没有收到候选人的投票请求，则自己转换状态为候选人。反之收到了，则重置定时器。
	- 收到RequestVote RPC后
		- 若term<currentTerm，返回false
		- 若term>=currentTerm（都变为candidate的时候可能出现=），则看votedFor是否为null，不是则返回当前投票votedFor是否等于请求的候选人Id。是则比较谁的log更新，即比较lastLogTerm和lastLogIndex。比自己新或是一样新则返回true，并设置votedFor。
	- 收到AppendEntries RPC后
		- 若term<currentTerm，返回false
		- 若prevLogIndex和prevLogTerm能够匹配到自己的log[]中的某条log，则将entries[]应用到prevLogIndex和prevLogTerm后面的日志。否则不能匹配则返回false。
		- 如果leaderCommit > commitIndex，并且此次RPC将要返回true则设置commitIndex= leaderCommit。（你要保证应用到状态机的log和leader一样才行）
	- 经过一系列的AppendEntries RPC后，日志最终会和leader同步
- **leader遵守的规则**
	- 成为leader后，向其他所有服务器发送带空操作log的AppendEntries RPC，在空闲时间重复发送以防止选举超时。
	- 如果收到来自客户端的请求，向本地日志增加条目，并试图复制给其他机器，在收到大多数机器回复后响应客户端成功并设置该日志为已提交，更新commitIndex。
	- 如果follower崩溃了或者运行缓慢或者是网络丢包了，leader会无限的重试AppendEntries RPC（甚至在它向客户端响应之后）直到所有的follower最终存储了所有的日志条目。
	- 每次给follower AppendEntries RPC时会把nextIndex后的所有日志发出去，若follower匹配prevLogIndex和prevLogTerm后返回成功，则更新对应follower的nextIndex和matchIndex。若返回false，则nextIndex减1，下次AppendEntries RPC时重新发送。

## 参考链接
[raft英文原版文献](https://link.zhihu.com/?target=https%3A//raft.github.io/raft.pdf)
[raft中文](http://www.infoq.com/cn/articles/raft-paper)
[raft小动画](http://thesecretlivesofdata.com/raft/#intro)