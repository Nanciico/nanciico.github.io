# 算法第四版 —— 符号表


## 符号表 API

``` Text
符号表
public class ST<Key, Value>

                    ST()                            创建符号表
         void       put(Key key, Value val)         向符号表加入键值对（若 value 为 null 则移除该 key）
        Value       get(Key key)                    获取 key 对应的 value，不存在返回 null
         void       delete(Key key)                 从符号表移除键值对
      boolean       contains(Key key)               查询是否存在 key 对应的 value
      boolean       isEmpty()                       判断符号表是否为空
          int       size()                          符号表中键值对个数
Iterable<Key>       keys()                          符号表中所有的 key
```

---

