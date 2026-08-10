# 16. Top K 高频元素（Top K Frequent Elements）

> 频率：★★★★★  难度：中等  LeetCode 347

## 题目描述
给你一个整数数组 `nums` 和一个整数 `k`，请你返回其中出现频率前 `k` 高的元素。

## 小白先理解
先数出每个数字出现了几次，然后再找出出现次数最多的前 k 个。

所以这题自然分两步：
1. 统计频率
2. 找 TopK

---

## 解法一：哈希 + 排序

### 思路
- 用哈希表统计次数
- 转成数组后按频率排序
- 取前 k 个

### Go
```go
import "sort"

func topKFrequentSort(nums []int, k int) []int {
    freq := map[int]int{}
    for _, num := range nums {
        freq[num]++
    }

    arr := make([][2]int, 0, len(freq))
    for num, cnt := range freq {
        arr = append(arr, [2]int{num, cnt})
    }

    sort.Slice(arr, func(i, j int) bool {
        return arr[i][1] > arr[j][1]
    })

    res := []int{}
    for i := 0; i < k; i++ {
        res = append(res, arr[i][0])
    }
    return res
}
```

### Python
```python
def top_k_frequent_sort(nums, k):
    freq = {}
    for num in nums:
        freq[num] = freq.get(num, 0) + 1

    arr = sorted(freq.items(), key=lambda x: x[1], reverse=True)
    return [arr[i][0] for i in range(k)]
```

---

## 解法二：最小堆（推荐）

### 核心思路
用一个大小为 `k` 的最小堆：
- 堆里只保留当前前 k 高频元素
- 如果新元素频率更高，就把堆顶弹掉

### 复杂度
比全排序更适合 TopK 问题。

### 一句话理解
不是把所有人都排一遍名次，而是只维护“前 k 名候选人”。

---

## 一句话记忆
**TopK 问题：先统计，再考虑堆。**
