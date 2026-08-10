# 33. 腐烂的橘子（Rotting Oranges）

> 频率：★★★★☆  难度：中等  LeetCode 994

## 小白先理解
坏橘子每过一分钟，会把四周的新鲜橘子传染坏。
问最少多少分钟后，所有橘子都坏掉；如果做不到，返回 -1。

这题本质是：
**从多个起点同时做 BFS，按层扩散。**

## 解法一：模拟每一分钟全图扫描
- 每分钟都把整张图扫一遍
- 很慢

## 解法二：多源 BFS（推荐）
### Go
```go
func orangesRotting(grid [][]int) int {
    rows, cols := len(grid), len(grid[0])
    queue := [][2]int{}
    fresh := 0
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == 2 {
                queue = append(queue, [2]int{i, j})
            } else if grid[i][j] == 1 {
                fresh++
            }
        }
    }
    if fresh == 0 { return 0 }
    dirs := [][2]int{{1,0},{-1,0},{0,1},{0,-1}}
    minutes := 0
    for len(queue) > 0 {
        size := len(queue)
        changed := false
        for i := 0; i < size; i++ {
            x, y := queue[0][0], queue[0][1]
            queue = queue[1:]
            for _, d := range dirs {
                nx, ny := x+d[0], y+d[1]
                if nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] == 1 {
                    grid[nx][ny] = 2
                    fresh--
                    changed = true
                    queue = append(queue, [2]int{nx, ny})
                }
            }
        }
        if changed { minutes++ }
    }
    if fresh > 0 { return -1 }
    return minutes
}
```
### Python
```python
from collections import deque

def oranges_rotting(grid):
    rows, cols = len(grid), len(grid[0])
    queue = deque()
    fresh = 0
    for i in range(rows):
        for j in range(cols):
            if grid[i][j] == 2:
                queue.append((i, j))
            elif grid[i][j] == 1:
                fresh += 1
    if fresh == 0:
        return 0
    minutes = 0
    dirs = [(1,0),(-1,0),(0,1),(0,-1)]
    while queue:
        size = len(queue)
        changed = False
        for _ in range(size):
            x, y = queue.popleft()
            for dx, dy in dirs:
                nx, ny = x+dx, y+dy
                if 0 <= nx < rows and 0 <= ny < cols and grid[nx][ny] == 1:
                    grid[nx][ny] = 2
                    fresh -= 1
                    changed = True
                    queue.append((nx, ny))
        if changed:
            minutes += 1
    return -1 if fresh > 0 else minutes
```

## 一句话记忆
**腐烂的橘子 = 多源 BFS 按层扩散。**
