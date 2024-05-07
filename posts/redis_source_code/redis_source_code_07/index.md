# Redis 整数集合 (intset)


在 Redis 7.2 以前， set 数据类型底层实现的数据结构为 intset 或者 hashtable。Redis 7.2 中给 set 增加了 listpack 编码方式。当 set 中的元素都是整数并且元素数目较少的时候，会使用 intset 作为底层实现，否则会使用 hashtable 或者 listpack 来作为底层实现。

intset 是 Redis 用来保存整数值的集合抽象数据结构，它可以保存类型为 int16_t、int32_t 或者 int64_t 的整数值（设置多个类型来节省内存，当整数元素超过编码范围区间时，会进行类型升级扩容，此时升级的时间复杂度为 O(N) ），并且保证集合中不会出现重复元素。各个元素在 intset 里是按照从小到大的有序的存储（里面的搜索是用的二分查找，时间复杂度为 O(log N)），并且不包含重复值。

---

## intset 结构体定义

intset 包含两部分，header 元数据（encoding、length）和实际的整数数组。

``` C
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

typedef struct intset {
    uint32_t encoding;          // 数据编码，表示 intset 每个整数用几个字节来编码存储
    uint32_t length;            // 元素个数，即 contents 数组长度
    int8_t contents[];          // 柔性数组，存放实际数据，数据项不重复，且升序排序
} intset;
```

## intset 编码升级

Redis 中 intset 是可以存储三种类型的整数，在小的时候都是用小的编码，当插入新元素比较过 intset 的 encoding 后，发现插入元素类型更大时，就会去进行整个 intset 结构的升级。即将 contents 里面的所有元素，类型都从小类型升级为大类型。

intset 不支持降级。

## 源码

### intsetSearch 查找元素

``` C
/* Search for the position of "value". Return 1 when the value was found and
 * sets "pos" to the position of the value within the intset. Return 0 when
 * the value is not present in the intset and sets "pos" to the position
 * where "value" can be inserted. */
// 搜索指定值 value 在 intset 中的位置
// 找到则返回 1，并设置 pos 值为指定位置下标
// 否则返回 0，并设置 pos 值为可插入位置
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        // 根据 intset 最大值和最小值，快速判断元素是否存在
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    // 二分查找
    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        // 如果找到了，设置 pos 指针，返回 1
        if (pos) *pos = mid;
        return 1;
    } else {
        // 没找到，设置 pos 指针为可插入位置，返回 0
        if (pos) *pos = min;
        return 0;
    }
}
```

### intsetAdd 添加元素

``` C
/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    // 如果 intset 需要进行编码升级，调用 intsetUpgradeAndAdd 方法
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        // 如果查找到该元素，元素已存在，插入失败
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        // 进行插入，对 intset 扩容
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        // 如果 pos == is->length 就不用进行内存移动，直接在数组最后插入
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    // 在 pos 下标插入 value 元素，is->contents[pos] = value，并且维护 is->length 长度
    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

### intsetUpgradeAndAdd 升级并添加元素

``` C
/* Upgrades the intset to a larger encoding and inserts the given integer. */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

### intsetMoveTail 移动元素

intsetMoveTail 方法将 from 对应位置往后的元素，移动到 to 位置。

``` C
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    // is->length - from 算出这个位置往后有多少个元素，即要移动的元素个数
    uint32_t bytes = intrev32ifbe(is->length)-from;
    // 获取当前的编码
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        // 获取 from 的起始地址，即移动范围的初始地址
        src = (int64_t*)is->contents+from;
        // 获取 to 的起始地址，即需要移动到的目的地初始地址
        dst = (int64_t*)is->contents+to;
        // 元素个数 * 元素大小，计算要移动的总字节数
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    // 将 src 位置往后 bytes 个字节，移动到 dst 位置
    memmove(dst,src,bytes);
}
```

### intsetRemove 删除元素

``` C
/* Delete integer from intset */
intset *intsetRemove(intset *is, int64_t value, int *success) {
    // 获取 value 类型
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    // 如果删除整数的类型 > 当前编码类型，不存在
    // 二分查找判断元素是否存在，存在才进行删除，否则直接返回
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        // 如果 pos 插入位置小雨 intset 长度，即在中间删除
        // 调用 intsetMoveTail 函数，将 pos+1 位置开始的元素，移动到 pos 的位置上
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        // 覆盖后，删除了目标元素，对 intset 进行缩容，并更新 intset 长度
        is = intsetResize(is,len-1);
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```

