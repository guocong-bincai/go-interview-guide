# 位运算高频题

> 考察频率：★★★★☆  优先级：P0  Go + Python 双语

## TODO（待填写）

## 1. 基础技巧（必背）
- [ ] `n & (n-1)`：消去最低位的 1（统计位数/判断 2 的幂）
- [ ] `n & (-n)`：取最低位的 1（lowbit，树状数组用）
- [ ] `a ^ a = 0`，`a ^ 0 = a`：异或自消性
- [ ] Go 中 `^x` 是按位取反（C 语言中是 `~x`）
- [ ] Go 没有 `>>>` 无符号右移，用 `uint(x) >> 1`

## 2. 高频题

### LeetCode 136 - 只出现一次的数字 I
- [ ] 所有数异或，成对的消掉，剩下单个
- [ ] Go + Python 代码

### LeetCode 137 - 只出现一次的数字 II
- [ ] 其他数出现 3 次，统计每一位上 1 的个数 mod 3
- [ ] 进阶：有限状态机（ones/twos）
- [ ] Go + Python 代码

### LeetCode 260 - 只出现一次的数字 III
- [ ] 两个单独数字，用异或分组
- [ ] Go + Python 代码

### LeetCode 201 - 数字范围按位与
- [ ] Brian Kernighan：不断消去 n 最低位 1，直到 m <= n
- [ ] Go + Python 代码

### LeetCode 191 - 位 1 的个数（汉明重量）
- [ ] `n & (n-1)` 循环 / `bits.OnesCount`
- [ ] Go + Python 代码

### LeetCode 338 - 比特位计数
- [ ] DP：`dp[i] = dp[i >> 1] + (i & 1)`
- [ ] Go + Python 代码

### LeetCode 190 - 颠倒二进制位
- [ ] 逐位读取，逆向写入
- [ ] Go + Python 代码

## 3. 位运算在工程中的应用
- [ ] 权限系统：位掩码表示权限（READ=1, WRITE=2, EXEC=4）
- [ ] 布隆过滤器底层：位数组操作
- [ ] Redis `BITSET` / `BITCOUNT`：用户签到统计
- [ ] Go `math/bits` 标准库：`bits.OnesCount64` 等

## 高频追问
- [ ] 为什么 `n & (n-1)` 能消去最低位的 1？从二进制补码角度解释。
- [ ] 判断一个数是否是 2 的幂，有几种方法？哪种最优？
- [ ] Go 中如何安全地做无符号右移？
