---
title: "Redis 异步删除解决大 Key 过期阻塞问题可行性分析"
date: 2022-10-12
# weight: 1
# aliases: ["/first"]
tags: ["Redis"]
author: "Shuyang"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
summary: "通过分析 Redis 异步删除源码，判断异步删除能否解决大 Key 过期阻塞主线程的问题。"
description: "通过分析 Redis 异步删除源码，判断异步删除能否解决大 Key 过期阻塞主线程的问题。"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## 源码分析

同步删除与异步删除的方法入口分别为 delCommand 方法与 unlinkCommand 方法。

``` C
void delCommand(client *c) {
    delGenericCommand(c,server.lazyfree_lazy_user_del);
}

void unlinkCommand(client *c) {
    delGenericCommand(c,1);
}
```

这两个方法都调用 delGenericCommand 方法，server.lazyfree_lazy_user_del 可通过配置文件配置，配置后可以使 del 命令与 unlink 命令完全相同。

delGenericCommand 方法判断传入的 lazy 参数值决定异步删除或者同步删除。

``` C delGenericCommand
/* This command implements DEL and LAZYDEL. */
void delGenericCommand(client *c, int lazy) {
    int numdel = 0, j;

    for (j = 1; j < c->argc; j++) {
        expireIfNeeded(c->db,c->argv[j],0);
        // 判断传入的 lazy 值选择异步删除或同步删除
        int deleted  = lazy ? dbAsyncDelete(c->db,c->argv[j]) :
                              dbSyncDelete(c->db,c->argv[j]);
        if (deleted) {
            signalModifiedKey(c,c->db,c->argv[j]);
            notifyKeyspaceEvent(NOTIFY_GENERIC,
                "del",c->argv[j],c->db->id);
            server.dirty++;
            numdel++;
        }
    }
    addReplyLongLong(c,numdel);
}
```

同步删除和异步删除都是调用 dbGenericDelete 方法，仅传入的 async 参数不同。

``` C 同步删除与异步删除方法
/* Delete a key, value, and associated expiration entry if any, from the DB */
int dbSyncDelete(redisDb *db, robj *key) {
    return dbGenericDelete(db, key, 0);
}

/* Delete a key, value, and associated expiration entry if any, from the DB. If
 * the value consists of many allocations, it may be freed asynchronously. */
int dbAsyncDelete(redisDb *db, robj *key) {
    return dbGenericDelete(db, key, 1);
}
```

dbGenericDelete 方法首先将 key 在 expires 字典中删除并释放内存，再在 dict 字典中移除该 key，但不释放内存。
通过 async 参数判断是否需要异步释放内存，若需要则会调用 freeObjAsync 方法进行异步释放内存，若不需要异步释放内存，则在 dictFreeUnlinkedEntry 方法中直接释放。
若进入 freeObjAsync 方法但不满足异步释放条件，也会在 dictFreeUnlinkedEntry 方法中直接释放。

``` C dbGenericDelete & dictFreeUnlinkedEntry
/* Helper for sync and async delete. */
static int dbGenericDelete(redisDb *db, robj *key, int async) {
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    // 删除 expires 字典中该 key，但不会删除 SDS 结构，因为该 SDS 在 dict 字典中被共享。
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    // 数据库字典中移除 key，不释放内存。
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        /* Tells the module that the key has been unlinked from the database. */
        moduleNotifyKeyUnlink(key,val,db->id);
        /* We want to try to unblock any client using a blocking XREADGROUP */
        if (val->type == OBJ_STREAM)
            signalKeyAsReady(db,key,val->type);
        if (async) {
            // 异步释放内存
            freeObjAsync(key, val, db->id);
            dictSetVal(db->dict, de, NULL);
        }
        if (server.cluster_enabled) slotToKeyDelEntry(de, db);
        // 释放内存
        dictFreeUnlinkedEntry(db->dict,de);
        return 1;
    } else {
        return 0;
    }
}

/* You need to call this function to really free the entry after a call
 * to dictUnlink(). It's safe to call this function with 'he' = NULL. */
void dictFreeUnlinkedEntry(dict *d, dictEntry *he) {
    if (he == NULL) return;
    dictFreeKey(d, he);
    dictFreeVal(d, he);
    zfree(he);
}
```

重点看 freeObjAsync 方法，先计算该 key 的异步删除阈值，若大于阈值 64，则为该 key 创建异步删除任务。

``` C freeObjAsync
/* If there are enough allocations to free the value object asynchronously, it
 * may be put into a lazy free list instead of being freed synchronously. The
 * lazy free list will be reclaimed in a different bio.c thread. If the value is
 * composed of a few allocations, to free in a lazy way is actually just
 * slower... So under a certain limit we just free the object synchronously. */
#define LAZYFREE_THRESHOLD 64

/* Free an object, if the object is huge enough, free it in async way. */
void freeObjAsync(robj *key, robj *obj, int dbid) {
    // 计算异步删除阈值
    size_t free_effort = lazyfreeGetFreeEffort(key,obj,dbid);
    /* Note that if the object is shared, to reclaim it now it is not
     * possible. This rarely happens, however sometimes the implementation
     * of parts of the Redis core may call incrRefCount() to protect
     * objects, and then call dbDelete(). */
    if (free_effort > LAZYFREE_THRESHOLD && obj->refcount == 1) {
        atomicIncr(lazyfree_objects,1);
        // 任务超过异步删除阈值，创建异步删除任务
        bioCreateLazyFreeJob(lazyfreeFreeObject,1,obj);
    } else {
        decrRefCount(obj);
    }
}
```

