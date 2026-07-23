![[0bde1c6d35de6865da3bba2d960f043b.png]]

#### 线性一致性
* 1.  如果一个操作在另一个操作开始前就结束了，那么这个操作必须在执行历史中出现在另一个操作前面。
	* 一个系统的执行历史是一系列的客户端请求，或许这是来自多个客户端的多个请求。如果执行历史整体可以按照一个顺序排列，且排列顺序与客户端请求的实际时间相符合，那么它是线性一致的。当一个客户端发出一个请求，得到一个响应，之后另一个客户端发出了一个请求，也得到了响应，那么这两个请求之间是有顺序的，因为一个在另一个完成之后才开始。一个线性一致的执行历史中的操作是非并发的，也就是时间上不重合的客户端请求与实际执行时间匹配。并且，每一个读操作都看到的是最近一次写入的值。
	* 1.  对于一个操作来说，从请求发出到收到回复，是一个时间段。因为操作中包含很多步骤，至少包含：网络传输、数据处理、数据真正写入数据库、数据处理、网络传输。那么**操作真正完成（数据真正写入数据库）可以看成是一个时间点**。对于线性一致性理解还是有难度，肯定还是有些疑惑的。建议[阅读](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.6-qiang-yi-zhi-linearizability)
* raft如何达成线性一致性
	* 每个 client 都需要一个唯一的标识符，它的每个不同命令需要有一个顺序递增的 commandId，clientId 和这个 commandId，clientId 可以唯一确定一个不同的命令，从而使得各个 raft 节点可以记录保存各命令是否已应用以及应用以后的结果。即对于每个clinet，都有一个唯一标识，对于每个client，只执行递增的命令。




##### 持久化
持久化就是把不能丢失的数据保存到磁盘。
* 持久化内容
	* raft节点的部分信息
		* `m_currentTerm` ：当前节点的Term，避免重复到一个Term，可能会遇到重复投票等问题。
		* `m_votedFor` ：当前Term给谁投过票，避免故障后重复投票。
		* `m_logs` ：raft节点保存的全部的日志信息。
	* kvDb的快照
		* `m_lastSnapshotIncludeIndex` ：快照的信息，快照最新包含哪个日志Index
		* `m_lastSnapshotIncludeTerm` ：快照的信息，快照最新包含哪个日志Term，与m_lastSnapshotIncludeIndex 是对应的
	* Snapshot是kvDb的快照，也可以看成是日志，因此:全部的日志 = m_logs + snapshot因为Snapshot是kvDB生成的，kvDB肯定不知道raft的存在，而什么term、什么日志Index都是raft才有的概念，因此snapshot中肯定没有term和index信息。所以需要raft自己来保存这些信息。
* 为什么持久化这些内容
	* 共识安全
		* 除了snapshot相关的部分，其他部分都是为了共识安全。
		* 其他的信息为什么不用持久化，比如说：身份、commitIndex、applyIndex等等。applyIndex不持久化是经典raft的实现，在一些工业实现上可能会优化，从而持久化。即applyIndex不持久化不会影响“共识”
	* 优化
		* 而snapshot是因为日志一个一个的叠加，会导致最后的存储非常大，因此使用snapshot来压缩日志
* 为什么snashot可以压缩日志？
	* 日志是追加写的，对于一个变量的重复修改可能会重复保存，理论上对一个变量的反复修改会导致日志不断增大。而snapshot是原地写，即只保存一个变量最后的值，自然所需要的空间就小了。
* 什么时候持久化
	* 需要持久化的内容发送改变的时候就要注意持久化。比如`term` 增加，日志增加等等。具体的可以查看代码仓库中的`void Raft::persist()` 相关内容。
* 谁来调用持久化
	* 只要能保证需要持久化的内容能正确持久化。仓库代码中选择的是raft类自己来完成持久化。因为raft类最方便感知自己的term之类的信息有没有变化。
	* 注意，虽然持久化很耗时，但是持久化这些内容的时候不要放开锁，以防其他线程改变了这些值，导致其它异常。
* ### 具体怎么实现持久化|使用哪个函数持久化
	* 持久化需要考虑：速度、大小、二进制安全
	* 仓库实现目前采用的是使用boost库中的持久化实现，将需要持久化的数据序列化转成`std::string` 类型再写入磁盘。 当然其他的序列化方式也少可行的

## kvServer
* kvServer是干什么的
	* kvServer其实是个中间组件，负责沟通kvDB和raft节点。
* 外部请求怎么打进来呢？
	* Server来负责呀，加入后变成了 ？ 图片
* ### kvServer怎么和上层kvDB沟通，怎么和下层raft节点沟通
	* 通过这两个成员变量实现：
	* std :: shared_ptr < LockQueue < ApplyMsg>  >  applyChan; // kvServer和raft节点的通信管道
			  使用的是unordered_map来代替上层的kvD 
	   * std::unordered_map< s td::string,  std::string> m_kvDB; //kvDB，用unordered_map来替代
	            其中`LockQueue` 是一个并发安全的队列，这种方式其实是模仿的go中的channel机制。在raft类中[这里](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp/blob/e400068e64c3ee01f4e72039dfa9c0f198363441/src/raftCore/include/raft.h#L59C5-L59C56)可以看到，raft类中也拥有一个applyChan，kvSever和raft类都持有同一个applyChan，来完成相互的通信。
* ### kvServer怎么处理外部请求
	* kvServer负责与外部clerk通信。
	* 一个外部请求的处理可以简单的看成两步
		* 接收外部请求与返回外部响应
			* 请求和返回的操作我们可以通过http、自定义协议等等方式实现，但是既然我们已经写出了rpc通信的一个简单的实现（源代码可见：[这里](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp/tree/main/example/rpcExample)），那就使用rpc来实现吧。
		* 本机内部与raft和kvDB协商如何处理该请求
* ## RPC如何实现调用
	* 1.写protoc文件，并生成对应的文件，Raft类使用的protoc文件和生成的文件见：[这里](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp/tree/main/src/raftRpcPro)
	* 2.继承生成的文件的类 `class Raft : public raftRpcProctoc::raftRpc`
	* 3.重写rpc方法即可

