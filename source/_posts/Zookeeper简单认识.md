---
title: Zookeeper简单认识
date: 2021-04-13 16:33:09
tags:
    - Java
    - Zookeeper
categories:    
    - Zookeeper
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/zookeeper.jpg
---
## Zookeeper简单认识
### 什么是Zookeeper？
ZooKeeper 是一种分布式协调服务，用于管理大型主机。在分布式环境中协调和管理服务是一个复杂的过程。ZooKeeper 通过其简单的架构和 API 解决了这个问题。ZooKeeper 允许开发人员专注于核心应用程序逻辑，而不必担心应用程序的分布式特性。

### Zookeeper的角色

- **Leader**：领导者，核心，事务请求的唯一调度者保证事务集群处理的顺序性，集群内各个服务的调度者
- **Follower**：跟随着，处理客户的非事务请求，事务请求则交给领导者处理，参与领导选举投票
- **Observer**：观察者，观察Zookeeper的状态，能够独立处理非事务请求，不参与领导选举投票

### Zookeeper的特性

- 全局数据的一致：每个 server 保存一份相同的数据副本，client 无论链接到哪个 server，展示的数据都是一致的
- 可靠性
- 顺序性
- 数据更新原子性
- 实时性：ZooKeeper 保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息

### Zookeeper的好处

- 简单的分布式协调过程
- 同步
- 有序的信息
- 序列化
- 可靠性
- 原子性

### 层次命名空间

ZooKeeper节点称为 znode 。每个znode由一个名称标识，并用路径(/)序列分隔

1. 根节点下有两个逻辑命名空间config和workers
2. config 命名空间用于集中式配置管理，workers 命名空间用于命名
3. 在config命名空间下，每个znode最多可存储1MB的数据。这与UNIX文件系统相类似，除了父znode也可以存储数据。这种结构的主要目的是存储同步数据并描述znode的元数据。此结构称为**ZooKeeper数据模型**。

Znode兼具文件和目录两种特点。既像文件一样维护着数据长度、元信息、ACL、时间戳等数据结构，又像目录一样可以作为路径标识的一部分。每个Znode由三个部分组成：

- **stat**：此为状态信息，描述该Znode版本、权限等信息。
- **data**：与该Znode关联的数据
- **children**：该Znode下的节点

Znode的其它信息：

- **版本号**： 每个znode都有版本号，这意味着每当与znode相关联的数据发生变化时，其对应的版本号也会增加。当多个zookeeper客户端尝试在同一znode上执行操作时，版本号的使用就很重要。
- **操作控制列表**(ACL)：ACL基本上是访问znode的认证机制。它管理所有znode读取和写入操作。
- **时间戳**：时间戳表示创建和修改znode所经过的时间。它通常以毫秒为单位。ZooKeeper从“事务ID"(zxid)标识znode的每个更改。Zxid 是唯一的，并且为每个事务保留时间，以便你可以轻松地确定从一个请求到另一个请求所经过的时间。
- **数据长度**：存储在znode中的数据总量是数据长度。你最多可以存储1MB的数据。

### Znode的类型
Znode被分为持久（persistent）节点，顺序（sequential）节点和临时（ephemeral）节点。

- **持久节点**：即使在创建该特定znode的客户端断开连接后，持久节点仍然存在。默认情况下，除非另有说明，否则所有znode都是持久的。
- **临时节点**：客户端活跃时，临时节点就是有效的
- **顺序节点**：顺序节点可以是持久的或临时的

参考：https://www.w3cschool.cn/zookeeper/zookeeper_overview.html
