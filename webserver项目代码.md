####  1. Epoll 边沿触发模式（ET）
 ```cpp
// 设置epoll事件监听列表
struct epoll_event ev, events[MAX_EVENTS];
int epollfd = epoll_create1(0);
if (epollfd < 0) {
    perror("epoll_create1 failed");
    exit(EXIT_FAILURE);
}

ev.events = EPOLLIN | EPOLLET;  // 监听读事件，使用边沿触发
ev.data.fd = listen_fd;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) {
    perror("epoll_ctl: listen_sock");
    exit(EXIT_FAILURE);
}

while (true) {
    int nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
    for (int n = 0; n < nfds; ++n) {
        if (events[n].data.fd == listen_fd) {
            // 处理新的连接
        } else {
            // 处理数据读写
        }
    }
}
 ```
**解释**：这段代码初始化了一个epoll事件循环，用于非阻塞地监听和处理网络连接和数据传输。通过设置 `EPOLLIN | EPOLLET`，它使用边沿触发模式来提高处理效率，这意味着只有状态变化时才会触发事件，减少了不必要的事件处理。

#### 2. 线程池实现与管理
 ```cpp
template<typename Task>
class ThreadPool {
private:
    std::vector<std::thread> workers;
    std::queue<Task> tasks;

    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;

public:
    explicit ThreadPool(size_t threads) : stop(false) {
        for(size_t i = 0; i < threads; ++i)
            workers.emplace_back(
                [this] {
                    while (true) {
                        Task task;
                        {
                            std::unique_lock<std::mutex> lock(this->queue_mutex);
                            this->condition.wait(lock, [this]{ return this->stop || !this->tasks.empty(); });
                            if (this->stop && this->tasks.empty())
                                return;
                            task = std::move(this->tasks.front());
                            this->tasks.pop();
                        }
                        task();
                    }
                }
            );
    }
};
 ```
**解释**：这段代码创建了一个线程池，其中每个工作线程可以从任务队列中取出任务并执行。通过使用 `std::condition_variable`，它有效地让线程在没有任务时等待，从而节省了CPU资源。这种设计模式提高了服务器处理请求的能力，减少了线程创建和销毁的开销。

#### 3. 异步日志系统
 ```cpp
 class AsyncLogger {
private:
    std::mutex mutex_;
    std::condition_variable cond_;
    std::queue<std::string> bufferQueue_;
    std::thread loggingThread_;
    bool isRunning_;

    void run() {
        while (isRunning_) {
            std::unique_lock<std::mutex> lock(mutex_);
            cond_.wait(lock, [this]{ return !bufferQueue_.empty() || !isRunning_; });

            while (!bufferQueue_.empty()) {
                auto buffer = bufferQueue_.front();
                bufferQueue_.pop();
                // 写入日志到文件系统或其他媒介
                // logToFile(buffer);
            }
            lock.unlock();
        }
    }

public:
    AsyncLogger() : isRunning_(true), loggingThread_(&AsyncLogger::run, this) {}
    ~AsyncLogger() {
        if (isRunning_) {
            isRunning_ = false;
            cond_.notify_one();
            loggingThread_.join();
        }
    }

    void log(const std::string& message) {
        std::lock_guard<std::mutex> guard(mutex_);
        bufferQueue_.push(message);
        cond_.notify_one();
    }
};
  ```
  **解释**：这段代码展示了一个异步日志系统的实现。利用条件变量和互斥锁来同步数据，保证多线程环境下的线程安全。日志消息首先被加入到队列中，然后由后台线程统一写入到文件或其他媒介，这样可以减少对前端业务处理的干扰。


### 11. 同步异步
**回答**：
-   **同步**：调用方发起请求后等待响应，调用和响应在同一线程中完成。
-   **异步**：调用方发起请求后立即返回，响应在不同的线程或回调函数中完成。

### 12. 阻塞非阻塞
**回答**：
-   **阻塞**：调用方在请求资源时，会一直等待，直到资源可用。
-   **非阻塞**：调用方请求资源时，如果资源不可用，立即返回错误或状态，调用方可以继续其他操作。

### 13. 可以同步非阻塞吗
**回答**：
-   **可以**。例如，在非阻塞 I/O 中，可以通过轮询或回调实现同步逻辑。即使操作是非阻塞的，调用者可以主动等待完成，也就实现了同步行为。

### 15. epoll 是同步的还是异步的？
**回答**：
-   **epoll** 是同步的。它是事件驱动的，但调用 `epoll_wait` 时，仍然需要在用户线程中同步处理这些事件。