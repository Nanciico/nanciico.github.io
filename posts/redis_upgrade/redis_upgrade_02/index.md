# Redis 内存碎片整理调研


---

## 内存碎片产生原因

内存碎片产生有两种原因

### 内存分配原因

> To store user keys, Redis allocates at most as much memory as the maxmemory setting enables (however there are small extra allocations possible).

Redis 在分配内存时有可能会分配少量额外空间。

Redis 封装的 zmalloc 方法会调用 ztrymalloc_usable 方法额外分配 PREFIX_SIZE 大小的空间。

``` C
/* Try allocating memory, and return NULL if failed.
 * '*usable' is set to the usable size if non NULL. */
void *ztrymalloc_usable(size_t size, size_t *usable) {
    /* Possible overflow, return NULL, so that the caller can panic or handle a failed allocation. */
    if (size >= SIZE_MAX/2) return NULL;
    void *ptr = malloc(MALLOC_MIN_SIZE(size)+PREFIX_SIZE);
 
    if (!ptr) return NULL;
#ifdef HAVE_MALLOC_SIZE
    size = zmalloc_size(ptr);
    update_zmalloc_stat_alloc(size);
    if (usable) *usable = size;
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    if (usable) *usable = size;
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

其次在默认内存分配器 Jemalloc 下，会按照固定大小分配内存，比如需要申请 6 bytes，但实际分配 8 bytes。

### 频繁修改 Redis 中数据

> Redis will not always free up (return) memory to the OS when keys are removed. This is not something special about Redis, but it is how most malloc() implementations work. For example if you fill an instance with 5GB worth of data, and then remove the equivalent of 2GB of data, the Resident Set Size (also known as the RSS, which is the number of memory pages consumed by the process) will probably still be around 5GB, even if Redis will claim that the user memory is around 3GB. This happens because the underlying allocator can't easily release the memory. For example often most of the removed keys were allocated in the same pages as the other keys that still exist.

在删除数据时，Redis 不会直接将该部分内存归还操作系统，原因是有可能有其他数据落在相同页上，这部分删除的内存也会产生内存碎片。

---

## 内存碎片率

mem_fragmentation_ratio = used_memory_rss / used_memory

可以理解为 Redis 向操作系统申请的内存与 Redis 实际使用内存的比。

理想情况下内存碎片率维持在 1.03 最好。正常情况下在 1-1.5 之间。

---

## 内存碎片整理功能默认配置

``` conf

activedefrag = 0                        # 内存碎片整理总开关，默认不开启。
 
active-defrag-cycle-min = 1             # 内存碎片整理的 CPU 时间占总 CPU 时间不低于 1%。
active-defrag-cycle-max = 25            # 内存碎片整理的 CPU 时间占总 CPU 时间不高于 25%。
active-defrag-threshold-lower = 10      # 内存碎片率小于 10%，不进行内存碎片整理。
active-defrag-threshold-upper = 100     # 在 100% 碎片率时达到最大的碎片整理力度。
 
active-defrag-max-scan-fields = 1000    # key 中包含的 field 大于 1000 会被单独处理。
 
active-defrag-ignore-bytes = 100mb      # 碎片总空间少于 100mb 不进行内存整理。
```

---

## 内存碎片整理的实现

内存碎片整理功能通过 activeDefragCycle 函数实现，该函数通过 serverCron 函数调用，在开启该功能后会被定时调用。一次完整的内存碎片整理过程需要多次调用 activeDefragCycle 函数，即会横跨多次定时任务。

### 控制占用 CPU 时间比率限制的实现

通过 active-defrag-cycle-min、active-defrag-cycle-max、active-defrag-threshold-lower、active-defrag-threshold-upper 四个配置能够控制内存碎片整理功能占用 CPU 时间的百分比。

该百分比最终会被计算为每次执行 activeDefragCycle 函数的最大时间限制 timeLimit，从而控制每次执行内存碎片整理功能的时间。

``` C
#define INTERPOLATE(x, x1, x2, y1, y2) ( (y1) + ((x)-(x1)) * ((y2)-(y1)) / ((x2)-(x1)) )
#define LIMIT(y, min, max) ((y)<(min)? min: ((y)>(max)? max: (y)))
 
