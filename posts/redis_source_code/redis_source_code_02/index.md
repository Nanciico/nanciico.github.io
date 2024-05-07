# Redis 链表 (adlist)


---

## 链表的结构

Redis 中链表的结构是这样的：一个 list 结构存储链表的元数据，每个链表结点 listNode 之间通过指针相连，head 与 tail 分别指向链表的头结点和尾结点。

{{< image src="/images/redis_source_code/02_01.jpg" caption="链表的结构" title="链表的结构" >}}

---

## 链表结构定义

### listNode

链表结点 listNode，包含一个前驱结点指针，一个后继结点指针，和一个 `void *` 指针指向具体的元素。

``` C
typedef struct listNode {
    // 前驱结点指针
    struct listNode *prev;
    // 后继结点指针
    struct listNode *next;
    // 指向具体元素
    void *value;
} listNode;
```

### listIter

链表迭代器 listIter 包含一个指向下一个结点的指针，和遍历方向。

``` C
/* Directions for iterators */
#define AL_START_HEAD 0
#define AL_START_TAIL 1

typedef struct listIter {
    // 指向下一个结点的指针
    listNode *next;
    // 遍历方向，从头开始还是从尾开始
    int direction;
} listIter;
```

### list

list 结构使用 dup/free/match 函数指针实现多态。不同类型的链表里的链表结点是不同的，多态的使用使得链表能够根据不同的链表结点自定义链表的 dup/free/match 操作。

``` C
typedef struct list {
    // 链表头结点
    listNode *head;
    // 链表尾结点
    listNode *tail;
    // 链表可以自定义 复制/释放/比较 函数
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    // 链表结点数量
    unsigned long len;
} list;
```

比如在 slowlog 模块中，使用 `listSetFreeMethod` 设置链表的 free 函数为 `slowlogFreeEntry` 来释放 slowlog 结点的内存。

``` C
/* Free a slow log entry. The argument is void so that the prototype of this
 * function matches the one of the 'free' method of adlist.c.
 *
 * This function will take care to release all the retained object. */
void slowlogFreeEntry(void *septr) {
    slowlogEntry *se = septr;
    int j;

    for (j = 0; j < se->argc; j++)
        decrRefCount(se->argv[j]);
    zfree(se->argv);
    sdsfree(se->peerid);
    sdsfree(se->cname);
    zfree(se);
}

/* Initialize the slow log. This function should be called a single time
 * at server startup. */
void slowlogInit(void) {
    server.slowlog = listCreate();
    server.slowlog_entry_id = 0;
    listSetFreeMethod(server.slowlog,slowlogFreeEntry);
}
```

---

## 应用

- 服务端使用链表来保存所有连接的客户端，连接的从库客户端等。如 server.clients 、server.slaves
- 慢查询 slowlog 使用链表来存储慢查询条目。server.slowlog

---

