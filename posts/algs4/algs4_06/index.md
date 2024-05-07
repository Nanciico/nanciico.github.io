# 算法第四版 —— 初级排序算法


---

## 排序算法类的模版

``` Java
public class Sort {

    public static void sort(Comparable[] a) {
        // 排序算法具体实现
    }

    private static boolean less(Comparable v, Comparable w) {
        return v.compareTo(w) < 0;
    }

    private static void exch(Comparable[] a, int i, int j) {
        Comparable temp = a[i]; a[i] = a[j]; a[j] = temp;
    }

    private static void show(Comparable[] a) {
        for (int i = 0; i < a.length; i++)
            StdOut.print(a[i] + " ");
        StdOut.println();
    }

    private static boolean isSorted(Comparable[] a) {
        for (int i = 1; i < a.length; i++) {
            if (less(a[i], a[i - 1])) return false;
        }
        return true;
    }
}
```

---

## 选择排序

### 选择排序算法描述

首先，找到数组中最小的元素，其次，将它和数组的第一个元素交换位置。再次，在剩下的元素中找到最小的元素，将它与数组的第二个元素交换位置。直到将整个数组排序。

### 选择排序的实现

``` Java
public class Selection {

    public static void sort(Comparable[] a) {
        int n = a.length;
        for (int i = 0; i < n; i++) {
            int min = i;
            for (int j = i + 1; j < n; j++) {
                if (less(a[j], a[min])) min = j;
            }
            exch(a, i, min);
            assert isSorted(a, 0, i);
        }
        assert isSorted(a);
    }
}
```

算法将第 i 小的元素放到 a[i] 中。数组的第 i 个位置的左边是 i 个最小的元素且它们不会再被访问。

### 选择排序的特点

对于长度为 N 的数组，选择排序需要大约 (N^2) / 2 次比较和 N 次交换。

1. 运行时间和输入无关。
2. 数据移动是最少的。数据移动与数组长度线性相关。

---

## 插入排序

### 插入排序算法描述

初始状态，把第一个元素看做只有一个元素的有序序列，从第二个元素开始及其之后的元素是未排序序列。对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。在从后向前扫描过程中，需要反复把已排序元素逐步向后挪位，为最新元素提供插入空间。

### 插入排序的实现

``` Java
public class Insertion {

    public static void sort(Comparable[] a) {
        int n = a.length;
        for (int i = 1; i < n; i++) {
            for (int j = i; j > 0 && less(a[j], a[j - 1]); j--) {
                exch(a, j, j - 1);
            }
            assert isSorted(a, 0, i);
        }
        assert isSorted(a);
    }

    public static void sort(Comparable[] a, int lo, int hi) {
        for (int i = lo + 1; i < hi; i++) {
            for (int j = i; j > lo && less(a[j], a[j - 1]); j--) {
                exch(a, j, j - 1);
            }
        }
        assert isSorted(a, lo, hi);
    }
}
```

对于 1 到 N - 1 之间的每个 i，将 a[i] 与 a[0] 到 a[i - 1] 中比它小的所有元素依次有序地交换。在索引 i 由左向右变化的过程中，它左侧的元素总是有序的，当 i 到达数组右端时完成排序。

在内循环中将较大的元素都向右移动而不总是交换两个元素，可以大幅提高插入排序的速度。

``` Java
public class InsertionSortWithoutExchange {

    public static void sort(Comparable[] a) {
        int n = a.length;
        for (int i = 1; i < n; i++) {
            Comparable value = a[i];
            int j;
            for (j = i; j > 0 && less(value, a[j - 1]); j--) {
                a[j] = a[j - 1];
            }
            a[j] = value;
        }
        assert isSorted(a, 0, n - 1);
    }
}
```

先找出最小的元素并将其置于数组的最左边，这样能够规避判断边界条件 `j > 0` 从而提高插入排序速度，这个元素被称为哨兵。

``` Java
public class InsertionWithSentinel {

    public static void sort(Comparable[] a) {
        int min = 0;
        for (int i = 1; i < a.length; i++)
            if (less(a[i], a[min])) min = i;
        exch(a, 0, min);

        for (int i = 2; i < a.length; i++) {
            for (int j = i; less(j, j - 1); j--) {
                exch(a, j, j - 1);
            }
        }
    }
}
```

### 插入排序的特点

