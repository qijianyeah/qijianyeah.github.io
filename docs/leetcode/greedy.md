# 贪心算法

### 6.1 柠檬水找零

**题目**：顾客付 5/10/20 美元，初始无零钱，判断能否给所有人正确找零。

**贪心策略**：

- 收 5 元：直接收
- 收 10 元：需找 5 元
- 收 20 元：优先用 10+5 找零，否则用 3 张 5 元

```java
public boolean lemonadeChange(int[] bills) {
    int five = 0, ten = 0;
    for (int bill : bills) {
        if (bill == 5) {
            five++;
        } else if (bill == 10) {
            if (five == 0) return false;
            five--;
            ten++;
        } else {
            if (five > 0 && ten > 0) {
                five--;
                ten--;
            } else if (five >= 3) {
                five -= 3;
            } else {
                return false;
            }
        }
    }
    return true;
}
```

---

### 6.2 三角形的最大周长

**题目**：给定正数数组，返回能构成非零面积三角形的最大周长，否则返回 0。

**贪心**：排序后从大到小检查，若 `A[i-2] + A[i-1] > A[i]` 则三者可构成三角形，周长最大。

```java
public int largestPerimeter(int[] A) {
    Arrays.sort(A);
    for (int i = A.length - 1; i >= 2; i--) {
        if (A[i - 2] + A[i - 1] > A[i]) {
            return A[i - 2] + A[i - 1] + A[i];
        }
    }
    return 0;
}
```

---

### 6.3 优势洗牌

**题目**：数组 A 重新排列，使 `A[i] > B[i]` 的下标数最大化（田忌赛马）。

**贪心 + 排序**：

1. 将 A、B 分别排序
2. A 中比 B 当前最小值大的最小元素去「赢」B 的最小值
3. 其余 A 的元素放入剩余池，填到 A 赢不了的 B 位置

![优势洗牌贪心匹配策略](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224278.png)

```java
public static int[] advantageCount(int[] A, int[] B) {
    int[] sortedA = A.clone();
    Arrays.sort(sortedA);
    int[] sortedB = B.clone();
    Arrays.sort(sortedB);
    Map<Integer, Deque<Integer>> assigned = new HashMap<>();
    for (int b : B) assigned.put(b, new LinkedList<>());
    Deque<Integer> remaining = new LinkedList<>();
    int j = 0;
    for (int a : sortedA) {
        if (a > sortedB[j]) {
            assigned.get(sortedB[j++]).add(a);
        } else {
            remaining.add(a);
        }
    }
    int[] ans = new int[B.length];
    for (int i = 0; i < B.length; i++) {
        if (!assigned.get(B[i]).isEmpty())
            ans[i] = assigned.get(B[i]).removeLast();
        else
            ans[i] = remaining.removeLast();
    }
    return ans;
}
```

- 时间复杂度：O(n log n)
- 空间复杂度：O(n)

---
