# LeetCode 算法题详解

---

## 目录

- [一、链表](#一链表)
  - [1.1 反转链表](#11-反转链表)
  - [1.2 环形链表](#12-环形链表)
- [二、数组与双指针](#二数组与双指针)
  - [2.1 寻找数组的中心索引](#21-寻找数组的中心索引)
  - [2.2 删除排序数组中的重复项](#22-删除排序数组中的重复项)
  - [2.3 两数之和](#23-两数之和)
  - [2.4 三个数的最大乘积](#24-三个数的最大乘积)
  - [2.5 合并两个有序数组](#25-合并两个有序数组)
  - [2.6 子数组最大平均数](#26-子数组最大平均数)
  - [2.7 最长连续递增序列](#27-最长连续递增序列)
- [三、数学与数论](#三数学与数论)
  - [3.1 统计 N 以内的素数](#31-统计-n-以内的素数)
  - [3.2 x 的平方根](#32-x-的平方根)
  - [3.3 排列硬币](#33-排列硬币)
- [四、递归与动态规划](#四递归与动态规划)
  - [4.1 斐波那契数列](#41-斐波那契数列)
  - [4.2 预测赢家](#42-预测赢家)
  - [4.3 打家劫舍](#43-打家劫舍)
  - [4.4 香槟塔](#44-香槟塔)
- [五、二叉树](#五二叉树)
  - [5.1 二叉树的最小深度](#51-二叉树的最小深度)
  - [5.2 二叉树遍历](#52-二叉树遍历)
- [六、贪心算法](#六贪心算法)
  - [6.1 柠檬水找零](#61-柠檬水找零)
  - [6.2 三角形的最大周长](#62-三角形的最大周长)
  - [6.3 优势洗牌](#63-优势洗牌)
- [七、图论与并查集](#七图论与并查集)
  - [7.1 省份数量](#71-省份数量)
- [八、模拟与分类讨论](#八模拟与分类讨论)
  - [8.1 井字游戏](#81-井字游戏)
  - [8.2 Dota2 参议院](#82-dota2-参议院)

---

## 一、链表

### 1.1 反转链表

**题目**：反转一个单链表。

**示例**：输入 `1→2→3→4→5`，输出 `5→4→3→2→1`。

#### 解法一：迭代

从前往后遍历链表，将当前节点的 `next` 指向上一个节点。需要三个指针：

| 变量 | 作用 |
|------|------|
| `prev` | 上一个已反转节点 |
| `curr` | 当前待处理节点 |
| `next` | 提前保存 `curr.next`，防止断链 |

步骤：

1. `next = curr.next`
2. `curr.next = prev`
3. `prev = curr`
4. `curr = next`

![反转链表迭代指针移动示意](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224916.png)

![反转链表节点状态变化](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224650.png)

```java
public static ListNode iterate(ListNode head) {
    ListNode prev = null, curr = head, next;
    while (curr != null) {
        next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

#### 解法二：递归

将大问题（整链表反转）拆成性质相同的小问题（两个节点反转）。递归到末尾后，执行 `head.next.next = head; head.next = null`。

```java
public static ListNode recursion(ListNode head) {
    if (head == null || head.next == null) {
        return head;
    }
    ListNode newHead = recursion(head.next);
    head.next.next = head;
    head.next = null;
    return newHead;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(n)（递归栈）

---

### 1.2 环形链表

**题目**：判断链表中是否存在环（某个节点可通过连续跟踪 `next` 再次到达）。

#### 解法一：哈希表

遍历链表，将每个节点放入 `HashSet`，若 `add` 返回 `false` 说明已访问过，存在环。

```java
public static boolean hasCycle(ListNode head) {
    Set<ListNode> seen = new HashSet<>();
    while (head != null) {
        if (!seen.add(head)) {
            return true;
        }
        head = head.next;
    }
    return false;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(n)

#### 解法二：快慢指针（Floyd 判圈）

慢指针每次走 1 步，快指针每次走 2 步。若有环，两指针必相遇；若无环，快指针先到达 `null`。

![环形链表快慢指针相遇示意](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224137.png)

```java
public static boolean hasCycle2(ListNode head) {
    if (head == null || head.next == null) {
        return false;
    }
    ListNode slow = head;
    ListNode fast = head.next;
    while (slow != fast) {
        if (fast == null || fast.next == null) {
            return false;
        }
        slow = slow.next;
        fast = fast.next.next;
    }
    return true;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

## 二、数组与双指针

### 2.1 寻找数组的中心索引

**题目**：找到数组中某个下标，使得该下标左右两侧元素之和相等。

**思路**：先求数组总和 `sum1`，从左向右遍历，维护左侧累加和 `sum2`。当 `sum1 == sum2` 时，当前下标即为中心索引；每步更新 `sum1 -= nums[i]`、`sum2 += nums[i]`。

```java
public static int pivotIndex(int[] nums) {
    int sum1 = Arrays.stream(nums).sum();
    int sum2 = 0;
    for (int i = 0; i < nums.length; i++) {
        sum2 += nums[i];
        if (sum1 == sum2) {
            return i;
        }
        sum1 -= nums[i];
    }
    return -1;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

### 2.2 删除排序数组中的重复项

**题目**：有序数组原地删除重复元素，使每个元素只出现一次，返回新长度。要求 O(1) 额外空间。

**双指针**：`i` 为慢指针（已去重区域的末尾），`j` 为快指针。当 `nums[j] != nums[i]` 时，将 `nums[j]` 复制到 `nums[i+1]` 并递增 `i`。

```java
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int i = 0;
    for (int j = 1; j < nums.length; j++) {
        if (nums[j] != nums[i]) {
            i++;
            nums[i] = nums[j];
        }
    }
    return i + 1;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

### 2.3 两数之和

**题目**：在升序数组 `numbers` 中找到两个数之和等于 `target`，返回下标（1-indexed）。

| 解法 | 思路 | 时间 | 空间 |
|------|------|------|------|
| 暴力 | 双重循环 | O(n²) | O(1) |
| 哈希表 | `map` 存值→下标，查 `target - num` | O(n) | O(n) |
| 二分 | 固定一个值，对另一个二分查找 | O(n log n) | O(1) |
| 双指针 | 左小右大，和大了右移，和小了左移 | O(n) | O(1) |

**哈希表**（适用于无序数组）：

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        if (map.containsKey(target - nums[i])) {
            return new int[]{map.get(target - nums[i]), i};
        }
        map.put(nums[i], i);
    }
    return new int[0];
}
```

**双指针**（适用于有序数组）：

```java
public int[] twoPoint(int[] numbers, int target) {
    int low = 0, high = numbers.length - 1;
    while (low < high) {
        int sum = numbers[low] + numbers[high];
        if (sum == target) {
            return new int[]{low + 1, high + 1};
        } else if (sum < target) {
            low++;
        } else {
            high--;
        }
    }
    return new int[]{-1, -1};
}
```

---

### 2.4 三个数的最大乘积

**题目**：在整型数组中找出三个数组成的最大乘积。

**关键观察**：

- 全为非负数：最大的三个数相乘
- 全为非正数：最大的三个数相乘（负负得正）
- 有正有负：可能是「三个最大正数」或「两个最小负数 × 最大正数」

**解法一：排序** — 排序后比较 `nums[0]*nums[1]*nums[n-1]` 与 `nums[n-3]*nums[n-2]*nums[n-1]`

**解法二：线性扫描** — 一次遍历维护两个最小值和三个最大值

```java
public static int getMaxMin(int[] nums) {
    int min1 = Integer.MAX_VALUE, min2 = Integer.MAX_VALUE;
    int max1 = Integer.MIN_VALUE, max2 = Integer.MIN_VALUE, max3 = Integer.MIN_VALUE;
    for (int x : nums) {
        if (x < min1) { min2 = min1; min1 = x; }
        else if (x < min2) { min2 = x; }
        if (x > max1) { max3 = max2; max2 = max1; max1 = x; }
        else if (x > max2) { max3 = max2; max2 = x; }
        else if (x > max3) { max3 = x; }
    }
    return Math.max(min1 * min2 * max1, max1 * max2 * max3);
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

### 2.5 合并两个有序数组

**题目**：将 `nums2` 合并进 `nums1`（`nums1` 长度为 `m+n`，前 `m` 个有效），使结果有序。

| 解法 | 思路 | 时间 | 空间 |
|------|------|------|------|
| 合并后排序 | `System.arraycopy` + `Arrays.sort` | O((m+n)log(m+n)) | O(1) |
| 双指针从前往后 | 拷贝 `nums1`，两指针比较填入 | O(m+n) | O(m) |
| 双指针从后往前 | 从尾部比较，无需额外数组 | O(m+n) | O(1) |

**从后往前双指针**（推荐）：

```java
public void merge(int[] nums1, int m, int[] nums2, int n) {
    int p1 = m - 1, p2 = n - 1, p = m + n - 1;
    while (p1 >= 0 && p2 >= 0) {
        nums1[p--] = (nums1[p1] < nums2[p2]) ? nums2[p2--] : nums1[p1--];
    }
    System.arraycopy(nums2, 0, nums1, 0, p2 + 1);
}
```

---

### 2.6 子数组最大平均数

**题目**：找出长度为 `k` 的连续子数组的最大平均数。

**滑动窗口**：先计算前 `k` 个元素之和，然后窗口右移时 `sum = sum - nums[i-k] + nums[i]`，维护最大和。

![滑动窗口求子数组最大平均数](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202606280224281.png)

```java
public double findMaxAverage(int[] nums, int k) {
    int sum = 0;
    for (int i = 0; i < k; i++) {
        sum += nums[i];
    }
    int maxSum = sum;
    for (int i = k; i < nums.length; i++) {
        sum = sum - nums[i - k] + nums[i];
        maxSum = Math.max(maxSum, sum);
    }
    return 1.0 * maxSum / k;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

### 2.7 最长连续递增序列

**题目**：在未经排序的数组中找到最长且连续递增的子序列长度。

**贪心**：当 `nums[i] <= nums[i-1]` 时，递增序列中断，重置起点为 `i`；否则更新最大长度 `ans = max(ans, i - start + 1)`。

```java
public static int findLength(int[] nums) {
    int ans = 0, start = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > 0 && nums[i] <= nums[i - 1]) {
            start = i;
        }
        ans = Math.max(ans, i - start + 1);
    }
    return ans;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

## 三、数学与数论

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

## 四、递归与动态规划

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

## 五、二叉树

### 5.1 二叉树的最小深度

**题目**：从根到最近叶子节点的最短路径上的节点数。

#### 解法一：深度优先（DFS）

递归求左右子树最小深度，叶子节点返回 1。

```java
public int minDepth(TreeNode root) {
    if (root == null) return 0;
    if (root.left == null && root.right == null) return 1;
    int minDepth = Integer.MAX_VALUE;
    if (root.left != null) minDepth = Math.min(minDepth(root.left), minDepth);
    if (root.right != null) minDepth = Math.min(minDepth(root.right), minDepth);
    return minDepth + 1;
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(log n)

#### 解法二：广度优先（BFS）

层序遍历，遇到第一个叶子节点立即返回深度（BFS 天然适合求最短路径）。

```java
public int minDepth(TreeNode root) {
    if (root == null) return 0;
    Queue<QueueNode> queue = new LinkedList<>();
    queue.offer(new QueueNode(root, 1));
    while (!queue.isEmpty()) {
        QueueNode nodeDepth = queue.poll();
        TreeNode node = nodeDepth.node;
        int depth = nodeDepth.depth;
        if (node.left == null && node.right == null) return depth;
        if (node.left != null) queue.offer(new QueueNode(node.left, depth + 1));
        if (node.right != null) queue.offer(new QueueNode(node.right, depth + 1));
    }
    return 0;
}
```

---

### 5.2 二叉树遍历

| 遍历方式 | 顺序 | 访问时机 |
|----------|------|----------|
| 前序 | 根 → 左 → 右 | 第一次成为栈顶 |
| 中序 | 左 → 根 → 右 | 第二次成为栈顶 |
| 后序 | 左 → 右 → 根 | 第三次成为栈顶 |
| 层序 | 逐层从左到右 | BFS |

#### 递归遍历

```java
public static void inorder(TreeNode root) {
    if (root == null) return;
    inorder(root.left);
    System.out.println(root.val);  // 中序
    inorder(root.right);
}
```

#### 迭代遍历

**前序**（栈，先右后左入栈）：

```java
public static void preOrder2(TreeNode head) {
    if (head == null) return;
    Stack<TreeNode> stack = new Stack<>();
    stack.push(head);
    while (!stack.isEmpty()) {
        head = stack.pop();
        System.out.println(head.val);
        if (head.right != null) stack.push(head.right);
        if (head.left != null) stack.push(head.left);
    }
}
```

**中序**（左子树全部入栈后出栈打印）：

```java
public static void inOrderIterative(TreeNode head) {
    Stack<TreeNode> stack = new Stack<>();
    while (!stack.isEmpty() || head != null) {
        if (head != null) {
            stack.push(head);
            head = head.left;
        } else {
            head = stack.pop();
            System.out.println(head.val);
            head = head.right;
        }
    }
}
```

**后序**（栈 + `prev` 记录上一个打印节点）：

```java
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> res = new ArrayList<>();
    Deque<TreeNode> stack = new LinkedList<>();
    TreeNode prev = null;
    while (root != null || !stack.isEmpty()) {
        while (root != null) {
            stack.push(root);
            root = root.left;
        }
        root = stack.pop();
        if (root.right == null || root.right == prev) {
            res.add(root.val);
            prev = root;
            root = null;
        } else {
            stack.push(root);
            root = root.right;
        }
    }
    return res;
}
```

#### 层序遍历

```java
public static void levelOrder(TreeNode root) {
    if (root == null) return;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        for (int i = queue.size(); i > 0; i--) {
            TreeNode node = queue.poll();
            System.out.println(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }
}
```

#### 线索二叉树

N 个节点的二叉树有 2N 个指针，其中 N-1 个被父子关系占用，剩余 N+1 个为空。线索二叉树利用这些空指针存放前驱/后继信息，提高遍历效率。

#### Morris 遍历

O(1) 空间的中序/前序/后序遍历。利用叶子节点的空右指针建立临时线索，遍历完成后恢复。

**Morris 中序核心逻辑**：

1. 若当前节点无左子树 → 打印，向右
2. 若有左子树 → 找左子树的最右节点
   - 最右节点的右指针为空 → 建立线索指向当前节点，向左
   - 最右节点的右指针指向当前节点 → 断开线索，打印当前节点，向右

```java
public static void morrisIn(TreeNode cur) {
    TreeNode mostRight;
    while (cur != null) {
        mostRight = cur.left;
        if (mostRight != null) {
            while (mostRight.right != null && mostRight.right != cur) {
                mostRight = mostRight.right;
            }
            if (mostRight.right == null) {
                mostRight.right = cur;
                cur = cur.left;
                continue;
            } else {
                mostRight.right = null;
            }
        }
        System.out.print(cur.val + " ");
        cur = cur.right;
    }
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

## 六、贪心算法

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

## 七、图论与并查集

### 7.1 省份数量

**题目**：n 个城市，邻接矩阵 `isConnected[i][j]=1` 表示直连。求省份（连通分量）数量。

| 解法 | 思路 | 时间 | 空间 |
|------|------|------|------|
| DFS | 遍历未访问城市，递归标记连通城市 | O(n²) | O(n) |
| BFS | 队列逐层扩散标记 | O(n²) | O(n) |
| 并查集 | 相连城市合并，统计根节点数 | O(n² α(n)) | O(n) |

**DFS**：

```java
public int findCircleNum(int[][] isConnected) {
    int n = isConnected.length;
    boolean[] visited = new boolean[n];
    int circles = 0;
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            dfs(isConnected, visited, n, i);
            circles++;
        }
    }
    return circles;
}

public void dfs(int[][] isConnected, boolean[] visited, int n, int i) {
    for (int j = 0; j < n; j++) {
        if (isConnected[i][j] == 1 && !visited[j]) {
            visited[j] = true;
            dfs(isConnected, visited, n, j);
        }
    }
}
```

**并查集**（路径压缩 + 按秩合并）：

```java
static int find(int x, int[] parent) {
    if (parent[x] == x) return x;
    parent[x] = find(parent[x], parent);  // 路径压缩
    return parent[x];
}

static void merge(int x, int y, int[] parent, int[] rank) {
    int i = find(x, parent), j = find(y, parent);
    if (i == j) return;
    if (rank[i] <= rank[j]) parent[i] = j;
    else parent[j] = i;
    rank[j]++;
}
```

---

## 八、模拟与分类讨论

### 8.1 井字游戏

**题目**：判断 3×3 井字棋局面是否可能由合法对局产生。

**分类讨论**：

1. O 的数量等于 X，或 O 比 X 少 1（X 先手）
2. 若 X 三连且 O 数量等于 X → 非法（X 赢后不应再下 O）
3. 若 O 三连且 O 数量不等于 X → 非法

```java
public static boolean validBoard(String[] board) {
    int xCount = 0, oCount = 0;
    for (String row : board) {
        for (char c : row.toCharArray()) {
            if (c == 'X') xCount++;
            if (c == 'O') oCount++;
        }
    }
    if (oCount != xCount && oCount != xCount - 1) return false;
    if (win(board, "XXX") && oCount != xCount - 1) return false;
    if (win(board, "OOO") && oCount != xCount) return false;
    return true;
}

public static boolean win(String[] board, String flag) {
    for (int i = 0; i < 3; i++) {
        if (flag.equals("" + board[i].charAt(0) + board[i].charAt(1) + board[i].charAt(2)))
            return true;
        if (flag.equals("" + board[0].charAt(i) + board[1].charAt(i) + board[2].charAt(i)))
            return true;
    }
    if (flag.equals("" + board[0].charAt(0) + board[1].charAt(1) + board[2].charAt(2)))
        return true;
    if (flag.equals("" + board[0].charAt(2) + board[1].charAt(1) + board[2].charAt(0)))
        return true;
    return false;
}
```

---

### 8.2 Dota2 参议院

**题目**：R（Radiant）和 D（Dire）参议员轮流投票，每人可禁止对手一名参议员或宣布胜利。预测最终哪方获胜。

**模拟 + 队列**：分别维护 R、D 的下标队列。每轮队首两人对决，下标小者胜并重新入队（下标 +n），直到一方队列为空。

```java
public String predictPartyVictory(String senate) {
    int n = senate.length();
    Queue<Integer> radiant = new LinkedList<>();
    Queue<Integer> dire = new LinkedList<>();
    for (int i = 0; i < n; i++) {
        if (senate.charAt(i) == 'R') radiant.offer(i);
        else dire.offer(i);
    }
    while (!radiant.isEmpty() && !dire.isEmpty()) {
        int r = radiant.poll(), d = dire.poll();
        if (r < d) radiant.offer(r + n);
        else dire.offer(d + n);
    }
    return !radiant.isEmpty() ? "Radiant" : "Dire";
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(n)
