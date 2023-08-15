# Redis 简单动态字符串SDS


## SDS 定义

`sds` 的定义以及 `sdshdr` 的定义代码如下：

``` C
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

`sds` 实际就是字符指针 `char *` 类型。

`sds` 通过五种类型的 Header 来满足不同长度字符串的使用需求。除了 `sdshdr5` 类型是为不可变的字符串准备的(比如一个存储 key 的字符串)，其余的 Header 都有 `len`、`alloc` 和 `flags` 元数据域和 `buf` 存储域。

- `len` 保存字符串长度(不包括 C 字符串末尾的 '\0' )；
- `alloc` 保存 `buf` 数组的长度；
- `flags` 占一个字节，前 3 位保存 Header 类型，后三位未使用；
- `buf` 保存具体的字符串。实际长度为 alloc + 1 (最后一位的 '\0')。

``` C
// flags 的定义
#define SDS_TYPE_5  0   // 0000 0000
#define SDS_TYPE_8  1   // 0000 0001
#define SDS_TYPE_16 2   // 0000 0010
#define SDS_TYPE_32 3   // 0000 0011
#define SDS_TYPE_64 4   // 0000 0100
```

### SDS 的内存构造

下图使用一个 `sdshdr16` 类型的 Header 演示 sds 的内存构造。

{{< image src="/images/redis_code/01_01.jpg" caption="SDS 内存构造" title="SDS 内存构造" >}}

### 内存对齐

`__attribute__ ((__packed__))` 令编译器取消内存对齐，从而节省内存，并方便使用 `buf[-1]` 获取 `flags` 域的地址。

下图使用 `sdshdr16` 的 sds，内存对齐系数为 4，演示内存对齐与内存不对齐的区别。

{{< image src="/images/redis_code/01_02.jpg" caption="内存对齐与内存不对齐的区别" title="内存对齐与内存不对齐的区别" >}}

## SDS API

### 创建字符串

`sdsnew` 函数创建一个字符串，返回指向 sds 的指针。

``` C
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 * If SDS_NOINIT is used, the buffer is left uninitialized;
 *
 * The string is always null-terminated (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * mystring = sdsnewlen("abc",3);
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    // sdshdr 指针
    void *sh;
    sds s;
    // 通过字符串长度，决定使用哪种 sdshdr
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    // 计算 sdshdr 的内存大小
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
    // malloc 分配的字节数
    size_t usable;

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    // 分配内存，大小为 Header 长度、字符串长度、'\0' 之和
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        // 将 sh 指针向后的内存区域初始化为 0
        memset(sh, 0, hdrlen+initlen+1);
    // 指向 buf
    s = (char*)sh+hdrlen;
    // 指向 flags
    fp = ((unsigned char*)s)-1;
    // 计算字符数组实际可用长度，使其不会超过类型的上限。
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        // ... SDS_TYPE_16 SDS_TYPE_32 SDS_TYPE_64
    }
    if (initlen && init)
        // 将 init 指向的字符串拷贝到指针 s 中
        memcpy(s, init, initlen);
    // 末尾赋值 '\0'
    s[initlen] = '\0';
    return s;
}

sds sdsnewlen(const void *init, size_t initlen) {
    return _sdsnewlen(init, initlen, 0);
}

sds sdstrynewlen(const void *init, size_t initlen) {
    return _sdsnewlen(init, initlen, 1);
}

/* Create an empty (zero length) sds string. Even in this case the string
 * always has an implicit null term. */
sds sdsempty(void) {
    return sdsnewlen("",0);
}

/* Create a new sds string starting from a null terminated C string. */
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}
```

### 字符串空间预分配

在 sds 可能会有空间内存增长时，会调用 `sdsMakeRoomFor` 方法进行 sds 空间检查和扩容。

为了尽可能避免内存的重新申请/分配，redis 的 buf 数组如果有变动，通常会预分配额外的空间保证下次字符串变化时，如果空间足够，就不需要再次进行内存分配。

常量 `SDS_MAX_PREALLOC` 值为 1024 * 1024。当新申请的 sds 大小小于 1MB 时，会多分配一倍的内存；当新申请的 sds 大小大于等于 1MB 时，函数多分配 1MB 内存。

``` C
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 * If there's already sufficient free space, this function returns without any
 * action, if there isn't sufficient free space, it'll allocate what's missing,
 * and possibly more:
 * When greedy is 1, enlarge more than needed, to avoid need for future reallocs
 * on incremental growth.
 * When greedy is 0, enlarge just enough so that there's free space for 'addlen'.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    // 旧 sdshdr 和新 sdshdr 指针
    void *sh, *newsh;
    // 旧 sds 可用内存 (sh->alloc - sh->len)
    size_t avail = sdsavail(s);
    // 旧 sds 长度，新 sds 长度，请求的长度
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    // 如果可用空间 avail 足够，直接返回 (避免内存的重新分配)
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    reqlen = newlen = (len+addlen);
    assert(newlen > len);   /* Catch size_t overflow */
    if (greedy == 1) {
        // 如果小于 1MB，就扩容两倍 newlen
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        // 否则，只扩展 1MB
        else
            newlen += SDS_MAX_PREALLOC;
    }

    // 根据长度计算 sdshdr 类型
    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    // 新的 sdshdr 大小
    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    if (oldtype==type) {
        // 如果新旧 sdshdr 类型一致, 对 sh 的空间重新分配, 不会改变原来里面的数据
        // https://man.cx/realloc
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        // sdshdr 类型改变，不能使用 realloc
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        // 将旧 sds 内容拷贝到新 sds 中
        memcpy((char*)newsh+hdrlen, s, len+1);
        // 释放旧 sdshdr 空间
        s_free(sh);
        // 旧 sds 指针指向新 sds
        s = (char*)newsh+hdrlen;
        // 赋值 flags
        s[-1] = type;
        // 设置 sh->len
        sdssetlen(s, len);
    }
    // 设置 sh->alloc
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}

