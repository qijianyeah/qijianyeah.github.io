# 图论与并查集

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
