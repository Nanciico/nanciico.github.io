# 「算法第四版」背包、队列和栈


---

## 背包

背包 API：

``` Text
背包
public class Bag<Item> implements Iterable<Item>

             Bag()                  创建一个背包
    void     add(Item item)         添加一个元素
 boolean     isEmpty()              背包是否为空
     int     size()                 背包中元素数量
```

背包是一种不支持从中删除元素的集合数据类型——他的目的是帮助用例收集元素并迭代遍历所有收集到的元素。

### 背包的链表实现

``` Java
import java.util.Iterator;

public class Bag<Item> implements Iterable<Item> {

    private Node first;

    private class Node {
        Item item;
        Node next;
    }

    public void add(Item item) {
        Node oldFirst = first;
        first = new Node();
        first.item = item;
        first.next = oldFirst;
    }

    @Override
    public Iterator<Item> iterator() {
        return new ListIterator();
    }

    private class ListIterator implements Iterator<Item> {

        private Node current = first;

        @Override
        public boolean hasNext() {
            return current != null;
        }

        @Override
        public Item next() {
            Item item = current.item;
            current = current.next;
            return item;
        }
    }
}
```

---

## 先进先出（FIFO）队列

队列 API：

``` Text
先进先出（FIFO）队列
public class Queue<Item> implements Interable<Item>

             Queue()                创建空队列
    void     enqueue(Item item)     添加一个元素
    Item     dequeue()              删除最早添加的元素
 boolean     isEmpty()              队列是否为空
     int     size()                 队列中的元素数量
```

先进先出队列是一种基于先进先出（FIFO）策略的集合类型。

### 队列的实现

#### 队列的链表实现

``` Java
import java.util.Iterator;

public class Queue<Item> implements Iterable<Item> {

    private Node first;
    private Node last;
    private int size;

    private class Node {
        Item item;
        Node next;
    }

    public boolean isEmpty() {
        return first == null;
    }

    public int size() {
        return size;
    }

    public void enqueue(Item item) {
        Node oldLast = last;
        last = new Node();
        last.item = item;
        last.next = null;
        if (isEmpty()) {
            first = last;
        } else {
            oldLast.next = last;
        }
        size++;
    }

    public Item dequeue() {
        Item item = first.item;
        first = first.next;
        if (isEmpty()) {
            last = null;
        }
        size--;
        return item;
    }

    @Override
    public Iterator<Item> iterator() {
        return new ListIterator();
    }

    private class ListIterator implements Iterator<Item> {

        private Node current = first;

        @Override
        public boolean hasNext() {
            return current != null;
        }

        @Override
        public Item next() {
            Item item = current.item;
            current = current.next;
            return item;
        }
    }
}
```

---

## 后进先出（LIFO）栈

栈 API：

``` Text
public class Stack<Item> implements Iterable<Item>

             Stack()                创建一个空栈
    void     push()                 添加一个元素
    Item     pop()                  删除最近添加的元素
 boolean     isEmpty()              栈是否为空
     int     size()                 栈中元素数量
```

### 栈的实现

#### 栈的数组实现

``` Java
import java.util.Iterator;

public class ResizingArrayStack<Item> implements Iterable<Item> {

    private Item[] array = (Item[]) new Object[1];
    private int size = 0;

    public boolean isEmpty() {
        return size == 0;
    }

    public int size() {
        return size;
    }

    public void push(Item item) {
        if (size == array.length) {
            resize(2 * array.length);
        }
        array[size++] = item;
    }

    public Item pop() {
        Item item = array[--size];
        array[size] = null;
        if (size > 0 && size == array.length / 4) {
            resize(array.length / 2);
        }
        return item;
    }

    private void resize(int length) {
        Item[] temp = (Item[]) new Object[length];
        for (int i = 0; i < size; i++) {
            temp[i] = array[i];
        }
        array = temp;
    }

    @Override
    public Iterator<Item> iterator() {
        return new ReverseArrayIterator();
    }

    private class ReverseArrayIterator implements Iterator<Item> {

        private int index = size;

        @Override
        public boolean hasNext() {
            return index > 0;
        }

        @Override
        public Item next() {
            return array[--index];
        }
    }
}
```

#### 栈的链表实现

``` Java
import java.util.Iterator;

public class Stack<Item> implements Iterable<Item> {

    private Node first;
    private int size;

    private class Node {
        Item item;
        Node next;
    }

    public boolean isEmpty() {
        return first == null;
    }

    public int size() {
        return size;
    }

    public void push(Item item) {
        Node oldFirst = first;
        first = new Node();
        first.item = item;
        first.next = oldFirst;
        size++;
    }

    public Item pop() {
        Item item = first.item;
        first = first.next;
        size--;
        return item;
    }

    @Override
    public Iterator<Item> iterator() {
        return new ListIterator();
    }

    private class ListIterator implements Iterator<Item> {

        private Node current = first;

        @Override
        public boolean hasNext() {
            return current != null;
        }

        @Override
        public Item next() {
            Item item = current.item;
            current = current.next;
            return item;
        }
    }
}
```

### 栈的应用

Dijkstra 双栈算术表达式求值算法

``` Java
public class Evaluate {

    public static void main(String[] args) {
        Stack<String> ops = new Stack<>();
        Stack<Double> vals = new Stack<>();

        while (!StdIn.isEmpty()) {
            String s = StdIn.readString();
            switch (s) {
                case "(" -> {
                    ;
                }
                case "+", "-", "*", "/", "sqrt" -> ops.push(s);
                case ")" -> {
                    String op = ops.pop();
                    double v = vals.pop();
                    v = switch (op) {
                        case "+" -> vals.pop() + v;
                        case "-" -> vals.pop() - v;
                        case "*" -> vals.pop() * v;
                        case "/" -> vals.pop() / v;
                        case "sqrt" -> Math.sqrt(v);
                        default -> v;
                    };
                    vals.push(v);
                }
                default -> vals.push(Double.parseDouble(s));
            }
        }
        StdOut.println(vals.pop());
    }
}
```

---

