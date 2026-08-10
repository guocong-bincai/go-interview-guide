# 图论高频题：DFS/BFS/并查集

> 考察频率：★★★☆☆  优先级：P2  Go + Python 双语
> 题型识别：有向/无向图遍历、连通分量、环检测、拓扑排序、最短路

---

## 面试官考察意图

考察候选人的**图建模能力**和**模板化思维**。
初级工程师能套模板写出正确解，但说不清楚何时选 DFS/BFS/并查集；高级工程师能直接给出时间/空间复杂度分析，并能根据题目约束（图是否稠密、是否需要最短路）快速选择最优算法。

---

## 核心答案（30 秒版）

图论题三大武器：

| 武器 | 适用场景 | 核心模板 |
|------|----------|----------|
| **DFS（递归/栈）** | 连通分量、岛屿、路径搜索、环检测 | visited + 回溯 |
| **BFS（队列）** | 最短路、层序遍历、拓扑排序 | queue + 距离数组 |
| **并查集（Union Find）** | 连通分量、冗余连接、等价类分组 | find + union + 路径压缩 |

**选择口诀：最短路用 BFS，连通分量用并查集，枚举所有路径用 DFS。**

---

## 深度展开

### 1. 图的表示：邻接表 vs 邻接矩阵

**面试关注点：** 为什么 LeetCode 图题几乎都用邻接表？

```go
// Go 邻接表（最常用）
type Graph struct {
    n     int
    adj   [][]int  // adj[i] = i 的邻居列表
}

// 无向图构建
func buildGraph(edges [][]int, n int) *Graph {
    g := &Graph{n: n, adj: make([][]int, n)}
    for _, e := range edges {
        g.adj[e[0]] = append(g.adj[e[0]], e[1])
        g.adj[e[1]] = append(g.adj[e[1]], e[0]) // 无向图双向
    }
    return g
}
```

```python
# Python 邻接表
from collections import defaultdict, deque

class Graph:
    def __init__(self, n: int):
        self.n = n
        self.adj = defaultdict(list)

    def add_edge(self, u: int, v: int, directed=False):
        self.adj[u].append(v)
        if not directed:
            self.adj[v].append(u)
```

| 表示法 | 空间 | 适用场景 |
|--------|------|----------|
| **邻接表** | O(V+E) | 稀疏图（绝大多数 LeetCode 题）|
| **邻接矩阵** | O(V²) | 稠密图、需要 O(1) 判断连通性 |

---

### 2. DFS 模板（递归 + 访问标记）

**适用：** 连通分量计数、岛屿问题、路径枚举、图的深拷贝

```go
// Go DFS 模板
func dfs(g *Graph, node int, visited []bool) {
    visited[node] = true
    fmt.Print(node, " ")

    for _, neighbor := range g.adj[node] {
        if !visited[neighbor] {
            dfs(g, neighbor, visited)
        }
    }
}

// 遍历所有连通分量
func exploreAll(g *Graph) int {
    visited := make([]bool, g.n)
    count := 0

    for i := 0; i < g.n; i++ {
        if !visited[i] {
            count++
            dfs(g, i, visited)
        }
    }
    return count
}
```

```python
# Python DFS 模板（递归）
def dfs(node: int, visited: list):
    visited[node] = True
    print(node, end=" ")

    for neighbor in graph.adj[node]:
        if not visited[neighbor]:
            dfs(neighbor, visited)

# 遍历所有连通分量
def count_components(n, edges):
    graph = Graph(n)
    for u, v in edges:
        graph.add_edge(u, v)
    
    visited = [False] * n
    count = 0

    for i in range(n):
        if not visited[i]:
            count += 1
            dfs(i, visited)
    
    return count
```

**DFS 递归深度警惕：** Go 默认栈大小约 1-2GB（栈自动增长），但递归过深（>10000 层）可能导致栈溢出。生产中优先用**显式栈**模拟。

---

### 3. BFS 模板（队列 + 距离数组）

**适用：** 最短路（边权为 1）、拓扑排序、层序扩展

```go
// Go BFS 最短路模板
func bfsShortest(g *Graph, start int) []int {
    dist := make([]int, g.n)
    for i := range dist {
        dist[i] = -1
    }
    dist[start] = 0

    queue := make([]int, 0)
    queue = append(queue, start)

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        for _, neighbor := range g.adj[node] {
            if dist[neighbor] == -1 { // 未访问
                dist[neighbor] = dist[node] + 1
                queue = append(queue, neighbor)
            }
        }
    }
    return dist
}
```

```python
# Python BFS 最短路模板
from collections import deque

def bfs_shortest(n, graph, start):
    dist = [-1] * n
    dist[start] = 0
    q = deque([start])

    while q:
        node = q.popleft()
        for neighbor in graph.adj[node]:
            if dist[neighbor] == -1:
                dist[neighbor] = dist[node] + 1
                q.append(neighbor)
    
    return dist
```

---

### 4. 并查集模板（Union Find）

