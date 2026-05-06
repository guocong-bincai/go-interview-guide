# 31. 只出现一次的数字（Single Number）

> 频率：★★★★☆  难度：简单  LeetCode 136

## 题目描述
给定一个非空整数数组，除了某个元素只出现一次外，其余每个元素均出现两次。找出那个只出现一次的元素。

## 面试为什么爱问
- 这是位运算的**最经典入门题**
- 考察对异或运算的理解：`a^a=0, a^0=a`
- 成对抵消的思想是高频考点，可推广到"找出落单的数"

## 核心考点
- [ ] 异或运算的交换律和结合律
- [ ] 不用额外空间（O(1) 空间）
- [ ] 扩展：如果出现 3 次怎么办

## 小白先理解
**异或就是"相同为 0，不同为 1"**

两个核心性质：
1. `x ^ x = 0` —— 任何数和自身异或等于 0
2. `x ^ 0 = x` —— 任何数和 0 异或等于自身

所以：
```
[2, 1, 4, 2, 1]
= 2^1^4^2^1
= (2^2) ^ (1^1) ^ 4
= 0 ^ 0 ^ 4
= 4
```
所有成双成对的都抵消了，只剩那个孤独的数。

---

## 解法一：异或（推荐）

### 核心思路
遍历数组，全部异或。成对的数全部抵消，剩下就是答案。

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

### 复杂度
- 时间：O(n)
- 空间：O(1)

---

## 解法二：哈希表计数

### Go
```go
func singleNumber_hash(nums []int) int {
    freq := make(map[int]int)
    for _, num := range nums {
        freq[num]++
    }
    for num, count := range freq {
        if count == 1 {
            return num
        }
    }
    return 0 // never reached
}
```

### Python
```python
def single_number_hash(nums):
    from collections import Counter
    counter = Counter(nums)
    for num, count in counter.items():
        if count == 1:
            return num
```

---

## 两种解法对比
| 维度 | 异或 | 哈希表 |
|------|------|--------|
| 时间复杂度 | O(n) | O(n) |
| 空间复杂度 | **O(1)** ✅ | O(n) |
| 面试推荐 | **优先** | 备选 |

---

## 易错点
| 错误 | 问题 |
|------|------|
| 用 `+` `-` 代替异或 | 无法处理负数 |
| 不理解异或可交换 | `2^1^4` = `1^2^4` = 任意顺序都可以 |
| 数组为空 | 题目保证非空 |

---

## 面试追问

**Q：进阶版——除了一个数出现一次，其他都出现三次，怎么找？**
用位运算按位统计。每一位上，如果出现 3 次的和 mod 3 等于 0，结果那一位就是 0，否则是 1。

**Q：如果有两个数只出现一次，其他都出现两次，怎么找？**
先全员异或得到 `a^b`，然后找到 `a^b` 二进制中任意一个 1 的位置（假设是第 k 位），按第 k 位是 0 还是 1 把数组分成两组，分别异或就得到两个数。

**Q：为什么不用排序？因为排序是 O(n log n)**
排序可以，但慢一个数量级。异或是 O(n) 最优解。

---

## 一句话记忆
**只出现一次的数字 = 全部异或，利用 a^a=0 和 a^0=a 的性质，成对抵消。**