# 引用计数算法


## 什么是引用计数

> In computer science, reference counting is a programming technique of storing the number of references, pointers, or handles to a resource, such as an object, a block of memory, disk space, and others.
>
> In garbage collection algorithms, reference counts may be used to deallocate objects that are no longer needed.

对于一个实现了引用计数的对象，引用计数算法能够跟踪有多少引用处于活动状态。当引用的数量降至 0，我们可以安全的释放该对象的内存。

引用计数也是实现基本的垃圾回收算法的方式。

**如果一个对象在多线程环境中需要安全关闭，可以考虑使用引用计数算法。**

## 引用计数算法的实现

[ElasticSearch 代码中实现了基本的的引用计数算法。](https://github.com/elastic/elasticsearch/blob/main/libs/core/src/main/java/org/elasticsearch/core/AbstractRefCounted.java)

``` Java
public abstract class AbstractRefCounted implements RefCounted {

    private final AtomicInteger refCount = new AtomicInteger(1);
    private final String name;

    public AbstractRefCounted(String name) {
        this.name = name;
    }

    @Override
    public final void incRef() {
        if (tryIncRef() == false) {
            alreadyClosed();
        }
    }

    @Override
    public final boolean tryIncRef() {
        do {
            int i = refCount.get();
            if (i > 0) {
                if (refCount.compareAndSet(i, i + 1)) {
                    return true;
                }
            } else {
                return false;
            }
        } while (true);
    }

    @Override
    public final boolean decRef() {
        int i = refCount.decrementAndGet();
        assert i >= 0;
        if (i == 0) {
            try {
                closeInternal();
            } catch (Exception e) {
                throw e;
            }
            return true;
        }
        return false;
    }

    protected void alreadyClosed() {
        throw new IllegalStateException(name + " is already closed can't increment refCount current count [" + refCount.get() + "]");
    }

    /**
     * Returns the current reference count.
     */
    public int refCount() {
        return this.refCount.get();
    }

    /** gets the name of this instance */
    public String getName() {
        return name;
    }

    /**
     * Method that is invoked once the reference count reaches zero.
     *
     */
    protected abstract void closeInternal();
}
```

## 分布式锁设计中如何使用引用计数

在分布式锁的设计中，本地锁生命周期通过引用计数进行管理，保证本地锁的安全关闭。

### 如何获取本地锁对象

- 如果能够从字典中获取本地锁对象则直接返回该对象；
- 如果字典中没有本地锁对象，创建一个本地锁对象并加入到字典中，返回该对象；
- 如果上述两个操作都失败了，则重试，直到成功获取本地锁对象。

``` C#
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

### 如何安全关闭本地锁对象

只需要实现 `CloseInternal()` 方法，在引用计数降至 0 时，从字典中将该对象移除就可以了。

``` C#
internal class LockEntry : AbstractRefCounted
{
    internal LockEntry(string name)
        : base(name)
    {
    }

    protected override void CloseInternal()
    {
        LocalLockManager.Instance.Remove(Name);
    }
}
```