bioCreateLazyFreeJob 方法创建任务并调用 bioSubmitJob 方法提交任务到 job 数据结构中。

``` C
void bioCreateLazyFreeJob(lazy_free_fn free_fn, int arg_count, ...) {
    va_list valist;
    /* Allocate memory for the job structure and all required
     * arguments */
    bio_job *job = zmalloc(sizeof(*job) + sizeof(void *) * (arg_count));
    job->free_args.free_fn = free_fn;

    va_start(valist, arg_count);
    for (int i = 0; i < arg_count; i++) {
        job->free_args.free_args[i] = va_arg(valist, void *);
    }
    va_end(valist);
    // 提交任务
    bioSubmitJob(BIO_LAZY_FREE, job);
}

void bioSubmitJob(int type, bio_job *job) {
    // 互斥锁
    pthread_mutex_lock(&bio_mutex[type]);
    // 添加任务至末尾
    listAddNodeTail(bio_jobs[type],job);
    bio_pending[type]++;
    // 发送信号唤醒阻塞线程
    pthread_cond_signal(&bio_newjob_cond[type]);
    pthread_mutex_unlock(&bio_mutex[type]);
}
```

执行后台任务方法 bioProcessBackgroundJobs，详细过程不在讨论的主题中。

``` C
void *bioProcessBackgroundJobs(void *arg) {
    bio_job *job;
    unsigned long type = (unsigned long) arg;
    sigset_t sigset;

    /* Check that the type is within the right interval. */
    if (type >= BIO_NUM_OPS) {
        serverLog(LL_WARNING,
            "Warning: bio thread started with wrong type %lu",type);
        return NULL;
    }

    switch (type) {
    case BIO_CLOSE_FILE:
        redis_set_thread_title("bio_close_file");
        break;
    case BIO_AOF_FSYNC:
        redis_set_thread_title("bio_aof_fsync");
        break;
    case BIO_LAZY_FREE:
        redis_set_thread_title("bio_lazy_free");
        break;
    }

    redisSetCpuAffinity(server.bio_cpulist);

    makeThreadKillable();

    pthread_mutex_lock(&bio_mutex[type]);
    /* Block SIGALRM so we are sure that only the main thread will
     * receive the watchdog signal. */
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGALRM);
    if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        serverLog(LL_WARNING,
            "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

    while(1) {
        listNode *ln;

        /* The loop always starts with the lock hold. */
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        /* Pop the job from the queue. */
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        /* It is now possible to unlock the background system as we know have
         * a stand alone job structure to process.*/
        pthread_mutex_unlock(&bio_mutex[type]);

        /* Process the job accordingly to its type. */
        if (type == BIO_CLOSE_FILE) {
            if (job->fd_args.need_fsync) {
                redis_fsync(job->fd_args.fd);
            }
            close(job->fd_args.fd);
        } else if (type == BIO_AOF_FSYNC) {
            /* The fd may be closed by main thread and reused for another
             * socket, pipe, or file. We just ignore these errno because
             * aof fsync did not really fail. */
            if (redis_fsync(job->fd_args.fd) == -1 &&
                errno != EBADF && errno != EINVAL)
            {
                int last_status;
                atomicGet(server.aof_bio_fsync_status,last_status);
                atomicSet(server.aof_bio_fsync_status,C_ERR);
                atomicSet(server.aof_bio_fsync_errno,errno);
                if (last_status == C_OK) {
                    serverLog(LL_WARNING,
                        "Fail to fsync the AOF file: %s",strerror(errno));
                }
            } else {
                atomicSet(server.aof_bio_fsync_status,C_OK);
            }
        } else if (type == BIO_LAZY_FREE) {
            job->free_args.free_fn(job->free_args.free_args);
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        zfree(job);

        /* Lock again before reiterating the loop, if there are no longer
         * jobs to process we'll block again in pthread_cond_wait(). */
        pthread_mutex_lock(&bio_mutex[type]);
        listDelNode(bio_jobs[type],ln);
        bio_pending[type]--;

        /* Unblock threads blocked on bioWaitStepOfType() if any. */
        pthread_cond_broadcast(&bio_step_cond[type]);
    }
}
```

结论：异步删除能够解决主线程阻塞问题。

## 惰性删除与异步删除

Redis 惰性删除策略是否采用异步删除策略？

在惰性删除中，Redis 在操作 Key 时会先判断该 Key 是否过期，若过期则会删除该 Key。

