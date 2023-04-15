# 「算法第四版」二分查找




## 前提条件

查找的数组是有序的。

## 递归写法

``` Java
public class BinarySearch {

    public static int rank(int key, int[] a) {
        return rank(key, a, 0, a.length - 1);
    }

    public static int rank(int key, int[] a, int lo, int hi) {
        if (lo > hi) return -1;
        int mid = lo + (hi - lo) / 2;
        if (key < a[mid]) return rank(key, a, lo, hi - 1);
        else if (key > a[mid]) return rank(key, a, lo + 1, hi);
        else return mid;
    }
}
```

## 循环写法

``` Java
public class BinarySearch {

    public static int rank(int key, int[] a) {
        int lo = 0;
        int hi = a.length - 1;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (key < a[mid]) hi = lo - 1;
            else if (key > a[mid]) lo = hi + 1;
            else return mid;
        }
        return -1;
    }
}
```

## 二分查找 key 的最小索引

``` Java
public class BinarySearch {

    public static int rank(int key, int[] a) {
        return rank(key, a, 0, a.length - 1);
    }

    public static int rank(int[] array, int key, int lo, int hi) {
        while (lo <= hi) {
            int mid = lo + ((hi - lo) >> 1);
            if (key <= array[mid]) {
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        }
        if (lo == array.length) return -1;
        return array[lo] == key ? lo : -1;
    }
}
```

## 二分查找极小(大)值

``` Java
public class BinarySearch {

    public static int partialMin(int[] array) {
        assert array != null && array.length > 0;
        int lo = 0;
        int hi = array.length - 1;
        while (lo <= hi) {
            int mid = lo + ((hi - lo) >> 1);
            if (mid == 0 || mid == array.length - 1) break;
            if (array[mid] < array[mid - 1] && array[mid] < array[mid + 1]) {
                return mid;
            } else if (array[mid - 1] <= array[mid + 1]) {
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        }
        return -1;
    }
}
```

