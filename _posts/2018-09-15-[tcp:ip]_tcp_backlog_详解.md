---
title: [TCP/IP] Tcp backlog 详解
data: 2018-09-15
categories:
- TCP/IP
tags:
- TCP/IP
- tcp
---

## Tcp backlog 详解

### 详解

> 在调用系统底层的`socket`方法`int listen(int sockfd, int backlog)`时，有一个`backlog`参数。如果想了解这个参数的作用，那么就需要回顾一下tcp的连接过程。

> ![](https://images2015.cnblogs.com/blog/927655/201612/927655-20161215133843776-605308204.png)
> 如图，在内核系统中维护了两个队列，分别是`syns queue`和`accept queue`。
> 
> * **syns queue**：用于保存半连接状态的请求（即状态为`SYN RECEIVED`的连接进入此队列）。其大小通过`/proc/sys/net/ipv4/tcp_max_syn_backlog`指定。
> * **accept queue**：用于保存全连接状态的请求（即当状态变更为`ESTABLISHED`时移到`accept`队列）。其大小为`/proc/sys/net/core/somaxconn`和使用`listen`函数时传入的参数的最小值。

> 知道了这两个队列，我们看一下连接过程：

> 1. `client`发送`SYN`到`server`，并将自身状态修改为`SYN_SEND`，如果`server`收到请求，则将状态修改为`SYN_RCVD`，并将该请求放入`syns queue`。
> 2. `server`回复`SYN+ACK`给`client`，如果`client`收到请求，则将状态修改为`ESTABLISHED`，并发送`ACK`给`server`。 
> 3. `server`收到`ACK`，将状态修改为`ESTABLISHED`，并把该请求从`syns queue`中放到`accept queue`。

> 到此我们知道`backlog`是用来控制`accept queue`长度的, 但是又出现了新的问题, `syns queue`和`accept queue`队列如果满了会发生什么情况?

> * `accept queue`已满, 经常发生在服务器或者服务器进程非常繁忙的情况下，进程没法足够快地调用`accept()`系统调用从中取出已完成连接。 此时一个已完成新连接需要从`syns queue`移动到`accept queue`, 即收到3次握手中最后一个`ACK`包, 那么会忽略这个包。`SYN RECEIVED`状态下有一个计时器实现：如果`ACK`包没有收到(或者是忽略的)，此时服务端无法判断是自己的`SYN+ACK`包丢失, 还是客户端的`ACK`包丢失, 因此协议栈会重发`SYN/ACK`包, 重试次数由`/proc/sys/net/ipv4/tcp_synack_retries`决定。若达到重试次数仍然未成功, 那么客户端会收到一个`RST`包宣告连接失败。
> * `syns queue`已满, 是`HTTP`服务器经常面临的问题，在客户端收到`SYN+ACK`后, 此时客户端变为`ESTABLISHED`, 此时为半开状态，若由于客户端恶意不响应`ACK`或者由于其他原因导致客户端无法收到`ACK`, 此时服务端无法判断是自己的`SYN+ACK`包丢失, 还是客户端的`ACK`包丢失, 因此协议栈会如上重发`SYN/ACK`包。 若需要维护的半开连接过多，会导致服务端无法正常服务。此时也就衍生出了`SYN Flood`攻击问题(SYN Flood 的具体问题可以参考下面列出的链接)。

> 大多数情况下`accept queue`都是空的，因为一旦有一个新连接进入队列，阻塞等待的`accept`系统调用将返回，然后连接从队列中取出。

### 参考文章

> * [浅谈tcp socket的backlog参数](https://blog.csdn.net/oyueyang1/article/details/80451535)
> * [[译文]深入理解Linux TCP backlog](https://www.jianshu.com/p/7fde92785056)
> * [从TCP三次握手说起——浅析TCP协议中的疑难杂症（真心不错）](https://blog.csdn.net/changyourmind/article/details/53127100)
> * [什么是SYN Flood攻击?](https://www.cnblogs.com/popduke/p/5823801.html#3ref)
> * [SYN Cookie的原理和实现](https://blog.csdn.net/zhangskd/article/details/16986931)