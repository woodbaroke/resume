![[e3c47cc1b10d447a2d8471c7e5be4b79.jpg]]
![[a33bba955d5d7372c4752faf20de018c.jpg]]

#### 共识
* 共识涉及多个服务器对状态机状态达成一致。一旦对状态机状态做出决定，就是最终决定。为保证安全性，已经被集群共识的值可以保证后面不会被覆盖
* 一致性算法 ：保证大部分服务器可用时保持运行。如果小于一半节点故障不对集群运行产生影响，一半以上则集群停止对外服务，永远不会返回不正确结果
* 满足特性
	* 可信的网络条件下（通信节点都是真实的）保证共识的一致性
	* 多数节点存活时，保持可用性
	* 不依赖于绝对时间。raft用自定的term任期时间作为逻辑时钟代替绝对时间。因为节点故障网络报文也会受到干扰而延迟，依赖绝对时间会带来问题
	* 多数节点一致后就返回结果，不受个别慢节点影响
		* 对于raft来说 判断的操作是存储到日志log中，一个操作就是log中的一个entry

#### raft概念
* 状态机
	* raft上层应用，此处是kv数据库
* 日志
	* log： raft保存的外部命令是以日志保存
	* entry： 日志有很多，可以看成一个连续数据，一个数据就是一个entry
* 提交日志 commit
	* raft保存日志后，经过复制同步才能应用到状态机，这个应用过程就是提交
* 节点身份
	* follower
	* candidate
	* leader
* term
	* 任期，即raft逻辑时钟
	* raft使用term对比比较日志 身份，心跳的新旧。term与leader身份绑定，即某个节点是leader更应该说是集群某个term的leader。term用连续数字表示，会在folloer发起选举时候+1（成为candidate试图成为leader）
		* 胜利当选：半数节点给当前candidate投票
		* 失败：如果没有任何candidate当选，选举超时后又会开始另一个term的选举
	* 
* 选举
	* follower变为leader
* 日志term
	* 日志提交惠济路日志再什么时候 即哪个term记录，用于后续日志的新旧比较
* 心跳、日志同步
	* leader向follower发送心跳（AppendEntryRPC）用于告诉follower自己的存在以及通过心跳来携带日志以同步

##### 安全性
* 发生故障时，一个节点无法知道当前最新的Term是多少，在故障恢复后，节点就可以通过其他节点发送过来的心跳中的Term信息查明一些过期信息。当发现自己的Term小于其他节点的Term时，这意味着“自己已经过期”
	* leader、Candidate：退回follower并更新term到较大的那个Term
		* 因为通过Term信息知道自己过期，意味着自己可能发生了网络隔离等故障，那么在此期间整个Raft集群可能已经有了新的leader、**提交了新的日志**，此时自己的日志是有缺失的，如果不退回follower，那么可能会导致整个集群的日志缺失，不符合安全性。
	* follower：更新Term信息到较大的那个Term
* 一个节点在一个term只有一个选票——保证了一个任期只有一个节点
* 集群同时最多只会有一个可以读写的 leader
	* 集群中由于发生了网络分区，一个分区中的leader会维护分区的运行，而另一个分区会因为没有leader而发生选举产生新的leader，任何情况下，最多只有一个分区拥有绝大部分节点 ，那么只有一个分区能写入日志
* 1.  `Leader Append-Only` ：leader 的日志是只增的。
* 1.  `Log Matching` ：如果两个节点的日志中有两个 entry 有相同的 index 和 term，那么它们就是相同的 entry。
	* 因为在Raft中，每个新的日志条目都必须按照顺序追加到日志中。在运行过程中会根据这条性质来检查follower的日志与leader的日志是否匹配，如果不匹配的话leader会发送自己的日志去覆盖follower对应不匹配的日志。
* 1.  `Leader Completeness` ：一旦一个操作被提交了，那么在之后的 term 中，该操作都会存在于日志中。
* 1.  `State Machine Safety` ：状态机一致性，一旦一个节点应用了某个 index 的 entry 到状态机，那么其他所有节点应用的该 index 的操作都是一致的。