**适用：** 连通分量、等价类分组、冗余连接、岛屿数量（替代 DFS/BFS）

#### Go 实现

```go
// 并查集 Go 实现（路径压缩 + 按秩合并）
type UnionFind struct {
    parent []int
    rank   []int
    count  int // 连通分量数
}

func NewUnionFind(n int) *UnionFind {
    parent := make([]int, n)
    rank := make([]int, n)
    for i := range parent {
        parent[i] = i
        rank[i] = 1
    }
    return &UnionFind{parent: parent, rank: rank, count: n}
}

// 路径压缩（递归）
func (uf *UnionFind) find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.find(uf.parent[x]) // 路径压缩
    }
    return uf.parent[x]
}

// 按秩合并
func (uf *UnionFind) union(x, y int) bool {
    rx, ry := uf.find(x), uf.find(y)
    if rx == ry {
        return false // 已连通
    }

    // 小秩挂到大秩下面
    if uf.rank[rx] < uf.rank[ry] {
        uf.parent[rx] = ry
    } else if uf.rank[rx] > uf.rank[ry] {
        uf.parent[ry] = rx
    } else {
        uf.parent[ry] = rx
        uf.rank[rx]++
    }
    uf.count--
    return true
}

func (uf *UnionFind) connected(x, y int) bool {
    return uf.find(x) == uf.find(y)
}
```

#### Python 实现

```python
# 并查集 Python 实现
class UnionFind:
    def __init__(self, n: int):
        self.parent = list(range(n))
        self.rank = [1] * n
        self.count = n

    def find(self, x: int) -> int:
        # 路径压缩（迭代）
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]  # 压缩一层
            x = self.parent[x]
        return x

    def union(self, x: int, y: int) -> bool:
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return False

        # 按秩合并
        if self.rank[rx] < self.rank[ry]:
            self.parent[rx] = ry
        elif self.rank[rx] > self.rank[ry]:
            self.parent[ry] = rx
        else:
            self.parent[ry] = rx
            self.rank[rx] += 1

        self.count -= 1
        return True

    def connected(self, x: int, y: int) -> bool:
        return self.find(x) == self.find(y)
```

---

## 高频真题一：LeetCode 200 — 岛屿数量

**题意：** '1' 是陆地，'0' 是水，统计独立岛屿数量。

```go
// Go DFS 版本（击败 98%）
func numIslands(grid [][]byte) int {
    if len(grid) == 0 || len(grid[0]) == 0 {
        return 0
    }
    m, n := len(grid), len(grid[0])
    count := 0

    var dfs func(i, j int)
    dfs = func(i, j int) {
        if i < 0 || i >= m || j < 0 || j >= n || grid[i][j] == '0' {
            return
        }
        grid[i][j] = '0' // 沉没陆地，避免重复访问
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                count++
                dfs(i, j)
            }
        }
    }
    return count
}

// Go 并查集版本
func numIslandsUF(grid [][]byte) int {
    if len(grid) == 0 || len(grid[0]) == 0 { return 0 }
    m, n := len(grid), len(grid[0])
    uf := NewUnionFind(m * n)

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                idx := i*n + j
                if i+1 < m && grid[i+1][j] == '1' {
                    uf.union(idx, (i+1)*n+j)
                }
                if j+1 < n && grid[i][j+1] == '1' {
                    uf.union(idx, i*n+(j+1))
                }
            }
        }
    }

    // 统计根节点数量（每个根 = 一个岛屿）
    roots := make(map[int]bool)
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                roots[uf.find(i*n+j)] = true
            }
        }
    }
    return len(roots)
}
```

```python
# Python DFS 版本
def numIslands(grid):
    if not grid or not grid[0]:
        return 0
    m, n = len(grid), len(grid[0])
    count = 0

    def dfs(i, j):
        if not (0 <= i < m and 0 <= j < n) or grid[i][j] == '0':
            return
        grid[i][j] = '0'  # 沉没
        dfs(i+1, j); dfs(i-1, j)
        dfs(i, j+1); dfs(i, j-1)

    for i in range(m):
        for j in range(n):
            if grid[i][j] == '1':
                count += 1
                dfs(i, j)
    return count

# Python 并查集版本
def numIslandsUF(grid):
    if not grid or not grid[0]:
        return 0
    m, n = len(grid), len(grid[0])
    uf = UnionFind(m * n)

    for i in range(m):
        for j in range(n):
            if grid[i][j] == '1':
                idx = i * n + j
                if i + 1 < m and grid[i+1][j] == '1':
                    uf.union(idx, (i+1)*n + j)
                if j + 1 < n and grid[i][j+1] == '1':
                    uf.union(idx, i*n + (j+1))

    roots = set()
    for i in range(m):
        for j in range(n):
            if grid[i][j] == '1':
                roots.add(uf.find(i*n + j))
    return len(roots)
```

**复杂度：** 时间 O(m×n)，空间 O(m×n)（递归栈或 UF 数组）。

