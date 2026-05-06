# 11. 课程表（Course Schedule）

> 频率：★★★★★  难度：中等  LeetCode 207

## 题目描述
有 `numCourses` 门课程，编号从 `0` 到 `numCourses-1`。

给你一个数组 `prerequisites`，其中 `prerequisites[i] = [a, b]` 表示：
想学课程 `a`，必须先学课程 `b`。

请你判断：是否可能完成所有课程。

## 小白先理解
比如：

```text
0 <- 1
```

表示学 1 之前要先学 0，这是可以完成的。

但如果是：

```text
0 <- 1 <- 2 <- 0
```

就形成了一个圈，谁都得先学别人，永远学不完。

所以这题本质就是：
**判断有向图里有没有环。**

---

## 解法一：DFS 判环

### 思路
每个节点有三种状态：
- 0：没访问过
- 1：正在访问
- 2：访问完成

如果 DFS 时遇到一个“正在访问”的节点，说明有环。

### Go
```go
func canFinishDFS(numCourses int, prerequisites [][]int) bool {
    graph := make([][]int, numCourses)
    for _, p := range prerequisites {
        graph[p[1]] = append(graph[p[1]], p[0])
    }

    visited := make([]int, numCourses)
    var dfs func(int) bool
    dfs = func(node int) bool {
        if visited[node] == 1 {
            return false
        }
        if visited[node] == 2 {
            return true
        }
        visited[node] = 1
        for _, next := range graph[node] {
            if !dfs(next) {
                return false
            }
        }
        visited[node] = 2
        return true
    }

    for i := 0; i < numCourses; i++ {
        if !dfs(i) {
            return false
        }
    }
    return true
}
```

### Python
```python
def can_finish_dfs(numCourses, prerequisites):
    graph = [[] for _ in range(numCourses)]
    for a, b in prerequisites:
        graph[b].append(a)

    visited = [0] * numCourses

    def dfs(node):
        if visited[node] == 1:
            return False
        if visited[node] == 2:
            return True
        visited[node] = 1
        for nxt in graph[node]:
            if not dfs(nxt):
                return False
        visited[node] = 2
        return True

    for i in range(numCourses):
        if not dfs(i):
            return False
    return True
```

---

## 解法二：BFS 拓扑排序（推荐）

### 核心思路
如果一个图没有环，就一定能找到“入度为 0”的点。

步骤：
1. 统计每门课的入度
2. 把所有入度为 0 的课放进队列
3. 每次弹出一门课，相当于“学完它”
4. 它后面的课程入度减 1
5. 如果最后学完了所有课，说明没有环

### Go
```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    graph := make([][]int, numCourses)
    indegree := make([]int, numCourses)

    for _, p := range prerequisites {
        a, b := p[0], p[1]
        graph[b] = append(graph[b], a)
        indegree[a]++
    }

    queue := []int{}
    for i := 0; i < numCourses; i++ {
        if indegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    count := 0
    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]
        count++
        for _, next := range graph[cur] {
            indegree[next]--
            if indegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    return count == numCourses
}
```

### Python
```python
from collections import deque

def can_finish(numCourses, prerequisites):
    graph = [[] for _ in range(numCourses)]
    indegree = [0] * numCourses

    for a, b in prerequisites:
        graph[b].append(a)
        indegree[a] += 1

    queue = deque([i for i in range(numCourses) if indegree[i] == 0])
    count = 0

    while queue:
        cur = queue.popleft()
        count += 1
        for nxt in graph[cur]:
            indegree[nxt] -= 1
            if indegree[nxt] == 0:
                queue.append(nxt)

    return count == numCourses
```

## 一句话记忆
**课程表 = 判断图里有没有环；推荐用拓扑排序。**
