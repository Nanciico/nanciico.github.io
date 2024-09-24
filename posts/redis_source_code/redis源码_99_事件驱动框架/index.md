# Redis 事件驱动框架


---

## Redis 事件循环的主要功能

1. 接受新的客户端连接
2. 响应客户端连接命令

---

## 事件循环的初始化

Redis 启动时会创建事件循环数据结构。

``` C
// src/server.c
void initServer(void) {
    // ...
    
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    if (server.el == NULL) {
        serverLog(LL_WARNING,
            "Failed creating the event loop. Error message: '%s'",
            strerror(errno));
        exit(1);
    }
    
    // ...
}
```

`aeCreateEventLoop` 初始化并返回 `aeEventLoop` 数据结构。

`aeEventLoop` 数据结构中两个最重要的域：`aeFileEvent *events` 和 `aeFiredEvent *fired`。

``` C
// src/ae.h
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

---

## 事件处理器是如何注册的？

事件循环有两个主要事件需要处理：

1. 新的连接建立并可用
2. 现有连接进入可读状态

事件循环初始化后，会在事件循环中注册一个 `aeFileProc *accept_handler`，该事件处理器会在新的连接建立时执行。

``` C
// src/server.c
/* Create an event handler for accepting new connections in TCP or TLS domain sockets.
 * This works atomically for all socket fds */
int createSocketAcceptHandler(connListener *sfd, aeFileProc *accept_handler) {
    int j;

    for (j = 0; j < sfd->count; j++) {
        if (aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,sfd) == AE_ERR) {
            /* Rollback */
            for (j = j-1; j >= 0; j--) aeDeleteFileEvent(server.el, sfd->fd[j], AE_READABLE);
            return C_ERR;
        }
    }
    return C_OK;
}
```

在连接已经建立后，会为连接设置一个读事件处理器。

``` C
// src/connection.h
/* Register a read handler, to be called when the connection is readable.
 * If NULL, the existing handler is removed.
 */
static inline int connSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    return conn->type->set_read_handler(conn, func);
}
```

---

## 如何触发事件处理器？

Redis 使用操作系统的多路复用 IO 来等待事件。

``` C
// src/ae.h
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

事件循环启动，会一直进行循环并执行已经就绪的事件。

``` C
// src.ae.c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

当事件已就绪，会被放置在 `aeFiredEvent *fired` 域。

``` C
// src/ae.h
/* State of an event based program */
typedef struct aeEventLoop {
    // ...

    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */

    // ...
} aeEventLoop;
```

`aeProcessEvents` 从 `aeFiredEvent *fired` 域中读取事件。

``` C
// src/ae.c
/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * If flags is 0, the function does nothing and returns.
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * if flags has AE_DONT_WAIT set, the function returns ASAP once all
 * the events that can be handled without a wait are processed.
 * if flags has AE_CALL_AFTER_SLEEP set, the aftersleep callback is called.
 * if flags has AE_CALL_BEFORE_SLEEP set, the beforesleep callback is called.
 *
 * The function returns the number of events processed. */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    // ...


    /* Call the multiplexing API, will return only on timeout or when
     * some event fires. */
    numevents = aeApiPoll(eventLoop, tvp);

    // ...

    for (j = 0; j < numevents; j++) {
        int fd = eventLoop->fired[j].fd;
        aeFileEvent *fe = &eventLoop->events[fd];
        int mask = eventLoop->fired[j].mask;
        int fired = 0; /* Number of events fired for current fd. */

        /* Normally we execute the readable event first, and the writable
         * event later. This is useful as sometimes we may be able
         * to serve the reply of a query immediately after processing the
         * query.
         *
         * However if AE_BARRIER is set in the mask, our application is
         * asking us to do the reverse: never fire the writable event
         * after the readable. In such a case, we invert the calls.
         * This is useful when, for instance, we want to do things
         * in the beforeSleep() hook, like fsyncing a file to disk,
         * before replying to a client. */
        int invert = fe->mask & AE_BARRIER;

        /* Note the "fe->mask & mask & ..." code: maybe an already
         * processed event removed an element that fired and we still
         * didn't processed, so we check if the event is still valid.
         *
         * Fire the readable event if the call sequence is not
         * inverted. */
        if (!invert && fe->mask & mask & AE_READABLE) {
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
            fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
        }

        /* Fire the writable event. */
        if (fe->mask & mask & AE_WRITABLE) {
            if (!fired || fe->wfileProc != fe->rfileProc) {
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }

        /* If we have to invert the call, fire the readable event now
         * after the writable one. */
        if (invert) {
            fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            if ((fe->mask & mask & AE_READABLE) &&
                (!fired || fe->wfileProc != fe->rfileProc))
            {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }
    }

    // ...
}
```

`fe->rfileProc` 会调用在 `aeCreateFileEvent` 注册的事件处理器。

``` C
// src/ae.c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    // ...

    aeFileEvent *fe = &eventLoop->events[fd];

    // ...

    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;

    // ...
}
```

事件循环会一直循环执行：

1. 等待事件就绪
2. 执行 `fired` 事件的处理器
3. 重复...

---

