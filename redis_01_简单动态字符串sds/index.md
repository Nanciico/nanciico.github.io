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

### sds 的内存构造

下图使用一个 `sdshdr16` 类型的 Header 演示 sds 的内存构造。

{{< image src="/images/redis_code/01_01.jpg" caption="SDS 内存构造" title="SDS 内存构造" >}}

### 内存对齐

`__attribute__ ((__packed__))` 令编译器取消内存对齐，从而节省内存，并方便使用 `buf[-1]` 获取 `flags` 域的地址。

下图使用 `sdshdr16` 的 sds，内存对齐系数为 4，演示内存对齐与内存不对齐的区别。

{{< image src="/images/redis_code/01_02.jpg" caption="内存对齐与内存不对齐的区别" title="内存对齐与内存不对齐的区别" >}}

## SDS API

### 创建字符串

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