/* Enlarge the free space at the end of the sds string more than needed,
 * This is useful to avoid repeated re-allocations when repeatedly appending to the sds. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    return _sdsMakeRoomFor(s, addlen, 1);
}

/* Unlike sdsMakeRoomFor(), this one just grows to the necessary size. */
sds sdsMakeRoomForNonGreedy(sds s, size_t addlen) {
    return _sdsMakeRoomFor(s, addlen, 0);
}
```

### 字符串空间惰性释放(重分配)

在 sds 长度减少，Redis 不会立即回收减少部分的内存空间，在下次需要增加 sds 长度时，可以一定程度上避免内存的分配。

比如在 `sdstrim` 中，不会释放内存空间：

``` C
/* Remove the part of the string from left and from right composed just of
 * contiguous characters found in 'cset', that is a null terminated C string.
 *
 * After the call, the modified sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call.
 *
 * Example:
 *
 * s = sdsnew("AA...AA.a.aa.aHelloWorld     :::");
 * s = sdstrim(s,"Aa. :");
 * printf("%s\n", s);
 *
 * Output will be just "HelloWorld".
 */
sds sdstrim(sds s, const char *cset) {
    char *end, *sp, *ep;
    size_t len;

    sp = s;
    ep = end = s+sdslen(s)-1;
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > sp && strchr(cset, *ep)) ep--;
    len = (ep-sp)+1;
    // sp 长度为 len 的内存拷贝到 s
    if (s != sp) memmove(s, sp, len);
    // 维护末尾的 '\0'
    s[len] = '\0';
    // 维护 sh->len， sh->alloc 没有更改
    sdssetlen(s,len);
    return s;
}
```

调用 `sdsRemoveFreeSpace` 可以主动回收空间。例如客户端输入缓冲区大于 4KB 并且客户端空闲时间超过 2s，会调用 `sdsRemoveFreeSpace` 对 `c->querybuf` 进行进行多余空间的释放。

``` C
/* The client query buffer is an sds.c string that can end with a lot of
 * free space not used, this function reclaims space if needed.
 *
 * The function always returns 0 as it never terminates the client. */
int clientsCronResizeQueryBuffer(client *c) {

    // ...

    /* Only resize the query buffer if the buffer is actually wasting at least a
     * few kbytes */
    if (sdsavail(c->querybuf) > 1024*4) {
        /* There are two conditions to resize the query buffer: */
        if (idletime > 2) {
            /* 1) Query is idle for a long time. */
            c->querybuf = sdsRemoveFreeSpace(c->querybuf, 1);
        }
        // ...
    }

    // ...

    return 0;
}
```

`sdsRemoveFreeSpace` 实际上是调用的 `sdsResize` 方法。

``` C
/* Reallocate the sds string so that it has no free space at the end. The
 * contained string remains not altered, but next concatenation operations
 * will require a reallocation.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdsRemoveFreeSpace(sds s, int would_regrow) {
    return sdsResize(s, sdslen(s), would_regrow);
}

/* Resize the allocation, this can make the allocation bigger or smaller,
 * if the size is smaller than currently used len, the data will be truncated.
 *
 * The when the would_regrow argument is set to 1, it prevents the use of
 * SDS_TYPE_5, which is desired when the sds is likely to be changed again.
 *
 * The sdsAlloc size will be set to the requested size regardless of the actual
 * allocation size, this is done in order to avoid repeated calls to this
 * function when the caller detects that it has excess space. */
