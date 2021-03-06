---
title: 分布式技术原理与算法解析——笔记
subtitle: 分布式技术原理与算法解析笔记
layout: post
tags: [分布式]
---

之前学习的分布式技术知识没有形成系统性的知识网络，趁着这个专栏好好梳理一下。

分布式技术的重要性不用多说，现在大部分的技术都是以分布式技术为基石。掌握了它，才能更好地掌握相关的、具体的技术实现原理，内在的设计驱动。

主要的内容如下：

四横：

- 分布式计算：解决用用的分布式计算问题。基于分布式计算模式，包括批处理任务计算、离线计算、在线计算、融合计算等，根据应用类型构建智能的分布式计算框架；
- 分布式数据存储与管理：解决数据的分布式和多远化问题。分布式数据库、分布式文件系统、分布式缓存，支持不同类型数据的存储和管理；
- 分布式通信：解决进程之间的分布式通信问题。通过消息队列、远程调用、订阅发布等方式，实现简单高效的通信；
- 分布式资源池化：解决资源的分布式和异构性问题。CPU、内存、GPU、网络等物理资源的虚拟化，形成逻辑资源池以便统一管理。

四纵：

- 分布式协同：解决分布式状态及数据的一致性问题。分布式互斥、分布式选举、分布式共识、分布式事务等；
- 分布式调度：解决资源与请求者的匹配问题。单体调度、双层调度、共享调度等。
- 分布式追踪与高可用：解决分布式定位、可靠性问题。分布式日志搜索、分布式问题建模、负载均衡、流量控制、故障隔离、故障恢复等等；
- 分布式部署：解决服务分布式部署问题。自动化、智能化部署。

在一定的资源上，进行一定通信，通过一定计算，完成一定数据的加工处理，从而提供对外服务。

四纵是四横的基础，无论是计算、存储、通信还是资源，都需要解决协同、调度、追踪高可用还有部署的问题。



## 分布式系统的性能指标



性能、资源、可用性和扩展性是分布式系统的重要指标。

### 性能（Performance）

主要用于衡量一个系统处理各种任务的能力。

不同的系统、服务要达成的目的不同，关注的性能自然也不尽相同，甚至互相矛盾。常见的戏能指标：

- 吞吐量（Throughput）
- 响应时间（Response Time）
- 完成时间（Turnaround Time）

#### 吞吐量

指系统在一定时间内可以处理的任务数。这个指标可以非常直观的体现一个系统的性能。常见的吞吐指标有QPS（Queries Per Second）、TPS（Transactions Per Second）和BPS（Bits Per Second）。

- QPS：查询数每秒，用于衡量一个系统每秒处理的查询数。这个指标通常用于读操作，越高表明对读操作的支持越好。所以，在设计一个分布式系统的时候，如果应用主要是读操作，那么重要的考虑是如何提高QPS，来支持高频的读操作。
- TPS：事务数每秒，用于很亮一个系统每秒处理的事务数。这个指标通常用于写操作，越高越好。如果一个应用的主要是写操作，那么需要重点考虑如何提高TPS，来支持高频写操作。
- BPS：比特数每秒，用于衡量一个系统每秒处理的数据量。杜宇一些网络系统、数据管理系统，我们不能简单的按照请求数或者事务数来衡量其性能。BPS更能反映系统的吞吐量。

#### 响应时间

指系统响应一个请求或者输入需要花费的时间。响应时间直接影响到用户体验，对于时延敏感的业务非常重要。

#### 完成时间

指系统真正完成一个请求或者处理需要花费的时间。任务并行模式的出现的其中一个目的，就是缩短整个任务的完成时间。



### 资源占用（Resource Usage）

