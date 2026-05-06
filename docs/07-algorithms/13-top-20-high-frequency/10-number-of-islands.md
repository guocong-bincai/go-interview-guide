# 10. 岛屿数量（Number of Islands）

> 频率：★★★★★  难度：中等  LeetCode 200

## 题目描述
给你一个由 `'1'`（陆地）和 `'0'`（水）组成的二维网格，请你计算网格中岛屿的数量。

岛屿总是被水包围，并且每座岛屿只能由水平或垂直方向上相邻的陆地连接形成。

## 小白先理解
这题本质是：
**数一数一共有多少块连在一起的陆地。**

每次你找到一块新陆地，就把和它连成一片的陆地全部“淹没”掉，这样不会重复数。

---

## 解法一：BFS

### 思路
- 遍历整个网格
- 遇到一个 `'1'`，说明发现了一座新岛
- 岛屿数量 +1
- 然后用 BFS 把和它相连的所有 `'1'` 全部访问掉

### Go
```go
func numIslandsBFS(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }
    rows, cols := len(grid), len(grid[0])
    dirs := [][2]int{{1,0},{-1,0},{0,1},{0,-1}}
    ans := 0

    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                ans++
                queue := [][2]int{{i,j}}
                grid[i][j] = '0'
                for len(queue) > 0 {
                    x, y := queue[0][0], queue[0][1]
                    queue = queue[1:]
                    for _, d := range dirs {
                        nx, ny := x+d[0], y+d[1]
                        if nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] == '1' {
                            grid[nx][ny] = '0'
                            queue = append(queue, [2]int{nx, ny})
                        }
                    }
                }
            }
        }
    }
    return ans
}
```

### Python
```python
from collections import deque

def num_islands_bfs(grid):
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    ans = 0
    dirs = [(1,0),(-1,0),(0,1),(0,-1)]

    for i in range(rows):
        for j in range(cols):
            if grid[i][j] == '1':
                ans += 1
                queue = deque([(i, j)])
                grid[i][j] = '0'
                while queue:
                    x, y = queue.popleft()
                    for dx, dy in dirs:
                        nx, ny = x + dx, y + dy
                        if 0 <= nx < rows and 0 <= ny < cols and grid[nx][ny] == '1':
                            grid[nx][ny] = '0'
                            queue.append((nx, ny))
    return ans
```

---

## 解法二：DFS（推荐）

### 核心思路
遇到一个 `'1'`：
- 说明发现一座新岛
- 用 DFS 把这一整片陆地全部标记为 `'0'`
- 以后再遍历到这些格子，就不会重复计数

### Go
```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }

    rows, cols := len(grid), len(grid[0])
    var dfs func(int, int)
    dfs = func(i, j int) {
        if i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] != '1' {
            return
        }
        grid[i][j] = '0'
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }

    ans := 0
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                ans++
                dfs(i, j)
            }
        }
    }
    return ans
}
```

### Python
```python
def num_islands(grid):
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])

    def dfs(i, j):
        if i < 0 or i >= rows or j < 0 or j >= cols or grid[i][j] != '1':
            return
        grid[i][j] = '0'
        dfs(i + 1, j)
        dfs(i - 1, j)
        dfs(i, j + 1)
        dfs(i, j - 1)

    ans = 0
    for i in range(rows):
        for j in range(cols):
            if grid[i][j] == '1':
                ans += 1
                dfs(i, j)

    return ans
```

### 复杂度
- 时间：`O(m*n)`
- 空间：`O(m*n)`（递归栈最坏情况）

---

## 一句话记忆
**数岛屿 = 找到一块陆地就把整座岛淹掉。**
