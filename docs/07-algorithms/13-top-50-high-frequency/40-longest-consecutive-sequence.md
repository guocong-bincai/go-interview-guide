# 40. 最长连续序列（Longest Consecutive Sequence）

> 频率：★★★☆☆  难度：中等  LeetCode 128

## 小白先理解
给定一个无序数组，找到数字连续的最长序列长度。
比如 `[100,4,200,1,3,2]` 中，最长连续序列是 `[1,2,3,4]`，长度 4。

## 解法一：排序
- 排序后线性扫描
- 简单但不是最优

## 解法二：哈希集合（推荐）
### 核心思路
把所有数放进集合里。
如果一个数 `x-1` 不在集合中，说明它可能是连续序列起点。
然后从它往后一直找 `x+1, x+2...`。

### Go
```go
func longestConsecutive(nums []int) int {
    set := map[int]bool{}
    for _, num := range nums { set[num] = true }
    ans := 0
    for num := range set {
        if !set[num-1] {
            cur := num
            length := 1
            for set[cur+1] {
                cur++
                length++
            }
            if length > ans { ans = length }
        }
    }
    return ans
}
```
### Python
```python
def longest_consecutive(nums):
    s = set(nums)
    ans = 0
    for num in s:
        if num - 1 not in s:
            cur = num
            length = 1
            while cur + 1 in s:
                cur += 1
                length += 1
            ans = max(ans, length)
    return ans
```

## 一句话记忆
**最长连续序列 = 只从“起点”开始往后扩。**
