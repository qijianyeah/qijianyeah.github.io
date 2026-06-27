# 递归与动态规划

### 4.1 斐波那契数列

**定义**：F(0)=0, F(1)=1, F(n)=F(n-1)+F(n-2)。

| 解法 | 思路 | 时间 | 空间 |
|------|------|------|------|
| 暴力递归 | 直接递归 | O(2^n) | O(n) |
| 记忆化递归 | 数组缓存已算结果 | O(n) | O(n) |
| 双指针迭代 | 只保留前两项 | O(n) | O(1) |

![斐波那契暴力递归重复计算树](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224885.png)

![斐波那契记忆化递归避免重复计算](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224014.png)

**记忆化递归**：

```java
private static int recurse(int[] arr, int num) {
    if (num == 0) return 0;
    if (num == 1) return 1;
    if (arr[num] != 0) return arr[num];
    arr[num] = recurse(arr, num - 1) + recurse(arr, num - 2);
    return arr[num];
}
```

**双指针迭代**（最优）：

```java
public static int iterate(int num) {
    if (num == 0) return 0;
    if (num == 1) return 1;
    int low = 0, high = 1;
    for (int i = 2; i <= num; i++) {
        int sum = low + high;
        low = high;
        high = sum;
    }
    return high;
}
```

---

### 4.2 预测赢家

**题目**：两人轮流从数组任意一端取一个数，假设双方都最优，判断玩家 1 是否能赢或平局。

**博弈 DP**：定义 `dp[i][j]` 为玩家在区间 `[i,j]` 相对对手的最大净胜分。

```java
public boolean PredictTheWinner(int[] nums) {
    int length = nums.length;
    int[][] dp = new int[length][length];
    for (int i = 0; i < length; i++) {
        dp[i][i] = nums[i];
    }
    for (int i = length - 2; i >= 0; i--) {
        for (int j = i + 1; j < length; j++) {
            dp[i][j] = Math.max(nums[i] - dp[i + 1][j], nums[j] - dp[i][j - 1]);
        }
    }
    return dp[0][length - 1] >= 0;
}
```

- 时间复杂度：O(n²)
- 空间复杂度：O(n²)，可优化为 O(n)

---

### 4.3 打家劫舍

**题目**：不能偷相邻房屋，求一夜能偷到的最高金额。

**状态转移**：`dp[i] = max(dp[i-2] + nums[i], dp[i-1])`

**空间优化**（滚动变量）：

```java
public int rob(int[] nums) {
    if (nums.length == 1) return nums[0];
    int first = nums[0], second = Math.max(nums[0], nums[1]);
    for (int i = 2; i < nums.length; i++) {
        int temp = second;
        second = Math.max(first + nums[i], second);
        first = temp;
    }
    return second;
}
```

**环形房屋**（首尾相连）：分别对 `[0, n-2]` 和 `[1, n-1]` 求最大值，取较大者。

**二叉树打家劫舍**：DFS 返回 `[选当前节点的最大值, 不选当前节点的最大值]`。

```java
public int[] dfs(TreeNode node) {
    if (node == null) return new int[]{0, 0};
    int[] l = dfs(node.left);
    int[] r = dfs(node.right);
    int selected = node.val + l[1] + r[1];
    int notSelected = Math.max(l[0], l[1]) + Math.max(r[0], r[1]);
    return new int[]{selected, notSelected};
}
```

---

### 4.4 香槟塔

**题目**：金字塔形香槟塔，每层满 250ml 后溢出均分到左右下层。求第 `query_row` 行第 `query_glass` 个杯子的香槟占比。

**模拟**：逐层计算每个杯子中的香槟量，溢出量 `(A[r][c] - 1.0) / 2.0` 分给下层左右两杯。

```java
public double champagneTower(int poured, int query_row, int query_glass) {
    double[][] A = new double[102][102];
    A[0][0] = poured;
    for (int r = 0; r <= query_row; r++) {
        for (int c = 0; c <= r; c++) {
            double q = (A[r][c] - 1.0) / 2.0;
            if (q > 0) {
                A[r + 1][c] += q;
                A[r + 1][c + 1] += q;
            }
        }
    }
    return Math.min(1, A[query_row][query_glass]);
}
```

---
