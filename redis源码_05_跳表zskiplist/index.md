# Redis 跳表(zskiplist)


---

## Redis 跳表的实现

Redis 中的跳表实现基于 William Pugh 的论文 [Skip Lists: A Probabilistic Alternative to Balanced Trees](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf) 实现的，但做了如下修改：

- 允许分数 score 重复；
- 比较操作不仅会比较 score，还会比较成员数据（如果 score 相同的话）；
- 最底层链表结点有反向指针，是一个双向链表，能够支持从尾到头遍历跳表。

{{< image src="/images/redis_source_code/05_01.jpg" caption="Redis 跳表的结构" title="Redis 跳表的结构" >}}

### 跳表的结构定义

zskiplistNode 是跳表结点的结构体：

1. ele：存储成员字符串；
2. score：结点的分值，当 score 相同时，会使用 ele 排序，整体是升序；
3. backward：指向前驱结点的指针，可以实现从表尾向表头方向的遍历，一个结点只有第一层有前驱结点；
4. level：层级数组，每个数组元素代表跳表的一层，每一层包含指向后驱结点的指针和跨度，跳表结点的层数是随机的；
    - forward：指向后继结点，可以实现从表头向表尾方向的遍历；
    - span：记录当前结点和后继结点之间跨越了多少个结点。

``` C
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```

zskiplist 是跳表的结构体：

1. header、tail 指针指向跳表的头、尾结点；
2. length 记录跳表结点个数；
3. level 记录跳表的层级，即目前跳表中层数最高的那个数据结点的层级，遍历的时候从这个层级开始遍历。

``` C
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

zset 数据结构，在数据量较少时，使用 listpack 实现；一般情况为跳表。

zset 由一个字典和一个跳表组成：

1. 字典结构保存成员和分数的映射关系，查询单独成员分数的时间复杂度 O(1)；
2. 跳表结构保存存储有序的数据。

``` C
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

---

## 跳表 API

### zslCreate

zslCreate 创建一个空跳表。

``` C
/* Create a new skiplist. */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    // 创建哨兵结点，一个 32 层高的空结点
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

### zslCreateNode

zslCreateNode 创建一个跳表结点。

``` C
/* Create a skiplist node with the specified number of levels.
 * The SDS string 'ele' is referenced by the node after the call. */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
```

### zslGetRank

zslGetRank 获取结点排名。结点排名 rank 是通过结点 span 跨度计算出来的。

``` C
/* Find the rank for an element by both score and key.
 * Returns 0 when the element cannot be found, rank otherwise.
 * Note that the rank is 1-based due to the span of zsl->header to the
 * first element. */
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    // 从顶层开始搜索
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            // span 累加计算 rank
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        // 目标结点存在，返回 rank
        if (x->ele && x->score == score && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
    }
    // 没有找到结点，返回 0
    return 0;
}
```

### 范围查找

针对范围查找，zrangespec 结构体存储查找范围 min / max 和是否包含 min / max 的信息，并使用一些辅助函数进行范围判断：

- zslValueGteMin：根据 spec->minex 判断 value 是否满足下界要求；
- zslValueLteMax：根据 spec->maxex 判断 value 是否满足上界要求；
- zslIsInRange：判断跳表是否存在有在 range 范围内的结点。

``` C
/* Struct to hold an inclusive/exclusive range spec by score comparison. */
typedef struct {
    double min, max;
    int minex, maxex; /* are min or max exclusive? */
} zrangespec;

// 如果 spec->minex 成立，判断 value > spec->min 否则判断 value >= spec->min
int zslValueGteMin(double value, zrangespec *spec) {
    return spec->minex ? (value > spec->min) : (value >= spec->min);
}

int zslValueLteMax(double value, zrangespec *spec) {
    return spec->maxex ? (value < spec->max) : (value <= spec->max);
}

/* Returns if there is a part of the zset is in range. */
int zslIsInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;

    /* Test for ranges that will always be empty. */
    // range 范围非法，返回 0
    if (range->min > range->max ||
            (range->min == range->max && (range->minex || range->maxex)))
        return 0;
    // 如果尾结点为空 或者 尾结点值比范围内最小值还大，返回 0
    x = zsl->tail;
    if (x == NULL || !zslValueGteMin(x->score,range))
        return 0;
    // 如果头结点为空 或者 头结点值比范围内最大值还小，返回 0
    x = zsl->header->level[0].forward;
    if (x == NULL || !zslValueLteMax(x->score,range))
        return 0;
    return 1;
}
```

zslFirstInRange 和 zslLastInRange 两个方法分别实现了查找 range 分数范围内的第一个结点或者最后一个结点。

``` C
/* Find the first node that is contained in the specified range.
 * Returns NULL when no element is contained in the range. */
