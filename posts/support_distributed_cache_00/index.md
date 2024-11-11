# 添加分布式缓存 —— 原缓存流程


## 添加分布式缓存的原因

原系统只支持本地缓存，并且经常存在清理缓存的需求，清理缓存会将所有本地缓存全部清空，从而导致大量请求打到数据库。

---

## 原缓存流程

### 写入操作

{{< image src="/images/support_distributed_cache/00_01.jpg" caption="原缓存流程——写入操作" title="原缓存流程——写入操作" >}}

1. 发送 Kafka 消息，获取版本号
2. 写数据库
3. 客户端消费 Kafka 消息，修改本地缓存版本 local_version

### 查询操作

{{< image src="/images/support_distributed_cache/00_02.jpg" caption="原缓存流程——查询操作" title="原缓存流程——查询操作" >}}

---

