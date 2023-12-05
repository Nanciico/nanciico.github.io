# Redis 分布式锁设计 —— 内存泄漏问题排查


---

## 内存泄漏现象

在进行性能测试的过程中，发现分布式锁测试程序的内存一直在上涨。起初怀疑是续约线程阻塞未消费续约对象导致续约对象堆积从而使内存上涨，然而事实并没有这么简单。

{{< image src="/images/distributed_lock/05_01.jpg" caption="分布式锁测试 —— 内存暴涨" title="分布式锁测试 —— 内存暴涨" >}}

怀疑分布式锁存在内存泄漏问题，于是获取程序内存 Dump，通过 WinDBG 分析内存问题。

---

## 内存泄漏排查

内存泄漏大部分是因为某些对象仍然存在指针指向它，从而未被垃圾回收器识别并清理。因此排查思路就是：

1. 查找内存中占用内存最大、最多的对象；
2. 排查该对象是否存在 gcroot 指向的路径。

首先使用 `!dumpheap -stat` 命令查看内存中占用最大或者最多的对象。

{{< image src="/images/distributed_lock/05_02.jpg" caption="内存中占用最大或者最多的对象" title="内存中占用最大或者最多的对象" >}}

可以看到占用内存最多的对象类型是 `System.Object[]` 类型。

其次使用命令 `!DumpHeap /d -mt 000007fef6155e70` 查看该类所有对象的地址、大小信息。

最后任意从上一步选择一个对象的地址，使用 `!gcroot 0000000008355cc8` 命令查看对象的 GC 指针信息。

{{< image src="/images/distributed_lock/05_03.jpg" caption="对象 GC 指针引用信息" title="对象 GC 指针引用信息" >}}

这里能够判断是 `System.EventHandler` 导致的内存泄漏问题。

---

## EventHandler 内存泄漏问题

EventHandler 导致内存泄漏问题，是 .NET 中一个非常常见的问题。

原因很容易解释：当 EventHandler 被订阅时，事件的发布者通过 EventHandler 委托持有对订阅者的引用（假设委托是一个实例方法）。

如果发布者的生存时间比订阅者长，那么即使没有其他对订阅者的引用，它也会使订阅者保持被引用状态。

如果使用相同的处理程序取消订阅事件，这将删除处理程序和可能的泄漏。然而，这实际上很少是一个问题，因为通常发布者和订阅者无论如何都有大致相同的生命周期。

在这个问题上，由于事件发布者是一个单例，生命周期是整个程序运行时间，事件的订阅者是 DistributedLock 对象，因此发布者一直持有订阅者的引用，从而使得订阅者无法被垃圾回收，导致内存泄漏。

---

## 参考资料

[Why and How to avoid Event Handler memory leaks?](https://stackoverflow.com/questions/4526829/why-and-how-to-avoid-event-handler-memory-leaks)

[Are you afraid of event handlers because of C# memory leak? Fear not!
](https://www.spicelogic.com/Blog/net-event-handler-memory-leak-16)

---

