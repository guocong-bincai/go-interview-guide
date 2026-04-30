# 50. 螺旋矩阵 II（Spiral Matrix II）

> 频率：★★★☆☆  难度：中等  LeetCode 59

## 小白先理解
和“螺旋矩阵”相反，这题不是读矩阵，而是生成一个 `n x n` 的矩阵，把数字 1 到 n² 按螺旋顺序填进去。

## 解法一：模拟 + visited
- 边走边填
- 用 visited 记录

## 解法二：四边界填充（推荐）
### 核心思路
维护上、下、左、右四条边界，按右、下、左、上的顺序一圈一圈填数字。

### Go
```go
func generateMatrix(n int) [][]int {
    matrix := make([][]int, n)
    for i := range matrix { matrix[i] = make([]int, n) }
    top, bottom := 0, n-1
    left, right := 0, n-1
    num := 1
    for top <= bottom && left <= right {
        for j := left; j <= right; j++ { matrix[top][j] = num; num++ }
        top++
        for i := top; i <= bottom; i++ { matrix[i][right] = num; num++ }
        right--
        for j := right; j >= left && top <= bottom; j-- { matrix[bottom][j] = num; num++ }
        bottom--
        for i := bottom; i >= top && left <= right; i-- { matrix[i][left] = num; num++ }
        left++
    }
    return matrix
}
```
### Python
```python
def generate_matrix(n):
    matrix = [[0] * n for _ in range(n)]
    top, bottom = 0, n - 1
    left, right = 0, n - 1
    num = 1
    while top <= bottom and left <= right:
        for j in range(left, right + 1):
            matrix[top][j] = num
            num += 1
        top += 1
        for i in range(top, bottom + 1):
            matrix[i][right] = num
            num += 1
        right -= 1
        if top <= bottom:
            for j in range(right, left - 1, -1):
                matrix[bottom][j] = num
                num += 1
            bottom -= 1
        if left <= right:
            for i in range(bottom, top - 1, -1):
                matrix[i][left] = num
                num += 1
            left += 1
    return matrix
```

## 一句话记忆
**螺旋矩阵 II = 四边界一圈一圈填数字。**
