# 「算法第四版」归并排序


---

## 自顶向下的归并排序

``` Java
public class Merge {

    public static void sort(Comparable[] a) {
        Comparable[] aux = new Comparable[a.length];
        sort(a, aux, 0, a.length - 1);
    }

    // mergesort a[lo..hi] using auxiliary array aux[lo..hi]
    private static void sort(Comparable[] a, Comparable[] aux, int lo, int hi) {
        if (hi <= lo) return;
        int mid = lo + (hi - lo) / 2;
        sort(a, aux, lo, mid);
        sort(a, aux, mid + 1, hi);
        merge(a, aux, lo, mid, hi);
    }

    // stably merge a[lo .. mid] with a[mid+1 ..hi] using aux[lo .. hi]
    private static void merge(Comparable[] a, Comparable[] aux, int lo, int mid, int hi) {
        // copy to aux[]
        for (int k = lo; k <= hi; k++) {
            aux[k] = a[k];
        }

        // merge back to a[]
        int i = lo, j = mid + 1;
        for (int k = lo; k <= hi; k++) {
            if (i > mid) a[k] = aux[j++];
            else if (j > hi) a[k] = aux[i++];
            else if (less(aux[j], aux[i])) a[k] = aux[j++]; // to ensure stability
            else a[k] = aux[i++];
        }
    }
}
```

### 自顶向下归并排序索引

``` Java
public class Merge {

    // Returns a permutation that gives the elements in the array in ascending order.
    public static int[] indexSort(Comparable[] a) {
        int n = a.length;
        int[] index = new int[n];
        for (int i = 0; i < n; i++)
            index[i] = i;

        int[] aux = new int[n];
        sort(a, index, aux, 0, n - 1);
        return index;
    }

    // mergesort a[lo..hi] using auxiliary array aux[lo..hi]
    private static void sort(Comparable[] a, int[] index, int[] aux, int lo, int hi) {
        if (hi <= lo) return;
        int mid = lo + (hi - lo) / 2;
        sort(a, index, aux, lo, mid);
        sort(a, index, aux, mid + 1, hi);
        merge(a, index, aux, lo, mid, hi);
    }

    // stably merge a[lo .. mid] with a[mid+1 .. hi] using aux[lo .. hi]
    private static void merge(Comparable[] a, int[] index, int[] aux, int lo, int mid, int hi) {

        // copy to aux[]
        for (int k = lo; k <= hi; k++) {
            aux[k] = index[k];
        }

        // merge back to a[]
        int i = lo, j = mid + 1;
        for (int k = lo; k <= hi; k++) {
            if (i > mid) index[k] = aux[j++];
            else if (j > hi) index[k] = aux[i++];
            else if (less(a[aux[j]], a[aux[i]])) index[k] = aux[j++];
            else index[k] = aux[i++];
        }
    }
}
```

### 自顶向下归并排序的优化

1. 对小规模子数组使用插入排序。使用插入排序处理小规模的子数组（比如长度小于 15），一般可以将归并排序的运行时间缩短 10% ~ 15%；
2. 测试数组是否已经有序。添加判断条件若 `a[mid] <= a[mid + 1]`，则此时数组已经是有序的，可以跳过 merge 方法；
3. 不将元素复制到辅助数组。节省将数组复制到用于归并的辅助数组所用的时间。需要调用两种排序方法，一种将数据从输入数组排序到辅助数组，一种将数组从辅助数组排序到输入数组。在递归调用的每个层次交换输入数组和辅助数组的角色。

``` Java
public class Merge {

    private static final int CUTOFF = 7;  // cutoff to insertion sort

    public static void sort(Comparable[] a) {
        Comparable[] aux = a.clone();
        sort(aux, a, 0, a.length - 1);
    }

    private static void sort(Comparable[] src, Comparable[] dst, int lo, int hi) {
        // 1. 对小规模子数组使用插入排序
        if (hi <= lo + CUTOFF) {
            insertionSort(dst, lo, hi);
            return;
        }

        int mid = lo + (hi - lo) / 2;
        sort(dst, src, lo, mid);        // 递归调用的每个层次交换输入数组和辅助数组的角色
        sort(dst, src, mid + 1, hi);    // sort 方法每次递归的结果为：传入的参数 dst 有序

        // 2. 测试数组是否有序以跳过 merge 方法
        // using System.arraycopy() is a bit faster than loop
        if (!less(src[mid + 1], src[mid])) {
            System.arraycopy(src, lo, dst, lo, hi - lo + 1);
            return;
        }

        merge(src, dst, lo, mid, hi);
    }

    private static void merge(Comparable[] src, Comparable[] dst, int lo, int mid, int hi) {
        // 3. 不将元素复制到辅助数组
        int i = lo, j = mid + 1;
        for (int k = lo; k <= hi; k++) {
            if (i > mid) dst[k] = src[j++];
            else if (j > hi) dst[k] = src[i++];
            else if (less(src[j], src[i])) dst[k] = src[j++];   // to ensure stability
            else dst[k] = src[i++];
        }
    }

    // sort from a[lo] to a[hi] using insertion sort
    private static void insertionSort(Comparable[] a, int lo, int hi) {
        for (int i = lo; i <= hi; i++)
            for (int j = i; j > lo && less(a[j], a[j - 1]); j--)
                exch(a, j, j - 1);
    }
}
```

