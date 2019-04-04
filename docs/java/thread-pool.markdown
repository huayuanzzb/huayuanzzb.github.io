---
layout: post
title:  线程池
date:   2019-04-04 17:25:26 +0800
nav_order: 1
parent: java
categories: java
---

```ThreadPoolExecutor``` 主要参数解释：

* corePoolSize: 线程池中用于执行任务的线程数。

如果```allowCoreThreadTimeOut = false```, 则线程池中最多```corePoolSize```个活跃线程用于执行新任务;
如果```allowCoreThreadTimeOut = true```, 则线程池中的线程在等待```keepAliveTime```后仍未有新任务提交到线程池, 回收该线程.

* maximumPoolSize: 线程中最大的线程数. 

该参数只有在queue满时才有意义:
线程池初始化时, 不指定 queue size, 默认 queue 为无界, 所有没有来得及执行的任务都会在queue中排队;
线程池初始化时，指定 queue size 和 maximumPoolSize, 若线程池中有 corePoolSize 个正在运行的线程，又提交了 queue size 个任务在 queue 中排队, 此时, 线程池最多还能再接受 (maximumPoolSize - corePoolSize) 个任务, 接下来提交的任务会被reject. 此时我们可以理解为线程池满负荷运行.

* keepAliveTime: 两种情况下会用到：
```allowCoreThreadTimeOut = true```, 所有空闲线程等待```keepAliveTime```后会被回收;
线程池满负荷转换为低负荷时, ```corePoolSize```之外的空闲线程等待```keepAliveTime```后被回收

通过可以看出，满足一下三个条件的线程就是固定大小的线程池:
1. 不指定queue size, 所有新任务都会在queue中排队
2. allowCoreThreadTimeOut = false, 即使线程空闲也不会回收
查看```Executors.newFixedThreadPool(int nThreads)```源码也可看到确实是这样实现的.

### 参考
https://stackoverflow.com/questions/17659510/core-pool-size-vs-maximum-pool-size-in-threadpoolexecutor
