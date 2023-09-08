# 「算法第四版」欧几里得算法求最大公因数


---

## 自然语言描述

计算两个**非负整数** p 和 q 的最大公约数：若 q 是 0，则最大公约数为 p。否则，将 p 除以 q 得到余数 r，p 和 q 的最大公约数即为 q 和 r 的最大公约数。

---

## 递归写法

``` Java
public class Euclid {

    public static int gcd(int p, int q) {
        if (q == 0) return p;
        int r = p % q;
        return gcd(q, r);
    }
}
```

---

## 循环写法

``` Java
public class Euclid {

    public static int gcd(int p, int q) {
        if (q == 0) return p;
        while (q != 0) {
            int r = p % q;
            p = q;
            q = r;
        }
        return p;
    }
}
```

---

