# 数学与数论

### 3.1 统计 N 以内的素数

**素数**：只能被 1 和自身整除的数（0、1 除外）。

#### 解法一：暴力

对每个数 `i` 判断是否为素数，只需检查 `[2, √i]` 范围内的因子。

```java
public boolean isPrime(int x) {
    for (int i = 2; i * i <= x; i++) {
        if (x % i == 0) return false;
    }
    return true;
}
```

- 时间复杂度：O(n√n)

#### 解法二：埃拉托斯特尼筛（埃氏筛）

从 2 开始，将每个素数的倍数标记为合数。优化：内层循环从 `i*i` 开始（更小的倍数已被更小的素数标记）。

```java
public static int eratosthenes(int n) {
    boolean[] isComposite = new boolean[n];
    int ans = 0;
    for (int i = 2; i < n; i++) {
        if (!isComposite[i]) {
            ans++;
            for (int j = i * i; j < n; j += i) {
                isComposite[j] = true;
            }
        }
    }
    return ans;
}
```

- 时间复杂度：O(n log log n)
- 空间复杂度：O(n)

---

### 3.2 x 的平方根

**题目**：不使用 `sqrt(x)`，求 x 平方根的整数部分。

#### 解法一：二分查找

在 `[0, x]` 范围内二分，找最大的 `mid` 使得 `mid * mid <= x`。

```java
public static int binarySearch(int x) {
    int l = 0, r = x, index = -1;
    while (l <= r) {
        int mid = l + (r - l) / 2;
        if ((long) mid * mid <= x) {
            index = mid;
            l = mid + 1;
        } else {
            r = mid - 1;
        }
    }
    return index;
}
```

- 时间复杂度：O(log n)

#### 解法二：牛顿迭代

由 `i + x/i = 2i` 推导出迭代公式 `i = (i + x/i) / 2`，收敛到平方根。

```java
public static double sqrts(double i, int x) {
    double res = (i + x / i) / 2;
    if (res == i) return i;
    return sqrts(res, x);
}
```

---

### 3.3 排列硬币

**题目**：n 枚硬币摆成阶梯形（第 k 行 k 枚），求完整行数。即求最大的 `k` 使 `k(k+1)/2 <= n`。

| 解法 | 思路 | 时间 |
|------|------|------|
| 迭代 | 逐行减去硬币 | O(√n) |
| 二分 | 对行数二分，比较 `mid*(mid+1)/2` 与 n | O(log n) |
| 牛顿迭代 | 解方程 `x(x+1)/2 = n` | O(log n) |

```java
public static int arrangeCoins2(int n) {
    int low = 0, high = n;
    while (low <= high) {
        long mid = (high - low) / 2 + low;
        long cost = (mid + 1) * mid / 2;
        if (cost == n) return (int) mid;
        else if (cost > n) high = (int) mid - 1;
        else low = (int) mid + 1;
    }
    return high;
}
```

---
