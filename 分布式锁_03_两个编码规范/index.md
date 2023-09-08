# 两个编码规范


---

## 不要生吞异常

Unlock 发送请求，如果发生 Socket 异常导致逻辑上解锁但实际上未解锁，是否需要将异常抛出？

- 若不处理异常，Redis 中的 Key 会在设定的 30s 后过期，过期后可以将其与解锁看做等效，因此可以不处理异常；
- 若需要处理异常，则将该异常抛出，用户在使用 Unlock 时需要处理该异常。

原则：不要在代码中假设某种异常不会发生，或者忽略某中异常是无所谓的。

如果不将异常抛出或没有输出到日志，程序在出现莫名其妙的异常后会难以定位问题。因此不要忽略异常。

---

## 区分“判断”与“校验”

在代码中，如果只有满足某种条件才能继续向后执行，应该使用校验。如果对于某个条件需要执行不同的分支，应该使用判断。

不要用判断取代校验，这会导致忽略某些异常。

``` C#
// 校验，正确的用法
internal void Unlock(string name)
{
    if (!(_lockEntries.TryGetValue(name, out var entry) && entry.IsEntered()))
    {
        throw new ExitLocalLockException("Exit local lock occurs an exception, " +
            "the local lock is missing or the local lock is not held by the current thread. " +
            $"LockName: [{name}]");
    }

    entry.Exit();
    entry.DecRef();
}

// 判断，会忽略异常。
internal void Unlock(string name)
{
    if (_lockEntries.TryGetValue(name, out var entry) && entry.IsEntered())
    {
        entry.Exit();
        entry.DecRef();
    }
}
```

---

