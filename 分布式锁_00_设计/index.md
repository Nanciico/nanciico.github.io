# Redis 分布式锁设计 —— 前言


---

## Redis键值存储设计

采用Hash结构作为分布式锁的存储结构。

- KEY存储分布式锁名LockName；
- FIELD存储分布式锁GUID:ThreadId；（每个分布式锁对象会生成一个GUID，ThreadId为Lock操作时当前线程Id。）
- VALUE存储分布式锁的重入次数。

---

## 请求量优化设计

应用尝试获取分布式锁失败时，需要重试争抢锁，直到成功获取分布式锁。在重试争抢分布式锁期间会产生不必要的请求，需要优化这部分请求，降低Redis服务器压力。

在单应用多线程争抢同一把分布式锁锁的场景下，使用本地锁保证该应用在同一时刻只有一个线程重试争抢分布式锁。

``` mermaid
flowchart LR

    A([Start]) --> B(Try to get<br>distributed lock)
    B --> C{Succeed to get<br>distributed lock?}
    C -->|Yes| D([End])
    C -->|No| F(Try to get<br>local lock)

    subgraph Retry
        direction LR
        F --> G(Try to get<br>distributed lock)
        G --> H{Succeed to get<br>distributed lock?}
        H -->|No| G
        H -->|Yes| I(Release<br>local lock)
    end

    I --> D
```

### 本地锁管理设计

使用本地锁管理分布式锁，此时引出了本地锁管理的问题。

所有的本地锁存放在一个 ConcurrentDictionary 中。当线程需要重试以获取分布式锁时，需要先获取本地锁对象（获取不到锁对象则需要创建本地锁对象）并持有本地锁。线程重试获取分布式锁成功后需要释放本地锁，且如果此时没有线程等待本地锁对象，则需要将本地锁对象从字典中删除。

本地锁生命周期通过引用计数进行管理，保证本地锁的安全关闭。

---

## 分布式锁续约设计

如果每个成功获取分布式锁的线程都新开一个守护线程进行锁续约，则会导致应用新开大量的线程造成资源浪费。因此需要统一管理需要续约的分布式锁。

分布式锁默认30s过期，每10s续约一次（过期时间的 1/3，这样每个分布式锁能够容忍续约失败一次）。

固定开 2-3 个线程对所有的分布式锁进行续约，使用阻塞优先队列管理需要续约的分布式锁对象。

- 续约线程从阻塞优先队列获取分布式锁对象；
- 如果分布式锁对象已经 Unlock，则不续约该锁；
- 如果分布式锁对象在续约过程中发生异常，放回阻塞优先队列，该分布式锁默认会在10s后再次续约；
- 如果分布式锁对象续约成功，放回阻塞优先队列，该分布式锁默认会在10s后再次续约；
- 如果分布式锁对象续约失败且未 Unlock。即在续约时分布式锁不存在，判定分布式锁已经失效。停止续约该分布式锁，并使用Cancel机制通知用户线程分布式锁已失效，用户线程可以调用 RenewFailed 方法判断分布式锁是否失效。

``` mermaid
flowchart LR

    A(Lock)
    A --> Q[/PriorityBlockingQueue\]
    Q -->|Poll| T((Renew Threads))
    T -->|Succeed to renew<br>renew occured an exception<br>Offer| Q
    T --> B(Already unlocked)
    T --> C(Lock has already lost)
```

---

## 分布式锁失效场景

由于分布式锁使用单点锁实现，并且由于 Redis 本身的局限性，导致有如下锁失效场景：

- 异步复制：在锁未同步到从节点时主节点宕机，此时客户端认为拿到锁，但主从切换后锁已丢失；
- 缓存淘汰LRU/LFU有淘汰锁的可能性；
- 同一把锁连续两次续约失败且期间并未解锁，锁会因过期而导致失效。

其中，异步复制导致的锁失效问题，可以使用 WAIT 命令解决，但本设计不实现。

---

