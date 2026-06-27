# 数组与双指针

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
