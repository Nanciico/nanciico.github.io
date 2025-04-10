# 算法第四版 —— 符号表


符号表：描述一种抽象机制，通过这种机制，我们保存信息（value），之后可以通过指定一个键（key）来搜索和检索这些信息。

---

## 符号表的基础 API

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

## 有序的符号表

### 有序符号表 API

``` Text
有序符号表扩展 API
public class ST<Key extends Comparable<Key>, Value>

          Key       min()                           smallest key
          Key       max()                           largest key
          Key       floor(Key key)                  largest key less than or equal to key
          Key       ceiling(Key key)                smallest key greater than or equal to key
          int       rank(Key key)                   number of keys less than key
          Key       select(int k)                   key of rank k
         void       deleteMin()                     delete smallest key
         void       deleteMax()                     delete largest key
          int       size(Key lo, Key hi)            number of keys in [lo..hi]
Iterable<Key>       keys(Key lo, Key hi)            keys in [lo..hi], in sorted order
```


