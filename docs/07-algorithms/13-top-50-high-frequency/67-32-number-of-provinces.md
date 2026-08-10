# 32. 省份数量（Number of Provinces）

> 频率：★★★★☆  难度：中等  LeetCode 547

## 小白先理解
城市之间有的互相连通，直接连也算，间接连也算。
问最后一共能分成几个连通块。

这题本质就是：
**数图里有几个连通分量。**

## 解法一：DFS
### Go
```go
func findCircleNum(isConnected [][]int) int {
    n := len(isConnected)
    visited := make([]bool, n)
    var dfs func(int)
    dfs = func(i int) {
        for j := 0; j < n; j++ {
            if isConnected[i][j] == 1 && !visited[j] {
                visited[j] = true
                dfs(j)
            }
        }
    }
    ans := 0
    for i := 0; i < n; i++ {
        if !visited[i] {
            visited[i] = true
            dfs(i)
            ans++
        }
    }
    return ans
}
```
### Python
```python
def find_circle_num(isConnected):
    n = len(isConnected)
    visited = [False] * n

    def dfs(i):
        for j in range(n):
            if isConnected[i][j] == 1 and not visited[j]:
                visited[j] = True
                dfs(j)

    ans = 0
    for i in range(n):
        if not visited[i]:
            visited[i] = True
            dfs(i)
            ans += 1
    return ans
```

## 解法二：并查集
- 把连通的城市合并到一个集合里
- 最后看有多少个根节点

## 一句话记忆
**省份数量 = 连通分量个数。**
