#### 线程模型
* 单线程reactor
	* event-loop库，待事件发生后**原地**调用对应的event handler
	* 一对一，一个event-loop只能使用一个核，故此类程序要么是IO-bound
	* 把多段逻辑按事件触发顺序交织在一个系统线程中
* N：1线程库 （又称Fiber 其实就是协程
	* 把N个用户线程映射入一个系统线程
	* 同时只运行一个用户线程，调用阻塞函数时才会切换至其他用户线程
	* N:1线程库与单线程reactor在能力上等价，但事件回调被替换为了上下文(栈,寄存器,signals)，运行回调变成了跳转至上下文
* 多线程reactor
	* 由一个或多个线程分别运行event dispatcher，待事件发生后把event handler交给一个worker线程执行![[Pasted image 20240904141515.png]]
* ## M:N线程库
	* 即把M个用户线程映射入N个系统线程。
	* 可以决定一段代码何时开始在哪运行，并何时结束，相比多线程reactor在调度上具备更多的灵活度。
	* 相比N:1线程库，M:N线程库在使用上更类似于系统线程，需要用锁或消息传递保证代码的线程安全。

#### bthread是brpc使用的M:N线程库
* ”M:N“是指M个bthread会映射至N个pthread，一般M远大于N
* 目的
	* 提高程序的并发度的同时，降低编码难度，并在核数日益增多的CPU上提供更好的scalability和cache locality。


#### FAQ
* ##### bthread是协程(coroutine)吗？
	* 不是。通常协程指N：1线程库。即所有的协程运行于一个系统线程中，计算能力和各类eventloop库等价
		* 协程优势
			* 由于不跨线程，协程之间的切换不需要系统调用，可以非常快(100ns-200ns)，受cache一致性的影响也小
		* 代价
			* 协程无法高效地利用多核，代码必须非阻塞，否则所有的协程都被卡住，对开发者要求苛刻
			* 协程的这个特点使其适合写运行时间确定的IO服务器，典型如http server
	* bthread是一个M:N线程库，一个bthread被卡住不会影响其他bthread。
		* 关键技术两点：work stealing调度和butex
			* 前者让bthread更快地被调度到更多的核心上
			* 后者让bthread和pthread可以相互等待和唤醒
		* 这两点协程都不需要
