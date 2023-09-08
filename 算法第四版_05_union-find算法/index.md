# 「算法第四版」Union-Find 算法


---

## 动态连通性问题及其应用

若整数对 (p, q) 是“相连”的，则：

- 自反性：p 和 p 是相连的；
- 对称性：如果 p 和 q 是相连的，那么 q 和 p 也是相连的；
- 传递性：如果 p 和 q 是相连的且 q 和 r 是相连的，那么 p 和 r 也是相连的。

当且仅当两个对象相连时它们才属于同一个等价类。

判断一对新的对象是否“相连”，这样的问题叫**动态连通性**问题。

### 网络

在大型计算机网络中，输入的整数表示主机(**触点**)，整数对表示网络(**连接**)，同一网络中的主机属于同一等价类(**连通分量**)。

在社交网络中，整数可以表示社交网络中的人，整数对表示朋友关系。

### 数学集合

将每个整数看做属于不同的数学集合，每一个整数对则需要先判断是否在同一数学集合中。若不是，则将 p 所属集合与 q 所属集合归并到同一集合。

---

## Union-Find 算法的 API

``` Text
public class UF

             UF(int N)                  以整数标志 [0, N-1] 初始化 N 个触点
       void  union(int p, int q)        在 p 和 q 之间添加一条连接
        int  find(int p)                p (0 <= p <= N-1) 所在的分量标识符
    boolean  connected(int p, int q)    p 和 q 是否连通
        int  count()                    连通分量的数量
```

---

## Union-Find 算法的实现

基本的 UF 算法实现见如下代码，union 与 find 的详细实现分别见 Quick-Find，Quick-Union，Weighted Quick-Union，Quick-Union with Path Compresson 的实现。

``` Java
public class UF {

    private int[] id;
    private int count;

    public UF(int n) {
        count = n;
        id = new int[n];
        for (int i = 0; i < n; i++)
            id[i] = i;
    }

    public void union(int p, int q);

    public int find(int p);

    public boolean connected(int p, int q) { return find(p) == find(q); }

    public int count() { return count; }

    private void validate(int p) {
        int n = id.length;
        if (p < 0 || p >= n) {
            throw new IllegalArgumentException("index " + p + " is not between 0 and " + (n-1));
        }
    }

    public static void main(String[] args) {
        int n = StdIn.readInt();
        UF uf = new UF(n);
        while (!StdIn.isEmpty()) {
            int p = StdIn.readInt();
            int q = StdIn.readInt();
            if (uf.find(p) == uf.find(q)) continue;
            uf.union(p, q);
            StdOut.println(p + " " + q);
        }
        StdOut.println(uf.count() + " components");
    }
}
```

### Quick-Find

``` Java
public class QuickFindUF {

    private int[] id;    // id[i] = component identifier of i
    private int count;   // number of components

    public QuickFindUF(int n) {
        count = n;
        id = new int[n];
        for (int i = 0; i < n; i++)
            id[i] = i;
    }

    public void union(int p, int q) {
        validate(p);
        validate(q);
        int pID = id[p];   // needed for correctness
        int qID = id[q];   // to reduce the number of array accesses

        // p and q are already in the same component
        if (pID == qID) return;

        for (int i = 0; i < id.length; i++)
            if (id[i] == pID) id[i] = qID;
        count--;
    }

    public int find(int p) {
        validate(p);
        return id[p];
    }
}
```

Quick-Find 算法中，find 方法的时间复杂度为 O(1)，但 union 方法的时间复杂度为 O(n)。

### Quick-Union

``` Java
public class QuickUnionUF {

    private int[] parent;  // parent[i] = parent of i
    private int count;     // number of components

    public QuickUnionUF(int n) {
        parent = new int[n];
        count = n;
        for (int i = 0; i < n; i++) {
            parent[i] = i;
        }
    }

    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) return;
        parent[rootP] = rootQ;
        count--;
    }

    public int find(int p) {
        validate(p);
        while (p != parent[p])
            p = parent[p];
        return p;
    }
}
```

Quick-Union 算法中，h 表示树的高度，find 方法的时间复杂度为 O(h)，union 方法的时间复杂度为 2·O(h) + 1 = O(h)。

### Weighted Quick-Union

``` Java
public class WeightedQuickUnionUF {

    private int[] parent;   // parent[i] = parent of i
    private int[] size;     // size[i] = number of elements in subtree rooted at i
    private int count;      // number of components

    public WeightedQuickUnionUF(int n) {
        count = n;
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
            size[i] = 1;
        }
    }

    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) return;

        // make smaller root point to larger one
        if (size[rootP] < size[rootQ]) {
            parent[rootP] = rootQ;
            size[rootQ] += size[rootP];
        }
        else {
            parent[rootQ] = rootP;
            size[rootP] += size[rootQ];
        }
        count--;
    }

    public int find(int p) {
        validate(p);
        while (p != parent[p])
            p = parent[p];
        return p;
    }
}
```

加权 Quick-Union 算法在最坏情况下，即归并的两个树大小总是相等，此时能够保证对数级别的性能 O(logN)。

加权 Quick-Union 算法能够在合理时间范围内处理大规模动态连通性问题。

### Weighted Quick-Union with Path Compresson

``` Java
public class WeightedQuickUnionWithPathCompression {

    private final int[] parent;
    private final int[] size;
    private int count;

    WeightedQuickUnionWithPathCompression(int n) {
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
            size[i] = 1;
        }
        count = n;
    }

    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) return;
        if (size[rootP] < size[rootQ]) {
            parent[rootP] = rootQ;
            size[rootQ] += size[rootP];
        } else {
            parent[rootQ] = rootP;
            size[rootP] += size[rootQ];
        }
        count--;
    }

    public int find(int p) {
        validate(p);
        int root = p;
        while (root != parent[root]) {
            root = parent[root];
        }
        // Path Compression
        while (parent[p] != root) {
            int temp = parent[p];
            parent[p] = root;
            p = temp;
        }
        return root;
    }
}
```

路经压缩的加权 Quick-Union 算法，find 方法能够得到几乎完全扁平化的树，使得算法的均摊成本**接近** O(1)。

路经压缩的加权 Quick-Union 算法是本题的最优算法。

---

