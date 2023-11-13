# volatile 变量与线程安全


---

## 一个线程不安全的现象

一个数组实现的环形缓冲区，变量 readPos 和 writePos 分别记录下一个读取的索引和下一个写入的索引。当缓冲区为空时，消费者会在数据存入缓冲区前等待。当缓冲区满时，生产者会等待将数据存入缓冲区。

``` Java
public class RingBuffer<Item> {

    private final Item[] buffer;
    private int readPos;
    private int writePos;

    RingBuffer(int capacity) {
        this.buffer = (Item[]) new Object[capacity];
        this.readPos = 0;
        this.writePos = 0;
    }

    public void write(Item item) {
        while (isFull())
            ;
        buffer[writePos] = item;
        writePos = (writePos + 1) % buffer.length;
    }

    public Item read() {
        while (isEmpty())
            ;
        Item item = buffer[readPos];
        readPos = (readPos + 1) % buffer.length;
        return item;
    }

    private boolean isEmpty() {
        return readPos == writePos;
    }

    private boolean isFull() {
        return ((writePos + 1) % buffer.length) == readPos;
    }

    public static void main(String[] args) {
        RingBuffer<Integer> ringBuffer = new RingBuffer<>(10);
        Thread writer1 = new Thread(() -> {
            for (int item = 0; item < Integer.MAX_VALUE; item++) {
                ringBuffer.write(item);
            }
        });
        writer1.start();
        while (true) {
            StdOut.println(ringBuffer.read());
        }
    }
}
```

在运行此测试用例时发现两个线程都容易进入死循环。写入线程一直认为缓冲区是满的，消费线程一直认为缓冲区是空的。经过排查，此现象是 readPos 和 writePos 变量不一致导致的。

在写入线程中，writePos 变量只会被写入线程修改，因此该变量对于写入线程来说始终是最新值。而写入线程调用 isFull 方法的 readPos 变量会被读取线程修改，导致写入线程中 readPos 变量是旧数据。

在读取线程中，readPos 变量只会被读取线程修改，因此该变量对于读取线程来说始终是最新值。而读取线程调用 isEmpty 方法的 writePos 变量会被写入线程修改，导致读取线程中 writePos 变量是旧数据。

---

## 解决方案

将 readPos 和 writePos 改为 volatile 变量，在这个场景中能够保证这两个变量的线程安全。

那么 volatile 变量在此场景中是如何保证线程安全的呢？

---

## volatile 变量机制

### 可见行保证

对于非 volatile 变量，JVM 不会保证线程修改变量会被立即从 CPU 缓存中回写到主内存中。使得另一个线程可能会从主内存读取到该变量的旧值。

对于 volatile 变量，JVM 会保证线程每次都会从主内存中读取该变量。并且对该变量的修改会被立即回写到主内存。此时其余所有线程都会看到该变量的最新值。

### happens-before 保证

happens-before 保证会对指令重排序进行限制。

1. 对 volatile 变量进行写入操作之前的所有指令不会因指令重排序导致这些指令在写入操作的后面；
2. 对 volatile 变量进行读取操作之后的所有指令不会因指令重排序导致这些指令在写入操作的之前。

即本应在 volatile 变量读取与写入操作之间的指令，不会因为指令重排序导致这些指令在变量读取与写入操作之外。

---

## volatile 变量何时是线程安全的？

在以下两个场景，volatile 变量是线程安全的：

1. 当只有一个线程向 volatile 变量写入，其余多个线程仅读取该变量时，总会读取最新的数据，此时是线程安全的；
2. 当多个线程向 volatile 变量写入并且对变量的操作是原子操作（被写入的新值不依赖旧值），此时是线程安全的。

---

## 一个 volitile 变量例子

``` C#
public class Singleton {

    private volatile static Singleton singleton;

    private Singleton (){}

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

---

## 参考资料

[Concurrency in Java: "synchronized" and "volatile" keywords](https://codeburps.com/post/synchronized-and-volatile-keyword)

[Volatile Variables and Thread Safety](https://www.baeldung.com/java-volatile-variables-thread-safety)

---

