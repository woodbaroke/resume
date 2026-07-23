![](file:///E:\qq mobile file\704449833\Image\C2C\Image1\82f36c7393a72c0bd2182098530537e8.JPG)![](file:///E:\qq mobile file\704449833\Image\C2C\Image1\246179d8158648ca412bcc7124b9d4e6.JPG)![](file:///E:\qq mobile file\704449833\Image\C2C\Image1\b0c8e86e92f74890a2893a1bf5bccb22.JPG)![](file:///E:\qq mobile file\704449833\Image\C2C\Image1\b147cc669753eb27739b211777de4496.JPG)

![[82f36c7393a72c0bd2182098530537e8.jpg]]
![[246179d8158648ca412bcc7124b9d4e6.jpg]]![[b0c8e86e92f74890a2893a1bf5bccb22.jpg]]![[b147cc669753eb27739b211777de4496.jpg]]

###### clerk的主要功能及代码
* 功能
	* clerk相当于就是一个外部的客户端了，其作用就是向整个raft集群发起命令并接收响应。
* 代码实现
	* clerk与kvServer需要建立网络链接，那么既然我们实现了一个简单的RPC，那么我们不妨使用RPC来完成这个过程。
	* 注意
		* 对于RPC返回对端不是leader的话，就需要另外再调用另一个kvServer的RPC重试，直到遇到leader。

### 跳表
* 网络上讲解跳表的博客实在多如牛毛，前人之述备矣。
授人以鱼不如授人以渔，本人学习跳表主要参考的资料是：
[卡哥的跳表项目](https://github.com/youngyangyang04/Skiplist-CPP)
[小林的网站](https://www.xiaolincoding.com/redis/data_struct/data_struct.html#%E8%B7%B3%E8%A1%A8)
### 如何植入
卡哥的跳表在：[https://github.com/youngyangyang04/Skiplist-CPP](https://github.com/youngyangyang04/Skiplist-CPP) ，我们首先把文件添加到我们的项目中。


#### 优化
* 整个项目运行中的一些辅助的小组件的实现思路以及一些优化的思路。这些组件实现版本多样，而且与其他模块没有关联，相对也比较简单，但正是如此，这方面可以多学习一下比较优秀的实现，然后稍微测试测试，面试的时候拿出来说一说。
* ### LockQueue的实现
	* 代码在[这里](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp/blob/d2eef5c6248375a428a14efb627964af087eb69f/src/common/include/util.h#L55)。其实就是按照线程池最基本的思路，使用锁和条件变量来控制queue。
	* 那么一个可能的问题就是由于使用了条件变量和锁，可能在内核和用户态会来回切换，有没有更优秀的尝试呢？
		* 比如：无锁队列，使用自旋锁优化，其他。。。。
		* 这里推荐大家可以多试试不同的实现方式，然后测试对比，面试的话很有说法的。
* ### Defer函数等辅助函数的实现
	* 在代码中经常会看到[Defer类](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp/blob/d2eef5c6248375a428a14efb627964af087eb69f/src/common/include/util.h#L23)，这个类的作用其实就是在函数执行完毕后再执行传入Defer类的函数，是收到go中defer的启发。
	* 主要是RAII的思想，如果面试的时候提到了RAII，那么就可以说到这个Defer，然后就牵扯过来了。
* ### 怎么使用boost库完成序列化和反序列化的
	* 主要参考[BoostPersistRaftNode](https://github.com/youngyangyang04/KVstorageBaseRaft-cpp/blob/34962a7d50e1f47c7b853b9e83ffdeb7d61d2da3/src/raftCore/include/raft.h#L149)类的定义和使用。在本篇正文部分如何植入跳表的部分讲解过了，这里就不重复了。
* ## 是否还有可做的工作
	* 在代码层面可以做的工作还有很多，主要但不限于包括：
	1.  现有实现的更优雅的版本。
	2.  可能的性能测试，比如火焰图分析系统的耗时。
	3.  一些组件的引入和优化，比如LockQueue更好的实现，日志库，异步RPC等等。最近在星球中不是正好有大佬分享了协程库的实现，在raft中到处都是多线程，那么是否可以引入协程库呢哈哈哈