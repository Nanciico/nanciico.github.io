# Redis 字典 (dict)


---

## 字典结构定义

### 哈希表 dict

``` C
struct dict {
    // 指向字典类型的指针
    dictType *type;
    // 一个包含两个哈希表的数组， (dictEntry **) 指向一个哈希表
    dictEntry **ht_table[2];
    // 哈希表的节点数量
    unsigned long ht_used[2];
    // 记录 rehash 的进度
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */

    /* Keep small vars at end for optimal (minimal) struct padding */
    // 记录 rehash 暂停
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */

    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as defined
                                 * by dictType's dictEntryBytes. */
};
```

### 哈希表结点 dictEntry

``` C
struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 指向下一个 dictEntry 的 next 指针，当发生哈希冲突时，使用的链表法串成一个链表
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
    // 任意数量的字节（从指针对齐的地址开始），其大小由 dictType 的 dictEntryMetadataBytes() 返回。
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as returned
                                 * by dictType's dictEntryMetadataBytes(). */
};

typedef struct {
    void *key;
    dictEntry *next;
} dictEntryNoValue;
```

### 字典类型 dictType

``` C
typedef struct dictType {
    // 哈希方法
    uint64_t (*hashFunction)(const void *key);
    // key 拷贝方法
    void *(*keyDup)(dict *d, const void *key);
    // value 拷贝方法
    void *(*valDup)(dict *d, const void *obj);
    // key 比较函数
    int (*keyCompare)(dict *d, const void *key1, const void *key2);
    // key 析构方法
    void (*keyDestructor)(dict *d, void *key);
    // value 析构方法
    void (*valDestructor)(dict *d, void *obj);
    // 
    int (*expandAllowed)(size_t moreMem, double usedRatio);
    /* Flags */
    /* The 'no_value' flag, if set, indicates that values are not used, i.e. the
     * dict is a set. When this flag is set, it's not possible to access the
     * value of a dictEntry and it's also impossible to use dictSetKey(). Entry
     * metadata can also not be used. */
    unsigned int no_value:1;
    /* If no_value = 1 and all keys are odd (LSB=1), setting keys_are_odd = 1
     * enables one more optimization: to store a key without an allocated
     * dictEntry. */
    unsigned int keys_are_odd:1;
    /* TODO: Add a 'keys_are_even' flag and use a similar optimization if that
     * flag is set. */

    /* Allow each dict and dictEntry to carry extra caller-defined metadata. The
     * extra memory is initialized to 0 when allocated. */
    size_t (*dictEntryMetadataBytes)(dict *d);
    size_t (*dictMetadataBytes)(void);
    /* Optional callback called after an entry has been reallocated (due to
     * active defrag). Only called if the entry has metadata. */
    void (*afterReplaceEntry)(dict *d, dictEntry *entry);
} dictType;
```

---

## 字典内存结构

// TODO 内存结构图

---

