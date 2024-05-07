# Redis 对象 (redisObject)


---

## redisObject 结构体

Redis 使用对象来表示数据库中的键和值，每当我们在Redis数据库中新创建一个键值对时，至少会创建两个对象，分别是键值对中的键对象和值对象。键对象总是一个字符串对象，而值对象，则根据使用而不同。而每个对象，在 Redis 中，都是由一个 redisObject（位于 server.h）的结构体来表示，也简称为 robj。

``` C
struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
};
```

一个 robj 对象包含 5 个字段：

- type：对象的数据类型，占 4 位，分别对应 Redis 对外暴露的数据类型：字符串、列表、集合、有序集合、哈希、模块、流

- encoding：对象内部编码，占 4 位，表示对象的具体底层实现的数据结构 encoding，一个 type 可能会对应多种实现的 encoding，存在不同的内部表达方式，例如字符串对象可以有 embstr / int / raw 三种编码方式

- lru：用于做 LRF / LFU 缓存淘汰的，占 24 位

- refcount：对象的引用计数，占 4 字节，跟大部分编程语言里的引用计数概念是一致的，它允许一些对象是可以共享的。例如一些小整数就可以提前申明共用作为缓存池，而不用重复创建对象

- ptr：指向底层实现数据结构的指针，占 8 字节，例如一个代表 list 的 robj，它的 ptr 可能指向一个 quicklist

---

### 创建对象 createObject

初始化创建一个 robj，设置一些默认值。encoding 一般在调用方会再次修改这个属性为对应的编码值。

``` C
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;
    o->lru = 0;
    return o;
}
```

### 设置 LRU 或者 LFU 策略

``` C
void initObjectLRUOrLFU(robj *o) {
    // 如果是共享对象，不设置策略
    if (o->refcount == OBJ_SHARED_REFCOUNT)
        return;
    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    // 根据 LRU 或者 LFU 策略设置 o->lru 属性
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes() << 8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return;
}
```

### 共享对象 makeObjectShared

``` C
/* Set a special refcount in the object to make it "shared":
 * incrRefCount and decrRefCount() will test for this special refcount
 * and will not touch the object. This way it is free to access shared
 * objects such as small integers from different threads without any
 * mutex.
 *
 * A common pattern to create shared objects:
 *
 * robj *myobject = makeObjectShared(createObject(...));
 *
 */
robj *makeObjectShared(robj *o) {
    serverAssert(o->refcount == 1);
    o->refcount = OBJ_SHARED_REFCOUNT;
    return o;
}
```

---

## 类型和编码

### 类型

``` C
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

/* The "module" object type is a special one that signals that the object
 * is one directly managed by a Redis module. In this case the value points
 * to a moduleValue struct, which contains the object value (which is only
 * handled by the module itself) and the RedisModuleType struct which lists
 * function pointers in order to serialize, deserialize, AOF-rewrite and
 * free the object.
 *
 * Inside the RDB file, module types are encoded as OBJ_MODULE followed
 * by a 64 bit module type ID, which has a 54 bits module-specific signature
 * in order to dispatch the loading to the right module, plus a 10 bits
 * encoding version. */
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */
#define OBJ_TYPE_MAX 7  /* Maximum number of object types */
```

### 编码

对象的 ptr 指针，指向了对象的底层实现数据结构，这些数据结构则由对象的 encoding 属性决定。encoding 数据记录了对象所使用的具体编码，或者说这个对象使用了什么数据结构作为底层实现。encoding 的值可以是下面列出的常量中的一个。

``` C
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
```

---

## 字符串对象

字符串对象，编码可以是 embstr、int 和 raw 这三种中的一种。

### 编码过程

针对一些字符串对象，Redis 在存储起来之前，会先经历一个编码过程，尝试将它编码转换为更节省内存的 encoding：