对于随机排列的长度为 N 且逐渐不重复的数组，平均情况下插入排序需要 ~(N^2)/4 次比较与交换。最坏情况下需要 ~(N^2)/2 次比较与交换。最好情况下需要 N - 1 次比较和 0 次交换。

插入排序所需时间取决于元素的初始顺序。因此插入排序适合实际应用中部分有序的非随机数组。

插入排序的比较和交换的次数相等。

---

## 希尔排序

### 希尔排序算法描述

希尔排序是更高效的插入排序。

对于大规模乱序数组插入排序很慢，因为它只会交换相邻的元素，因此元素只能一点一点地从数维的一端移动到另一端。

希尔排序使数组中任意间隔为 h 的元素都是有序的。这样的数组被称为 h 有序数组。在进行排序时，如果 h 很大，我们就能将元素移动到很远的地方，为实现更小的 h 有序创造方便。对于任意以 1 结尾的 h 序列，我们都能够将数组排序。这就是希尔排序。

希尔排序比插入排序更高效的原因：排序之初，各个子数组都很短，排序之后子数组都是部分有序的，这两种情况都很适合插入排序。

### 希尔排序的实现

递增序列为 (3^k - 1) / 2 的希尔排序。

``` Java
public class Shell {

    public static void sort(Comparable[] a) {
        int n = a.length;

        // 3x+1 increment sequence:  1, 4, 13, 40, 121, 364, 1093, ...
        int h = 1;
        while (h < n / 3) h = 3 * h + 1;

        while (h >= 1) {
            // h-sort the array
            for (int i = h; i < n; i++) {
                for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
                    exch(a, j, j - h);
                }
            }
            assert isHsorted(a, h);
            h /= 3;
        }
        assert isSorted(a);
    }

    // is the array h-sorted?
    private static boolean isHsorted(Comparable[] a, int h) {
        for (int i = h; i < a.length; i++)
            if (less(a[i], a[i - h])) return false;
        return true;
    }
}
```

也可以将希尔排序的递增序列存储在一个数组中。

``` Java
class ShellSortKeepIncrementSequence {

    public static void sort(Comparable[] a) {
        int n = a.length;
        int[] incrementSequence = getIncrementSequence(n);
        for (int h : incrementSequence) {
            for (int i = h; i < n; i++) {
                for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
                    exch(a, j, j - h);
                }
            }
        }
    }

    private static int[] getIncrementSequence(int n) {
        int size = (int) (Math.log(2 * n + 1) / Math.log(3));
        int[] incrementSequence = new int[size];
        for (int h = 0, i = size - 1; i >= 0; i--) {
            h = 3 * h + 1;
            incrementSequence[i] = h;
        }
        return incrementSequence;
    }
}
```

一个更加高效的递增序列为 1，5，19，41, 109，209，505，929，2161，3905，8929，16 001，36 289，64 769，146 305，260 609。该序列通过 `9 * (4^k) - 9 * (2^k) + 1` 与 `(4^k) - 3 * (2^k) + 1` 综合得到 。在实际应用中可能可以将性能改进 20% ~ 40%。

```Java
class ShellSortHighPerformanceIncrementSequence {

    public static void sort(Comparable[] a) {
        int n = a.length;
        Stack<Integer> incrementSequence = getIncrementSequence(n);
        while (!incrementSequence.isEmpty()) {
            int h = incrementSequence.pop();
            for (int i = h; i < n; i++) {
                for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
                    exch(a, j, j - h);
                }
            }
        }
    }

    private static Stack<Integer> getIncrementSequence(int n) {
        Stack<Integer> sequence = new Stack<>();
        int value = -1;
        int k = 0;
        while (true) {
            value = (int) (9 * Math.pow(4, k) - 9 * Math.pow(2, k) + 1);
            if (value < n) sequence.push(value);
            value = (int) (Math.pow(4, k + 2) - 3 * Math.pow(2, k + 2) + 1);
            if (value < n) sequence.push(value);
            if (value >= n) break;
            k++;
        }
        return sequence;
    }
}
```

### 希尔排序的特点

希尔排序比插入排序和选择排序要快得多，并且数组越大，优势越大。

对于中等大小的数组它的运行时间是可以接受的。它的代码量很小，且不需要使用额外的内存空间。

解决一个排序问题时，可以先用希尔排序，然后再考虑是否值得将它替换为更加复杂的排序算法。

---

