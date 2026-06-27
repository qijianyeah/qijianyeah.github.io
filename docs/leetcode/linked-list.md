# 链表

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
