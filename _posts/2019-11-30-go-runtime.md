---
title: 深入理解GO———runtime
subtitle: go runtime
tags: [go,runtime]
layout: post
---

[转载地址](https://mp.weixin.qq.com/s/5l4EG5fUEYBlbQMsegmL1g)

# golang runtime

基于2019.02发布的go 1.12 linux amd64版本, 主要介绍了Runtime一些原理和实现的一些细节：

1. **Golang Runtime是个什么?** Golang Runtime的发展历程, 每个版本的改进
2. **Go调度**: 协程结构体, 上下文切换, 调度队列, 大致调度流程, 同步执行流又不阻塞线程的网络实现等
3. **Go内存**: 内存结构, mspan结构, 全景图及分配策略等
4. **Go GC**: Golang GC停顿大致的一个发展历程, 三色标记实现的一些细节, 写屏障, 三色状态, 扫描及元信息, 1.12版本相对1.5版本的改进点, GC Pacer等
5. **实践**: 观察调度, GC信息, 一些优化的方式, 几点问题排查的思路, 几个有意思的问题排查
6. **总结**: 贯穿Runtime的思想总结



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesqBJiaZW3qsMSSDGJibD8sXjU1ayMJYZV6TZJHldq36fLm1U9EicDoRv3g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 为什么去了解runtime呢?

1. **可以解决一些棘手的问题**: 在写这个PPT的时候, 就有一位朋友在群里发了个pprof图, 说同事写的代码有问题, CPU利用率很高., 找不出来问题在哪, **我看了下pprof图, 说让他找找是不是有这样用select的**, 一查的确是的. 平时也帮同事解决了一些和并发, 调度, GC有关的问题
2. **好奇心**: 大家写久了go, 惊叹于它的简洁, 高性能外, 必然对它是怎么实现的有很多好奇. 协程怎么实现, GC怎么能并发, 对象在内存里是怎么存在的? 等等
3. 技术深度的一种 

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesnXySYJB2QBha8sgaJVjiaTXKswzX4ujW1snyJPric9zdAjaaAhricoYSQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesK4Hu4fGDueqRM8JU2iaoaRZYsLxeeDot4jNfiaVVHl2MU4wA6EyMLQhw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesCbiaS8m4Wz6JSgP0K3rV3vorn1wcB10A7bFeekAaAIAkQynEJAOaT8Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**go的runtime代码在go sdk的runtime目录下**. 

主要有所述的4块功能. 提到runtime, 大家可能会想起java, python的runtime. 

不过go和这两者不太一样：

- java, python的runtime是虚拟机, 
- go的runtime和用户代码一起编译到一个可执行文件中. 

用户代码和runtime代码除了代码组织上有界限外, 运行的时候并没有明显的界限. 

如上所示, 一些常用的关键字被编译成runtime包下的一些函数调用.

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesMLO6V2DMfKA4kSrLOJA9I6G8fTMN2e7ZZBBcttxvncAonoBWia5busg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

左边标粗的是一些更新比较大的版本. 右边的GC STW仅供参考.

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesBSmg3SkkibwFvO8XUFLiblJnBXOUg68rianR47QBje6BPXKq5fUvBTDUQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesVHmbZeHkgxibhiajwwmdJlw3IH4BrqO0pSkD0bYSp0LTibGXslNvq2a9g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesG35yPug4bM3WSAQsWt8c6LasQ7JbwJZcwI7fb0nvibjJP2Cv82creQQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**Goroutine是一种用户态线程**。不断的共享，不断的减少切换成本。（可参考现代操作系统的文章，线程的实现：内核态、用户态）。

**我们去看调度的一个进化, 从进程到线程再到协程, 其实是一个不断共享, 不断减少切换成本的过程**. 

**go实现的协程为有栈协程**, **go协程的用法和线程的用法基本类似**. 

很多人会疑问, 协程到底是个什么东西? 用户态的调度感觉很陌生, 很抽象, 到底是个什么东西?

要理解调度, 要理解两个概念: **运行**和**阻塞**.

 特别是在协程中, 这两个概念不容易被正确理解. 

我们理解概念时往往会代入自身感受, 觉得线程或协程运行就是像我们吭哧吭哧的处理事情, 线程或协程阻塞就是做事情时我们需要等待其他人, 然后就在这等着了. 要是其他人搞好了, 那我们就继续做当前的事. **其实主体对象搞错了**. 

**正确的理解应该是我们处理事情时就像CPU, 而不是像线程或者协程**. 假如我当前在写某个服务, 发现依赖别人的函数还没有ready, 那就把写服务这件事放一边. 点开企业微信, 我去和产品沟通一些问题了. 我和产品沟通了一会后, 检查一下, 发现别人已经把依赖的函数提交了, 然后我就最小化企业微信, 切到IDE, 继续写服务A了.

对操作系统有过一些了解, 知道**linux下的线程其实是task_struct结构**, **线程其实并不是真正运行的实体, 线程只是代表一个执行流和其状态**。**真正运行驱动流程往前的其实是CPU**. 

CPU在时钟的驱动下, 根据PC寄存器从程序中取指令和操作数, 从RAM中取数据, 进行计算, 处理, 跳转, 驱动执行流往前。 CPU并不关注处理的是线程还是协程, 只需要设置PC寄存器, 设置栈指针等(这些称为上下文), 那么CPU就可以欢快的运行这个线程或者这个协程了。

线程的运行, 其实是被运行。 其阻塞, 其实是切换出调度队列, 不再去调度执行这个执行流。其他执行流满足其条件, 便会把被移出调度队列的执行流重新放回调度队列。

**协程同理, 协程其实也是一个数据结构, 记录了要运行什么函数, 运行到哪里了**。

**go在用户态实现调度, 所以go要有代表协程这种执行流的结构体, 也要有保存和恢复上下文的函数, 运行队列.** 理解了阻塞的真正含义, 也就知道能够比较容易理解, 为什么go的锁, channel这些不阻塞线程. 对于实现的同步执行流效果, 又不阻塞线程的网络, 接下来也会介绍.

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesiaUS9iaFubwRw1BUY8h2DicYhHaZqnKVb7xNg3vdSqY9qOLYPFHmgf1Jw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



我们go一个func时一般这样写

```go
go func1(arg1 type1,arg2 type2){....}(a1,a2)
```

**一个协程代表了一个执行流**, **执行流有需要执行的函数(对应上面的func1), 有函数的入参(a1, a2), 有当前执行流的状态和进度(对应CPU的PC寄存器和SP寄存器), 当然也需要有保存状态的地方, 用于执行流恢复**. 

真正代表协程的是runtime.g结构体. 每个go func都会编译成runtime.newproc函数, **最终有一个runtime.g对象放入调度队列**. 

上面的func1函数的指针设置在runtime.g的startfunc字段, 参数会在newproc函数里拷贝到stack中, sched用于保存协程切换时的pc位置和栈位置. 协程切换出去和恢复回来需要保存上下文, 恢复上下文, 这些由以下两个汇编函数实现. 以上就能实现协程这种执行流, 并能进行切换和恢复. (下图中的struct和函数都做了精简)

## GM模型

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesNb23ibDcUKibwjicwhvwMZFGAp7H73NEk3U7oEno3LJwTVHnzpMnia32qA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

有了协程的这种执行流形式, 那待运行的协程放在哪呢? 在Go1.0的时候:

1. **调度队列schedt是全局的**, **对该队列的操作均需要竞争同一把锁, 导致伸缩性不好**.
2. **新生成的协程也会放入全局的队列, 大概率是被其他m(可以理解为底层线程的一个表示)运行了, 内存亲和性不好.** 当前协程A新生成了协程B, 然后协程A比较大概率会结束或者阻塞, 这样m直接去执行协程B, 内存的亲和性也会好很多.
3. **因为mcache与m绑定, 在一些应用中(比如文件操作或其他可能会阻塞线程的系统调用比较多), m的个数可能会远超过活跃的m个数, 导致比较大的内存浪费.**.

那是不是可以给m分配一个队列, 把阻塞的m的mcache给执行go代码的m使用? Go 1.1及以后就是这样做的.

## GPM模型

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesUG3FhZkIXysRR3TvJeicoYtiaX6nwmyR9r9sQBxrzxvdgmL1TotVlceg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在1.1中调度模型更改为**GPM模型**, 引入**逻辑Process的概念**, **表示执行Go代码所需要的资源, 同时也是执行Go代码的最大的并行度**. 这个概念可能很多人不知道怎么理解.

 P涉及到几点, 队列和mcache, 还有P的个数的选取. 

首先**为什么把全局队列打散**, 以及**mcache为什么跟随P**, 这个在GM模型那一页就讲的比较清楚了. 

然后**为什么P的个数默认是CPU核数**: Go尽量提升性能, 那么在一个n核机器上, 如何能够最大利用CPU性能呢? 当然是同时有n个线程在并行运行中, 把CPU喂饱, 即所有核上一直都有代码在运行.

 **在go里面, 一个协程运行到阻塞系统调用, 那么这个协程和运行它的线程m, 自然是不再需要CPU的, 也不需要分配go层面的内存.** 

**只有一直在并行运行的go代码才需要这些资源, 即同时有n个go协程在并行执行, 那么就能最大的利用CPU, 这个时候需要的P的个数就是CPU核数. (注意并行和并发的区别)**

## 协程状态及流转

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesS2Y1aG3tIhDQzZQvukhwU4LtlZwPWHBydRiab0ttTWlkM5Hib0PRwHKQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

协程的状态其实和线程状态类似,状态转换和发生状态转换的时机如图所示. 还是需要注意: **协程只是一个执行流, 并不是运行实体.**



### **调度**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesadxyJHssdfYoemU6rSG1h1PxP8sRvezGN6WmTicYLVWdytJoZ6pBVPw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

并没有一个一直在运行调度的调度器实体. 当一个协程切换出去或新生成的m, go的运行时从stw中恢复等情况时, 那么接下来就需要发生调度. go的调度是通过线程(m)执行runtime.schedule函数来完成的.

### sysmon协程

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujes2A0Jl5lLksnkrPAEYl8QL3eHp9rb4LtNpEzAtP6lxwUCuQpzqxOpXg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在linux内核中有一些执行定时任务的线程, 比如定时写回脏页的pdflush, 定期回收内存的kswapd0, 以及每个cpu上都有一个负责负载均衡的migration线程等. 

在go运行时中也有类似的协程, sysmon. 功能比较多: 定时从netpoll中获取ready的协程, 进行抢占, 定时GC,打印调度信息,归还内存等定时任务

## 协作式抢占

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesZmwFaIiaJDia6s3ygpicShkWUKCO1qZSPLkcBq0aSPLbsUDUKh8ibJy7pA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

go目前(1.12)还没有实现非协作的抢占. **基本流程是sysmon协程标记某个协程运行过久, 需要切换出去, 该协程在运行函数时会检查栈标记, 然后进行切换.**



## 同步执行流不阻塞线程的网络的实现

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesGLsnvMpF3Sh6J1GfufmNZrDd9bTWPTBO0sMn0VjG2khD2K0qW99pDg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**go写后台最舒服的就是能够以同步写代码的方式操作网络, 但是网络操作不阻塞线程**. 

主要是结合了**非阻塞的fd**, **epoll**以及**协程的切换和恢复**. 

**linux提供了网络fd的非阻塞模式**, **对于没有ready的非阻塞fd执行网络操作时, linux内核不阻塞线程, 会直接返回EAGAIN, 这个时候将协程状态设置为wait, 然后m去调度其他协程**. 

**go在初始化一个网络fd的时候, 就会把这个fd使用epollctl加入到全局的epoll节点中. 同时放入epoll中的还有polldesc的指针**.

```go
func netpollopen(fd uintptr, pd *pollDesc) int32 {var ev epollevent    ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET*(**pollDesc)(unsafe.Pointer(&ev.data)) = pdreturn-epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)}
```

**在sysmon中, schedule函数中, start the world中等情况下, 会执行netpoll 调用epollwait系统调用**.

把**ready的网络事件从epoll中取出来**, **每个网络事件可以通过前面传入的polldesc获取到阻塞在其上的协程, 以此恢复协程为runnable**.



## 调度相关结构体

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujes3HCPAHvQPvLwBH2gDM9FmLv3ibM1KQKakOZVPerXicichrVkGLOr3Mrfg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 调度综述

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesejicmZfXKJIf05G5vKfWOicHvgSKv5D2qbn8FZUMv2dHgLic6x2JNfuaQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesb3xbc9ZP5k5WESN6V673MMOcWx6ClQ7FwOs3RZribrnNkrCe3oj3lTg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

# 内存分配

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujeszEicLjoHmvicvBePcka8hItascIDXRCewhSZf0kYD4TelkIvdFK0me2Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 内存分配简介

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesaMgHUa3ibxtFCzgzeDITm06jI0Z3Kic5ibI3hu8CZgICHibfJncGGdLjuA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**Go的分配采用了类似tcmalloc的结构.** 

**特点**: 

- **使用一小块一小块的连续内存页, 进行分配某个范围大小的内存需求**. 比如某个连续8KB专门用于分配17-24字节,以此减少内存碎片. 

- **线程拥有一定的cache, 可用于无锁分配**.

- **同时Go对于GC后回收的内存页, 并不是马上归还给操作系统, 而是会延迟归还, 用于满足未来的内存需求**.



## 内存空间结构

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesfia0SxKTAjWBS85yPxCjBLHOWdgukEfbKklqKEib82nqFAS37Zb4CLgA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在1.10以前go的堆地址空间是线性连续扩展的, 比如在1.10(linux amd64)中, 最大可扩展到512GB. 

因为go在gc的时候会根据拿到的指针地址来判断是否位于go的heap的, 以及找到其对应的span, 其判断机制需要gc heap是连续的. 

但是连续扩展有个问题, cgo中的代码(尤其是32位系统上)可能会占用未来会用于go heap的内存. 这样在扩展go heap时, mmap出现不连续的地址, 导致运行时throw. 

**在1.11中, 改用了稀疏索引的方式来管理整体的内存. 可以超过512G内存, 也可以允许内存空间扩展时不连续.** 

**在全局的mheap struct中有个arenas二阶数组**, 在linux amd64上,一阶只有一个slot, 二阶有4M个slot, 每个slot指向一个heapArena结构, 每个heapArena结构可以管理64M内存, 所以在新的版本中, go可以管理4M*64M=256TB内存, 即目前64位机器中48bit的寻址总线全部256TB内存.

###  **span机制**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesOYR1U9Oplj9CNqvTTEkWFRSpxhrJneQr7ozVKfYwzXVW9ISdcQM3Ew/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

前面提到了**go的内存分配类似于tcmalloc**, **采用了span机制来减少内存碎片**. 

**每个span管理8KB整数倍的内存, 用于分配一定范围的内存需求.**



### **内存分配全景**



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesHsMydVkRs6Bykx0srVZ7ybWYNYX2wEQ9rNCnnl49MHiczQEGlyWd22w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

多层次的分配Cache, 每个P上有一个mcache, mcache会为每个size最多缓存一个span, 用于无锁分配. 

**全局每个size的span都有一个mcentral, 锁的粒度相对于全局的heap小很多, 每个mcentral可以看成是每个size的span的一个全局后备cache.** 

在**gc完成后, 会把P中的span都flush到mcentral中, 用于清扫后再分配**. P有需要span时, 从对应size的mcentral获取. 获取不到再上升到全局的heap.



**几种特殊的分配器**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesm9naeSuPVDTpHUZuMPLdAD0u7r6huBg2xLNPIDw96kJbv0fm82IicGA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

- 对于很小的对象分配, go做了个优化, 把小对象合并, 以移动指针的方式分配. 

- 对于栈内存有stackcache分配, 也有多个层次的分配, 同时stack也有多个不同size. 

用于分配stack的内存也是位于go gc heap, 用mspan管理, 不过这个span的状态和用于分配对象的mspan状态不太一样, 为mSpanManual. 

我们可以思考一个问题, go的对象是分配在go gc heap中, 并由mcache, mspan, mcentral这些结构管理, 那么mcache, mspan, mcentral这些结构又是哪里管理和分配的呢? 

肯定不是自己管理自己. 这些都是由特殊的分配fixalloc分配的, 每种类型有一个fixalloc, 大致原理就是通过mmap从进程空间获取一小块内存(百KB的样子), 然后用来分配这个固定大小的结构.



**内存分配综合**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesTQdw8uQQT5I3zZzDboUAGaicibdTEhkzRzgUmojclsngzxDO6bxE11Ag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**GC**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesicrtqPsRqqhmCY1icPpg9x6ic0ILj8TtSsY8BTlfEQia3FDr3CznKEA7Ew/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**Golang GC简述**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesGkg2okQAmBK4hKlJ69Zia2ULF5vomZT4puqzx1JScgwAj3eA20TKICg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**GC简介**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesvMfBB4CiawVrOtjMdUWus6DqSQA8V5ySM5HrnvibFIvZf1MaMb8icsjbw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

GC并不是个新事物, 使得GC大放光彩的是Java语言.

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujeslvZZk99bExB462vWS5mxCmNfFSfgdufbzJo2BiajfZ1j4fWJibwnksCw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesNtgOYxytD4sCDvyVxnwhIdMDYJfSZuPXt2wtJpXMO5PXZsE7cWlNjg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## Golang GC发展

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesFK6icxiceT3Kos0ibuVqUWZicqg5JP4rUsNnjgicxd0iahoGq8smeBQQ7PAg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

上面是几个比较重要的版本. 左图是根据twitter工程师的数据绘制的(堆比较大), 从1.4的百ms级别的停顿到1.8以后的小于1ms. 

右图是我对线上服务(Go 1.11编译)测试的一个结果, 是一个批量拉取数据的服务, 大概3000qps, 服务中发起的rpc调用大概在2w/s. 可以看到大部分情况下GC停顿小于1ms, 偶尔超过一点点. 

整体来说golang gc用起来是很舒心的, 几乎不用你关心.

### **三色标记**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesWshDIJ8xiatpmZUSVYlIb3I2xdcpvgqFT9WoAZBCVSic4JDgpVRSVXjQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

go采用的是并发三色标记清除法. 

图展示的是一个简单的原理.

有几个问题可以思考一下: 

- 并发情况下, 会不会漏标记对象? 

- 对象的三色状态存放在哪? 

- 如何根据一个对象来找到它引用的对象?

### **写屏障**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesL9ib2XxJwtY6PudAmDuMaVIicpsFdOsCzIPapjl6k0D8D8PjRM3FfgSg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

GC最基本的就是正确性: 不漏标记对象, 程序还在用的对象都被清除了, 那程序就错误了. 有一点浮动垃圾是允许的. 

在并发情况下, 如果没有一些措施来保障, 那可能会有什么问题呢? 

看左边的代码和图示, 第2步标记完A对象, A又没有引用对象, 那A变成黑色对象. 

在第3步的时候, muator(程序)运行, 把对象C从B转到了A, 

第4步, GC继续标记, 扫描B, 此时B没有引用对象, 变成了黑色对象. 我们会发现C对象被漏标记了.

**如何解决这个问题?**

 go使用了写屏障, 这里的**写屏障是指由编译器生成的一小段代码.** 在**gc时对指针操作前执行的一小段代码**, 和CPU中维护内存一致性的写屏障不太一样哈. 所以有了写屏障后, 第3步, A.obj=C时, 会把C加入写屏障buf. 最终还是会被扫描的.



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujessAFmCZEOnynibvK4MKKj7WbrqZOsydFsZquanhkATkMHaTzcicvwD0qg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这里感受一下写屏障具体生成的代码. 

我们可以看到在写入指针slot时, 对写屏障是否开启做了判断, 如果开启了, 会跳转到写屏障函数, 执行加入写屏障buf的逻辑. 

1.8中写屏障由Dijkstra写屏障改成了混合式写屏障, 使得GC停顿达到了1ms以下.



**三色状态**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesLXCwQnWaJ9ZibbZice43picibQicpah9EpNshgT5gsSaZyUwdxjpmjpQ0BA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

并没有这样一个集合把不同状态对象放到对应集合中. 只是一个逻辑上的意义.

### 扫描和元信息

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesbqtVNKeruibso6BeQKLK61DlDu2GgTibs8K6HRibdBTXic5oBe0Q4U8Beg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

gc拿到一个指针, 如何把这个指针指向的对象其引用的子对象都加到扫描队列呢? 而且go还允许内部指针, 似乎更麻烦了. 

我们分析一下, 要知道对象引用的子对象, 从对象开始到对象结尾, 把对象那一块内存上是指针的放到扫描队列就好了. 

那我们是不是得知道对象有多大, 从哪开始到哪结束, 同时要知道内存上的8个字节, 哪里是指针, 哪里是普通的数据. 

**首先go的对象是mspan管理的, 我们如果能知道对象属于哪个mspan, 就知道对象多大, 从哪开始, 到哪结束了.** 

**前面我们讲到了areans结构, 可以通过指针加上一定的偏移量, 就知道属于哪个heap arean 64M块. 再通过对64M求余, 结合spans数组, 即可知道属于哪个mspan了.**

结合heapArean的bitmap和每8个字节在heapArean中的偏移, 就可知道对象每8个字节是指针还是普通数据(这里的bitmap是在分配对象时根据type信息就设置了, type信息来源于编译器生成)

**GC流程**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujes6l48pMKuIdOpRrWXdnnAYM7lkXXIiaZHYOwYWuTYbxCEv6HZqMFP33Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

1.5和1.12的GC大致流程相同. 

上图是golang官方的ppt里的图, 下图是我根据1.12源码绘制的. 

从最坏可能会有百ms的gc停顿到能够稳定在1ms以下, 这之间GC做了很多改进. 

右边是我根据官方issues整理的一些比较重要的改进. 1.6的分布式检测, 1.7将栈收缩放到了并发扫描阶段, 1.8的混合写屏障, 1.12更改了mark termination检测算法, mcache flush移除出mark termination等等...

## Golang GC Pacer

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesRPZs40BNUcPSpv32K8p2ibbLsYq63ObkJYicQUmVRXa8a1zx84G6PARw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**大家对并发GC除了怎么保证不漏指针有疑问外, 可能还会疑问, 并发GC如何保证能够跟得上应用程序的分配速度? 会不会分配太快了, GC完全跟不上, 然后OOM?**

这个就是**Golang GC Pacer的作用.** 

**Go的GC是一种比例GC, 下一次GC结束时的堆大小和上一次GC存活堆大小成比例. 由GOGC控制, 默认100, 即2倍的关系, 200就是3倍, 以此类推.** 

**假如上一次GC完成时, 存活对象1000M, 默认GOGC 100, 那么下次GC会在比较接近但小于2000M的时候(比如1900M)开始, 争取在堆大小达到2000M的时候结束.** 

**这之间留有一定的裕度, 会计算待扫描对象大小(根据历史数据计算)与可分配的裕度的比例, 应用程序分配内存根据该比例进行辅助GC, 如果应用程序分配太快了, 导致credit不够, 那么会被阻塞, 直到后台的mark跟上来了,该比例会随着GC进行不断调整.** 

**GC结束后, 会根据这一次GC的情况来进行负反馈计算, 计算下一次GC开始的阈值. 如何保证按时完成GC呢?** 

**GC完了后, 所有的mspan都需要sweep, 类似于GC的比例, 从GC结束到下一次GC开始之间有一定的堆分配裕度, 会根据还有多少的内存需要清扫, 来计算分配内存时需要清扫的span数这样的一个比例.**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesCcab82skBv2LIJoRny3sGFsQQPpYKWluuKoyznQVUWJ9VJ36GqX09Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**实践与总结**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesaKzH66DkADjk7jS9SmveUgEcZrvQU2mpibicRJAqJxXbtR05kayibcx0A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 观察调度

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesicVFRY9XTE1ETiaC3ryXyg5ZzGmUJCbACAI9OpeA0nCLLOkHGniaVSgibQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)观察一下调度, 加一些请求. 

