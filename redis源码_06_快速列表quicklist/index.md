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

fill 和 compress 成员可以通过配置文件配置：

- fill 成员对应配置：list-max-listpack-size
  - -5: 每个 quidklistNode 结点的 listpack 字节大小不能超过 64 Kb
  - -4: 每个 quidklistNode 结点的 listpack 字节大小不能超过 32 Kb
  - -3: 每个 quidklistNode 结点的 listpack 字节大小不能超过 16 Kb
  - -2: 每个 quidklistNode 结点的 listpack 字节大小不能超过 8 Kb
  - -1: 每个 quidklistNode 结点的 listpack 字节大小不能超过 4 Kb
  - 整数表示 listpack 结构最多包含的 entry 个数
- compress 成员对应配置：list-compress-depth，表示压缩结点的深度
  - 0 表示不压缩
  - 1 表示 quicklist 列表的两端各有 1 个节点不压缩，中间的节点压缩
  - 2 表示 quicklist 列表的两端各有 2 个节点不压缩，中间的节点压缩
  - 以此类推

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

_quicklistInsert 以一个已存在的 listpack entry 前或后插入一个 entry 结点：

- 当前 quicklistNode 结点的 listpack 可以插入
  - 插入已存在的 entry 前 (after = 0)
  - 插入已存在的 entry 后 (after = 1)