sds sdsResize(sds s, size_t size, int would_regrow) {
    // 旧 sdshdr 和 新 sdshdr 指针
    void *sh, *newsh;
    // 新 sdshdr 类型 和 旧 sdshdr 类型
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    // 新 sdshdr 长度 和 新 sdshdr 长度
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    sh = (char*)s-oldhdrlen;

    /* Return ASAP if the size is already good. */
    // 大小无变化，不需要 resize
    if (sdsalloc(s) == size) return s;

    /* Truncate len if needed. */
    if (size < len) len = size;

    /* Check what would be the minimum SDS header that is just good enough to
     * fit this string. */
    type = sdsReqType(size);
    if (would_regrow) {
        /* Don't use type 5, it is not good for strings that are expected to grow back. */
        if (type == SDS_TYPE_5) type = SDS_TYPE_8;
    }
    hdrlen = sdsHdrSize(type);

    /* If the type is the same, or can hold the size in it with low overhead
     * (larger than SDS_TYPE_8), we just realloc(), letting the allocator
     * to do the copy only if really needed. Otherwise if the change is
     * huge, we manually reallocate the string to use the different header
     * type. */
    // 如果 新旧 sdshdr 类型一样，或者旧类型能够容纳新类型，使用 realloc
    int use_realloc = (oldtype==type || (type < oldtype && type > SDS_TYPE_8));
    size_t newlen = use_realloc ? oldhdrlen+size+1 : hdrlen+size+1;
    // jemalloc 优化
    int alloc_already_optimal = 0;
    #if defined(USE_JEMALLOC)
        /* je_nallocx returns the expected allocation size for the newlen.
         * We aim to avoid calling realloc() when using Jemalloc if there is no
         * change in the allocation size, as it incurs a cost even if the
         * allocation size stays the same. */
        alloc_already_optimal = (je_nallocx(newlen, 0) == zmalloc_size(sh));
    #endif

    if (use_realloc && !alloc_already_optimal) {
        // 使用 realloc
        newsh = s_realloc(sh, newlen);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else if (!alloc_already_optimal) {
        // 使用 malloc
        newsh = s_malloc(newlen);
        if (newsh == NULL) return NULL;
        // 复制
        memcpy((char*)newsh+hdrlen, s, len);
        // 释放原 sds
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
    }
    s[len] = 0;
    sdssetlen(s, len);
    sdssetalloc(s, size);
    return s;
}
```

## 字符串编码

Redis 字符串会使用不同的编码格式存储不同类型的数据。若字符串是数字，则使用 int 编码；若设置的字符串大小小于等于 44 个字节，则使用紧凑的编码类型 embstr；若设置的字符串大小大于 44 个字节，则使用 raw 类型进行编码。

### redisObject

在学习 Redis 编码之前，需要先了解 redisObject 结构体。在 Redis 中，所有的 key/value 都是一个 `redisObject` 结构体。该结构体的定义如下。

``` C
#define LRU_BITS 24

struct redisObject {
    // 4 bits 表示类型， string / list / set / zset / hash / module / stream
    unsigned type:4;
    // 4 bits 表示编码，如 string 可以有 raw / embstr / int 等编码方式
    unsigned encoding:4;
    // 24 bits
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    // 4 * 8 = 32 bits 对象的引用计数
    int refcount;
    // 8 bytes (64-bit system)，指向实际对象地址指针
    void *ptr;
};
```

可以算出 redisObject 的大小为 16 字节。4 bits 的 encoding 域保存对象编码类型。

### 字符串紧凑编码 embstr

embstr 类型的介绍：[Introduction of a new string encoding: EMBSTR](https://github.com/redis/redis/commit/894eba07c8484c0f34b09d54a84e69314c37c427 "Introduction of a new string encoding: EMBSTR")

在字符串大小小于等于 44 字节时，使用 embstr 对字符串进行编码。以 44 字节作为分界的原因需要对比一下 embstr 和 raw 在内存上的存储方式。

{{< image src="/images/redis_code/01_03.jpg" caption="embstr 和 raw 在内存上的存储方式" title="embstr 和 raw 在内存上的存储方式" >}}

在 embstr 编码中， robj 和 sds 在内存中是连续的，申请内存时只需要调用一次 malloc 方法，释放内存也只需调用一次 free 方法；raw 格式编码在内存地址中不连续，申请/释放内存时会进行两次 malloc/free 调用。

### 字符串对象的创建

创建字符串对象的代码如下所示：

``` C
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    // 长度小于等于 44 字节，使用 embstr 编码，否则使用 raw 编码
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}

/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    // malloc 一次性申请 robj + sdshdr8 + len + 1 字节的内存
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    // 设置 robj
    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = 0;

    sh->len = len;
    // alloc 设置为 len
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}

/* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
 * string object where o->ptr points to a proper sds string. */
robj *createRawStringObject(const char *ptr, size_t len) {
    // sdsnewlen 中会调用一次 malloc
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}

robj *createObject(int type, void *ptr) {
    // 第二次调用 malloc
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;
    o->lru = 0;
    return o;
}
```

