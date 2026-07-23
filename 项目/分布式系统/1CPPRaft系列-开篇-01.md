![[c4fe3f0fcb7289dcc8d8fcf3482115fd.png]]

### 项目背景
本项目致力于构建一种基于Raft一致性算法的分布式键值存储数据库，以确保数据的一致性、可用性和分区容错性。
* 目的：学习了Raft算法之后手动实现，**并基于此搭建了一个k-v存储的分布式数据库**。
* 主要工作：实现基本的Raft协议和键值存储功能。
	* 1.  **一致性：** 通过Raft算法确保数据的强一致性，使得系统在正常和异常情况下都能够提供一致的数据视图。
	* **可用性：** 通过分布式节点的复制和自动故障转移，实现高可用性，即使在部分节点故障的情况下，系统依然能够提供服务。
	* **分区容错：** 处理网络分区的情况，确保系统在分区恢复后能够自动合并数据一致性。
* 技术栈
	* 1.  **Raft一致性算法：** 作为核心算法，确保数据的一致性和容错性。
	*  **存储引擎：** 使用适当的存储引擎作为底层存储引擎，提供高效的键值对操作。目前选择的是跳表，但是可以替换为任意k-v数据库。
* 未来可以优化：性能优化、安全性增强、监控和管理工具的开发等。
* 思考
	* raft算法，虽然简单，但是如何保证在复杂情况下的容错，代码在不同运行阶段如果发生宕机等错误如何作
	* 关注raft算法本身实现与对外暴露接口

#### 项目划分
* **raft节点**：核心层，负责与其他机器raft节点联通，达成 分布式共识的目的
* **raftServer**： 负责raft节点与kv数据库中间的协调服务。 负责持久化kv数据库的数据
* **上层状态机** ： kv数据库 负责数据存储
* **持久层**：复杂数据落盘。
* **RPC通信**：重要过程中提供多节点快速通信能力

#### raft集群数据库
个机器通过网络通信构成集群，对外表现就像一台单机kv数据库一般，且少数节点故障不会影响集群工作
* 优势
	* 集群有容错能力，可保证各个机器kv数据库以相同顺序执行外部命令
* 劣势
	* 容错需要算法支持，程序会变复杂。
	* 此外需要额外对数据备份
	* 额外网络通信开销

#### 项目描述
本项目基于raft共识算法的分布式kv数据库，具备线性一致性和分区容错性，在少于半数节点发生故障时仍正常提供服务。使用个人实现的RPC通信框架MPRRPC和调表数据库完成RPC功能和KV存储功能

##### 主要工作
* 基于protobuf和自定义协议实现RPC通信框架，完成各节点之间的远程调用和数据传递功能
* 基于调表数据结构实现跳表数据库完成kv存储功能
* 实现raft协议的心跳与选举机制，通过定时线程触发心跳与选举任务，并维护集群的日志提交状态。实现日志读写与提交，由领导节点处理客户端读写请求，并将日志复制到跟随者节点。超过半数节点复制成功后提交日志，应用命令至状态机返回响应给客户端
* 实现客户端协议，包括在客户端协议中加入由客户端唯一编号和请求序号组成的请求id宝成线性一致性，以及客户端重试等功能

#### 收获
* 深入了解了分布式系统知识
* 熟悉了raft共识算法的原理和实现，加强对分布式系统中一致性，容错性等重要概念的理解
* 学习了rpc和kv存储数据的相关原理和实现

