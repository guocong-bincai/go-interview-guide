# 49. Pow(x, n)

> 频率：★★★☆☆  难度：中等  LeetCode 50

## 小白先理解
要求计算 `x` 的 `n` 次方。
如果直接乘 n 次，也能做，但太慢。

## 解法一：直接循环乘
- 时间 `O(n)`

## 解法二：快速幂（推荐）
### 核心思路
利用：
- `x^8 = (x^4)^2`
- `x^5 = x * x^4`

每次把指数减半，所以特别快。

### Go
```go
func myPow(x float64, n int) float64 {
    N := n
    if N < 0 {
        x = 1 / x
        N = -N
    }
    ans := 1.0
    for N > 0 {
        if N%2 == 1 {
            ans *= x
        }
        x *= x
        N /= 2
    }
    return ans
}
```
### Python
```python
def my_pow(x, n):
    if n < 0:
        x = 1 / x
        n = -n
    ans = 1.0
    while n > 0:
        if n % 2 == 1:
            ans *= x
        x *= x
        n //= 2
    return ans
```

## 一句话记忆
**Pow(x,n) = 快速幂，每次把指数砍半。**