- 如果当前 quicklistNode 结点的 listpack 由于 fill 的配置，无法继续插入
  - 已存在的 entry 是 quicklistNode 的头结点，当前 quicklistNode 结点的前驱指针不为空，并且 after = 0
    - 前驱结点可以插入，因此插在前驱结点的尾部
    - 前驱结点不可以插入，因此要在当前结点和前驱结点之间创建一个新结点保存要插入的 entry
  - 已存在的 entry 是 quicklistNode 的头结点，当前 quicklistNode 结点的前驱指针不为空，并且 after = 1
    - 后继结点可以插入，因此插在后继结点的头部
    - 后继结点不可以插入，因此要在当前结点和后继结点之间创建一个新结点保存要插入的 entry
  - 以上情况都不满足，当前结点是满的，且需要将 entry 插入在 listpack 中间位置，则需要分割当前 quicklistNode 结点，然后插入到其中一个新结点，如果条件允许，还会进行结点两侧小结点的合并

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
    // full：当前结点是否已满
    // at_tail：entry 在当前结点尾部
    // at_head：entry 在当前结点头部
    // avail_next：下一个结点可插入
    // avail_prev：上一个结点可插入
    int full = 0, at_tail = 0, at_head = 0, avail_next = 0, avail_prev = 0;
    int fill = quicklist->fill;
    // 当前 entry 所属 quicklistNode
    quicklistNode *node = entry->node;
    quicklistNode *new_node = NULL;

    if (!node) {
        /* we have no reference node, so let's create only node in the list */
        // entry 没有所属 quicklistNode，因此需要在快速列表中创建一个结点
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

    // 处理大元素(PLAIN NODE)
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
    // 此后的代码是决定在哪里插入和如何插入的代码

    if (!full && after) {
        // 当前结点没有满，且是后插入，则直接将新 entry 插入到当前 entry 后
        D("Not full, inserting after current position.");
        // 如果结点压缩，则将当前结点解压，并设置 recompress = 1
        quicklistDecompressNodeForUse(node);
        node->entry = lpInsertString(node->entry, value, sz, entry->zi, LP_AFTER, NULL);
        node->count++;
        // 更新 node->sz
        quicklistNodeUpdateSz(node);
        // 如果结点需要重新压缩，则进行重新压缩
        quicklistRecompressOnly(node);
    } else if (!full && !after) {
        // 当前结点未满，并且是前插入，测直接将新 entry 插入到当前 entry 前
        D("Not full, inserting before current position.");
        quicklistDecompressNodeForUse(node);
        node->entry = lpInsertString(node->entry, value, sz, entry->zi, LP_BEFORE, NULL);
        node->count++;
        quicklistNodeUpdateSz(node);
        quicklistRecompressOnly(node);
    } else if (full && at_tail && avail_next && after) {
        /* If we are: at tail, next has free space, and inserting after:
         *   - insert entry at head of next node. */
        // 如果当前结点是满的，且当前 entry 是尾部，且存在后继结点，且后继结点未满
        // 插入新 entry 到后继结点头部
        D("Full and tail, but next isn't full; inserting next node head");
        new_node = node->next;
        quicklistDecompressNodeForUse(new_node);
        new_node->entry = lpPrepend(new_node->entry, value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(new_node);
        // 记：为什么此处需要重新压缩 node 结点？
        quicklistRecompressOnly(node);
    } else if (full && at_head && avail_prev && !after) {
        
        /* If we are: at head, previous has free space, and inserting before:
         *   - insert entry at tail of previous node. */
        // 如果当前结点是满的，且当前 entry 是头部，且存在前驱结点，且前驱结点未满
        // 插入新 entry 到前驱结点尾部
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
        // 如果当前结点已满，且前驱、后继结点都没有空间
        // 则创建一个新结点并接入 quicklist
        D("\tprovisioning new node...");
        new_node = quicklistCreateNode();
        new_node->entry = lpPrepend(lpNew(0), value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
    } else if (full) {
        /* else, node is full we need to split it. */
        /* covers both after and !after cases */
        // 当前结点满了，且是在结点中间插入
        // 需要将这个满结点，根据当前 entry 位置，拆分成两个结点
        D("\tsplitting node...");
        quicklistDecompressNodeForUse(node);
        // 
        new_node = _quicklistSplitNode(node, entry->offset, after);
        if (after)
            new_node->entry = lpPrepend(new_node->entry, value, sz);
        else
            new_node->entry = lpAppend(new_node->entry, value, sz);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
        // 尝试合并：
        // - (center->prev->prev, center->prev)
        // - (center->next, center->next->next)
        // - (center->prev, center)
        // - (center, center->next)
        _quicklistMergeNodes(quicklist, node);
    }

    quicklist->count++;

    /* In any case, we reset iterator to forbid use of iterator after insert.
     * Notice: iter->current has been compressed in _quicklistInsert(). */
    resetIterator(iter); 
}
```

### quicklistPush

quicklistPush 方法向 quicklist 头尾两端插入元素，根据 where 参数选择向头结点添加还是尾结点添加元素。代码以向头结点添加元素来做分析。

``` C
/* Wrapper to allow argument-based switching between HEAD/TAIL pop */
void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
    /* The head and tail should never be compressed (we don't attempt to decompress them) */
    if (quicklist->head)
        assert(quicklist->head->encoding != QUICKLIST_NODE_ENCODING_LZF);
    if (quicklist->tail)
        assert(quicklist->tail->encoding != QUICKLIST_NODE_ENCODING_LZF);

    if (where == QUICKLIST_HEAD) {
        quicklistPushHead(quicklist, value, sz);
    } else if (where == QUICKLIST_TAIL) {
        quicklistPushTail(quicklist, value, sz);
    }
}

/* Add new entry to head node of quicklist.
 *
 * Returns 0 if used existing head.
 * Returns 1 if new head created. */
// 如果添加结点后未改变头结点指针，返回 0
// 如果添加结点后创建了一个新的头结点，返回 1
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;

    // 处理大元素
    if (unlikely(isLargeElement(sz))) {
        __quicklistInsertPlainNode(quicklist, quicklist->head, value, sz, 0);
        return 1;
    }

    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        // 当前结点允许插入元素，向头结点 listpack 头部插入元素，更新头结点大小
        quicklist->head->entry = lpPrepend(quicklist->head->entry, value, sz);
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        // 创建新结点
        quicklistNode *node = quicklistCreateNode();
        node->entry = lpPrepend(lpNew(0), value, sz);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    // 维护结点数量
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
```

#### _quicklistNodeAllowInsert

_quicklistNodeAllowInsert 方法判断 quicklistNode 是否允许插入新的元素。

``` C
/* Maximum estimate of the listpack entry overhead.
 * Although in the worst case(sz < 64), we will waste 6 bytes in one
 * quicklistNode, but can avoid memory waste due to internal fragmentation
 * when the listpack exceeds the size limit by a few bytes (e.g. being 16388). */
#define SIZE_ESTIMATE_OVERHEAD 8

// 判断 node 结点是否允许插入
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz) {
    // node 结点为空，不允许插入
    if (unlikely(!node))
        return 0;

    // 如果 node 是 PLAIN 类型结点，或者 sz 太大，不允许插入
    if (unlikely(QL_NODE_IS_PLAIN(node) || isLargeElement(sz)))
        return 0;

    /* Estimate how many bytes will be added to the listpack by this one entry.
     * We prefer an overestimation, which would at worse lead to a few bytes
     * below the lowest limit of 4k (see optimization_level).
     * Note: No need to check for overflow below since both `node->sz` and
     * `sz` are to be less than 1GB after the plain/large element check above. */
    // 估计添加元素后结点的大小，超过限制则不允许插入
    size_t new_sz = node->sz + sz + SIZE_ESTIMATE_OVERHEAD;
    if (unlikely(quicklistNodeExceedsLimit(fill, new_sz, node->count + 1)))
        return 0;
    return 1;
}
```

### quicklistPop

quicklistPop 方法

``` C
/* Default pop function
 *
 * Returns malloc'd value from quicklist */
