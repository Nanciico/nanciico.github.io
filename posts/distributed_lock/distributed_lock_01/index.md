# Redis 分布式锁设计 —— 基本特性的设计与实现


上一章节，我们简要介绍了分布式锁的特性。这一章，我们介绍分布式锁基本特性的设计与实现，包括：互斥性、避免死锁、高可用性和可重入性。

分布式锁使用 Redis 中的 Hash 数据结构实现。Hash 的 Key 保存分布式锁的名字，相同的 Key 代表同一把分布式锁。

---

## 互斥性与可重入性

分布式锁的互斥性，即在任何场景下，只能有一个客户端能够持有分布式锁。那么 Redis 服务端需要记录下是哪一个进程的哪一个线程持有分布式锁。

分布式锁的可重入性，即一个线程获取分布式锁后，还可以再次获取锁。那么 Redis 服务端需要记录下哪一个线程重入哪一把分布式锁多少次。

为满足这两个特性，我们做如下的设计：

1. Key 保存分布式锁的名字，相同的 Key 代表同一把分布式锁；
2. Field 保存*分布式锁对象的GUID*和*持有锁的线程ID*，每个分布式锁对象会生成一个GUID，这个GUID同时也记录了进程信息；
3. Value 保存重入次数。

这样便保证了分布式锁的互斥性，并实现了可重入性。

---

## 避免死锁

避免死锁，已经获取分布式锁的客户端崩溃，其他的客户端也应该能够获取分布式锁。如果分布式锁得不到释放，其他线程将会一直阻塞。

避免死锁的常用方法，便是给分布式锁一个过期时间，这样即使客户端崩溃，分布式锁过期后会得到释放，其他线程便能够得到执行。

我们的设计中，默认的过期时间为 30s。

---

## 加锁解锁操作的实现

通过互斥性、可重入性和避免死锁这 3 个特性，我们可以实现分布式锁的加锁与解锁操作。

分布式锁的加锁和解锁操作通过 Lua 实现，这是因为我们需要保证分布式锁的加锁与解锁操作是**原子操作**。

加锁操作脚本如下所示：

``` Lua
local function TryLock(key, field, leaseTime)
    if ((redis.call('EXISTS', key) == 0) or (redis.call('HEXISTS', key, field) == 1))
    then
        redis.call('HINCRBY', key, field, 1)
        redis.call('PEXPIRE', key, leaseTime)
        return redis.call('PTTL', key)
    end
    return -2
end
return TryLock(KEYS[1], ARGV[1], ARGV[2])
```

解锁操作脚本如下所示：

``` Lua
local function Unlock(key, field)
    if (redis.call('HEXISTS', key, field) == 0)
    then
        return 0
    end
    local counter = redis.call('HINCRBY', key, field, -1)
    if (counter < 1)
    then
        redis.call('DEL', key)
    end
    return 1
end
return Unlock(KEYS[1], ARGV[1], ARGV[2])
```

---

## 高可用性

分布式锁的高可用性通过 Redis 服务的高可用性来保障。Redis 服务采用集群模式保障 Redis 服务的高可用。

---

