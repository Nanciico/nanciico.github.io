# Redis 分布式锁设计 —— 异步续约机制


为了防止出现业务不能在锁过期时间内结束的问题，分布式锁的设计中引入了续约机制。

本章介绍分布式锁异步续约机制的设计。

---

## 分布式锁续约调研

一个续约的思路是，当线程成功获取分布式锁后，启动一个守护线程，每间隔一定的时间对分布式锁进行续约。当释放分布式锁后，关闭续约线程。如果每个成功获取分布式锁的线程都新开一个守护线程进行锁续约，则会导致应用新开大量的线程造成资源浪费。

开源项目 Redission 中分布式锁续约基于定时任务实现的（Netty 中的 [HashedWheelTimer](https://netty.io/4.0/api/io/netty/util/HashedWheelTimer.html) 工具类）。成功获取分布式锁后注册一个续约定时任务，由线程池中的线程调度执行定时任务，释放分布式锁时将定时任务取消。

---

## 分布式锁续约设计

首先是分布式锁续约间隔问题：分布式锁默认 30s 过期，每 10s 续约一次（过期时间的 1/3，这样每个分布式锁能够容忍续约失败一次）。

分布式锁续约流程如下：

1. 当线程第一次（重入次数为 1）成功获取分布式锁，生成一个 RenewEntry 的续约信息对象，对象以续约时间戳由小到大排序添加到一个阻塞优先队列中；

2. 启动一个固定线程的线程池，续约线程从阻塞优先队列中获取续约对象进行续约操作：

    - 如果续约对象对应的分布式锁已经释放，续约线程不做任何操作；
    - 如果续约成功，则更新下次的续约时间戳，重新将续约对象放入阻塞优先队列中（因为分布式锁未释放）；
    - 如果续约失败（非异常原因导致失败，比如 Key 过期或被删除等），则向客户端通知续约失败；
    - 如果续约异常（Socket、IO 等异常），连续两次续约异常需要向客户端通知续约失败，仅一次续约异常则更新下次的续约时间戳，重新将续约对象放入阻塞优先队列中。

{{< image src="/images/distributed_lock/03_01.jpg" caption="分布式锁续约设计" title="分布式锁续约设计" >}}

---

## 阻塞优先队列的实现

阻塞优先队列的实现的思路：在非阻塞优先队列的基础上，通过锁的通知和等待机制来实现阻塞优先队列。这里涉及到一些线程同步的知识。

[Monitor.Pulse](https://learn.microsoft.com/en-us/dotnet/api/system.threading.monitor.pulse?view=net-8.0)：通知在锁对象就绪队列中的一个线程，通知锁对象的状态已改变。只有持有锁对象的线程才能调用 Pulse 方法。

[Monitor.Wait](https://learn.microsoft.com/en-us/dotnet/api/system.threading.monitor.wait?view=net-8.0) ：释放当前占有的锁对象并阻塞当前线程，直到当前线程重新获取锁；如果指定了超时时间，则会在超时时间过去后，将当前线程放入锁对象就绪队列中。

``` C#
internal class RenewEntryPriorityBlockingQueue<T>
    where T : RenewEntry
{
    private readonly object _locker = new object();

    internal void Offer(T item)
    {
        Monitor.Enter(_locker);
        GrowIfNecessary();

        try
        {
            Insert(item);
            // Offer 线程在插入数据后
            // 通知等待的续约线程，已经有新数据加入到优先队列中
            Monitor.Pulse(_locker);
        }
        finally
        {
            Monitor.Exit(_locker);
        }
    }

    internal T Poll()
    {
        Monitor.Enter(_locker);

        T item;
        try
        {
            var waitTime = DistributedLockConfig.Instance.RenewThreadPollIntervalMilliseconds;
            while ((item = Peek()) == null || (waitTime = item.TimeToRenew()) > 0)
            {
                // 续约线程在优先队列无数据，或队列中第一个元素未到续约时间时
                // 需要进行阻塞等待
                Monitor.Wait(_locker, waitTime);
                waitTime = DistributedLockConfig.Instance.RenewThreadPollIntervalMilliseconds;
            }
            item = Delete();
        }
        finally
        {
            Monitor.Exit(_locker);
        }

        return item;
    }
}
```

上述代码中 Insert、Delete、Peek 等方法为优先队列的基本 API 操作。

---

## 续约失败通知的实现

续约失败通知的实现使用的 C# 提供的 Cancellation 机制。

---

## 分布式锁失效场景

由于分布式锁使用单点锁实现，并且由于 Redis 本身的局限性，导致有如下锁失效场景：

- 异步复制：在锁未同步到从节点时主节点宕机，此时客户端认为拿到锁，但主从切换后锁已丢失；
- 缓存淘汰LRU/LFU有淘汰锁的可能性；
- 同一把锁连续两次续约失败且期间并未解锁，锁会因过期而导致失效。

其中，异步复制导致的锁失效问题，可以使用 WAIT 命令解决（Redission 中有实现），但本设计不实现。

---