#### 领导人选举
* Raft是一个强`Leader` 模型，可以粗暴理解成Leader负责统领follower，如果Leader出现故障，那么整个集群都会对外停止服务，直到选举出下一个Leader。如果follower出现故障（数量占少部分），整个集群依然可以运行。
	* 节点之间通过网络通信，其他节点（follower）如何知道leader出现故障？
		* leader会定时向集群中剩下的节点（follower）发送AppendEntry（作为心跳，hearbeat ）以通知自己仍然存活。
			* AppendEntry作用
				*  心跳： 携带日志entry及其辅助信息，以控制日志的同步和日志向状态机提交
				* 通告leader的index和term等关键信息以便follower对比确认follower自己或者leader是否过期
		* 如果follower在一段时间内没有接收leader发送的AppendEntry，那么follower就会认为当前的leader 出现故障，从而发起选举。
			* 可以用一个定时器和一个标志位来实现，每到定时时间检查这期间有无AppendEntry 即可。
	* follower知道leader出现故障后如何选举出leader？
		* follower认为leader故障后只能通过：term增加，变成candidate，向其他节点发起RequestVoteRPC申请其他follower的选票，过一段时间之后会发生如下情况：
		* 1.  赢得选举，马上成为`leader` （此时term已经增加了）发现有符合要求的leader，自己马上变成follower 了，这个符合要求包括：leader的term≥自己的term 一轮选举结束，无人变成leader，那么循环这个过程，即：term增加，变成candidate
	* 符合什么条件的节点可以成为leader？
		* “选举限制”，有限制的目的是为了保证选举出的 leader 一定包含了整个集群中目前已 committed 的所有日志。
		* 当 candidate 发送 RequestVoteRPC 时，会带上最后一个 entry 的信息。 所有的节点收到该请求后，都会比对自己的日志，如果发现自己的日志更新一些，则会拒绝投票给该 candidate，即自己的日志必须要“不旧于”改candidate。
		* 这样的限制可以保证：成为leader的节点，其日志已经是多数节点中最完备的，即包含了整个集群的所有 committed entries。
	* 如何判断日志的老旧：
		* 比较两个东西：最新日志entry的term和对应的index。index即日志entry在整个日志的索引。
			* if 两个节点最新日志entry的term不同
			   term大的日志更新
			   else
			   最新日志entry的index大的更新
			   end

#### 日志同步、心跳
* 日志同步 和 心跳 是放在一个RPC函数（AppendEntryRPC）中来实现的
	* 心跳RPC 可以看成是没有携带日志的特殊的 日志同步RPC
* 对于一个follower，如果leader认为其日志已经和自己匹配了，那么在AppendEntryRPC中不用携带日志（再携带日志属于无效信息了，但其他信息依然要携带），反之如果follower的日志只有部分匹配，那么就需要在AppendEntryRPC中携带对应的日志。
* 问题
	* ##### 为什么不直接让follower拷贝leader的日志|leader发送全部的日志给follower？
		* eader发送日志的目的是让follower同步自己的日志，当然可以让leader发送自己全部的日志给follower，然后follower接收后就覆盖自己原有的日志，但是这样就会携带大量的无效的日志（因为这些日志follower本身就有）。
		* 因此 raft的方式是：先找到日志不匹配的那个点，然后只同步那个点之后的日志。
	  * ##### leader如何知道follower的日志是否与自己完全匹配？
		  * 在AppendEntryRPC中携带上 entry的index和对应的term（日志的term），可以通过比较最后一个日志的index和term来得出某个follower日志是否匹配
	  * ##### 如果发现不匹配，那么如何知道哪部分日志是匹配的，哪部分日志是不匹配的呢？
		  * eader每次发送AppendEntryRPC后，follower都会根据其entry的index和对应的term来判断某一个日志是否匹配。
		  * 在leader刚当选，会从最后一个日志开始判断是否匹配，如果匹配，那么后续发送AppendEntryRPC就不需要携带日志entry了。
		  * 如果不匹配，那么下一次就发送 倒数第2个 日志entry的index和其对应的term来判断匹配，如果还不匹配，那么依旧重复这个过程，即发送 倒数第3个 日志entry的相关信息
		  * 重复这个过程，知道遇到一个匹配的日志
  * raft对于日志可以保证其具有两个特点：
	  * 两个节点的日志中，有两个 entry 拥有相同的 index 和 term，那么它们一定记录了相同的内容/操作，即两个日志匹配
		  * 原因：仅有 leader 可以生成 entry
	  * 两个节点的日志中，有两个 entry 拥有相同的 index 和 term，那么它们前面的日志entry也相同
		  * 原因：eader 在通过 AppendEntriesRPC 和 follower 通讯时，除了带上自己的term等信息外，还会带上entry的index和对应的term等信息，follower在接收到后通过对比就可以知道自己与leader的日志是否匹配，不匹配则拒绝请求。leader发现follower拒绝后就知道entry不匹配，那么下一次就会尝试匹配前一个entry，直到遇到一个entry匹配，并将不匹配的entry给删除（覆盖）。
	* 注意：
		*  raft为了避免出现一致性问题，要求 leader 绝不会提交过去的 term 的 entry （即使该 entry 已经被复制到了多数节点上）。leader 永远只提交当前 term 的 entry， 过去的 entry 只会随着当前的 entry 被一并提交。
	* 优化：
		* ##### 寻找匹配加速（可选）
		在寻找匹配日志的过程中，在最后一个日志不匹配的话就尝试倒数第二个，然后不匹配继续倒数第三个。。。
		`leader和follower` 日志存在大量不匹配的时候这样会太慢，可以用一些方式一次性的多倒退几个日志，就算回退稍微多了几个也不会太影响，具体实现参考[7.3 快速恢复
		