# Redis 快速列表(quicklist)


---

## quicklist 结构体定义

### quicklistNode

quicklistNode 结构体定义了快速列表结点，它是一个 32 字节的结构体，包含指向前驱结点和后继结点的指针，一个数据指针 entry 指向实际存储数据的 listpack (或者一个压缩过后的 quicklistLZF)，另外还有一些描述结点元数据的成员。

``` C
/* Node, quicklist, and Iterator are the only data structures used currently. */

/* quicklistNode is a 32 byte struct describing a listpack for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max lp bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, PLAIN=1 (a single item as char array), PACKED=2 (listpack with multiple items).
 * recompress: 1 bit, bool, true if node is temporary decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 10 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    // 前驱结点指针 8 bytes
    struct quicklistNode *prev;
    // 后继结点指针 8 bytes
    struct quicklistNode *next;
    // 数据指针，指向 listpack 或者 quicklistLZF，8 bytes
    unsigned char *entry;
    // entry 占用字节大小 4 bytes
    size_t sz;             /* entry size in bytes */
    // 16 bits，listpack 里的结点数量
    unsigned int count : 16;     /* count of items in listpack */
    // 2 bits，listpack 编码方式， RAW 表示原始 listpack， LZF 表示压缩后的 listpack
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    // 2 bits，表示 entry 中的元素类别是 PLAIN 还是 PACKED
    unsigned int container : 2;  /* PLAIN==1 or PACKED==2 */
    // 1 bit，表示该结点在之前是否压缩过
    unsigned int recompress : 1; /* was this node previous compressed? */
    // 1 bit，表示该结点数据能否被压缩
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    // 1 bit，标记该结点不进行压缩
    unsigned int dont_compress : 1; /* prevent compression of entry that will be used later */
    // 9 bits，用于扩展
    unsigned int extra : 9; /* more bits to steal for future usage */
} quicklistNode;
```

### quicklistLZF

quicklistLZF 是 quicklistNode 中的 listpack 压缩后的结构：

- sz：指 compressed 数组的长度；
- compressed：柔性数组，存储着 listpack 被压缩后的具体的字节数组。

``` C
/* quicklistLZF is a 8+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->entry is compressed, node->entry points to a quicklistLZF */
// quicklistLZF 是一个 8+N 字节大小的结构体
// 压缩后的 listpack 大小存储在 LZF->sz 中，未压缩的长度存储在 quicklistNode->sz 中
typedef struct quicklistLZF {
    size_t sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```

### quicklistBookmark

quicklistBookmark 会作为 quicklist 结构体的最后一个成员，是一个柔性数组。在需要迭代一个很长的 quicklist 时，可以利用此结构实现部分迭代。

``` C
/* Bookmarks are padded with realloc at the end of of the quicklist struct.
 * They should only be used for very big lists if thousands of nodes were the
 * excess memory usage is negligible, and there's a real need to iterate on them
 * in portions.
 * When not used, they don't add any memory overhead, but when used and then
 * deleted, some overhead remains (to avoid resonance).
 * The number of bookmarks used should be kept to minimum since it also adds
 * overhead on node deletion (searching for a bookmark to update). */
typedef struct quicklistBookmark {
    quicklistNode *node;
    char *name;
} quicklistBookmark;
```

### quicklist

quicklist 是一个 40 字节大小的结构体(64 位操作系统下)，同时也是一个双向链表。

``` C
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: 0 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor.
 * 'bookmarks are an optional feature that is used by realloc this struct,
 *      so that they don't consume memory when not used. */
typedef struct quicklist {
    // 8 bytes，头指针
    quicklistNode *head;
    // 8 bytes，尾指针
    quicklistNode *tail;
    // 8 bytes，所有 listpacks 中所有结点总数
    unsigned long count;        /* total count of all entries in all listpacks */
    // 8 bytes，快速列表结点数
    unsigned long len;          /* number of quicklistNodes */
    // 16 bits (64 位操作系统)，每个结点的填充因数
    signed int fill : QL_FILL_BITS;       /* fill factor for individual nodes */
    // 16 bits (64 位操作系统)，两端结点不压缩的深度
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    // 4 bits，bookmarks 数组大小
    unsigned int bookmark_count: QL_BM_BITS;
    // 柔性数组，用于重新分配 quicklist 空间，不使用时不消耗内存
    quicklistBookmark bookmarks[];
} quicklist;
```

### quicklistIter

quicklistIter 是快速列表的迭代器，用于遍历 quicklist。

