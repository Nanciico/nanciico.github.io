# 添加分布式缓存 —— 缓存设计


## 缓存设计

### 写入操作

{{< image src="/images/support_distributed_cache/01_01.jpg" caption="分布式缓存设计——写入操作" title="分布式缓存设计——写入操作" >}}

1. 发送 Kafka 消息，获取版本号
2. **将版本号写入 Redis**
3. 写数据库
4. 客户端消费 Kafka 消息，修改本地缓存版本 local_version

### 查询操作

{{< image src="/images/support_distributed_cache/01_02.jpg" caption="分布式缓存设计——查询操作" title="分布式缓存设计——查询操作" >}}

1. 当本地缓存 local_cache 不存在，或者本地缓存校验版本失败，查询 Redis 缓存；
2. 若 Redis 缓存校验成功，使用 Redis 缓存更新本地缓存，返回 Redis 缓存数据，否则查询数据库；
3. 查询数据库数据，使用数据库数据更新 Redis 缓存和本地缓存，返回数据库数据。

#### 查询操作的设计细节

##### 为何 Redis 缓存需要与本地版本校验？

redis_cache 与 redis_version 校验成功，表示 Redis 缓存版本正确。

redis_cache 与 local_version 校验成功，表示 Redis 缓存数据不比本地缓存数据旧。

这样才能确保 Redis 缓存数据不是旧数据。

##### 为何要将 Redis 版本写入本地版本？

在这个场景下：发送 Kafka 消息成功，写 Redis 版本成功，写数据库失败，客户端重启

客户端重启使得本地缓存与本地版本丢失，由于写数据库失败，客户端会将数据库的旧数据写入本地缓存，造成该客户端一直读取旧数据。

通过将 Redis 版本写入本地版本，确保客户端重启不会丢失本地版本，保证 Redis 版本与本地版本的最终一致性。

---