#### Reference
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://github.com/youngyangyang04/Skiplist-CPP  
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/  
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://raft.github.io/  
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://cloud.tencent.com/developer/article/1860632  
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://erdengk.top/archives/fen-bu-shi-jian-dan-ru-men-zhi-shi  
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/  
![](file:///C:\Users\DELL\AppData\Roaming\Tencent\QQTempSys\[5UQ[BL(6~BS2JV6W}N6[%S.png)https://www.zhihu.com/people/tan-xin-yu-22


下面推荐一些相关的学习资料，甚至本项目部分内容都是源于下面内容：
1.  [卡哥的跳表](https://github.com/youngyangyang04/Skiplist-CPP)
2.  [mit6.824课程的汉化book](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/)
3.  [raft算法的可视化](https://raft.github.io/)
4.  [分布式系统之CAP理论](https://cloud.tencent.com/developer/article/1860632)
5.  [分布式简单入门知识集合](https://erdengk.top/archives/fen-bu-shi-jian-dan-ru-men-zhi-shi)
6.  [Raft的介绍](https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/)
7.  [大佬的知乎](https://www.zhihu.com/people/tan-xin-yu-22)
8.  [mit6.824的讲义](http://nil.csail.mit.edu/6.824/2020/notes/l-raft.txt)
9.  raft论文

  

## 最佳食用指南
#### 可以尝试思考的问题

  

1.  锁，能否在其中的某个地方提前放锁，或者使用多把锁来尝试提升性能？
2.  多线程发送，能不能直接在doHeartBeat或者doElection函数里面直接一个一个发送消息呢？
	1. 可以使用线程池 并发发送

  

#### 可以有的优化空间：

  

1.  线程池，而不是每次rpc都不断地创建新线程
2.  日志
3.  从节点读取日志

### 为什么协程可以高并发？

协程的主要优势是：

1.  **低开销**：协程是在用户态切换的，不涉及内核态的上下文切换，开销远小于线程。
2.  **非阻塞**：协程通过**异步 IO**机制，可以在一个协程中进行多个任务的切换，而不会阻塞整个线程的执行。
3.  **可控并发**：协程允许开发者更精细地控制任务的执行顺序和并发度，非常适合需要大量 IO 操作的应用。



#### 后期内容预告！

  

1.  剩余辅助函数的逻辑。
2.  持久化：raft哪些变量需要持久化
3.  rpc：如何实现一个简单的rpc通信

  

哦吼！差点忘记了，大家都喜欢show me the code（文本阅读源码太无聊），俺们的代码仓库在[这里](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp)，目前已经可以尝试一个rpc的简单运行了，运行方式在首页的README中。

  

走过路过不要忘记点一个star。

* 为了防止在同一时间有太多的 follower 转变为 candidate 导致无法选出绝对多数， Raft 采用了随机选举超时（`randomized election timeouts`）的机制，
* 每一个 candidate 在发起选举后，都会随机化一个新的选举超时时间， 一旦超时后仍然没有完成选举，则增加自己的 term，然后发起新一轮选举。 在这种情况下，应该能在较短的时间内确认出 leader。 （因为 term 较大的有更大的概率压倒其他节点）

Raft 保证下列两个性质：

-   如果在两个日志（节点）里，有两个 entry 拥有相同的 index 和 term，那么它们一定有相同的 cmd；
	- 仅有 leader 可以生成 entry”
-   如果在两个日志（节点）里，有两个 entry 拥有相同的 index 和 term，那么它们前面的 entry 也一定相同。
	- 一致性检查（consistency check）
		- leader 在通过 AppendEntriesRPC 和 follower 通讯时，会带上上一块 entry 的信息， 而 follower 在收到后会对比自己的日志，如果发现这个 entry 的信息（index、term）和自己日志内的不符合，则会拒绝该请求。
		- 一旦 leader 发现有 follower 拒绝了请求，则会与该 follower 再进行一轮一致性检查， 找到双方最大的共识点，然后用 leader 的 entries 记录覆盖 follower 所有在最大共识点之后的数据。
- raft 为了避免出现一致性问题，要求 leader 绝不会提交过去的 term 的 entry （即使该 entry 已经被复制到了多数节点上）。leader 永远只提交当前 term 的 entry， 过去的 entry 只会随着当前的 entry 被一并提交。

因此 leader 当选后，应当立刻发起 AppendEntriesRPC 提交一个 no-op entry。注意，这是一个 `Must`，不是一个 `Should`，否则会有许多 corner case 存在问题。比如：

-   读请求：leader 此时的状态机可能并不是最新的，若服务读请求可能会违反线性一致性，即出现 safety 的问题；若不服务读请求则可能会有 liveness 的问题。
-   配置变更：可能会导致数据丢失，具体原因和例子可以参考此 [博客](https://zhuanlan.zhihu.com/p/359206808)。

实际上，leader 当选后提交一个 no-op entry 日志的做法就是 Raft 算法解决 “幽灵复现” 问题的解法，感兴趣的可以看看此 [博客](https://mp.weixin.qq.com/s/jzx05Q781ytMXrZ2wrm2Vg)。

### [](https://tanxinyu.work/raft/#%E8%8A%82%E7%82%B9%E5%B4%A9%E6%BA%83 "节点崩溃")节点崩溃


## 日志压缩
因此需要定时去做 snapshot。

snapshot 会包括：

-   状态机当前的状态。
-   状态机最后一条应用的 entry 对应的 index 和 term。
-   集群最新配置信息。
-   为了保证 exactly-once 线性化语义的去重表（之后会介绍到）。


所有节点间仅通过三种类型的 RPC 进行通信：

-   `AppendEntriesRPC`：最常用的，leader 向 follower 发送心跳或同步日志。
-   `RequestVoteRPC`：选举时，candidate 发起的竞选请求。
-   `InstallsnapshotRPC`：用于 leader 下发 snapshot。

### AppendEntriesRPC[](https://tanxinyu.work/raft/#AppendEntriesRPC)

参数：

-   `term`：leader 当前的 term；
-   `leaderId`：leader 的 节点 id，让 follower 可以重定向客户端的连接；
-   `prevLogIndex`：前一块 entry 的 index；
-   `prevlogterm`：前一块 entry 的 term；
-   `entries[]`：给 follower 发送的 entry，可以一次发送多个，heartbeat 时该项可缺省；
-   `leaderCommit`：leader 当前的 `committed index`，follower 收到后可用于自己的状态机。

返回：

-   `term`：响应者自己的 term；
-   `success`：bool，是否接受请求。  
    该请求通过 leaderCommit 通知 follower 提交相应的 entries 到。通过 entries[] 复制 leader 的日志到所有的 follower。

实现细节：

1.  如果 `term < currentTerm`，立刻返回 false
2.  如果 prevLogIndex 不匹配，返回 false
3.  如果自己有块 entry 和新的 entry 不匹配（在相同的 index 上有不同的 term）， 删除自己的那一块以及之后的所有 entry；
4.  把新的 entries 添加到自己的 log；  
    5 。如果 `leaderCommit > commitindex`，将 commitIndex 设置为 `min(leaderCommit, last index)`， 并且提交相应的 entries。

### [](https://tanxinyu.work/raft/#RequestVoteRPC "RequestVoteRPC")RequestVoteRPC[](https://tanxinyu.work/raft/#RequestVoteRPC)

参数：

-   `term`：candidate 当前的 term；
-   `candidateId`：candidate 的节点 id
-   `lastlogindex`：candidate 最后一个 entry 的 index；
-   `lastlogterm`：candidate 最后一个 entry 的 term。
-   `isleaderTransfer`：用于表明该请求来自于禅让，无需等待 electionTimeout，必须立刻响应。
-   `isPreVote`：用来表明当前是 PreVote 还是真实投票

返回：

-   `term`：响应者当前的 term；
-   `voteGranted`：bool，是否同意投票。

实现细节：

1.  如果 `term < currentTerm`，返回 false；
2.  如果 votedFor 为空或者为该 `candidated id`，且日志项不落后于自己，则同意投票。

### [](https://tanxinyu.work/raft/#InstallsnapshotRPC "InstallsnapshotRPC")InstallsnapshotRPC[](https://tanxinyu.work/raft/#InstallsnapshotRPC)

参数：

-   `term`：leader 的 term
-   `leaderId`：leader 的 节点 id
-   `lastIncludedindex`：snapshot 中最后一块 entry 的 index；
-   `lastIncludedterm`：snapshot 中最后一块 entry 的 term；
-   `offset`：该份 chunk 的 offset；
-   `data[]`：二进制数据；
-   `done`：是否是最后一块 chunk

返回：

-   `term`：follower 当前的 term

实现细节：

1.  如果 `term < currentTerm` 就立即回复
2.  如果是第一个分块（offset 为 0）就创建一个新的快照
3.  在指定偏移量写入数据
4.  如果 done 是 false，则继续等待更多的数据
5.  保存快照文件，丢弃索引值小于快照的日志
6.  如果现存的日志拥有相同的最后任期号和索引值，则后面的数据继续保持
7.  丢弃整个日志
8.  使用快照重置状态机


### 2. **Raft 的设计应对措施**

Raft 通过几种机制来应对这种情况，确保集群最终的一致性：

#### (1) **心跳机制和选举超时**

-   原来的 Leader 会定期向其他节点发送心跳包（`AppendEntries` RPC），期望通过心跳确认自己仍然是 Leader。然而，由于它被隔离到了少数派中，它的心跳包无法获得大多数节点的确认。
-   隔离在多数派中的 Follower 节点如果在规定的选举超时时间内没有收到 Leader 的心跳，就会认为 Leader 已经失效，并触发新的选举。
-   多数派中的一个 Follower 会被选举为新的 Leader。

#### (2) **任期（Term）机制**

-   Raft 使用任期（`term`）来区分不同时间段的 Leader。每次选举产生新的 Leader 时，任期号都会增加。
-   当网络恢复时，原来的 Leader 会尝试与多数派通信，但它的任期号比新的 Leader 低。
-   一旦原来的 Leader 收到新 Leader 发送的具有更高任期的心跳或其他 RPC 请求（如 `AppendEntries` 或 `RequestVote`），它会意识到自己已经落后，**自动降级为 Follower**。
-   Raft 保证节点在发现有更高任期的 Leader 存在时，会自动放弃自己的 Leader 身份。

#### (3) **日志一致性检查**

-   原来的 Leader 被隔离期间，它的日志可能会有未提交的条目。
-   当网络恢复后，新的 Leader 会将其日志复制到原来的 Leader，并通过一致性检查确保所有节点的日志保持一致。
-   原来的 Leader 会根据新的 Leader 的日志进行回滚，丢弃那些在分区期间没有被多数节点提交的日志条目。

### 3. **总结应对措施**

-   **Leader 退位的条件**：旧的 Leader 并不会主动退位，但一旦它与新的 Leader 取得联系并发现新的 Leader 拥有更高的任期，它会自动退位，变为 Follower。
-   **数据一致性**：Raft 确保只有被多数派确认的日志条目才会提交并应用到状态机。任何旧的 Leader 在分区期间所做的操作，如果没有被多数派确认，将在恢复网络后被新的 Leader 撤销。
-   **避免脑裂**：Raft 通过严格的多数派机制、选举超时、任期、日志复制和回滚机制，避免了集群在发生网络分区时出现脑裂，并确保了数据的线性一致性。