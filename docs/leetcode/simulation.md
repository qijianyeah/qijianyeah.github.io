# 模拟与分类讨论

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
