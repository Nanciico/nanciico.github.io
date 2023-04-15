# 《算法第四版》算法分析


## 算法分析的方法

- Observe some feature of the natural world, generally with precise measurements.
- Hypothesize a model that is consistent with the observations.
- Predict events using the hypothesis.
- Verify the predictions by making further observations.
- Validate by repeating until the hypothesis and observations agree.

## 算法运行时间实验

程序执行速度的快慢通常取决于问题的规模。

通过观察代码初步预测程序的执行时间。

使用类似 DoublingTest 方法不断增加问题规模并计时，进入“预测——验证”循环。

``` Java
public class DoublingTest {
    public static double timeTrial(int N) {  
        // Time ThreeSum.count() for N random 6-digit ints.
        int MAX = 1000000;
        int[] a = new int[N];
        for (int i = 0; i < N; i++)
            a[i] = StdRandom.uniformInt(-MAX, MAX);
        Stopwatch timer = new Stopwatch();
        int cnt = ThreeSum.count(a);
        return timer.elapsedTime();
    }

    public static void main(String[] args) {  
        // Print table of running times.
        for (int N = 250; true; N += N) {  
            // Print time for problem size N.
            double time = timeTrial(N);
            StdOut.printf("%7d %5.1f\n", N, time);
        }
    }
}
```

分析实验数据，得到其运行时间的数学模型。

## 运行时间的数学模型

得到运行时间的数学模型步骤：（书：P114）

- 确定**输入模型**，定义问题规模
- 识别**内循环**
- 根据内循环中的操作确定**成本模型**
- 对于给定的输入，判断这些操作的执行频率

## 增长数量级

| 描述 | 增长的数量级 | 典型代码 | 说明 | 举例 |
| :----: | :----: | :----: | :----: | :----: |
| 常数级别 | 1 | a = b + c | 普通语句 | 两个数相加 |
| 对数级别 | logN | 二分查找 | 二分策略 | 二分查找 |
| 线性级别 | N | 循环 | 循环 | 循环查找最大元素 |
| 线性对数级别 | NlogN | 归并排序 Merge.sort 和 快速排序 Quick.sort | 归并排序，快速排序 | 归并排序，快速排序 |
| 平方级别 | N^2 | 选择排序 Selection.sort 和 插入排序 Insertion.sort | 双层循环 | 选择排序，插入排序 |
| 立方级别 | N^3 | ThreeSum 三层循环 | 三层循环 | 三层循环 |
| 指数级别 | 2^N | 第六章 | 穷举查找 | 检查所有子集 |

## 倍率实验

通过倍率实验能够简单有效地预测任意程序的性能并判断它们运行时间大致的增长数量级，但对比值没有极限的算法无效。

``` Java
public class DoublingRatio {
    public static double timeTrial(int N) {  
        // Time ThreeSum.count() for N random 6-digit ints.
        int MAX = 1000000;
        int[] a = new int[N];
        for (int i = 0; i < N; i++)
            a[i] = StdRandom.uniformInt(-MAX, MAX);
        Stopwatch timer = new Stopwatch();
        int cnt = ThreeSum.count(a);
        return timer.elapsedTime();
    }

    public static void main(String[] args) {
        double prev = timeTrial(125);
        for (int N = 250; true; N += N) {
            double time = timeTrial(N);
            StdOut.printf("%6d %7.1f ", N, time);
            StdOut.printf("%5.1f\n", time / prev);
            prev = time;
        }
    }
}
```

在有性能问题的情况家应该考虑对编写过的所有程序进行倍率实验，以便能找到性能问题。

