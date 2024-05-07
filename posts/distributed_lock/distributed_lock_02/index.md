# Redis 分布式锁设计 —— 加锁主流程


本章介绍分布式锁 Lock、TryLock 接口的加锁主流程。

---

## Lock 接口的加锁主流程

对于 Lock 接口，应该轮询尝试加锁，直到加锁成功，否则 Sleep 一段时间，以降低 Redis 请求量。由此我们可以得到初步的加锁代码如下：

``` csharp
public override void Lock(long leaseTime)
{
    Checker.CheckLeaseTime(leaseTime);

    while (true)
    {
        // 尝试向 Redis 发送请求加锁，返回加锁是否成功
        if (TryLockInternal(leaseTime))
        {
            return;
        }

        Thread.Sleep(DistributedLockConfig.Instance.RetryIntervalMilliseconds);
    }
}
```

但在多线程争抢同一分布式锁的场景下，上述加锁流程会向 Redis 发出不必要的请求：

{{< image src="/images/distributed_lock/02_01.jpg" caption="不必要的请求" title="不必要的请求" >}}

如上图，该场景下有 3 进程，每个进程包含 5 个线程，这些线程都去争抢同一把分布式锁，那么所有线程都会向 Redis 轮询请求分布式锁。然而经过分析，如果在一个进程内，有一个线程正在争抢分布式锁，其他线程并不需要去争抢，因此这些线程的轮询请求可以被优化掉。

这里使用一个内部锁去控制在一个进程内只有一个线程去轮询争抢分布式锁，优化后的效果如下图所示：

{{< image src="/images/distributed_lock/02_02.jpg" caption="使用内部锁优化请求量" title="使用内部锁优化请求量" >}}

如上图，红色实线表示该线程正持有分布式锁；虚线表示线程正轮询请求分布式锁。线程必须先持有内部锁后，才能轮询请求锁，请求锁成功后，需要释放内部锁。优化后的代码如下：

``` C#
public override void Lock(long leaseTime)
{
    Checker.CheckLeaseTime(leaseTime);

    // 尝试获取一次分布式锁
    if (TryLockInternal(leaseTime))
    {
        return;
    }

    // 获取分布式锁失败，需要获取内部锁后，轮询请求分布式锁
    // 只有一个线程会持有内部锁，保证只有一个线程进入轮询代码块
    LocalLockManager.Instance.Lock(_entryName);
    try
    {
        while (true)
        {
            if (TryLockInternal(leaseTime))
            {
                return;
            }

            Thread.Sleep(DistributedLockConfig.Instance.RetryIntervalMilliseconds);
        }
    }
    finally
    {
        // 释放内部锁
        LocalLockManager.Instance.Unlock(_entryName);
    }
}
```

---

## LocalLockManager

我们可以注意到优化后的加锁流程中，引入了一个单例的 LocalLockManager，其实际上是一个并发字典。引入这个类是因为，不同的分布式锁对应不同的内部锁，我们需要将这些内部锁对象统一管理，即通过 LocalLockManager 来创建或释放内部锁对象。

### LockEntry 生命周期管理

LockEntry 是通过引用计数算法管理的内部锁的封装。

当一个线程需要轮询重试时，需要先从 LocalLockManager 中获取对应的 LockEntry，再去尝试持有内部锁；如果 LockEntry 不存在，则会创建一个 LockEntry，并将其添加到 LocalLockManager 中，再尝试持有内部锁。

当一个线程轮询重试结束时，线程需要释放内部锁，这时线程还需要判断是否有其他线程正在等待内部锁。如果没有其他线程等待内部锁，则需要将这个内部锁从 LocalLockManager 中移除。这也是为何需要引用计数算法的原因，因为需要判断是否有其他线程等待，且需要保证移除操作的线程安全。

此处存在两个线程安全问题：

1. 从 LocalLockManager 中获取 LockEntry 时，可能出现多个线程均判断 LockEntry 不存在，从而导致之后的 LocalLockManager 添加操作线程不安全；
2. 线程成功获取 LockEntry，尝试增加 LockEntry 的引用计数时，另一个线程将该 LockEntry 从 LocalLockManager 中移除。