---

## 高频真题二：LeetCode 207 — 课程表（拓扑排序 / 环检测）

**题意：** 判断有向图中是否有环（课程 prerequisite 是否矛盾）。

```go
// Go 拓扑排序（Kahn 算法 + BFS）
func canFinish(numCourses int, prerequisites [][]int) bool {
    // 构建入度表和邻接表
    inDegree := make([]int, numCourses)
    adj := make([][]int, numCourses)

    for _, pre := range prerequisites {
        adj[pre[1]] = append(adj[pre[1]], pre[0]) // pre[1] -> pre[0]
        inDegree[pre[0]]++
    }

    // BFS：从入度为 0 的节点开始（无前置课程的课）
    queue := make([]int, 0)
    for i := 0; i < numCourses; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    visited := 0
    for len(queue) > 0 {
        course := queue[0]
        queue = queue[1:]
        visited++

        for _, next := range adj[course] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }
    return visited == numCourses // 如果所有课都能被访问到，说明无环
}
```

```python
# Python 拓扑排序
from collections import deque

def canFinish(numCourses, prerequisites):
    in_degree = [0] * numCourses
    adj = [[] for _ in range(numCourses)]

    for u, v in prerequisites:
        adj[v].append(u)
        in_degree[u] += 1

    q = deque([i for i in range(numCourses) if in_degree[i] == 0])
    visited = 0

    while q:
        course = q.popleft()
        visited += 1
        for nxt in adj[course]:
            in_degree[nxt] -= 1
            if in_degree[nxt] == 0:
                q.append(nxt)

    return visited == numCourses
```

**追问：Kahn 算法 vs DFS 环检测**

```go
// Go DFS 版本（另一种思路）
func canFinishDFS(numCourses int, prerequisites [][]int) bool {
    adj := make([][]int, numCourses)
    for _, pre := range prerequisites {
        adj[pre[1]] = append(adj[pre[1]], pre[0])
    }

    // 0=未访问 1=访问中 2=已完成
    state := make([]int, numCourses)

    var hasCycle func(int) bool
    hasCycle = func(u int) bool {
        if state[u] == 1 { return true }  // 发现环
        if state[u] == 2 { return false } // 已完成

        state[u] = 1 // 标记为访问中
        for _, v := range adj[u] {
            if hasCycle(v) {
                return true
            }
        }
        state[u] = 2 // 标记为已完成
        return false
    }

    for i := 0; i < numCourses; i++ {
        if hasCycle(i) {
            return false
        }
    }
    return true
}
```

**复杂度：** 时间 O(V+E)，空间 O(V+E)。

---

## 高频真题三：LeetCode 684 — 冗余连接（并查集）

**题意：** 在树中加一条边变成有环图，找到那条被加入后会形成环的边。

```go
// Go 并查集解法
func findRedundantConnection(edges [][]int) []int {
    uf := NewUnionFind(len(edges) + 1)

    for _, edge := range edges {
        u, v := edge[0], edge[1]
        if !uf.union(u, v) {
            return edge // 已经连通，再 union 就成环，返回这条边
        }
    }
    return nil // 不会到这里
}
```

```python
# Python 并查集解法
def findRedundantConnection(edges):
    uf = UnionFind(len(edges) + 1)
    for u, v in edges:
        if not uf.union(u, v):
            return [u, v]
    return []
```

**复杂度：** 时间 O(N α(N))，空间 O(N)，α 是 Ackermann 函数的反函数（实际约等于常数）。

---

## 高频追问

**Q：什么时候用 BFS 而不是 DFS？**
> 找最短路时用 BFS（边权为 1 的图）。DFS 找的是一条路径，但不保证最短。BFS 按层次扩展，第一次到达目标节点的层数就是最短距离。

**Q：并查集的时间复杂度为什么是 O(α(N))？**
> 因为用了路径压缩 + 按秩合并。α(N) 是 Ackermann 函数的反函数，对于 N < 2^65536（宇宙中原子数量级），α(N) ≤ 4。所以并查集几乎就是 O(1) 的 union/find。

**Q：DFS 递归会爆栈吗？怎么改？**
> 会。当图深度很大（如链状图 10⁵ 层）时递归会导致 stack overflow。改用**显式栈**（自己维护 stack 数组）或**BFS**。Go 的 goroutine 栈虽然会动态增长，但递归过深仍是危险的。

**Q：邻接表遍历的时间复杂度？**
> O(V+E)，因为每个顶点访问一次、每条边访问两次（无向图）。

---

## 延伸阅读

| 资源 | 链接 |
|------|------|
| LeetCode 图论 Tag | https://leetcode.cn/tag/graph/ |
| 并查集详解 | https://zhuanlan.zhihu.com/p/93613606 |
| 岛屿问题总结 | https://leetcode.cn/problems/number-of-islands/solution/ |
| 拓扑排序 Kahn 算法 | https://zh.wikipedia.org/wiki/拓扑排序 |