- 检查 type，确保只有 string 类型可以进来
- 检查 encoding，只会对 raw 和 embstr 进行操作
- 检查 refcount，因为如果引用计数器大于 1 的共享对象，在多处被引用，如果贸然进行编码，可能会改变 robj 对象指针，这对其他引用来说，是不安全的（要修改所有引用），所以不会对共享过的字符串对象进行编码
- 为了节省内存尝试进行各种 encoding 的转换
  - int：试图将字符串转 long 类型，如果能转返回 1，并且将转换后的值存储在 value 里。这里判断参与转换的字符串的长度是要 <= 20，因为一个 long 能表示的范围是 -9223372036854775808 到 9223372036854775807，包含负号的话十进制表示需要 20 个字符
  - int 共享整数：对于 int 编码如果是小整数，例如现在默认 [0, 10000) 范围里如果能使用共享整数就会使用缓存池里的共享对象，节省这些对象的创建和销毁。不过只在 maxmemory != 0 并且没有使用 LRU / LFU 内存淘汰策略下才会使用共享整数，因为 LRU / LFU 需要记录对象的最后一次访问时间，这样就不能使用共享对象；而 maxmemory = 0 说明不限制内存使用，自然也无内存淘汰策略啥事了
  - embstr：例如携带非数字字符，不能使用 int 编码，如果字符串长度 <= 44 就使用 embstr 编码字符串，前面有介绍过的
  - raw：最后没法子只能用 raw 编码，并且尝试回收 sds 空闲空间

``` C
/* Try to encode a string object in order to save space */
robj *tryObjectEncodingEx(robj *o, int try_trim) {
    long value;
    sds s = o->ptr;
    size_t len;

    /* Make sure this is a string object, the only type we encode
     * in this function. Other types use encoded memory efficient
     * representations but are handled by the commands implementing
     * the type. */
    // 确保对象类型是字符串类型
    serverAssertWithInfo(NULL,o,o->type == OBJ_STRING);

    /* We try some specialized encoding only for objects that are
     * RAW or EMBSTR encoded, in other words objects that are still
     * in represented by an actually array of chars. */
    // 检查判断 encoding 是否为 RAW 或者 EMBSTR，int 编码的字符串不做处理
    if (!sdsEncodedObject(o)) return o;

    /* It's not safe to encode shared objects: shared objects can be shared
     * everywhere in the "object space" of Redis and may end in places where
     * they are not handled. We handle them only as values in the keyspace. */
    // 共享对象，不做处理
     if (o->refcount > 1) return o;

    /* Check if we can represent this string as a long integer.
     * Note that we are sure that a string larger than 20 chars is not
     * representable as a 32 nor 64 bit integer. */
    len = sdslen(s);
    // 尝试将字符串转换为 long 类型，如果能够转换返回 1，并将转换的值存储在 value 里
    if (len <= 20 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        // 如果没开启 maxmemory，或者缓存淘汰策略没有使用 LRU/LFU
        // 且 value 值在 [0, OBJ_SHARED_INTEGERS) 之间
        // 使用 Redis 的小整数缓存，节约内存空间
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o);
            return shared.integers[value];
        } else {
            // 无法使用共享对象
            if (o->encoding == OBJ_ENCODING_RAW) {
                // 编码为 RAW，释放原本的 sds，赋值 ptr，转换编码为 int
                sdsfree(o->ptr);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*) value;
                return o;
            } else if (o->encoding == OBJ_ENCODING_EMBSTR) {
                // 编码为 EMBSTR，减少 o 的引用计数
                // 因为 EMBSTR 编码 robj 和 sds 内存是连续的
                // RAW 编码格式可以直接释放 sds 赋值 ptr
                // EMBSTR 只能调用方法重新根据整数创建字符串对象 robj
                decrRefCount(o);
                return createStringObjectFromLongLongForValue(value);
            }
        }
    }

    /* If the string is small and is still RAW encoded,
     * try the EMBSTR encoding which is more efficient.
     * In this representation the object and the SDS string are allocated
     * in the same chunk of memory to save space and cache misses. */
    // 如果 len <= 44 且 encoding 为 RAW，将它转为 EMBSTR
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        // 创建 EMBSTR 字符串对象，释放原本的 o 对象
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }

    /* We can't encode the object...
     * Do the last try, and at least optimize the SDS string inside */
    if (try_trim)
        // 尝试缩减 sds 里的空闲空间来节约内存
        trimStringObjectIfNeeded(o, 0);

    /* Return the original object. */
    return o;
}
```

### createEmbeddedStringObject

如果字符串长度 <= 44，会调用这个方法，用于生成更节省空间的字符串对象。

``` C

```