/* decide if defrag is needed, and at what CPU effort to invest in it */
void computeDefragCycles() {
    size_t frag_bytes;
    // 获得内存碎片率和内存碎片的总字节数
    float frag_pct = getAllocatorFragmentation(&frag_bytes);
    /* If we're not already running, and below the threshold, exit. */
    // 如果当前未进行内存碎片整理，且内存碎片率和内存碎片总字节数不满足阈值要求，退出
    if (!server.active_defrag_running) {
        if(frag_pct < server.active_defrag_threshold_lower || frag_bytes < server.active_defrag_ignore_bytes)
            return;
    }
 
    /* Calculate the adaptive aggressiveness of the defrag */
    // 自适应计算内存碎片清理的 cpu 占用百分比
    int cpu_pct = INTERPOLATE(frag_pct,
            server.active_defrag_threshold_lower,
            server.active_defrag_threshold_upper,
            server.active_defrag_cycle_min,
            server.active_defrag_cycle_max);
    // 将 cpu 占用百分比限制在 [active_defrag_cycle_min, active_defrag_cycle_max]
    cpu_pct = LIMIT(cpu_pct,
            server.active_defrag_cycle_min,
            server.active_defrag_cycle_max);
     /* We allow increasing the aggressiveness during a scan, but don't
      * reduce it. */
    if (cpu_pct > server.active_defrag_running) {
        // 记录比率
        server.active_defrag_running = cpu_pct;
        serverLog(LL_VERBOSE,
            "Starting active defrag, frag=%.0f%%, frag_bytes=%zu, cpu=%d%%",
            frag_pct, frag_bytes, cpu_pct);
    }
}
```

计算公式： cycle-min + (frag_pct - threshold_lower) * (cycle_max - cycle_min) / (threshold_upper - threshold_lower)。

若当前内存碎片率为 1.5，则计算出来的 cpu_pct = 14。

则 `timelimit = 1000000 * server.active_defrag_running / server.hz / 100 = 1000000 * 14 / 10 / 100 = 14,000 μs = 14 ms`。

该值计算出来后，会在每 16 次 scan，或每 512 次指针移动，或每 64 个 key 完成内存整理完毕后，判断运行时间是否超过运行时间限制，若超过则退出本次内存整理。

即在内存碎片率为 1.5 时，每次定时任务会花费 14ms 左右时间整理内存碎片。

### 内存碎片整理核心实现

内存整理功能通过 scan 键空间实现。每次 scan 时会调用 defragScanCallback 回调函数，执行 scan 出来的 key 的内存碎片清理工作。

`cursor = dictScan(db->dict, cursor, defragScanCallback, defragDictBucketCallback, db);`

defragScanCallback 调用 defragKey 函数，先尝试整理 key 对象，再判断 value 对象的编码从而调用相关函数整理 value 对象。

最终都会调用 activeDefragAlloc 函数进行内存整理。内存整理的过程为：分配新内存、内存复制、释放旧内存。

``` C
/* Defrag helper for generic allocations.
 *
 * returns NULL in case the allocation wasn't moved.
 * when it returns a non-null value, the old pointer was already released
 * and should NOT be accessed. */
void* activeDefragAlloc(void *ptr) {
    size_t size;
    void *newptr;
    if(!je_get_defrag_hint(ptr)) {
        server.stat_active_defrag_misses++;
        return NULL;
    }
    /* move this allocation to a new allocation.
     * make sure not to use the thread cache. so that we don't get back the same
     * pointers we try to free */
    size = zmalloc_size(ptr);
    // 分配新内存
    newptr = zmalloc_no_tcache(size);
    // 内存复制
    memcpy(newptr, ptr, size);
    // 释放旧内存
    zfree_no_tcache(ptr);
    return newptr;
}
```

这样就完成了单个内存区域的内存整理。

### 内存碎片整理大 Key 是否阻塞

假设有一大 key 其拥有的元素为 fields 个。

在处理到该 key 时，会比较 fields 与 active-defrag-max-scan-fields 的大小，若 fields > active-defrag-max-scan-fields，则将其标记为大 key 放在一个列表里，并跳过该 key 的内存整理。

``` C
if (dictSize(d) > server.active_defrag_max_scan_fields)
    defragLater(db, kde);
else
    defragged += activeDefragSdsDict(d, DEFRAG_SDS_DICT_VAL_IS_SDS);
```

在 scan 下一个桶之前，会检查列表里是否有大 key 未完成内存整理，若有则会单独为大 key 进行内存整理。

``` C
/* before scanning the next bucket, see if we have big keys left from the previous bucket to scan */
if (defragLaterStep(db, endtime)) {
    quit = 1; /* time is up, we didn't finish all the work */
    break; /* this will exit the function and we'll continue on the next cycle */
}
```

在整理大 key 的内存时，也会分多次整理。会在每 16 次迭代，或每 512 个指针移动，或每 64 个 field 内存整理完毕后，判断运行时间是否超过运行时间限制，若超过则退出本次内存整理。

``` C
do {
    int quit = 0;
    if (defragLaterItem(de, &defrag_later_cursor, endtime,db->id))
        quit = 1; /* time is up, we didn't finish all the work */
 
    /* Once in 16 scan iterations, 512 pointer reallocations, or 64 fields
        * (if we have a lot of pointers in one hash bucket, or rehashing),
        * check if we reached the time limit. */
    if (quit || (++iterations > 16 ||
                    server.stat_active_defrag_hits - prev_defragged > 512 ||
                    server.stat_active_defrag_scanned - prev_scanned > 64)) {
        if (quit || ustime() > endtime) {
            if(key_defragged != server.stat_active_defrag_hits)
                server.stat_active_defrag_key_hits++;
            else
                server.stat_active_defrag_key_misses++;
            return 1;
        }
        iterations = 0;
        prev_defragged = server.stat_active_defrag_hits;
        prev_scanned = server.stat_active_defrag_scanned;
    }
} while(defrag_later_cursor);
```

即一个大 key 的内存整理会分多次处理，不会长时间阻塞主线程。

---

## 手动内存整理 Memory Purge

该命令只在使用 jemalloc 内存分配器下生效。

该命令清理内存脏页，和上述内存碎片整理功能管理的不是相同区域。

---

## 结论

Redis 内存碎片整理功能是通过 scan 命令渐进式地整理每次迭代到的 key，每次调用的时间复杂度为 O(1), 完整执行完一次内存碎片整理功能的时间复杂度为 O(n)。

内存碎片整理大 key 会将大 key 分多次处理，不会长时间阻塞主线程。

内存碎片整理功能的核心作用是降低内存碎片率，提高内存利用率以节省内存成本。

但内存整理功能在主线程中执行，会阻塞主线程而降低 Redis 的性能。

---

## 参考资料

[Memory optimization | Redis](https://redis.io/docs/management/optimization/memory-optimization/)

[redis/defrag.c at unstable · redis/redis · GitHub](https://github.com/redis/redis/blob/unstable/src/defrag.c)

[MEMORY PURGE | Redis](https://redis.io/commands/memory-purge/)

---