``` C
typedef struct quicklistIter {
    // 迭代的 quicklist
    quicklist *quicklist;
    // 当前迭代的 quicklistNode
    quicklistNode *current;
    // 当前迭代的 listpack 中的 element 的指针
    unsigned char *zi; /* points to the current element */
    // 当前 listpack 的偏移量
    long offset; /* offset in current listpack */
    // 迭代方向
    int direction;
} quicklistIter;
```

### quicklistEntry

quicklistEntry 用来描述一个 listpack 的 一个 element。

``` C
typedef struct quicklistEntry {
    // 指向 listpack element 所属 quicklist
    const quicklist *quicklist;
    // 指向 listpack element 所属 quicklistNode
    quicklistNode *node;
    // 指向 listpack element 所属 listpack
    unsigned char *zi;
    // 若 listpack element 保存类型为字符串类型，value 和 sz 保存该值
    unsigned char *value;
    // 若 listpack element 保存类型为整形，longval 保存该值
    long long longval;
    size_t sz;
    // 标识 listpack element 在所处 listpack (即 zi 所指向的 listpack) 中的偏移量
    // 表示该 element 是该 listpack 的第几个元素
    int offset;
} quicklistEntry;
```

---

## quicklist API

### quicklistNew

quicklistNew 创建一个赋默认值的 quicklist：

1. 调用 quicklistCreate 创建一个空的快速列表；
2. 为快速列表设置默认值：

    - 设置每个结点的默认填充因数；
    - 设置每个结点的默认压缩深度。
3. 返回快速列表

``` C
/* Create a new quicklist with some default parameters. */
quicklist *quicklistNew(int fill, int compress) {
    quicklist *quicklist = quicklistCreate();
    quicklistSetOptions(quicklist, fill, compress);
    return quicklist;
}

/* Create a new quicklist.
 * Free with quicklistRelease(). */
quicklist *quicklistCreate(void) {
    struct quicklist *quicklist;

    quicklist = zmalloc(sizeof(*quicklist));
    quicklist->head = quicklist->tail = NULL;
    quicklist->len = 0;
    quicklist->count = 0;
    quicklist->compress = 0;
    quicklist->fill = -2;
    quicklist->bookmark_count = 0;
    return quicklist;
}

void quicklistSetOptions(quicklist *quicklist, int fill, int depth) {
    quicklistSetFill(quicklist, fill);
    quicklistSetCompressDepth(quicklist, depth);
}

#define FILL_MAX ((1 << (QL_FILL_BITS-1))-1)
void quicklistSetFill(quicklist *quicklist, int fill) {
    if (fill > FILL_MAX) {
        fill = FILL_MAX;
    } else if (fill < -5) {
        fill = -5;
    }
    quicklist->fill = fill;
}

#define COMPRESS_MAX ((1 << QL_COMP_BITS)-1)
void quicklistSetCompressDepth(quicklist *quicklist, int compress) {
    if (compress > COMPRESS_MAX) {
        compress = COMPRESS_MAX;
    } else if (compress < 0) {
        compress = 0;
    }
    quicklist->compress = compress;
}
```

### _quicklistInsert

