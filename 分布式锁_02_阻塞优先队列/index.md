# 阻塞优先队列


## 使用阻塞优先队列的原因

续约线程从阻塞优先队列中获取分布式锁对象进行续约。

- 如果阻塞优先队列为空，线程应该等待阻塞优先队列有数据后再来获取数据；
- 如果阻塞优先队列不为空，但未到分布式锁的续约时间，则线程需要等待分布式锁能够续约后再来获取数据；
- 如果阻塞优先队列不为空，且已经到了分布式锁的续约时间，此时线程应从阻塞优先队列中获取数据并进行续约操作。

因此在本场景下，线程有两个时机是需要阻塞的。

## 阻塞优先队列的实现

``` C#
public class LockPriorityBlockingQueue<T>
    where T : LockBase
{
    private int _capacity;
    private int _size;

    private T[] _heap;

    private readonly object _locker = new object();

    public LockPriorityBlockingQueue()
    {
        _capacity = 10;
        _size = 0;
        _heap = new T[_capacity + 1];
    }

    public void Offer(T item)
    {
        Monitor.Enter(_locker);
        GrowIfNecessary();

        try
        {
            Insert(item);
            Monitor.Pulse(_locker);
        }
        finally
        {
            Monitor.Exit(_locker);
        }
    }

    public T Poll()
    {
        Monitor.Enter(_locker);
        T item;
        try
        {
            var waitTime = Timeout.Infinite;
            while ((item = Peek()) == null || (waitTime = item.NextRenewTime) > 0)
            {
                Monitor.Wait(_locker, waitTime);
                waitTime = Timeout.Infinite;
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

其中 Insert、Delete、Peek 等方法为优先队列的基本操作。