---

## 自底向上的归并排序

### 自底向上归并排序数组

``` Java
public class Merge {

    public static void sort(Comparable[] a) {
        int n = a.length;
        Comparable[] aux = new Comparable[n];
        for (int len = 1; len < n; len *= 2) {
            for (int lo = 0; lo < n - len; lo += len + len) {
                int mid = lo + len - 1;
                int hi = Math.min(lo + len + len - 1, n - 1);
                merge(a, aux, lo, mid, hi);
            }
        }
    }

    private static void merge(Comparable[] a, Comparable[] aux, int lo, int mid, int hi) {

        // copy to aux[]
        for (int k = lo; k <= hi; k++) {
            aux[k] = a[k];
        }

        // merge back to a[]
        int i = lo, j = mid + 1;
        for (int k = lo; k <= hi; k++) {
            if (i > mid) a[k] = aux[j++];  // this copying is unnecessary
            else if (j > hi) a[k] = aux[i++];
            else if (less(aux[j], aux[i])) a[k] = aux[j++];
            else a[k] = aux[i++];
        }
    }
}
```

### 自底向上归并排序链表

``` Java
class LinkedListNatureMerge {

    public static void sort(LinkedList<Comparable> a) {
        if (a == null || a.size() < 2) return;

        LinkedList.Node sentinel = a.new Node(null, a.first);
        while (true) {
            LinkedList.Node prev = sentinel;
            LinkedList.Node lo = prev.next;
            LinkedList.Node mid = findAscIndex(lo, a.last);
            if (mid == a.last) break;

            while (mid != a.last) {
                LinkedList.Node hi = findAscIndex(mid.next, a.last);

                prev = merge(prev, lo, mid, hi);
                if (a.last.next != null) a.last = prev;     // 维护链表 last 指针

                lo = prev.next;
                mid = findAscIndex(lo, a.last);
            }
        }
        a.first = sentinel.next;        // 维护链表 first 指针
    }

    private static LinkedList.Node merge(LinkedList.Node prev, LinkedList.Node lo, LinkedList.Node mid, LinkedList.Node hi) {
        LinkedList.Node next = hi.next;

        LinkedList.Node p1 = lo;
        LinkedList.Node p2 = mid.next;

        mid.next = null;
        hi.next = null;

        while (p1 != null && p2 != null) {
            if (less(p2.item, p1.item)) {
                prev.next = p2;
                p2 = p2.next;
            } else {
                prev.next = p1;
                p1 = p1.next;
            }
            prev = prev.next;
        }

        if (p1 == null) {
            prev.next = p2;
        } else {
            prev.next = p1;
        }

        while (prev.next != null) {
            prev = prev.next;
        }
        prev.next = next;
        return prev;
    }

    private static LinkedList.Node findAscIndex(LinkedList.Node node, LinkedList.Node last) {
        if (node == null) return last;
        while (node.next != null) {
            if (less(node.next.item, node.item)) return node;
            node = node.next;
        }
        return last;
    }

    public static boolean isSorted(LinkedList<Comparable> a) {
        LinkedList.Node pointer = a.first;
        while (pointer.next != null) {
            if (less(pointer.next.item, pointer.item)) return false;
            pointer = pointer.next;
        }
        return true;
    }

    private static boolean less(Comparable v, Comparable w) {
        return v.compareTo(w) < 0;
    }

    static class LinkedList<Item extends Comparable> {
        Node first;
        Node last;
        private int size;

        LinkedList() {
            first = last = null;
            size = 0;
        }

        public void insert(Item item) {
            if (first == null) {
                first = last = new Node(item, null);
            } else {
                first = new Node(item, first);
            }
            size++;
        }

        public Item remove() {
            if (first == null) return null;
            Node temp = first;
            if (first == last) {
                first = last = null;
            } else {
                first = first.next;
            }
            size--;
            return temp.item;
        }

        public int size() {
            return size;
        }

        class Node {
            Item item;
            Node next;

            Node(Item item, Node next) {
                this.item = item;
                this.next = next;
            }
        }
    }
}
```

---

## 归并排序的特点

归并排序是***分治思想***的典型应用。

归并排序的算法复杂度：

1. 对于长度为 N 的任意数组，自底向上的归并排序需要 1/2NlgN 至 NlgN次比较，最多访问数组 6NlgN 次；
2. 没有任何基于比较的算法能够保证使用少于 lg(N!) ~ NlgN次比较将长度为 N 的数组排序；
3. 归并排序是一种渐进最优的基于比较排序的算法，归并排序在最坏情况下的比较次数和任意基于比较的排序算法所需的最少比较次数都是 ~NlgN。

---

