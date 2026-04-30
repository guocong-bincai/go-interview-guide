# 37. 搜索二维矩阵（Search a 2D Matrix）

> 频率：★★★★☆  难度：中等  LeetCode 74

## 小白先理解
矩阵满足：
- 每行升序
- 每行第一个数大于上一行最后一个数

所以整个矩阵其实可以看成一个“拉平后有序的一维数组”。

## 解法一：先找行再找列
- 先确定目标在哪一行
- 再在那一行里二分

## 解法二：整体二分（推荐）
### Go
```go
func searchMatrix(matrix [][]int, target int) bool {
    rows, cols := len(matrix), len(matrix[0])
    left, right := 0, rows*cols-1
    for left <= right {
        mid := left + (right-left)/2
        val := matrix[mid/cols][mid%cols]
        if val == target { return true }
        if val < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return false
}
```
### Python
```python
def search_matrix(matrix, target):
    rows, cols = len(matrix), len(matrix[0])
    left, right = 0, rows * cols - 1
    while left <= right:
        mid = (left + right) // 2
        val = matrix[mid // cols][mid % cols]
        if val == target:
            return True
        elif val < target:
            left = mid + 1
        else:
            right = mid - 1
    return False
```

## 一句话记忆
**二维矩阵整体有序时，可以当一维数组二分。**