int quicklistPop(quicklist *quicklist, int where, unsigned char **data,
                 size_t *sz, long long *slong) {
    unsigned char *vstr = NULL;
    size_t vlen = 0;
    long long vlong = 0;
    if (quicklist->count == 0)
        return 0;
    int ret = quicklistPopCustom(quicklist, where, &vstr, &vlen, &vlong,
                                 _quicklistSaver);
    if (data)
        *data = vstr;
    if (slong)
        *slong = vlong;
    if (sz)
        *sz = vlen;
    return ret;
}

/* pop from quicklist and return result in 'data' ptr.  Value of 'data'
 * is the return value of 'saver' function pointer if the data is NOT a number.
 *
 * If the quicklist element is a long long, then the return value is returned in
 * 'sval'.
 *
 * Return value of 0 means no elements available.
 * Return value of 1 means check 'data' and 'sval' for values.
 * If 'data' is set, use 'data' and 'sz'.  Otherwise, use 'sval'. */
int quicklistPopCustom(quicklist *quicklist, int where, unsigned char **data,
                       size_t *sz, long long *sval,
                       void *(*saver)(unsigned char *data, size_t sz)) {
    unsigned char *p;
    unsigned char *vstr;
    unsigned int vlen;
    long long vlong;
    int pos = (where == QUICKLIST_HEAD) ? 0 : -1;

    if (quicklist->count == 0)
        return 0;

    if (data)
        *data = NULL;
    if (sz)
        *sz = 0;
    if (sval)
        *sval = -123456789;

    quicklistNode *node;
    if (where == QUICKLIST_HEAD && quicklist->head) {
        node = quicklist->head;
    } else if (where == QUICKLIST_TAIL && quicklist->tail) {
        node = quicklist->tail;
    } else {
        return 0;
    }

    /* The head and tail should never be compressed */
    assert(node->encoding != QUICKLIST_NODE_ENCODING_LZF);

    if (unlikely(QL_NODE_IS_PLAIN(node))) {
        if (data)
            *data = saver(node->entry, node->sz);
        if (sz)
            *sz = node->sz;
        quicklistDelIndex(quicklist, node, NULL);
        return 1;
    }

    p = lpSeek(node->entry, pos);
    vstr = lpGetValue(p, &vlen, &vlong);
    if (vstr) {
        if (data)
            *data = saver(vstr, vlen);
        if (sz)
            *sz = vlen;
    } else {
        if (data)
            *data = NULL;
        if (sval)
            *sval = vlong;
    }
    quicklistDelIndex(quicklist, node, &p);
    return 1;
}
```

---

