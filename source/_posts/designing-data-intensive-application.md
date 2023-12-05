---
title: 读书笔记-数据密集型应用系统设计（一）
date: 2023-02-22 22:28:29
tags:
    - DB
categories:
    - DB
---

# 系统的核心设计目标
三个目标:
## 可靠(Reliability)
    >发生了某种意外，系统依旧可以正常运转
    可靠性是软件的一个保证，是一种态度，如果不能保证，那么会对营收和声誉都造成很大的影响

如何保证可靠性?有若干方法:
1. 用最小的方法来设计系统。通过精心设计的抽象层、API以及管理界面，是做正确的事情很轻松，做错误的事情很复杂
2. 对容易出现错的地方进行分离。提供一个真实的沙箱环境，导入真实的数据，能够完成快速的切换
3. 测试。重中之重
4. 出现人为失误，系统能够快速的恢复，减少故障影响。通过版本管理，能够实现版本快速回滚，并发布新代码上线
5. 监控系统。对错误能够快速的掌握和把控
6. 管理流程。

## 可扩展(Scalability)
    如果系统工程以某种方式增长，我们应对增长的措施有哪些？

1. 描述负载: 在一切开始前，我们需要能够正确的描述出我们系统所能承载的负载情况。然后才能正确的作出合理的解决方法。
2. 描述性能: 对于系统所需要面对的压力，我们需要能够得到请求各个指数，知道各个环节是什么情况。
   1. 延迟: 服务器处理时间
   2. 响应时间: 对应性能最直观的参数体现，包含整个流程
   3. 中位数: 优先拿取响应时间的中位数来分析
   4. 平均值: 无法了解用户具体的请求情况，因为是已经处理的数据
3. 应对方法: 

## 可维护(Maintainability)
1. 可运维性: 运维更简易
2. 简单性: 简化复杂度。只有简化，才能让我们更加快速的进行维护迭代
3. 可演变性: 易于改变。面对日益增长的需求，能够快速迭代也是需要考量的一环

## 数据模型
1. 文档模型
2. 关系模型: 基于sql这个模型。数据被组织成关系，在sql中称为表，其中每个关系都是元祖的无序集合。
3. 层次模型: 将所有数据表现为嵌套在记录中的记录，与json结构类似
4. 网络模型: 层次模型的推广。但是更新和查询变得异常的复杂而没有灵活性，层次模型也是。

### 文档数据库与关系数据
1. 文档数据库: 模式灵活性，更加符合程序的数据库
2. 关系数据库: 联结操作，多对一，一对多这种情况下，能够使得数据更加清晰