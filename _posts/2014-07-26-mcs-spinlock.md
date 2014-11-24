---
layout: post
title: "Linux同步机制--MCS自旋锁"
tagline: "Synchronization mechanisms in Linux: mcs_spinlock"
description: "Synchronization mechanisms in Linux: mcs_spinlock"
categories: [linux内核, 同步机制]
tags: [Linux内核, 同步机制, MCS自旋锁]
---
{% include JB/setup %}

## MCS自旋锁简介

MCS自旋锁, 顾名思义, 是一种自旋锁的实现方式。它的名字中的**"MCS"**来自于两位发明者的名字缩写: Mellor-Crummey和Scott。这种自旋锁的一个显著优点是:**减少了CPU cacheline bouncing的问题**。所谓**CPU cacheline bouncing**, 是指几个CPU的cacheline反复失效这一现象, 勿庸多言, 这种行为对性能是巨大的损耗。而传统的自旋锁实现都存在这一问题, 因为自旋锁的本质就是**CPU在一个表示锁状态的对象上进行紧密的轮询(tight loop), 以等待锁是空闲的状态, 进而获取之**。这个**对象**可以是一个bit, 也可以是一个整型变量, 不同实现采取不同方案。但传统的自旋锁实现都逃不开这样一个桎梏: 轮询时把该对象拷贝到本地CPU的cacheline, 检测其状态, 如果空闲, 获取该锁；如果不空闲, 则进入紧密轮询, 再把该对象拷贝到本地CPU的cacheline, 重新检测其状态, 以期别人已经释放该锁。这种现象在锁是个热点时更为严重, 多个CPU竞争该锁, 每个CPU都上演这一幕, 从而造成**CPU cacheline bouncing**的问题。

