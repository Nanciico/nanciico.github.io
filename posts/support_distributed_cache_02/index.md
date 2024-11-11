# 添加分布式缓存 —— 系统实现


## 优化首条 Kafka 消息不消费问题

### 现象

相同的代码，使用 .Net Build 不消费首条消息，但通过 MS Build 会消费首条消息。

### 问题定位

通过打点调试，定位到不消费消息的原因： XXXMetadataCache 单例类，调用类的静态方法，未触发单例初始化，不会注册 Handler 方法。从而导致消息虽然消费了，但是没有处理消息。

{{< image src="/images/support_distributed_cache/02_01.jpg" caption="打点调试——预期日志" title="打点调试——预期日志" >}}

{{< image src="/images/support_distributed_cache/02_02.jpg" caption="打点调试——实际日志" title="打点调试——实际日志" >}}

通过 ILSpy 工具，查看单例类反编译代码，发现该类上存在 `beforefieldinit` 标志。

{{< image src="/images/support_distributed_cache/02_03.jpg" caption="单例类代码" title="单例类代码" >}}

{{< image src="/images/support_distributed_cache/02_04.jpg" caption="单例类 IL 代码" title="单例类 IL 代码" >}}

通过添加静态构造函数，去掉类上的 `beforefieldinit` 标志，使得调用静态方法能够触发静态构造函数，从而解决该问题。

{{< image src="/images/support_distributed_cache/02_05.jpg" caption="优化后单例类代码" title="优化后单例类代码" >}}

{{< image src="/images/support_distributed_cache/02_06.jpg" caption="优化后单例类 IL 代码" title="优化后单例类 IL 代码" >}}

### beforefieldinit 标识

[What does beforefieldinit flag do?](https://stackoverflow.com/questions/610818/what-does-beforefieldinit-flag-do)

[C# and beforefieldinit](https://csharpindepth.com/articles/BeforeFieldInit)

[Implementing the Singleton Pattern in C#](https://csharpindepth.com/Articles/Singleton)

---

## 缓存设计中常见的问题

### 缓存穿透

缓存穿透是指用户请求的数据在缓存中不存在即没有命中，同时在数据库中也不存在，导致用户每次请求该数据都要去数据库中查询一遍，然后返回空。

如果有恶意攻击者不断请求系统中不存在的数据，会导致短时间大量请求落在数据库上，造成数据库压力过大，甚至击垮数据库系统。

缓存穿透常用解决方案：

1. 布隆过滤器；
2. 缓存空对象。

本项目中使用缓存空对象的解决方案。

### 缓存雪崩

缓存雪崩是指缓存中数据大批量到过期时间，而查询数据量巨大，请求直接落到数据库上，引起数据库压力过大甚至宕机。

本项目缓存雪崩的解决方案：

1. 通过给缓存设置的过期时间加上一个随机的时间，从而避免让缓存数据短时间大批量过期；
2. 清理缓存数据的时间点随机化，让不同客户端清理缓存的时间点均匀。

---

## Redis Lua 整数精度问题

Redis 缓存版本号，只允许写入高版本，这个需要使用 Lua 实现。

发现无法向 Redis 写入 `long.MaxValue`，因为 Redis 中 Lua 版本为 5.1 ，不支持整数，一律使用双精度浮点数表示整数，根据 IEEE754，能精确表达的整数范围为：[-2^53, 2^53]。

由于 2^53 比较大，评估版本号够用，所以此处不做修改。

---

## 双重检查锁

Redis 可以通过 Lua 保证只写入高版本号，但是本地版本存在并发问题。

``` C#
// 双重检查锁

private void InvalidateVersion(string key, long version)
{
    GetVersion(key, out var oldVersion, out var _);

    if (version > oldVersion)                                   // 提高性能
    {
        var lockTaken = false;

        try
        {
            _xlock.Lock(key, ref lockTaken);

            GetVersion(key, out oldVersion, out var _);

            if (version > oldVersion)                           // 并发正确性
            {
                PutVersion(key, version);
            }
        }
        finally
        {
            _xlock.Release(key, lockTaken);
        }
    }
}
```

这里使用双重检查锁的原因：

1. 由于大多数场景，判断条件： `version > oldVersion` 不成立，因此可以提高性能，避免进入内部代码块，节省获取锁的开销；
2. 因为第一次获取的版本可能是旧数据，所以获取锁之后，还需要再获取一遍本地版本进行判断，保证并发正确性。

---

