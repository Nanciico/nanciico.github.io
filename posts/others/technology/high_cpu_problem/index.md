# Redis 客户端在 NET8 容器环境下 CPU 高问题排查


---

## 现象

Redis 客户端在 NET8 Unix 容器环境下，CPU 比在 NET45 Windows 环境高。

性能测试数据：

| 环境 | 线程数 | 延迟 | QPS | CPU |
| :----: | :----: | :----: | :----: | :----: |
| 容器 NET 8.0 | 10 | 0.17ms | 58982 | 75% |
| Windows NET45 | 10 | 0.28ms | 35765 | 16.7% |

---

## 问题排查

通过 CPU Profile 文件，发现 `System.Net.Sockets.SocketAsyncEngine.EventLoop()` 方法占用了 43.59% 的 CPU。

{{< image src="/images/others/technology/02_01.jpg" caption="CPU Profile" title="CPU Profile" >}}

通过阅读代码，NET8 在 Unix 环境且多路复用为 epoll 或者 kqueue 时，会使用内部的 System.Net.Sockets.SocketAsyncEngine 优化网络 IO。

SocketAsyncEngine 负责从 epoll | kqueue 中获取通知，并将正确的工作项（Socket reads and writes）调度到线程池中。

{{< image src="/images/others/technology/02_02.jpg" caption="SocketAsyncEngine" title="SocketAsyncEngine" >}}

---

## 原因

SocketAsyncEngine 占用 1 线程，执行 SocketAsyncEngine.EventLoop() 方法，接收 SocketEvent 后，调用 ScheduleToProcessEvents() 方法将事件交给 ThreadPool 处理。

{{< image src="/images/others/technology/02_03.jpg" caption="SocketAsyncEngine.EventLoop()" title="SocketAsyncEngine.EventLoop()" >}}

{{< image src="/images/others/technology/02_04.jpg" caption="ScheduleToProcessEvents()" title="ScheduleToProcessEvents()" >}}

ThreadPool 线程池执行 Execute 方法，Execute 方法会再次调用 ScheduleToProcessEvents() 向线程池提交任务。

{{< image src="/images/others/technology/02_05.jpg" caption="Execute()" title="Execute()" >}}

因此在性能测试大量 IO 事件的场景下，会一直向 ThreadPool 提交任务，抬升 CPU。

---

