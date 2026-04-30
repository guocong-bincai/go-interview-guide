# 24. 螺旋矩阵（Spiral Matrix）

> 频率：★★★★☆  难度：中等  LeetCode 54

## 小白先理解
按“右、下、左、上”一圈一圈地把矩阵走完。
核心是维护四条边界：上、下、左、右。

## 解法一：visited 数组模拟走路
- 每走一步标记已访问
- 简单但额外空间大

## 解法二：四边界收缩（推荐）
### Go
```go
func spiralOrder(matrix [][]int) []int {
    if len(matrix) == 0 { return []int{} }
    top, bottom := 0, len(matrix)-1
    left, right := 0, len(matrix[0])-1
    res := []int{}
    for top <= bottom && left <= right {
        for j := left; j <= right; j++ { res = append(res, matrix[top][j]) }
        top++
        for i := top; i <= bottom; i++ { res = append(res, matrix[i][right]) }
        right--
        if top <= bottom {
            for j := right; j >= left; j-- { res = append(res, matrix[bottom][j]) }
            bottom--
        }
        if left <= right {
            for i := bottom; i >= top; i-- { res = append(res, matrix[i][left]) }
            left++
        }
    }
    return res
}
```
### Python
```python
def spiral_order(matrix):
    if not matrix:
        return []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1
    res = []
    while top <= bottom and left <= right:
        for j in range(left, right + 1): res.append(matrix[top][j])
        top += 1
        for i in range(top, bottom + 1): res.append(matrix[i][right])
        right -= 1
        if top <= bottom:
            for j in range(right, left - 1, -1): res.append(matrix[bottom][j])
            bottom -= 1
        if left <= right:
            for i in range(bottom, top - 1, -1): res.append(matrix[i][left])
            left += 1
    return res
```

## 一句话记忆
**螺旋矩阵 = 四条边界不断往里收。**