* ##### Q: 我应该在程序中多使用bthread吗？ brpc提供了[异步接口](https://github.com/apache/brpc/blob/master/docs/cn/client.md#%E5%BC%82%E6%AD%A5%E8%AE%BF%E9%97%AE)，所以一个常见的问题是：我应该用异步接口还是bthread？
	* 延时不高时你应该先用简单易懂的同步接口，不行的话用异步接口，只有在需要多核并行计算时才用bthread。
		* 异步即用回调代替阻塞，有阻塞的地方就有回调。
	* 除非你需要在一次RPC过程中[让一些代码并发运行](https://github.com/apache/brpc/blob/master/docs/cn/bthread_or_not.md)，你不应该直接调用bthread函数，把这些留给brpc做更好
* ##### bthread和pthread worker如何对应？
	* pthread worker在任何时间只会运行一个bthread，当前bthread挂起时，pthread worker先尝试从本地runqueue弹出一个待运行的bthread，若没有，则随机偷另一个worker的待运行bthread，仍然没有才睡眠并会在有新的待运行bthread时被唤醒。
* ##### bthread中能调用阻塞的pthread或系统函数吗？
	* 可以，只阻塞当前pthread worker。其他pthread worker不受影响
* ##### Q：一个bthread阻塞会影响其他bthread吗？
	* 不影响。若bthread因bthread API而阻塞，它会把当前pthread worker让给其他bthread。若bthread因pthread API或系统函数而阻塞，当前pthread worker上待运行的bthread会被其他空闲的pthread worker偷过去运行。
* ##### bthread会有[Channel](https://gobyexample.com/channels)吗？
	* 不会。chanell代表的是两点间的关系.可以理解为channel为客户端

### bthread_id是一个特殊的同步结构
* 可以互斥RPC过程中的不同环节，也可以O(1)时间内找到RPC上下文(即Controller)
* bthread_id解决在其他rpc框架中广泛存在的问题
	* -   在发送RPC过程中response回来了，处理response的代码和发送代码产生竞争。
	* -   设置timer后很快触发了，超时处理代码和发送代码产生竞争。
	* -   重试产生的多个response同时回来产生的竞争。
	* -   通过correlation_id在O(1)时间内找到对应的RPC上下文，而无需建立从correlation_id到RPC上下文的全局哈希表。
	* -   取消RPC
* bthread_id包括两部分，一个是用户可见的64位id，另一个是对应的不可见的bthread::Id结构体

### **bthread是M:N的“协程”，每个bthread之间的平等的，**
* 即对称协程，不区分区分caller和callee
	* 微信开源的libco就是N:1的，libco属于非对称协程，区分caller和callee
* 实现M:N其中关键就是：**工作窃取（Work Stealing）算法**
* 三大件：TaskControl、TaskGroup、TaskMeta。以下简称TC、TG、TM。
	* TaskControl进程内全局唯一
	* TaskGroup和线程数相当，每个线程(pthread）都有一个TaskGroup，brpc中也将TaskGroup称之为 worker
		* bthread并不严格从属于一个pthread，但是bthread在运行的时候还是需要在一个pthread中的worker中（也即TG）被调用执行的。
	* TM基本就是表征bthread上下文的真实结构体了。
* 如果能获取到thread local的TG（tls_task_group），那么直接用这个tg来运行任务：start_background()。
	* 主要就是从资源池取出一个TM对象m，然后对他进行初始化，将回调函数fn赋进去，将fn的参数arg赋进去等等。
	* 另外就是调用make_tid，计算出了一个tid，这个tid会作为出参返回，也会被记录到TM对象中
		* tid可以理解为一个bthread任务的任务ID号。
* 如果获取不到TG，说明当前还没有bthread的上下文（TC、TG都没有），所以调用start_from_non_worker()函数，其中再调用get_or_new_task_control()函数，从而创建出一个TC来。

#### TaskControl::worker_thread()
* static函数（回调函数一般为static）
	* 普通成员函数是与对象实例绑定的。当你调用一个普通的成员函数时，需要一个对象实例作为调用者，隐式地带有一个 `this` 指针，指向调用该函数的对象。
	* **静态成员函数** 是属于类本身的，而不是属于某个对象实例。静态成员函数只能访问类的静态成员，不能直接访问非静态的成员变量或成员函数。
	* thread创建时 需要有一个开始任务的函数指针(void*)开启任务，函数指针的好处是允许你在程序运行时动态选择要执行的函数，而不是在编译时确定。而普通成员函数有一个额外的隐式参数，即 `this` 指针，指向该函数所属对象的实例。

#### mutex、futex和butex
* mutex
	* 当一个线程要进入临界区时，必须先获取 `mutex` 锁。成功获取锁的线程可以访问共享资源，其他线程则会被阻塞，直到该线程释放锁。
	* 实现依赖于 CPU 提供的原子操作指令（如 `Test-And-Set`、`Compare-And-Swap`），确保在多线程环境下锁的操作是安全的。
* futex
	* **用户态优先**：`Futex` 首先尝试在用户态完成锁的获取和释放操作。如果锁没有被竞争，所有的操作都在用户态完成，线程不会陷入内核。
	* 当锁被竞争或出现阻塞时，`futex` 会进入内核态
	* 但是futex仍然是线程粒度的阻塞
		* 如果协程阻塞了，当前线程就会切回内核等待，该线程上的所有协程都阻塞了，无法尝试获取锁
* butex
	* 集合futex优点，适合高并发场景。
	* 且是协程粒度的锁