``` C
/* Insert a new entry before or after existing entry 'entry'.
 *
 * If after==1, the new value is inserted after 'entry', otherwise
 * the new value is inserted before 'entry'. */
REDIS_STATIC void _quicklistInsert(quicklistIter *iter, quicklistEntry *entry,
                                   void *value, const size_t sz, int after)
{
    // 插入 entry 的目标 quicklist
    quicklist *quicklist = iter->quicklist;
    int full = 0, at_tail = 0, at_head = 0, avail_next = 0, avail_prev = 0;
    int fill = quicklist->fill;
    // 当前 entry 所属 quicklistNode
    quicklistNode *node = entry->node;
    quicklistNode *new_node = NULL;

    if (!node) {
        /* we have no reference node, so let's create only node in the list */
        // 当前 entry 没有所属 quicklistNode，因此需要在快速列表中创建一个结点
        D("No node given!");
        if (unlikely(isLargeElement(sz))) {
            // sz 超过阈值，向 quicklist 插入一个 PlainNode（quicklist->container = 1）
            __quicklistInsertPlainNode(quicklist, quicklist->tail, value, sz, after);
            return;
        }
        // 创建一个新的空结点
        new_node = quicklistCreateNode();
        // 创建一个新 listpack，将 sz 大小的 value 插入 listpack
        // 新结点的 entry 指向新的 listpack
        new_node->entry = lpPrepend(lpNew(0), value, sz);
        // 向 quicklist 插入新结点
        __quicklistInsertNode(quicklist, NULL, new_node, after);
        // 更新维护相关计数器
        new_node->count++;
        quicklist->count++;
        return;
    }

    /* Populate accounting flags for easier boolean checks later */
    if (!_quicklistNodeAllowInsert(node, fill, sz)) {
        D("Current node is full with count %d with requested fill %d",
          node->count, fill);
        // 通过填充因数 fill 判断能否将 sz 大小的数据插入 node 中
        // 不能插入，则标记 full = 1
        full = 1;
    }

    // entry->offset：listpack element 在 listpack 中的偏移量
    // node->count：quicklistNode 中元素个数
    // 如果 after = 1，listpack element 在所处 listpack 最后
    // 则标记 at_tail = 1 表示 entry 在当前 listpack 最后
    // 如果 quicklistNode 的下一个结点允许插入
    // 则标记 avail_next = 1
    if (after && (entry->offset == node->count - 1 || entry->offset == -1)) {
        D("At Tail of current listpack");
        at_tail = 1;
        if (_quicklistNodeAllowInsert(node->next, fill, sz)) {
            D("Next node is available.");
            avail_next = 1;
        }
    }

    if (!after && (entry->offset == 0 || entry->offset == -(node->count))) {
        D("At Head");
        at_head = 1;
        if (_quicklistNodeAllowInsert(node->prev, fill, sz)) {
            D("Prev node is available.");
            avail_prev = 1;
        }
    }

    if (unlikely(isLargeElement(sz))) {
        if (QL_NODE_IS_PLAIN(node) || (at_tail && after) || (at_head && !after)) {
            __quicklistInsertPlainNode(quicklist, node, value, sz, after);
        } else {
            quicklistDecompressNodeForUse(node);
            new_node = _quicklistSplitNode(node, entry->offset, after);
            quicklistNode *entry_node = __quicklistCreatePlainNode(value, sz);
            __quicklistInsertNode(quicklist, node, entry_node, after);
            __quicklistInsertNode(quicklist, entry_node, new_node, after);
            quicklist->count++;
        }
        return;
    }

    /* Now determine where and how to insert the new element */
    if (!full && after) {
        D("Not full, inserting after current position.");
        quicklistDecompressNodeForUse(node);
        node->entry = lpInsertString(node->entry, value, sz, entry->zi, LP_AFTER, NULL);
        node->count++;
        quicklistNodeUpdateSz(node);
        quicklistRecompressOnly(node);
    } else if (!full && !after) {
        D("Not full, inserting before current position.");
        quicklistDecompressNodeForUse(node);
        node->entry = lpInsertString(node->entry, value, sz, entry->zi, LP_BEFORE, NULL);
        node->count++;
        quicklistNodeUpdateSz(node);
        quicklistRecompressOnly(node);
    } else if (full && at_tail && avail_next && after) {
        /* If we are: at tail, next has free space, and inserting after:
         *   - insert entry at head of next node. */
        D("Full and tail, but next isn't full; inserting next node head");
        new_node = node->next;
        quicklistDecompressNodeForUse(new_node);
        new_node->entry = lpPrepend(new_node->entry, value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(new_node);
        quicklistRecompressOnly(node);
    } else if (full && at_head && avail_prev && !after) {
        /* If we are: at head, previous has free space, and inserting before:
         *   - insert entry at tail of previous node. */
        D("Full and head, but prev isn't full, inserting prev node tail");
        new_node = node->prev;
        quicklistDecompressNodeForUse(new_node);
        new_node->entry = lpAppend(new_node->entry, value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(new_node);
        quicklistRecompressOnly(node);
    } else if (full && ((at_tail && !avail_next && after) ||
                        (at_head && !avail_prev && !after))) {
        /* If we are: full, and our prev/next has no available space, then:
         *   - create new node and attach to quicklist */
        D("\tprovisioning new node...");
        new_node = quicklistCreateNode();
        new_node->entry = lpPrepend(lpNew(0), value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
    } else if (full) {
        /* else, node is full we need to split it. */
        /* covers both after and !after cases */
        D("\tsplitting node...");
        quicklistDecompressNodeForUse(node);
        new_node = _quicklistSplitNode(node, entry->offset, after);
        if (after)
            new_node->entry = lpPrepend(new_node->entry, value, sz);
        else
            new_node->entry = lpAppend(new_node->entry, value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
        _quicklistMergeNodes(quicklist, node);
    }

    quicklist->count++;

    /* In any case, we reset iterator to forbid use of iterator after insert.
     * Notice: iter->current has been compressed in _quicklistInsert(). */
    resetIterator(iter); 
}
```

---

