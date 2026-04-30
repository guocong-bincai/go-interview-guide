# 18. 买卖股票最佳时机（Best Time to Buy and Sell Stock）

> 频率：★★★★☆  难度：简单  LeetCode 121

## 题目描述
给定一个数组 `prices`，其中 `prices[i]` 表示某支股票第 i 天的价格。
你只能选择某一天买入这只股票，并选择在未来某一天卖出。请你计算你能获得的最大利润。

## 小白先理解
本质就是：
**前面尽量低价买，后面尽量高价卖。**

但要注意：
- 买必须在卖前面

---

## 解法一：暴力枚举

### 思路
每一天都当作买入日，再去后面找最好的卖出日。

### 复杂度
- 时间：`O(n^2)`

---

## 解法二：维护最低价格（推荐）

### 核心思路
遍历数组时：
- 记录到目前为止最便宜的价格 `minPrice`
- 当前价格如果卖出，利润就是 `price - minPrice`
- 取最大值

### Go
```go
func maxProfit(prices []int) int {
    minPrice := prices[0]
    ans := 0

    for _, price := range prices {
        if price < minPrice {
            minPrice = price
        }
        if price-minPrice > ans {
            ans = price - minPrice
        }
    }
    return ans
}
```

### Python
```python
def max_profit(prices):
    min_price = prices[0]
    ans = 0

    for price in prices:
        min_price = min(min_price, price)
        ans = max(ans, price - min_price)

    return ans
```

## 一句话记忆
**股票题核心：边遍历边维护历史最低价。**
