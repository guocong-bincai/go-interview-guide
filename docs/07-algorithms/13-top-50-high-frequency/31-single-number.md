# 31. 只出现一次的数字（Single Number）

> 频率：★★★★☆  难度：简单  LeetCode 136

## 小白先理解
数组里除了一个数字只出现一次，其他数字都出现两次。
问你怎么找出那个只出现一次的数字。

比如：
```text
[2,2,1]
```
答案就是 `1`。

## 解法一：哈希表计数
- 统计每个数出现次数
- 找次数为 1 的

## 解法二：异或（推荐）
### 核心思路
异或有两个神奇性质：
- `a ^ a = 0`
- `a ^ 0 = a`

所以成对出现的数字会全部抵消，最后只剩那个单独的数。

### Go
```go
func singleNumber(nums []int) int {
    ans := 0
    for _, num := range nums {
        ans ^= num
    }
    return ans
}
```
### Python
```python
def single_number(nums):
    ans = 0
    for num in nums:
        ans ^= num
    return ans
```

## 一句话记忆
**只出现一次的数字 = 全部异或。**
