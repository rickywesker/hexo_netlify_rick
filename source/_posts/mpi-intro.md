---
title: 利用MPI实现All-Reduce
date: 2021-12-07 04:15:40
tags:
  - 编程
  - 项目
  - 并行编程
categories:
  - 知识蒸馏
---



# 利用MPI实现All-Reduce

## 实验介绍

是次实验使用两种方法来进行 **In-place Allreduce**, 分别是 **基于主从结构的广播** 以及 **Ring-Allreduce**。

## 基于主从结构的广播

这个方法比较自然，主要是以0号通信器作为Master节点，分别接受来自其他通信器的数据，并进行reduce操作。当所有节点的数据都被reduce之后，我们通过Master节点对所有节点进行广播，这样一来就完成了allreduce操作。

对于每一个通信器，我们把数据分成 {% mathjax %}(BUFFERSIZE/comm\_size){% endmathjax %}​ 份，每一份分别传给Master节点，直到所有节点完成通讯。

![微信截图_20211203004848.png](https://i.loli.net/2021/12/03/D5MXSWZoQ9acxgL.png)

### 分析

设 {% mathjax %}α{% endmathjax %}​ 表示2个通信节点（指集合通信中的一个node，比如1个mpi process即可认为是一个通信节点）间的latency， {% mathjax %}S{% endmathjax %}​表示要allreduce的数据块大小，{% mathjax %}B{% endmathjax %}​表示2个通信节点间的带宽，{% mathjax %}C{% endmathjax %}​表示每字节数据的计算耗时。另外以{% mathjax %}N{% endmathjax %}​表示节点个数。这样的情况下，我们的通信只需要2步，一步reduce，一步broadcast。它们的通信耗时都是{% mathjax %}α + S / B{% endmathjax %}​​。

所以，在Master节点上的计算耗时是{% mathjax %}N*S*C{% endmathjax %}

总体耗时是{% mathjax %}2*(α + S/B) + N*S*C{% endmathjax %}​​。

该算法最大的缺点就是Master 节点的带宽会成为瓶颈。

相关代码如下

![carbon _1_.png](https://i.loli.net/2021/12/03/MKLFqBsknZGRAd3.png)



## Ring-Allreduce

Ring-Allreduce的思想在于，将每块通信器（图源自网络）的数据切成{% mathjax %}COMM\_SIZE{% endmathjax %} 块。

![微信截图_20211203010609.png](https://i.loli.net/2021/12/03/6K2GlquHcsf8xNk.png)

最开始的时候，每个通信器只知道自己的数据，我们的目标在于把每一块的数据聚合，即
{% mathjax %}
node_i = \{a_i,b_i,c_i,d_i\} \ for\ i \in[0,3]
{% endmathjax %}
![微信截图_20211204024154.png](https://s2.loli.net/2021/12/04/m2MuPGs1v83h4Hl.png)

我们有四条通信通道，即通信器0到1，通信器1到2，以此类推直到通信器3到0，我们在每一次通信的时候，没有任何一个节点是idle的，每一轮通信，每个通信器都会进行发送信息并且接受信息。

### 第一轮通信

![微信截图_20211204024413.png](https://s2.loli.net/2021/12/04/5Iy3hrYmRSuzW68.png)

### 第N-1次通信

![image-20211204024605244](C:\Users\A\AppData\Roaming\Typora\typora-user-images\image-20211204024605244.png)

这时候我们需要再进行N-1次通信将reduce后的块发送给其他通信器。

![image-20211204024712356](C:\Users\A\AppData\Roaming\Typora\typora-user-images\image-20211204024712356.png)

## 分析

我们的通信是并发的，所有通信通道都被同时利用，没有冗余开销。

第一阶段可以称为 **Scatter Reduce**，每一步的通信耗时是{% mathjax %}α+S/(NB){% mathjax %}​，计算耗时是{% mathjax %}(S/N)*C{% endmathjax %}

第二阶段每一步的通信耗时也是{% mathjax %}α+S/(NB){% endmathjax %}，没有计算。这一阶段也可视为 **allgather**。

整体耗时大概是{% mathjax %}2*(N-1)*[α+S/(NB)] + (N-1)*[(S/N)*C]{% endmathjax %}​

## 实验结果

### Reduce and Broadcast

|         | 4MB      | 8MB      | 16MB     | 32MB     | 100M     |
| ------- | -------- | -------- | -------- | -------- | -------- |
| 4 core  | 0.030918 | 0.059034 | 0.143740 | 0.304914 | 0.950648 |
| 8 core  | 0.067289 | 0.119806 | 0.305247 | 0.684425 | 2.084829 |
| 16 core | 0.276184 | 0.553833 | 1.375983 | 2.875176 | 8.750173 |

### Ring

|         | 4MB      | 8MB      | 16MB     | 32MB     | 100MB    |
| ------- | -------- | -------- | -------- | -------- | -------- |
| 4 core  | 0.061386 | 0.145172 | 0.268803 | 0.513543 | 1.564591 |
| 8 core  | 0.069678 | 0.146112 | 0.272159 | 0.548818 | 1.759283 |
| 16 core | 0.896696 | 0.495705 | 0.883620 | 1.444635 | 3.877275 |

![results.png](https://s2.loli.net/2021/12/04/CkjKfAcd4gwhboJ.png)

可以看出，**Ring-Allreduce** 的效果在 **通信器数量多的时候** 具有更好的表现，也反映了我们的实验中所有通信都是并发的这样一个事实。 
