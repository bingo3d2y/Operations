# select and poll and epoll

不管你系统单进程能打开的最大fd数是不是1024， 反正select就支持这么多。 不管你系统单进程能打开的最大fd数是不是1024， 反正poll/epoll支持的数量没有限制。

Nginx epoll 模型

