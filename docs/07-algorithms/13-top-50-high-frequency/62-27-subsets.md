# 27. 子集（Subsets）

> 频率：★★★★☆  难度：中等  LeetCode 78

## 题目描述
给你一个整数数组 `nums`，返回该数组所有可能的子集（幂集）。

## 面试为什么爱问
- 这是回溯/DFS 的入门经典题
- 子集树 vs 排列树的典型代表
- 面试官会追问：和组合有什么区别？怎么去重？

## 核心考点
- [ ] 组合型回溯（从前往后选）
- [ ] 二叉树思维：每个元素有"选/不选"两条路
- [ ] 树层 vs 树枝去重

## 小白先理解
**本质：每个元素都有两种选择——选或不选**

`[1,2,3]` 的子集枚举过程像一棵二叉树：
```
                  []           ← 一开始什么也没选
          /               \
        [1]               []     ← 1 选或不选
      /     \           /     \
   [1,2]   [1]       [2]      []
  /    \    |         |       |
[1,2,3][1,2] [1,3][1] [2,3][2] [3][]
```
最终所有叶子节点就是所有子集。

**为什么要"从 start 开始选"？** 因为组合不考虑顺序，`[1,2]` 和 `[2,1]` 是同一个组合，所以选过的索引不再回头。

---

## 解法一：回溯枚举（推荐）

### 核心思路
从索引 0 开始，每个位置尝试选或不选。
- 选：加入 path，递归下一位
- 不选：直接递归下一位（path 不变）

### Go
```go
func subsets(nums []int) [][]int {
    res := [][]int{}
    path := []int{}

    var backtrack func(start int)
    backtrack = func(start int) {
        // 每进入一次递归，都生成一个新子集（因为每个节点都是一个子集）
        tmp := make([]int, len(path))
        copy(tmp, path)
        res = append(res, tmp)

        // 从 start 开始选，避免产生重复组合
        for i := start; i < len(nums); i++ {
            path = append(path, nums[i])   // 选 nums[i]
            backtrack(i + 1)               // 递归选下一个
            path = path[:len(path)-1]      // 不选 nums[i]，回溯
        }
    }

    backtrack(0)
    return res
}
```

### Python
```python
def subsets(nums):
    res = []
    path = []

    def backtrack(start):
        res.append(path[:])  # 记录当前子集
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1)
            path.pop()

    backtrack(0)
    return res
```

### 复杂度
- 时间：O(n × 2ⁿ)，每个子集生成需要 O(n) 拷贝
- 空间：O(n)，递归栈深度 + path 空间

---

## 解法二：位运算迭代

### 核心思路
有 n 个元素，就有 2ⁿ 个子集。每个子集对应一个 n 位二进制数。

### Go
```go
func subsets_bit(nums []int) [][]int {
    n := len(nums)
    total := 1 << n
    res := make([][]int, 0, total)

    for mask := 0; mask < total; mask++ {
        set := []int{}
        for i := 0; i < n; i++ {
            if mask&(1<<i) != 0 {  // 第 i 位是 1，说明选 nums[i]
                set = append(set, nums[i])
            }
        }
        res = append(res, set)
    }
    return res
}
```

### Python
```python
def subsets_bit(nums):
    n = len(nums)
    total = 1 << n
    res = []
    for mask in range(total):
        s = []
        for i in range(n):
            if mask & (1 << i):
                s.append(nums[i])
        res.append(s)
    return res
```

---

## 两种解法对比
| 维度 | 回溯 | 位运算 |
|------|------|--------|
| 时间复杂度 | O(n × 2ⁿ) | O(n × 2ⁿ) |
| 空间复杂度 | O(n) 递归栈 | O(1) |
| 适合场景 | 面试手写 | 快速枚举 |

---

## 易错点
| 错误 | 问题 |
|------|------|
| `res.append(path)` 直接引用 | path 后续会变，结果全错 |
| 递归时从 0 开始 | 产生重复排列 |
| 不理解"组合 vs 排列" | 子集是组合，不考虑顺序 |

---

## 面试追问

**Q：和组合总和（Combination Sum）有什么区别？**
子集是每个元素只能选一次；组合总和每个元素可以重复选（`start` 参数不变），还可以加目标和判断提前终止。

**Q：如何去重（比如 `nums = [1,2,2]`，子集不能重复）？**
排序后，在 for 循环里加 `if i > start && nums[i] == nums[i-1] { continue }` 跳过同一层重复元素。

**Q：子集 II（LeetCode 90）怎么做？**
在子集基础上，排序 + 树层去重。同一位如果和前一个值相同就跳过。

---

## 一句话记忆
**子集 = 每个元素选或不选，回溯从 start 开始遍历，避免产生重复组合。**