MCS自旋锁如何巧妙解决这一问题, 在下一节描述。虽然这一种锁相对当前Mainline kernel中的[ticket spinlock](http://lwn.net/Articles/267968/)实现具有上述的优势, 但它依然没代替前者成为内核中采取的方案, 原因是它的实现占用空间太多, 而自旋锁在内核中使用很多, 有些地方对该锁的大小非常敏感, 因此无法被采纳。不过, 在内核另一种加锁机制**mutex**的实现中, 就使用了它, 以进行优化。关于MCS自旋锁的更多介绍, 可以参阅[这篇论文](http://www.cise.ufl.edu/tr/DOC/REP-1992-71.pdf), 或lwn站点上的文章:  [MCS locks and qspinlocks](http://lwn.net/Articles/590243/)。

## MCS自旋锁原理

MCS自旋锁的特色就在于它**显著减少**了**CPU cacheline boucing**的问题。这一问题的根源在于在轮询中, 每个循环都要把锁对象复制到本地CPU cacheline中来检查, 如果能减少这一开销, 这一问题就能解决。MCS自旋锁正是这样做的, 每次轮询, 都是访问本地CPU的锁对象, 而不是全局的那个锁对象, 这样就**显著减少**了**CPU cacheline boucing**的问题。初听起来不可思议, 它如何保证本地CPU的锁对象能准确反映全局锁对象的状态的呢? 这正是它实现的高明之处。

另外, 这种实现方式, **甚至可以不需要存在一个全局的锁对象**! 后文将"全局锁对象"简称为"G", 将某CPU X的本地锁对象简称为"LX"。

一个MCS自旋锁像这样:

    struct mcs_spinlock {
        struct mcs_spinlock *next;
        int locked;
    }

其中**next**字段是一个指针, 指向下一个锁; **locked**字段则表示锁的状态, **0为空闲, 1为锁被占用**。

接下来将以一个双CPU的模型来描述该锁是如何工作的。一开始, 只有锁**G**, 如图:

              G
        +--------------+
        | next == NULL |
        +--------------+
        | locked == 0  |
        +--------------+

当CPU-1准备获取该锁时, 它用**G->next**字段, 与**L1**的地址, 进行**原子交换操作: xchg(G->next, L1)**, 交换操作是一个三步骤的操作, 使用原子方式, 避免看见中间状态, 使实现变复杂。交换后, 如图:

               G                  L1 == NULL
         +--------------+      +--------------+
         | next == &L1  |      | next == NULL |
         +--------------+      +--------------+
         | locked == 0  |      | locked == 0  |
         +--------------+      +--------------+

然后, CPU-1检查**L1**的值, 发现其为**NULL**, 这表示CPU-1获得了锁。

此时, 若CPU-2也想获得这个锁, 它进行一样的操作, 用本地锁对象**L2**, 与全局锁**G**进行**next字段的原子交换操作: xchg(I->next, L2->next)**。交换后, 如图:

               G                     L1                  L2 == &L1
         +--------------+      +--------------+      +--------------+
         | next == &L2  |      | next == &L2  |      | next == NULL |
         +--------------+      +--------------+      +--------------+
         | locked == 0  |      | locked == 0  |      | locked == 0  |
         +--------------+      +--------------+      +--------------+

然后, CPU-2检查**L2**的值, 发现其不为**NULL**, 而是存放着**L1**的地址, 于是, 它知道现在锁被CPU-1获取着, 于是, 它将**L1->next**标记为自己的地址: &L2。然后, **CPU-2将在本地锁对象L2上进行轮询, 检测L2->locked的状态!**

当CPU-1准备释放锁时, 它将和**G->next**进行一次**比较并交换操作: cmpxchg(G->next, L1, NULL)**。若**G->next**是**L1**, 则将**G->next**赋值为**NULL**, 并返回**L1**; 否则不交换, 只返回**G->next**的旧值。翻译为大白话, 就是CPU-1检查**G**是否还指向自己, 如果是, 则将**G->next**赋为空值, 表示释放锁；否则, 表示有人在等待。

此时, CPU-1发现返回的值是**L2**的地址, 而不是**NULL**, 知道CPU-2在等待。于是, 它将**L1->next**置为**NULL**, 并将**L2->locked**的值置为1, 这样就完成了锁的易主, CPU-1释放了锁, CPU-2获得锁。

而此时, CPU-2正在**L2**上轮询, 当它发现**L2->locked**变为1时, 它知道自己获取锁了, 停止轮询。

回顾整个过程, 有4点重要的内容:

 * 当要获取锁且锁繁忙时, CPU-2只要在本地锁对象上轮询, 检测本地锁状态即可, 当CPU-1释放锁时, 会更新CPU-2的锁状态, 于是, CPU-2就知道自己获得锁了。这就避免了**CPU cacheline bouncing**的问题。

 * 这个实现也是公平的, 要获取锁的CPU将**依次排队**, 以到达的次序获取锁。这是一个队列, 每个本地CPU的**next**字段指向下一个要获得锁的CPU。

 * 此外, 我们可以发现, 全局锁**G**的**locked**字段其实是用不着的, 因此, 实现上, 我们可以省去这个字段, 只保留一个指针, 此时, **G**退化为一个指示器, 指向当前队列最后一个锁对象, 即最后到达的要获取锁的CPU的锁对象。我们可以看到内核的代码实现中其实就是这么做的。详见内核源码: **kernel/locking/mcs_spinlock.h**。

 * 再进一步, 我们可以发现, 指向本地CPU的锁对象的指针其实是和CPU一一对应的。意思就是, 当我们发现指向CPU-2的本地锁对象指针时, 我们就知道CPU-2在等待这个锁。原因是**一个特定的自旋锁对于一个CPU来说, 是不可重入的**, 因为当一个CPU获取该锁时, CPU是关本地中断的, 也禁止抢占的, 这防止了从本地CPU的其他上下文, 如中断, 获取该锁的可能, 以避免死锁。基于此, 这个指针可以进一步替换为CPU号, 这就是内核中**可取消的MCS自旋锁**的做法, 这是下一篇文章的内容。

  <center><strong>* * * * * * 全文完 * * * * * * </strong></center>