[TOC]

## 1、Redis 有什么优点？

**1. 速度快**

因为数据存在内存中，类似于 HashMap ，HashMap 的优势就是查找和操作的时间复杂度都是O (1) 。

> Redis 本质上是一个 Key-Value 类型的内存数据库，很像Memcached ，整个数据库统统加载在内存当中进行操作，定期通过异步操作把数据库数据 flush 到硬盘上进行保存。
>
> 因为是纯内存操作，Redis 的性能非常出色，每秒可以处理超过 10 万次读写操作，是已知性能最快的 Key-Value 数据库。

**2. 支持丰富数据类型**

支持 String ，List，Set，Sorted Set，Hash 。

> Redis 的出色之处不仅仅是性能，Redis 最大的魅力是支持保存多种数据结构，此外单个 Value 的最大限制是1GB，不像 Memcached只能保存1MB的数据，因此Redis可以用来实现很多有用的功能。比方说：
>
> - 用他的 List 来做 FIFO 双向链表，实现一个轻量级的高性能消息队列服务。
> - 用他的 Set 可以做高性能的 tag 系统等等。

**3. 丰富的特性**

- 订阅发布 Pub / Sub 功能
- Key 过期策略
- 事务
- 支持多个 DB
- 计数
- …

并且在 Redis 5.0 增加了 Stream 功能，一个新的强大的支持多播的可持久化的消息队列，提供类似 Kafka 的功能。

**4. 持久化存储**

Redis 提供 RDB 和 AOF 两种数据的持久化存储方案，解决内存数据库最担心的万一 Redis 挂掉，数据会消失掉。

## 2、Redis 有什么缺点？

- 1、由于 Redis 是内存数据库，所以，单台机器，存储的数据量，跟机器本身的内存大小。虽然 Redis 本身有 Key 过期策略，但是还是需要提前预估和节约内存。如果内存增长过快，需要定期删除数据。

  > 另外，可使用 Redis Cluster、Codis 等方案，对 Redis 进行分区，从单机 Redis 变成集群 Redis 。

- 2、如果进行完整重同步，由于需要生成 RDB 文件，并进行传输，会占用主机的 CPU ，并会消耗现网的带宽。不过 Redis2.8 版本，已经有部分重同步的功能，但是还是有可能有完整重同步的。比如，新上线的备机。

- 3、修改配置文件，进行重启，将硬盘中的数据加载进内存，时间比较久。在这个过程中，Redis 不能提供服务。

## 3、Redis 和 Memcached 的区别有哪些？

**1. Redis 支持复杂的数据结构**

- Memcached 仅提供简单的字符串。
- Redis 提供复杂的数据结构，丰富的数据操作。

**2. Redis 原生支持集群模式**

- 在 Redis3.x 版本中，官方便能支持 Cluster 模式。
- Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据。

**3. 性能对比**

- Redis 只使用单核，而 Memcached 可以使用多核，所以平均每一个核上 Redis在存储小数据时比 Memcached 性能更高。
- 在 100k 以上的数据中，Memcached 性能要高于 Redis 。虽然 Redis 最近也在存储大数据的性能上进行优化，但是比起 Memcached，还是稍有逊色。

**4. 内存使用效率对比**

- 简单的 Key-Value 存储的话，Memcached 的内存利用率更高，可以使用类似内存池。
- 如果 Redis 采用 hash 结构来做 key-value 存储，由于其组合式的压缩， 其内存利用率会高于 Memcached 。

另外，Redis 和 Memcached 的内存管理方法不同。

- Redis 采用的是包装的 malloc/free ， 相较于 Memcached 的内存管理方法 tcmalloc / jmalloc 来说，要简单很多 。

**5. 网络 IO 模型**

- Memcached 是多线程，非阻塞 IO 复用的网络模型，原型上接近 Nignx 。
- Redis 使用单线程的 IO 复用模型，自己封装了一个简单的 AeEvent 事件处理框架，主要实现了 epoll, kqueue 和 select ，更接近 Apache 早期的模式。

**6. 持久化存储**

- Memcached 不支持持久化存储，重启时，数据被清空。
- Redis 支持持久化存储，重启时，可以恢复已持久化的数据。

## 4、请说说 Redis 的线程模型？

redis 内部使用文件事件处理器 `file event handler`，这个文件事件处理器是单线程的，所以 redis 才叫做单线程的模型。它采用 IO 多路复用机制同时监听多个 socket，根据 socket 上的事件来选择对应的事件处理器进行处理。

文件事件处理器的结构包含 4 个部分：

- 多个 socket
- IO 多路复用程序
- 文件事件分派器
- 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将 socket 产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。

来看客户端与 redis 的一次通信过程：

![redis-single-thread-model](http://www.iocoder.cn/images/Redis/2019_11_22/01.png)

- 客户端 socket01 向 redis 的 server socket 请求建立连接，此时 server socket 会产生一个 `AE_READABLE` 事件，IO 多路复用程序监听到 server socket 产生的事件后，将该事件压入队列中。文件事件分派器从队列中获取该事件，交给`连接应答处理器`。连接应答处理器会创建一个能与客户端通信的 socket01，并将该 socket01 的 `AE_READABLE` 事件与命令请求处理器关联。
- 假设此时客户端发送了一个 `set key value` 请求，此时 redis 中的 socket01 会产生 `AE_READABLE` 事件，IO 多路复用程序将事件压入队列，此时事件分派器从队列中获取到该事件，由于前面 socket01 的 `AE_READABLE` 事件已经与命令请求处理器关联，因此事件分派器将事件交给命令请求处理器来处理。命令请求处理器读取 socket01 的 `key value` 并在自己内存中完成 `key value` 的设置。操作完成后，它会将 socket01 的 `AE_WRITABLE` 事件与令回复处理器关联。
- 如果此时客户端准备好接收返回结果了，那么 redis 中的 socket01 会产生一个 `AE_WRITABLE` 事件，同样压入队列中，事件分派器找到相关联的命令回复处理器，由命令回复处理器对 socket01 输入本次操作的一个结果，比如 `ok`，之后解除 socket01 的 `AE_WRITABLE` 事件与命令回复处理器的关联。

这样便完成了一次通信。

http://www.cnblogs.com/barrywxx/p/8570821.html