``` C expireIfNeeded
/* This function is called when we are going to perform some operation
 * in a given key, but such key may be already logically expired even if
 * it still exists in the database. The main way this function is called
 * is via lookupKey*() family of functions.
 *
 * The behavior of the function depends on the replication role of the
 * instance, because by default replicas do not delete expired keys. They
 * wait for DELs from the master for consistency matters. However even
 * replicas will try to have a coherent return value for the function,
 * so that read commands executed in the replica side will be able to
 * behave like if the key is expired even if still present (because the
 * master has yet to propagate the DEL).
 *
 * In masters as a side effect of finding a key which is expired, such
 * key will be evicted from the database. Also this may trigger the
 * propagation of a DEL/UNLINK command in AOF / replication stream.
 *
 * On replicas, this function does not delete expired keys by default, but
 * it still returns 1 if the key is logically expired. To force deletion
 * of logically expired keys even on replicas, use the EXPIRE_FORCE_DELETE_EXPIRED
 * flag. Note though that if the current client is executing
 * replicated commands from the master, keys are never considered expired.
 *
 * On the other hand, if you just want expiration check, but need to avoid
 * the actual key deletion and propagation of the deletion, use the
 * EXPIRE_AVOID_DELETE_EXPIRED flag.
 *
 * The return value of the function is 0 if the key is still valid,
 * otherwise the function returns 1 if the key is expired. */
int expireIfNeeded(redisDb *db, robj *key, int flags) {
    if (!keyIsExpired(db,key)) return 0;

    /* If we are running in the context of a replica, instead of
     * evicting the expired key from the database, we return ASAP:
     * the replica key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys. The
     * exception is when write operations are performed on writable
     * replicas.
     *
     * Still we try to return the right information to the caller,
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time.
     *
     * When replicating commands from the master, keys are never considered
     * expired. */
    if (server.masterhost != NULL) {
        if (server.current_client == server.master) return 0;
        if (!(flags & EXPIRE_FORCE_DELETE_EXPIRED)) return 1;
    }

    /* In some cases we're explicitly instructed to return an indication of a
     * missing key without actually deleting it, even on masters. */
    if (flags & EXPIRE_AVOID_DELETE_EXPIRED)
        return 1;

    /* If clients are paused, we keep the current dataset constant,
     * but return to the client what we believe is the right state. Typically,
     * at the end of the pause we will properly expire the key OR we will
     * have failed over and the new primary will send us the expire. */
    if (checkClientPauseTimeoutAndReturnIfPaused()) return 1;

    /* Delete the key */
    // 删除 key
    deleteExpiredKeyAndPropagate(db,key);
    return 1;
}
```

expireIfNeeded 方法会调用 deleteExpiredKeyAndPropagate 方法删除 key。

删除 key 时会读取 server.lazyfree_lazy_expire 配置决定删除策略。server.lazyfree_lazy_expire 可在配置文件中配置，配置后惰性删除将采用异步删除策略。

``` C deleteExpiredKeyAndPropagate
/* Delete the specified expired key and propagate expire. */
void deleteExpiredKeyAndPropagate(redisDb *db, robj *keyobj) {
    mstime_t expire_latency;
    latencyStartMonitor(expire_latency);
    if (server.lazyfree_lazy_expire)
        // 采用异步删除策略
        dbAsyncDelete(db,keyobj);
    else
        dbSyncDelete(db,keyobj);
    latencyEndMonitor(expire_latency);
    latencyAddSampleIfNeeded("expire-del",expire_latency);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,"expired",keyobj,db->id);
    signalModifiedKey(NULL, db, keyobj);
    propagateDeletion(db,keyobj,server.lazyfree_lazy_expire);
    server.stat_expiredkeys++;
}
```

结论：Redis 惰性删除在配置后可采用异步删除策略。

## 定时删除与异步删除

定时任务 serverCron 方法最终会调用 activeExpireCycleTryExpire 方法，该方法仍会调用 deleteExpiredKeyAndPropagate 方法。

``` C activeExpireCycleTryExpire
/* Helper function for the activeExpireCycle() function.
 * This function will try to expire the key that is stored in the hash table
 * entry 'de' of the 'expires' hash table of a Redis database.
 *
 * If the key is found to be expired, it is removed from the database and
 * 1 is returned. Otherwise no operation is performed and 0 is returned.
 *
 * When a key is expired, server.stat_expiredkeys is incremented.
 *
 * The parameter 'now' is the current time in milliseconds as is passed
 * to the function to avoid too many gettimeofday() syscalls. */
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now) {
    long long t = dictGetSignedIntegerVal(de);
    if (now > t) {
        sds key = dictGetKey(de);
        robj *keyobj = createStringObject(key,sdslen(key));
        // 删除 key
        deleteExpiredKeyAndPropagate(db,keyobj);
        decrRefCount(keyobj);
        return 1;
    } else {
        return 0;
    }
}
```

结论：Redis 定时删除在配置后可采用异步删除策略。

## 结论

异步删除策略能够在删除大 Key 时避免主线程阻塞，惰性删除与定时删除在配置后均可采用异步删除策略，因此异步删除能够解决大 Key 过期引起的主线程阻塞问题。
