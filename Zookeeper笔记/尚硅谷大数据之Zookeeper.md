[TOC]

## 第一章zookeeper入门

### 1.1 概述

是一个开源的分布式协调服务。

从设计模式角度来理解，是一个基于观察者模式设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后接受观察者的注册，一旦这些数据的状态发送变化，zookeeper就讲负责通知已经在Zookeeper上注册的那些观察者做出相应的反应。

Zookeeper=文件系统+通知机制

### 1.2 特点

- 一个领导者（Leader），多个跟随者（Follower）组成的集群。
- 集群中只要有半数以上节点存活，Zookeeper集群就能正常服务。
- 全局数据一致性：每个server保存一份相同的数据副本，Client无论连接到哪个server，数据都是一致的。
- 更新请求顺序进行，来自同一个Client的更新请求按其发送顺序依次执行。
- 数据更新原子性，依次数据更新要么成功，要么失败。
- 实时性，在一定时间范围内，Client能读到最新数据。

### 1.3 数据结构

Zookeeper数据模型的结构与Unix文件系统很类似，整体上可以看作是一棵树，每个节点称作一个ZNode。每个ZNode默认能够存储1MB的数据，每个ZNode都可以通过其路径唯一标识。

### 1.4 应用场景

- 统一命名服务

  在分布式环境中，经常需要对应用/服务进行统一命名，便于识别。

  例如：IP不容易记住，而域名容易记住。

- 统一配置管理

  1）在分布式环境下，配置文件同步非常常见：

  ​	（1）一般要求一个集群中，所有节点的配置信息是一致的，比如Kafka集群。

  ​	（2）对配置文件修改后，希望能够快速同步到各个节点上。

  2）配置管理可交由Zookeeper实现。

  ​	（1）可将配置信息写入Zookeeper上的一个ZNode。

  ​	（2）各个客户端服务器监听这个ZNode。

  ​	（3）一旦ZNode中的数据被修改，Zookeeper将通知各个客户端服务器。

- 统一集群管理

  1）分布式环境中，实时掌握每个节点的状态是必要的。

  ​	（1）可根据节点实时状态做出一些调整。

  2）Zookeeper可以实现实时监控节点状态变化。

  ​	（1）可将节点信息写入Zookeeper上一个ZNode。

  ​	（2）监听这个ZNode可获取它的实时状态变化。

- 软负载均衡

  在Zookeeper中记录每台服务器的访问数，让访问数最少的服务器去处理最新的客户端请求。

### 1.5 下载地址



## 第二章Zookeeper安装

### 2.1 本地模式安装部署



### 2.2 配置参数解读

Zookeeper中的配置文件zoo.cfg中参数含义解读如下：

1. tickTime=2000：通信心跳数，Zookeeper服务器与客户端心跳时间，单位毫秒

   Zookeeper使用的基本时间，服务器之间或客户端之间维持心跳的时间间隔，也就是每个tickTime时间就会发送一个心跳，时间单位为毫秒。

   它用于心跳机制，并且设置最小的session超时时间为两倍心跳时间。（session的最小超时时间是2*tickTime）。

2. initLimit=10：LF初始通信时限

   集群中的Follower跟随者服务器与Leader领导者服务器之间初始化连接时能容忍的最多心跳数（tickTime的数量），用它来限定集群中Zookeeper服务器连接到 Leader的时限。

3. syncLimit=5：LF同步通信时限

   集群中Leader与Follower之间的最大响应时间单位，假如响应超过syncLimit*ticketTime，Leader认为Follower死掉，从服务器列表中删除Follower。

4. dataDir：数据目录文件+数据持久化路径

   主要用于保存Zookeeper中的数据。

## 第三章Zookeeper内部原理

### 3.1 选举机制 

1）半数机制：集群中半数以上的机器存活，集群可用，所以Zookeeper适合安装奇数台服务器。



### 3.2 节点类型

持久：客户端和服务器断开连接后，创建的节点不删除，除非我们主动删除。

临时：与客户端会话生命周期绑定，客户端与服务器连接断开后，创建的节点自动删除。

持久化顺序节点：

临时顺序节点：

### 3.3 Stat结构体

1）czxid：创建节点的事务zxid

​	每次修改Zookeeper状态都会收到一个zxid形式的时间戳，也就是Zookeeper事务ID。

​	事务ID是Zookeeper中所有修改总的次序，每个修改都有唯一的zxid，如果zxid1小于zxid2，那么zxid1在		 zxid2之前发生。

2）ctime：znode被创建的毫秒数（从1970年开始）。

3）mzxid：znode最后更新的事务zxid。

4）mtime：znode最后修改的毫秒数（从1970年开始）。

5）pzxid：znode最后更新的子节点zxid。

6）cvesion：znode子节点变化号，znode子节点修改次数。

7）dataversion：znode数据变化号。

8）aclVersion：znode控制访问列表的变化号。

9）ephemeralOwner：如果是临时节点，这个是znode拥有者的session id，如果不是临时节点则是0。

10）dataLength：znode的数据长度。

11）munChildren：znode子节点数量。

### 3.4 监听器原理



### 3.5 写数据流程



## 第四章Zookeeper实战

### 4.1 分布式安装部署



### 4.2 Shell命令操作

