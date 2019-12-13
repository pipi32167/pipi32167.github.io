# MySQL在NUMA架构下的负载均衡问题

[TOC]

[The MySQL “swap insanity” problem and the effects of the NUMA architecture](https://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/)

 [A brief update on NUMA and MySQL](https://blog.jcole.us/2012/04/16/a-brief-update-on-numa-and-mysql/)

这是以上两篇博客的笔记。

## 概念

### SMP/UMA

Uniform Memory Access

多cpu多核心通过总线访问统一的内存。 

### NUMA

Non-Uniform Memory Access 

非统一内存访问架构，和UMA相反，每个cpu都有自己的内存。 

## Linux如何处理NUMA系统

1. 枚举硬件，理解物理布局。 
2. 将处理器（不是核心）划分为“**节点**”。对于现代PC处理器来说，这意味着每个节点就是一个物理的处理器，不管处理器有多少核心。
3. 绑定内存到最近的节点上。
4. 收集节点间通讯的“成本”信息（节点间的“距离”）。 

## NUMA如何改变Linux

1. 每个进程和线程都从它们的父母那里继承了一个NUMA策略。这个被继承的策略可以在线程基础上进行修改，该策略定义了一个进程如何被调度至cpu甚至是核心，如何分配内存，以及前两者的严格程度。 
2. 每个线程都初始分配了一个运行的“优先”节点。这个线程可以在任意节点上运行（如果策略允许的话），但调度器尝试去保证它总是在在优先节点上运行。 
3. 为进程分配的内存总是落在某个特定节点上，默认是“当前”节点，也就是线程运行的那个优先节点。在SMP/UMA架构下所有的内存都被视为平等的，有相同的成本，但在NUMA架构下，系统需要去考虑这些内存来自哪儿，因为访问非本地内存意味着性能会受到影响，而且可能会导致缓存融合延迟。 
4. 即使系统需要，分配到某个节点的内存也**不会**转动到另一个节点去。一旦内存分配到了某个节点，那它就一直在那个节点上了。 

## 如何控制NUMA策略

使用numactl或者用在程序中调用libnuma进行精细控制NUMA策略。 

用numactl能够实现的简单事情有：

1. 使用特定策略分配内存 

   - --localalloc，在“本地”分配内存，这也是默认模式；

   - —preferred=node，优先在指定节点分配内存，但有需要的话可以在其他节点上分配； 

   - —membind=nodes，总是在指定节点或者节点集里分配内存； 

   - —interleaved=all 或者 —interleaved=nodes，在分配内存的时候，对所有的节点或者指定节点集公平地按照round-robin方式选择。 

2. 在指定节点或节点集上运行程序。 
  - —cpunodebind=nodes，物理cpu。 
  - —physcpubind=cpus，cpu核心。 

## MySQL的负载场景导致的问题

单进程多线程，使用了系统几乎所有的内存，并且还会尽可能多地去使用其他的系统资源。 

在一个基于NUMA的系统里，内存被分给多个节点，系统如何处理这个过程并不直观。系统的默认行为是把内存分配到和线程调度执行的相同节点上，这个默认行为在小内存的场景下工作得很好，但当你想要分配超过系统一半的内存到某个节点上，这是不可能的事情：在一个两节点的系统里，每个节点只有50%的内存。另外，因为许多不同的查询同时在所有的处理器上执行，没有一个处理器能够针对特定查询提前预先读取特定内存。 

[/proc/](http://linux.die.net/man/5/numa_maps)[pid](http://linux.die.net/man/5/numa_maps)[/numa_maps](http://linux.die.net/man/5/numa_maps) 在这个文件里查看mysqld的情况：

```bash 
2aaaaad3e000 default anon=13240527 dirty=13223315 swapcache=3440324 active=13202235 N0=7865429 N1=5375098 
```

- 2aaaaad3e000 —— 内存区域的虚拟地址。

- default —— 这个内存区域的NUMA策略。

- anon=number —— 匿名页的数量。 
- dirty=number —— 脏页（被修改）的数量。一般来说只有在即将被使用的时候内存才会被分配，因此是脏的，但如果fork了一个进程，可能会有许多的写时复制页（copy-on-write page），这些内存页不是脏页。
- swapcache=number —— 被换出去的页数量，这些页是干净的，因为它们被换出去了，因此有需要的时候可以被释放掉，但此时还在内存中。
- active=number —— 活动页的数量，活动页是在“活动列表”（**active list**）中的页。如果这个属性显示出来了，表示有些内存是不活动的（**inactive**）（匿名页数量 减去 活动页数量，得到非活动页数量），这意味着不活动页可能很快就会被swapper换页换出去。
- N0=number 和 N1=number —— 各自分配给节点0和节点1的页数量。

作者分析MySQL负载问题的时候，通过脚本[numa-maps-summary.pl](http://jcole.us/blog/files/numa-maps-summary.pl)获取到了以下信息：

```bash
N0        :      7983584 ( 30.45 GB)
N1        :      5440464 ( 20.75 GB)
active    :     13406601 ( 51.14 GB)
anon      :     13422697 ( 51.20 GB)
dirty     :     13407242 ( 51.14 GB)
mapmax    :          977 (  0.00 GB)
mapped    :         1377 (  0.01 GB)
swapcache :      3619780 ( 13.81 GB)
```

每个节点只有32GB内存，但是N0已经差不多用完了内存，导致后续需要内存的操作都要去N1的内存去分配。

## 简单的解决方案

`numactl --interleave all command`

修改mysqld_safe脚本：

```bash
# ...
cmd="$NOHUP_NICENESS"
# 在这里打补丁
cmd="/usr/bin/numactl --interleave all $cmd"
# ...
```

修改后内存使用得到了平衡，详情如下：

```bash
2aaaaad3e000 interleave=0-1 anon=13359067 dirty=13359067 
  N0=6679535 N1=6679532
```

```bash
N0        :      6814756 ( 26.00 GB)
N1        :      6816444 ( 26.00 GB)
anon      :     13629853 ( 51.99 GB)
dirty     :     13629853 ( 51.99 GB)
mapmax    :          296 (  0.00 GB)
mapped    :         1384 (  0.01 GB)
```

## 彻底的解决方案

1. 使用`numactl --interleave all command`。
2. 在mysqld启动前刷掉Linux的buffer  cache：`sysctl -q -w vm.drop_caches=3`。这对保证分配的公平性有所帮助，即使后台进程在系统缓存中有大量数据时重启。
3. 使用`MAP_POPULATE`（Linux 2.6.23+），如果系统不支持此操作则使用`memset`，强制操作系统在MySQL启动时为InnoDB的缓存池分配内存。这强迫立刻进行NUMA节点分配的决策，因为第二步的原因，此时buffer cache还是干净的。

在生产环境中的一台机器（144GB的RAM，120GB的InnoDB buffer pool）实行这个方案的结果是两个节点的分配页的差额只有152（0.00045%），实现了完美的平衡。

详情如下：

```bash
2aaaab2db000 interleave=0-1 anon=33358486 dirty=33358486
  N0=16679245 N1=16679241
```

```bash
N0        :     16870335 ( 64.36 GB)
N1        :     16870183 ( 64.35 GB)
active    :           81 (  0.00 GB)
anon      :     33739094 (128.70 GB)
dirty     :     33739094 (128.70 GB)
mapmax    :          221 (  0.00 GB)
mapped    :         1467 (  0.01 GB)
```



## 注意

读取/proc/pid/numa_maps文件会导致对应pid的进程停顿。在生产环境中慎用。