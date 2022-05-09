# epoll网络模型

- epoll性能卓越，是高性能网络框架的基础（nginx，Redis，rpc，libevent）
- epoll，没有内存拷贝，没有轮询，没有遍历

## epoll的3个主要函数

- 创建句柄：`int epoll_create(int size);`.epoll句柄可以理解为socket的集合，就像select中的bitmap和poll中的数组一样。
- 注册事件：`int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);`，意思就是告诉epoll需要监视哪些socket的哪些事件
- 等待事件：`int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);`,epoll事件返回之后，后续的处理和select和poll也是类似的。

