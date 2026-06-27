# 二叉树

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
