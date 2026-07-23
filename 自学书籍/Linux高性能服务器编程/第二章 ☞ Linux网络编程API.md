## 1.Linux网络编程基础API

*socket原义（ip，port） 这样的ip和端口 对，唯一的表示了通信的一段，这就是socket地址

####  socket地址API
######  字节序——分为大端小端
	Linux提供API来进行字节序转化——htonl表示“host to network long”，即长整形数据（32bit）主机字节序转为网络字节序
* 大端字节序——高位字节在低地址
	* 为统一发送数据总是转为大端字节序，称为网络字节序
* 小端字节序——低位字节在高地址
	* 现代pc多采用小端字节， 所以称为主机字节序

##### 通用socket地址
	scoket地址的结构体是 sockaddr,sa_family成员是地址族类型，__ss_padding成员用于存放socket地址值，__ss_align用于内存对齐
	struct sockaddr_storage
	{
	sa_family_t sa_family;
	unsigned long int__ss_align;
	char__ss_padding[128-sizeof(__ss_align)];
	}

####  socket基础API

###### 创建socket —— int socket(int domain,int type,int protocol);
	Linux的哲学是 所有东西都是文件。 socket就是可读可写可控制可关闭的文件描述符。
		domain表示哪个协议族，对于TCPIP使用PF_INET；type指定服务类型，主要有socket_stream流服务和socket_urgam(数据报服务)，对于TCPIP协议族前者代表传输层TCP，后者代表UDP。
scoket调用成功则返回一个socket文件描述符，失败则返回-1，并设置errno。

###### 命名socket —— int bind(int sockfd,const struct sockaddr* my_addr,socklen_t* addrlen);
	bind将my_addr所指地址分配给未命名的文件描述符sockfd， adddrlen指出该地址的长度
* bind成功时返回0，失败则返回-1并设置errno。两种errno：
	* eacces 被绑定的地址是受保护的地址。比如知名服务端口（0-1024
	* eaddrinuse 被绑定的地址正在被使用。比如绑到了一个正在处于TIME_wait状态的scoket地址

*将一个socket与socket地址绑定称为给socket命名

在服务器程序中，只有命名socket，客户端才能知道如何连接他。
但是在客户端中，无须命名socket，而是匿名的方式使用操作系统自动分配的socket地址。

###### 监听socket——int listen(int sockfd,int backlog);
*socket被命名之后，还不能马上接受客户连接。我们需要创建一个监听队列以存放待处理的客户连接请求。
	backlog参数指明内核监听队列的最大长度。 如果超过长度，服务器将不受理新客户连接，客户端也将收到econnrgefuse错误信息。
* listen成功时返回0，失败则返回-1并设置errno。

###### 接受连接——int accept(int sockfd,struct sockaddr* addr,socklen_t* addrlen);
*服务器通过系统调用从listen监听队列中接受一个连接
	sockfd是经过监听的socket；addr用来获取被接受连接的远程socket地址；该地址长度由addrlen指出。

* accept成功时返回一个新的socket，这个socket唯一的标识了这个连接。服务器可以通过读写这个socket来与被接受连接的客户端通信。 accept失败则返回-1并设置errno
* accept只是从监听队列中取出连接，无论连接处于何种状态，也不关心网络变化。
* 处于established状态的socket称为连接socket

##### 发起连接——int connect(int sockfd,const struct sockaddr* serv_addr,socklen_t addrlen);
*客户端则需要主动调用connect发起连接
	sockfd是socket调用返回的socket；ser_addr是服务器监听的socket地址；addrlen是这个地址长度
* connet成功则返回0；客户端可以通过读写这个sockfd来与服务器通信。失败则返回-1，设置errno：
	* ECONNREFUSED 目标端口不存在
	* ETIMEDOUT，连接超时
	

##### 关闭连接——int close(int fd);
*关闭一个连接实际上就是关闭该连接对应的socket
	fd参数是待关闭的socket。但是close调用并非立即关闭连接，而是fd-1，当fd为0时才关闭。
* 多进程程序中，一次fork系统调用默认将父进程打开的socket引用计数+1。故需要在父进程子进程都执行close操作关闭才可以。
* 如果必须立即关闭则使用 ——**int shutdown(int sockfd,int howto);
	* howto参数决定了shutdown行为。
	* 可以关闭读、关闭写、和读写同时关闭。
	* 成功返回0，失败返回-1

##### 数据读写
* 读操作——**ssize_t recv(int sockfd,void* buf,size_t len,int flags);
	* buf和len指定读缓冲区位置和大小
	* 成功则返回实际读取长度
	* 返回0表示对方关闭连接
	* 返回-1出错
* 写操作——**ssize_t send(int sockfd,const void* buf,size_t len,int flags);
	* 成功时返回实际写入数据长度

#### 网络信息API



## 2.高级IO函数
#### 创建文本描述符的函数
* pipe函数 ——**int pipe(int fd[2]);
	* 用于创建一个管道，实现进程间的通信
	* fd的两个文件描述符构成管道的两端。fd[0]只能用于从管道读出数据，fd[1]则只能用于往管道写入数据
	* 创建双向管道——**int socketpair(int domain,int type,int protocol,int fd[2]);
* dup函数——**int dup(int file_descriptor);
	* 创建一个新的文件描述符，该新文件描述符和原有文件描述符file_descriptor指向相同的文件、管道或者网络连接。
* readv函数——**ssize_t readv(int fd,const struct iovec* vector,int count)；
	* 将数据从文件描述符读到分散的内存块中
	* iovec结构体描述一块内存区，vector结构体数组则是多块；count参数是vector数组的长度
	* 成功时返回读出fd的字节数
	* 失败则返回-1.并设置error
* writev函数——**ssize_t writev(int fd,const struct iovec*vector,int count);
	* 将多块分散的数据块一并写入到文件描述符

## 3.服务器程序规范
* 通常以后台进程形式运行。后台进程=守护进程，父进程通常是pid=1的init进程
* 有一套日志系统，输出日志到文件
* 一般以某个专门的非root身份运行
* 可配置
* 启动时生成一个PID文件以记录后台进程的PID
* 需要考虑资源限制。如可用文件描述符总数和内存总量

##### 日志
* rsyslogd守护进程既能接收用户进程输出的日志，又能接收内核日志
* 设置日志掩码，使日志级别大于日志掩码的日志信息被系统忽略

###### 用户信息
* 真实用户ID（UID）、有效用户ID（EUID）、真实组ID（GID）和有效组ID（EGID）

##### 进程间的关系
*Linux下每个进程都隶属于一个进程组，因此它们除了PID信息外，还有进程组ID（PGID）。
* 获取指定进程的PGID——**pid_t getpgid(pid_t pid);
* 设置PGID——**int setpgid(pid_t pid,pid_t pgid);
* 每个进程组都有一个首领进程，其PGID和PID相同。进程组将一直存在，直到其中所有进程都退出，或者加入到其他进程组。

#### 会话
*一些有关联的进程组将形成一个会话（session）
* 创建会话——**pid_t setsid(void);
* ps命令可查看进程、进程组和会话之间的关系。ps和less命令的父进程是bash命令