我们可以看到虽然有1000个连接, 但是go只用了几个线程就能处理了, 表明go的网络的确是由epoll管理的. 

runqueue表示的是全局队列待运行协程数量, 后面的数字表示每个P上的待运行协程数. 

可以看到待处理的任务并没有增加, 表示虽然请求很多, 但完全能hold住. 

同时可以看到, 不同P上有的时候可能任务不均衡, 但是一会后, 任务又均衡了, 表示go的work stealing是有效的.

**观察GC**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesWnUTRDfoY2d54MJ6GDJbNPeYwVC4UUMjWG5V85c9ZVfPwvsH5JWCAw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

其中一些数据的含义, 在分享的时候没有怎么解释, 不过网上的解释几乎没有能完全解释正确. 

我这里敲一下. 其实一般关注堆大小和两个stw的wall time即可. 

gc 8913(第8913次gc) @2163.341s(在程序运行的第2163s) 1%(gc所有work消耗的历史累计CPU比例, 所以其实这个数据没太大意义) 0.13(第一个stw的wall time)+14(并发mark的wall time)+0.20(第二个stw的wall time) ms clock, 1.1(第一个stw消耗的CPU时间)+21(用户程序辅助扫描消耗的cpu时间)/22(分配用于mark的P消耗的cpu时间)/0(空闲的P用于mark的cpu时间)+1.6ms(第2个stw的cpu时间) cpu, 147(gc开始时的堆大小)->149(gc结束的堆大小)->75MB(gc结束时的存活堆大小), 151 MB goal(本次gc预计结束的堆大小), 8P(8个P)

