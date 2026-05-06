# 14. 跳跃游戏（Jump Game）

> 频率：★★★★★  难度：中等  LeetCode 55

## 题目描述
给你一个非负整数数组 `nums`，你最初位于数组的第一个下标。
数组中的每个元素代表你在该位置可以跳跃的最大长度。
请判断你是否能够到达最后一个下标。

## 小白先理解
比如：

```text
[2,3,1,1,4]
```

- 在位置 0，最多跳 2 步
- 继续往后，总能跳到最后

所以答案是 `true`。

而：

```text
[3,2,1,0,4]
```

跳到下标 3 时，那里是 `0`，后面过不去了。

---

## 解法一：DP

### 思路
用一个数组记录每个位置能不能到达。

### 问题
能做，但是没必要，太慢。

---

## 解法二：贪心（推荐）

### 核心思路
维护当前能到达的最远位置 `maxReach`。

遍历数组：
- 如果当前位置已经超过 `maxReach`，说明根本到不了这里，返回 false
- 否则更新最远位置：`max(maxReach, i + nums[i])`

### Go
```go
func canJump(nums []int) bool {
    maxReach := 0
    for i := 0; i < len(nums); i++ {
        if i > maxReach {
            return false
        }
        if i+nums[i] > maxReach {
            maxReach = i + nums[i]
        }
    }
    return true
}
```

### Python
```python
def can_jump(nums):
    max_reach = 0
    for i, num in enumerate(nums):
        if i > max_reach:
            return False
        max_reach = max(max_reach, i + num)
    return True
```

## 一句话记忆
**跳跃游戏看的是“最远能到哪”，不是每一步怎么跳。**