// 查找跳表 zsl 在 range 范围内的第一个结点
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;
    int i;

    /* If everything is out of range, return early. */
    if (!zslIsInRange(zsl,range)) return NULL;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* Go forward while *OUT* of range. */
        // 结点不在范围内，一直向前
        while (x->level[i].forward &&
            !zslValueGteMin(x->level[i].forward->score,range))
                x = x->level[i].forward;
    }

    /* This is an inner range, so the next node cannot be NULL. */
    x = x->level[0].forward;
    serverAssert(x != NULL);

    /* Check if score <= max. */
    if (!zslValueLteMax(x->score,range)) return NULL;
    return x;
}

/* Find the last node that is contained in the specified range.
 * Returns NULL when no element is contained in the range. */
// 查找跳表 zsl 在 range 范围内的最后一个结点
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;
    int i;

    /* If everything is out of range, return early. */
    if (!zslIsInRange(zsl,range)) return NULL;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* Go forward while *IN* range. */
        // 结点在范围内，一直向前
        while (x->level[i].forward &&
            zslValueLteMax(x->level[i].forward->score,range))
                x = x->level[i].forward;
    }

    /* This is an inner range, so this node cannot be NULL. */
    serverAssert(x != NULL);

    /* Check if score >= min. */
    if (!zslValueGteMin(x->score,range)) return NULL;
    return x;
}
```

### zslInsert

zslInsert 向跳表中插入结点：

1. 查找到要插入结点的位置，update 数组记录搜索路径；
2. 创建结点，插入到对应位置；
3. 维护每一层指针关系、跨度等信息。

``` C
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    // updata 数组记录搜索路径
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // rank 数组记录每一层 span 总和
    unsigned long rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        // 跳表最高层 rank 初始化为 0，下层 rank 初始化为上一层结果
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            // 计算本层到插入结点位置 span 总和
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        // 记录每一层最接近插入结点的结点
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    // 生成随机 level
    level = zslRandomLevel();
    // 如果新的层高大于跳表本身的层高，维护新层数据（rank 和 update）
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    // 创建新结点
    x = zslCreateNode(level,score,ele);
    // 维护每一层的链表和跨度
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    // 由于新增结点的原因，上层的跨度都需要加一
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置新结点 backward 为 update[0]
    // 若插入的新结点是跳表的第一个插入的结点，就指向 NULL
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        // 维护后继结点的 backward 指针指向自己
        x->level[0].forward->backward = x;
    else
        // 新结点没有后继结点，说明是链表最后一个结点，设置为跳表尾结点
        zsl->tail = x;
    // 维护跳表长度
    zsl->length++;
    return x;
}
```

随机生成结点层级的方法如下：

在跳表论文中，对于一个新结点我们期望晋升的概率为 50%。不过在 Redis 中，晋升概率为 25%。所以 Redis 中的跳表更加扁平化，层级相对较低，从概率上讲相当于一颗四叉树。ZSKIPLIST_P 值越大，跳表会越高，内存占用会越大，时间会越快，反之亦然。Redis 如此设置晋升概率是在空间和时间上做出权衡，避免消耗过多的内存，也能相对保证时间效率。

``` C
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    static const int threshold = ZSKIPLIST_P*RAND_MAX;
    int level = 1;
    while (random() < threshold)
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

### zslDelete

zslDelete 查找并删除结点：

1. 查找到要插入结点的位置，update 数组记录搜索路径；
2. 删除结点；
3. 维护每一层指针关系、跨度等信息。

``` C
/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele). */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    // 搜素删除结点位置，维护 update 数组
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    // 删除结点
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        // 如果 node 为 NULL 释放结点内存
        // 否则将 node 指针指向需要删除的结点
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}

/* Internal function used by zslDelete, zslDeleteRangeByScore and
 * zslDeleteRangeByRank. */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    // 从底层到高层遍历，维护 update 数组
    // update 数组指向删除结点，就维护指针和跨度
    // 否则仅仅维护跨度
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        // 维护后退指针
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    // 更新跳表层数，如果 header->forward 没有指向结点，说明该层为空，跳表层数减一
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    // 结点数量减一
    zsl->length--;
}
```

---