## 优化

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujes29GJZzwicfvljiclq3TXQWSicHX7HgldxSiaPSdKmKrGsG0VsPiancjN7wQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

个人建议, 没事不要总想着优化, 好好curd就好.



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesWGNvCoKEwfPY2udrNyF1RXibfTic05DpnkgBV7OQvd5ZicTicLewm50ribw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

当然还是有一些优化方法的..

**一点实践**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesGWgyZAicyGy2jYeMXeQVGbvyubxiaiakmepTdfAQCkouTqWVZyGwGYooA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

我们将pprof的开启集成到模板中, 并自动选择端口, 并集成了gops工具, 方便查询runtime信息, 同时在浏览器上可直接点击生成火焰图, pprof图, 非常的方便, 也不需要使用者关心.

**问题排查的一点思路**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesZrXOdAn2S2BYYoIzZ6uvKiaBOGM5UUZCicFgY8cBEUH3PPHde4iangnzw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**一次有意思的问题排查**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesj7Bj93fTgVpfGtDibCibIY7YrSenJbscxqEcSRC4iaWgJyUekqUvvlpng/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

负载, 依赖服务都很正常, CPU利用率也不高, 请求也不多, 就是有很多超时.



![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujessXeuy8xPNib3RD6TqIQXibDSJiayib353pJSYkhPCWzCqlImOMwaEzs4kg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

该服务在线上打印了debug日志, 因为早期的服务模板开启了gctrace, 框架把stdout重定向到一个文件了. 而输出gctrace时本来是到console的, 输出到文件了, 而磁盘跟不上, 导致gctrace日志被阻塞了.

这里更正一下ppt中的内容, 并不是因为gc没完成而导致其他协程不能运行, 而是后续gc无法开启, 导致实质上的stw. 打印gc trace日志时, 已经start the world了, 其他协程可以开始运行了. 但是在打印gctrace日志时, 还保持着开启gc需要的锁, 所以, 打印gc trace日志一直没完成, 而gc又比较频繁, 比如0.1s一次, 这样会导致下一次gc开始时无法获取锁, 每一个进入gc检查的p阻塞, 实际上就造成了stw.

# Runtime的一点个人总结

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesNQUonBbGGfIKBibPovPXK5CRfLeGGukiaeYbYPKOZJLNBP2Hia0hiawk4g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

并行, 纵向多层次, 横向多个class, 缓存, 缓冲, 均衡.

**参考文档**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SE7gZKCslX1Z2XzSBcURSfst1pbVujesNCcJLogtmSrWPat3qia32VY5wQz1WSgD6vegwCFTQUYY6iamt1m8zd2Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)