这里通过一个循环操作，和引用计数的 TryIncRef 操作，解决上述两个线程安全问题：

``` csharp
private LockEntry GetLockEntry(string name)
{
    while (true)
    {
        if (_lockEntries.TryGetValue(name, out var entry))
        {
            if (entry.TryIncRef())
            {
                return entry;
            }
        }

        var newEntry = new LockEntry(name);
        if (_lockEntries.TryAdd(name, newEntry))
        {
            return newEntry;
        }
    }
}
```

对于第一个线程安全问题，线程 A 与线程 B 同时执行 TryAdd 方法，仅会有一个线程添加成功，比如线程 A 添加成功，线程 B 则会进入下一个循环，这样就保证了 LocalLockManager 添加操作的线程安全。

对于第二个线程安全问题，线程 A 执行 TryGetValue 成功获取了 LockEntry，线程 B 将引用计数减至 0 后，将 LockEntry 从 LocalLockManager 中移除，此时线程 A 执行 TryIncRef 将失败，并进入下一个循环，重新尝试获取 LockEntry。

---

## 引用计数算法

> In computer science, reference counting is a programming technique of storing the number of references, pointers, or handles to a resource, such as an object, a block of memory, disk space, and others.
>
> In garbage collection algorithms, reference counts may be used to deallocate objects that are no longer needed.

对于一个实现了引用计数的对象，引用计数算法能够跟踪有多少引用处于活动状态。当引用的数量降至 0，我们可以安全的释放该对象的内存。

引用计数也是实现基本的垃圾回收算法的方式。

**如果一个对象在多线程环境中需要安全关闭，可以考虑使用引用计数算法。**

本实现借鉴了 ElasticSearch 中的实现：[ElasticSearch 引用计数算法。](https://github.com/elastic/elasticsearch/blob/main/libs/core/src/main/java/org/elasticsearch/core/AbstractRefCounted.java)

引用计数算法有两个核心操作：TryIncRef 和 DecRef。

``` csharp
internal abstract class AbstractRefCounted : IRefCounted
{
    private long _refCount = 1;

    public bool TryIncRef()
    {
        do
        {
            long i = Interlocked.Read(ref _refCount);
            if (i > 0)
            {
                if (Interlocked.CompareExchange(ref _refCount, i + 1, i) == i)
                {
                    return true;
                }
            }
            else
            {
                return false;
            }
        } while (true);
    }

    public bool DecRef()
    {
        long i = Interlocked.Decrement(ref _refCount);
        if (i == 0)
        {
            CloseInternal();
            return true;
        }
        return false;
    }
}
```

DecRef 方法能够保证在引用计数降至 0 时执行一次 CloseInternal 方法，由于 `Interlocked.Decrement()` 是原子操作，因此在多线程环境下能够保证只有一个线程执行 CloseInternal 方法，即保证引用计数对象的安全关闭。

---

## TryLockInternal 方法

简单介绍一下 TryLockInternal 方法。

TryLockInternal 方法向 Redis 服务端发送一次加锁请求，如果加锁成功，执行续约与重入相关代码，返回 true，否则返回 false。

``` csharp
protected bool TryLockInternal(long leaseTime)
{
    // 向 Redis 发送加锁请求
    var pttl = LuaHelper.TryLock(_keyspace, _tenantId, _lockName, GetClientId(), leaseTime);

    // 加锁成功
    // 在正常情况下，只有一个线程能够加锁成功，即只有一个线程能进入该代码块
    if (pttl > 0)
    {
        if (TryGetRenewEntry(out var oldEntry))
        {
            // 已注册续约队列，则仅增加重入次数
            oldEntry.IncreaseReentrantCount();
        }
        else
        {
            var renewEntry = new RenewEntry(this, GetClientId(), leaseTime, pttl);
            _threadIds.TryAdd(Thread.CurrentThread.ManagedThreadId, renewEntry);

            // 未注册续约队列，则加入到续约队列
            RenewManager.Instance.AddEntry(renewEntry);
        }

        return true;
    }

    return false;
}
```

---

