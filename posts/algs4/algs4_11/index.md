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

## 无序链表中的顺序查找

符号表中使用数据结构的一个简单选择是链表，每个节点存储一个键值对。

``` Java
public class SequentialSearchST<Key, Value> {
    private Node first; // first node in the linked list

    private class Node { // linked-list node
        Key key;
        Value val;
        Node next;

        public Node(Key key, Value val, Node next) {
        this.key = key;
        this.val = val;
        this.next = next;
        }
    }

    // Search for key, return associated value.
    public Value get(Key key) {
        for (Node x = first; x != null; x = x.next) {
            if (key.equals(x.key)) {
                return x.val; // search hit
            }   
        }

        return null; // search miss
    }

    // Search for key. Update value if found; grow table if new.
    public void put(Key key, Value val) {
        for (Node x = first; x != null; x = x.next) {
            if (key.equals(x.key)) {
                x.val = val;    // Search hit: update val.
                return;
            }
        }

        first = new Node(key, val, first); // Search miss: add new node.
    }
}
```

---

## 有序数组中的二分查找

有序符号表，使用的数据结构是一对平行的数组，一个存储键，一个存储值。

``` Java
public class BinarySearchST<Key extends Comparable<Key>, Value> {
    private Key[] keys;
    private Value[] vals;
    private int N;

    public BinarySearchST(int capacity) {
        keys = (Key[]) new Comparable[capacity];
        vals = (Value[]) new Object[capacity];
    }

    public int size() { return N; }

    public Value get(Key key) {
        if (isEmpty()) {
            return null;
        }

        int i = rank(key);
        if (i < N && keys[i].compareTo(key) == 0) {
            return vals[i];
        } else {
            return null;
        }
    }

    // Search for key. Update value if found; grow table if new.
    public void put(Key key, Value val) { 
        int i = rank(key);
        if (i < N && keys[i].compareTo(key) == 0) {
            vals[i] = val;
            return;
        }

        for (int j = N; j > i; j--) { 
            keys[j] = keys[j-1];
            vals[j] = vals[j-1];
        }
        keys[i] = key;
        vals[i] = val;
        N++;
    }

    public int rank(Key key) {
        if (hi < lo) {
            return lo;
        }

        int mid = lo + (hi - lo) / 2;
        int cmp = key.compareTo(keys[mid]);
        if (cmp < 0) {
            return rank(key, lo, mid-1);
        }
        else if (cmp > 0) {
            return rank(key, mid+1, hi);
        }
        else {
            return mid;
        }
    }
}
```

---

## 符号表各种实现的优缺点

| 使用的数据结构 | 实现 | 优点 | 缺点 |
| :--: | :--: | :--: | :--: |
| 链表（顺序查找） | SequentialSearchST | 适用于小型问题 | 对于大型符号表很慢 |
| 有序数组（二分查找） | BinarySearchST | 最优的查找效率和空间需求，能够进行有序性相关的操作 | 插入操作很慢 |
| 二叉查找树 | BST | 实现简单，能够进行有序性相关的操作 | 没有性能上界的保证，链接需要额外的空间 |
| 平衡二叉查找树 | RedBlackBST | 最优的查找和插入效率，能够进行有序性相关的操作 | 链接需要额外的空间 |
| 散列表 | SeparateChainHashST \ LinearProblingHashST | 能够快速地查找和插入常见类型的数据 | 需要计算每种类型的数据的散列，无法进行有序性相关的操作，链接和空结点需要额外空间 |

---

