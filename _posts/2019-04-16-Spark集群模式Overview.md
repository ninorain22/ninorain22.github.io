### 集群模式概述

这里讲述Spark如何在集群上运行

#### 组件
Spark程序在集群中作为独立的进程组来执行，通过main程序里的`SparkContext`来协调(即Driver Program)

为了在集群上运行，`SparkContext`可以连接到不同类型的`cluster managers`，一但连接上，Spark可以获得集群中节点的executors,
用于为你的application计算和存储数据。接着，它将发送你的application代码到这些executors上，最终，`SparkContext`发送`tasks`给这些executors去执行

![avatar](../imgs/cluster_mode_overview.png)

关于这个架构，有几个有用的注意事项：
1. 每个application都有自己的executor进程, 会在application的整个生命周期存在，并且多线程的执行`tasks`.
这样做的好处是实现应用的互相隔离，无论是从任务调度方面还是在executor方面。
然而，这也就意味着数据不能在不同的Spark application之间共享
2. Spark对于底层的cluster manager是不知情的，只要它可以正确的获得executors
3. 在生命周期内。Driver Program必须监听和接收任何从executors新来的连接，因此，Driver Program必须可以被worker nodes网络寻址
4. 由于是通过Driver来调度task到集群，因此，最好Driver Program运行在靠近worker nodes的机器上，最好是同一个网络区域。

#### Cluster Manager类型
+ Standalone
+ Apache Mesos
+ Hadoop YARN
+ Kubernetes