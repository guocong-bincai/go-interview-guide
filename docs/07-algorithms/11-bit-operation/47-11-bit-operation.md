# 位运算高频题

> 考察频率：★★★★☆  优先级：P0  Go + Python 双语

[🏠 首页](../../../README.md) · [📚 算法模块](../README.md)

---

## 1. 基础技巧（必背）

### `n & (n-1)`：消去最低位的 1

**原理：** 对于任意整数 `n`，`n-1` 会将 `n` 二进制中最右边的 `1` 变成 `0`，并将该位右侧所有 `0` 变成 `1`。两者做 `&` 运算，正好把最低位的 `1` 消掉。

```
例如 n = 12 (1100)
n-1 = 11 (1011)
n & (n-1) = 1000 = 8
```

**核心应用：**

| 应用 | 思路 | 复杂度 |
|------|------|--------|
| 统计 1 的位数 | 每次 `n & (n-1)` 消掉一个 1，计数 | O(k)，k 为 1 的个数 |
| 判断 2 的幂 | 2 的幂只有最高位是 1：`n & (n-1) == 0 && n > 0` | O(1) |

```go
// Go - 统计 1 的个数
func countBits(n int) int {
    count := 0
    for n > 0 {
        n &= n - 1 // 消去最低位的 1
        count++
    }
    return count
}

// Go - 判断 2 的幂
func isPowerOfTwo(n int) bool {
    return n > 0 && n&(n-1) == 0
}
```

```python
# Python - 统计 1 的个数
def count_bits(n: int) -> int:
    count = 0
    while n:
        n &= n - 1
        count += 1
    return count

# Python - 判断 2 的幂
def is_power_of_two(n: int) -> bool:
    return n > 0 and (n & (n - 1)) == 0
```

---

### `n & (-n)`：取最低位的 1（lowbit）

**原理：** `-n` 在二进制中是 `n` 的补码表示。对于正整数 `n`，`-n` = `~n + 1`（按位取反再加 1），这个操作会把最低位的 `1` 及其右侧的所有位全部反转，但左侧不变。因此 `n & (-n)` 正好得到最低位的 `1` 本身。

```
例如 n = 12 (1100)
-n = -12 的补码 = ...0100
n & (-n) = 0100 = 4
```

**核心应用：** 树状数组（Fenwick Tree）的 `lowbit` 操作。

```go
// Go - lowbit
func lowbit(n int) int {
    return n & -n
}

// Go - 基于 lowbit 求 1 的个数
func countBitsLowbit(n int) int {
    count := 0
    for n > 0 {
        n -= lowbit(n) // 去掉最低位的 1
        count++
    }
    return count
}
```

```python
# Python - lowbit
def lowbit(n: int) -> int:
    return n & -n

# Python - 基于 lowbit 求 1 的个数
def count_bits_lowbit(n: int) -> int:
    count = 0
    while n:
        n -= lowbit(n)
        count += 1
    return count
```

---

### 异或自消性：`a ^ a = 0`，`a ^ 0 = a`

异或是自己的逆运算：

- `a ^ a = 0`：任何数和自身异或结果为 0
- `a ^ 0 = a`：任何数和 0 异或结果不变自己
- 满足交换律和结合律：`a ^ b ^ a = b`

这是 LeetCode 136/260 等"单独出现数字"系列题目的理论基础。

```go
// Go - 异或基础
func xor基础() {
    a := 5 // 0101
    fmt.Println(a ^ a) // 0
    fmt.Println(a ^ 0) // 5
    // 交换律：a^b^a = (a^a)^b = 0^b = b
}
```

```python
# Python - 异或基础
def xor基础():
    a = 5  # 0101
    print(a ^ a)  # 0
    print(a ^ 0)  # 5
    # 交换律：a^b^a = (a^a)^b = 0^b = b
```

---

### Go 中 `^x` 是按位取反（C 语言中是 `~x`）

**Go：** `^x` 是 unary 操作符，表示对 `x` 按位取反（`^x = ^x & 0xFFFF...`）。
**C/Python：** `~x` 是按位取反。

```go
// Go
x := 0b1010 // 10
fmt.Println(^x) // ...11110101（补码），输出 -11

// 如果只想取低 8 位：
fmt.Println(^x & 0xFF) // 245
```

```python
# Python
x = 0b1010  # 10
print(~x)   # -11（Python 中 ~x = -x-1）
print(~x & 0xFF)  # 245
```

---

### Go 没有 `>>>` 无符号右移，用 `uint(x) >> 1`

Go 中 `>>` 对有符号数做算术右移（符号位扩展），对无符号数做逻辑右移。
要模拟无符号右移，必须先转为 `uint`：

```go
// Go - 模拟无符号右移
func unsignedRightShift(x int, n int) int {
    return int(uint(x) >> n)
}

// 负数的区别：
x := -4 // ...11111100
fmt.Println(x >> 1)      // -2（算术右移，符号扩展）
fmt.Println(uint(x) >> 1) // 很大的正数（逻辑右移）
```

```python
# Python - 无符号右移
# Python 的 >> 是逻辑右移（对正数）和算术右移（对负数，自动补符号位）
# Python 整数是无限精度，不会溢出
# 用 & mask 模拟无符号右移
def unsigned_right_shift(x: int, n: int) -> int:
    mask = (1 << 64) - 1  # 假设 64 位
    return ((x & mask) >> n)
```

---

## 2. 高频题

### LeetCode 136 - 只出现一次的数字 I

> **题目：** 一个数组中，除一个数字外其他数字都出现了两次，找出那个单独的数。
> **难度：** 简单
> **思路：** 所有数字异或，成对的数相互抵消，剩下单独的数字。

```go
// Go
func singleNumber(nums []int) int {
    result := 0
    for _, n := range nums {
        result ^= n
    }
    return result
}
```

```python
# Python
def single_number(nums: list[int]) -> int:
    result = 0
    for n in nums:
        result ^= n
    return result
```

**复杂度：** 时间 O(n)，空间 O(1)

**追问：** 如果有两个单独的数怎么办？ → LeetCode 260

---

### LeetCode 137 - 只出现一次的数字 II

> **题目：** 一个数组中，除一个数字外其他数字都出现了三次，找出那个单独的数。
> **难度：** 中等
> **思路：** 统计每一位上 1 的个数。出现 3 次的数字，每一位 1 的个数 % 3 == 0；单独的数贡献了相应的位。

```go
// Go - 方法一：统计每一位
func singleNumber137_1(nums []int) int {
    result := 0
    for i := 0; i < 64; i++ {
        sum := 0
        bit := 1 << i
        for _, n := range nums {
            if n&bit != 0 {
                sum++
            }
        }
        if sum%3 != 0 {
            result |= bit
        }
    }
    return result
}

// Go - 方法二：有限状态机（进阶）
// ones 表示当前为止出现 1 次的位
// twos 表示当前为止出现 2 次的位
// threes = ones & twos
// 当某一位出现 3 次时，从 ones 和 twos 中清除
func singleNumber137_2(nums []int) int {
    ones, twos := 0, 0
    for _, n := range nums {
        twos |= ones & n // 之前出现 1 次的位 AND 当前位 -> 出现 2 次
        ones ^= n        // 异或更新出现 1 次的位
        threes := ones & twos
        ones &^= threes // 出现 3 次的位清除
        twos &^= threes
    }
    return ones
}
```

```python
# Python - 方法一：统计每一位
def single_number_137_1(nums: list[int]) -> int:
    result = 0
    for i in range(64):
        bit = 1 << i
        sum_bits = sum(1 for n in nums if n & bit)
        if sum_bits % 3:
            result |= bit
    return result

# Python - 方法二：有限状态机（进阶）
def single_number_137_2(nums: list[int]) -> int:
    ones, twos = 0, 0
    for n in nums:
        twos |= ones & n
        ones ^= n
        threes = ones & twos
        ones &= ~threes
        twos &= ~threes
    return ones
```

**复杂度：** 时间 O(n)，空间 O(1)

**状态机解释：**
- `ones` = `~twos` & `~n` & `n` ... 实际上逻辑就是 tracking 出现 1 次的状态
- 直观理解：每个位有 3 种状态（出现 0/1/3 次 mod 3），用两个二进制位编码

---

### LeetCode 260 - 只出现一次的数字 III

> **题目：** 数组中有两个数字只出现一次，其他数字都出现两次。找出这两个数字。
> **难度：** 中等
> **思路：** 先全员异或得到 `diff = a ^ b`（两个单独数的异或）。找到 `diff` 最低位的 1（`diff & -diff`），用它把数组分成两组，各组分别异或得到两个单独的数。

```go
// Go
func singleNumber260(nums []int) []int {
    diff := 0
    for _, n := range nums {
        diff ^= n
    }

    // 取最低位的 1（分组依据）
    lowbit := diff & -diff

    a, b := 0, 0
    for _, n := range nums {
        if n&lowbit == 0 {
            a ^= n
        } else {
            b ^= n
        }
    }
    return []int{a, b}
}
```

```python
# Python
def single_number_260(nums: list[int]) -> list[int]:
    diff = 0
    for n in nums:
        diff ^= n

    lowbit = diff & -diff

    a, b = 0, 0
    for n in nums:
        if n & lowbit == 0:
            a ^= n
        else:
            b ^= n
    return [a, b]
```

**复杂度：** 时间 O(n)，空间 O(1)

**追问：** 为什么 `diff & -diff` 能找到最低位的 1？
> 因为 `-diff` = `~diff + 1`，最低位的 1 及其右侧全反，最低位的 1 位置两数相同为 1，其余左侧不变。`diff & (-diff)` 只保留最低位的 1。

---

### LeetCode 201 - 数字范围按位与

> **题目：** 求 `[m, n]` 范围内所有数字的按位与。
> **难度：** 中等
> **思路：** Brian Kernighan 算法。不断消去 `n` 最低位的 1，直到 `n <= m`。剩余的 `m`（或 `n`）就是答案。

```go
// Go
func rangeBitwiseAnd(m int, n int) int {
    for n > m {
        n &= n - 1 // 消去 n 最低位的 1
    }
    return n
}
```

```python
# Python
def range_bitwise_and(m: int, n: int) -> int:
    while n > m:
        n &= n - 1  # 消去 n 最低位的 1
    return n
```

**原理：** 二进制下，`[m, n]` 范围的数按位与，只有公共前缀保留。一旦某位出现 0，后续所有位都会变成 0。Brian Kernighan 算法不断消去右侧的 1，等价于找到公共前缀。

**复杂度：** 时间 O(log n)（最坏 O(32)），空间 O(1)

**追问：** 不用循环怎么做？→ 用 `m ^ n` 找到最高不同位，然后 `m >> shift << shift`

```go
// Go - 另一种：移位法
func rangeBitwiseAndShift(m int, n int) int {
    shift := 0
    for m != n {
        m >>= 1
        n >>= 1
        shift++
    }
    return m << shift
}
```

---

### LeetCode 191 - 位 1 的个数（汉明重量）

> **题目：** 给定无符号整数，返回其二进制表示中 1 的个数。
> **难度：** 简单

```go
// Go - 方法一：n & (n-1) 循环
func hammingWeight1(n uint32) int {
    count := 0
    for n != 0 {
        n &= n - 1
        count++
    }
    return count
}

// Go - 方法二：标准库（面试推荐）
func hammingWeight2(n uint32) int {
    return bits.OnesCount32(n)
}

// Go - 方法三：逐位
func hammingWeight3(n uint32) int {
    count := 0
    for i := 0; i < 32; i++ {
        if n&1 == 1 {
            count++
        }
        n >>= 1
    }
    return count
}
```

```python
# Python - 方法一：n & (n-1) 循环
def hamming_weight_1(n: int) -> int:
    count = 0
    while n:
        n &= n - 1
        count += 1
    return count

# Python - 方法二：内置函数
def hamming_weight_2(n: int) -> int:
    return bin(n).count('1')

# Python - 方法三：Python 3.8+ int.bit_count()
def hamming_weight_3(n: int) -> int:
    return n.bit_count()
```

**复杂度：** 时间 O(k)（k 为 1 的个数）或 O(32)，空间 O(1)

**面试建议：** Go 用 `math/bits.OnesCount32`，Python 用 `int.bit_count()`，但也要能手写 `n & (n-1)` 循环。

---

### LeetCode 338 - 比特位计数

> **题目：** 给定 `n`，对于 `0 <= i <= n` 的每个 `i`，返回其二进制表示中 1 的个数。
> **难度：** 中等
> **思路：** DP。`i` 的 1 的个数 = `i >> 1` 的 1 的个数 + `i & 1`。

```go
// Go
func countBits(n int) []int {
    dp := make([]int, n+1)
    for i := 1; i <= n; i++ {
        dp[i] = dp[i>>1] + (i & 1)
    }
    return dp
}

// 或者用 n & (n-1) 的关系：dp[i] = dp[i & (i-1)] + 1
func countBitsAlt(n int) []int {
    dp := make([]int, n+1)
    for i := 1; i <= n; i++ {
        dp[i] = dp[i&(i-1)] + 1
    }
    return dp
}
```

```python
# Python
def count_bits(n: int) -> list[int]:
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i >> 1] + (i & 1)
    return dp

# 或者用 n & (n-1) 的关系
def count_bits_alt(n: int) -> list[int]:
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i & (i - 1)] + 1
    return dp
```

**复杂度：** 时间 O(n)，空间 O(n)

---

### LeetCode 190 - 颠倒二进制位

> **题目：** 将 32 位无符号整数的二进制位顺序颠倒，返回新的整数。
> **难度：** 简单

```go
// Go - 逐位读取逆向写入
func reverseBits(n uint32) uint32 {
    var result uint32 = 0
    for i := 0; i < 32; i++ {
        result = (result << 1) | (n & 1)
        n >>= 1
    }
    return result
}

// Go - 用库
func reverseBitsStd(n uint32) uint32 {
    return bits.Reverse32(n)
}
```

```python
# Python - 逐位读取逆向写入
def reverse_bits(n: int) -> int:
    result = 0
    for i in range(32):
        result = (result << 1) | (n & 1)
        n >>= 1
    return result
```

**复杂度：** 时间 O(32) = O(1)，空间 O(1)

---

## 3. 位运算在工程中的应用

### 权限系统：位掩码表示权限

用二进制位表示权限，每种权限占一个 bit：

```go
// Go - 权限位掩码
const (
    READ   = 1 << 0  // 0001 = 1
    WRITE  = 1 << 1  // 0010 = 2
    EXEC   = 1 << 2  // 0100 = 4
    DELETE = 1 << 3  // 1000 = 8
)

// 检查权限
func hasPermission(userPerm, wantPerm int) bool {
    return userPerm & wantPerm == wantPerm
}

// 添加权限
func addPermission(userPerm, newPerm int) int {
    return userPerm | newPerm
}

// 移除权限
func removePermission(userPerm, rmPerm int) int {
    return userPerm &^ rmPerm // 等价于 userPerm & ~rmPerm
}

// 切换权限
func togglePermission(userPerm, togglePerm int) int {
    return userPerm ^ togglePerm
}
```

```python
# Python - 权限位掩码
READ = 1 << 0    # 0001 = 1
WRITE = 1 << 1   # 0010 = 2
EXEC = 1 << 2    # 0100 = 4
DELETE = 1 << 3  # 1000 = 8

def has_permission(user_perm: int, want_perm: int) -> bool:
    return (user_perm & want_perm) == want_perm

def add_permission(user_perm: int, new_perm: int) -> int:
    return user_perm | new_perm

def remove_permission(user_perm: int, rm_perm: int) -> int:
    return user_perm & ~rm_perm

def toggle_permission(user_perm: int, toggle_perm: int) -> int:
    return user_perm ^ toggle_perm
```

**优点：** 一个 int 即可表示 32 种权限，权限检查 O(1)，内存极省。

---

### 布隆过滤器底层：位数组操作

布隆过滤器使用一个大位数组 + 多个哈希函数。位数组操作是其核心：

```go
// Go - 布隆过滤器位数组操作示例
type BloomFilter struct {
    bitArray []uint64
    size     uint64
}

// 添加元素：设置 k 个位
func (bf *BloomFilter) Add(data []byte) {
    for _, hash := range bf.getHashes(data) {
        bf.bitArray[hash/64] |= 1 << (hash % 64)
    }
}

// 查询元素：检查 k 个位是否都为 1
func (bf *BloomFilter) Contains(data []byte) bool {
    for _, hash := range bf.getHashes(data) {
        if bf.bitArray[hash/64]&(1<<(hash%64)) == 0 {
            return false
        }
    }
    return true
}
```

---

### Redis `BITSET` / `BITCOUNT`：用户签到统计

Redis 的位图操作本质是对大字符串的位操作，适用于用户签到、活跃统计等场景：

```bash
# Redis 命令示例
SETBIT user:1001:sign:2024  0  1   # 1月1日签到
SETBIT user:1001:sign:2024  31 1   # 2月1日签到
BITCOUNT user:1001:sign:2024       # 统计全年签到天数
BITCOUNT user:1001:sign:2024 0 15  # 统计前16天签到天数
```

```go
// Go - 模拟用户签到统计（位数组思路）
type SignRecord struct {
    bitmap []byte // 每天占 1 bit
}

func (s *SignRecord) Sign(day int) {
    byteIndex := day / 8
    bitIndex := day % 8
    if byteIndex >= len(s.bitmap) {
        s.bitmap = append(s.bitmap, make([]byte, (byteIndex-len(s.bitmap)+1))...)
    }
    s.bitmap[byteIndex] |= 1 << bitIndex
}

func (s *SignRecord) Count() int {
    return bits.OnesCountBytes(s.bitmap)
}

func (s *SignRecord) Check(day int) bool {
    byteIndex := day / 8
    bitIndex := day % 8
    if byteIndex >= len(s.bitmap) {
        return false
    }
    return s.bitmap[byteIndex]&(1<<bitIndex) != 0
}
```

---

### Go `math/bits` 标准库

Go 的 `math/bits` 包提供了丰富的位运算工具，是面试和工程中的利器：

| 函数 | 作用 |
|------|------|
| `bits.OnesCount64(x)` | 统计 64 位整数中 1 的个数 |
| `bits.Len64(x)` | 返回最高有效位的位置（不含符号位） |
| `bits.Reverse64(x)` | 颠倒 64 位整数的位顺序 |
| `bits.RotateLeft64(x, n)` | 循环左移 n 位 |
| `bits.LeadingZeros64(x)` | 左侧前导零个数 |
| `bits.TrailingZeros64(x)` | 右侧尾随零个数（等价于 lowbit 位置） |

```go
// Go - math/bits 常用示例
func demoBits() {
    x := uint64(12) // 1100

    fmt.Println(bits.OnesCount64(x))             // 2
    fmt.Println(bits.Len64(x))                   // 4
    fmt.Println(bits.Reverse64(x))               // 0011...（颠倒）
    fmt.Println(bits.TrailingZeros64(x))         // 2（lowbit = 4，即 2^2）
    fmt.Println(bits.RotateLeft64(x, 1))         // 24 (1100 -> 1001 -> 11000)
    fmt.Println(bits.LeadingZeros64(x))          // 60（64 位下左侧 0 的个数）
}
```

---

## 高频追问

### 为什么 `n & (n-1)` 能消去最低位的 1？从二进制补码角度解释。

对于正整数 `n`，二进制如 `xxxx1000`（最低位的 1 在第 k 位）。

`n-1` 的操作：**将最低位的 1 变成 0，并将该位右侧所有位变成 1**：
- `n     = xxxx1000`
- `n-1   = xxxx0111`

两者 `&` 运算：
- `xxxx1000 & xxxx0111 = xxxx0000`

正好消去了最低位的 1。

**补码角度：** `-n = ~n + 1`。`~n` 将所有位取反，`+1` 向左传播直到遇到最低位的 0。`n & -n` 结果就是最低位的 1 本身（其他位在 `~n` 和 `n` 中互反，只有该位相同）。

---

### 判断一个数是否是 2 的幂，有几种方法？哪种最优？

| 方法 | 代码 | 复杂度 |
|------|------|--------|
| 循环除 2 | `while n%2==0: n/=2; return n==1` | O(log n) |
| `n & (n-1)` | `n > 0 and n & (n-1) == 0` | O(1)，最优 |
| `n & -n` | `n > 0 and n == n & -n` | O(1) |
| 查表 | 预先算好 2^0..2^30 | O(1) 但需空间 |

**最优：`n & (n-1) == 0 && n > 0`**
- 2 的幂只有最高位是 1：`1000, 100, 10`
- `n-1` 会把唯一的 1 消掉：`0111, 011, 01`
- 两者 `&` = 0

**注意：** 0 不是 2 的幂，负数也不是。

---

### Go 中如何安全地做无符号右移？

Go 没有 `>>>` 操作符（Java 中有）。处理无符号右移：

1. **显式转为 `uint` 再移位：**
   ```go
   n := -4
   result := uint(n) >> 1  // 逻辑右移，结果是很大的正数
   ```

2. **掩码限制位数（常用）：**
   ```go
   const mask = uint(^uint32(0)) >> 1 // 0x7FFFFFFF
   result := uint(int32(n)&mask) >> shift
   ```

3. **用 `math/bits` 包的 `RotateLeft` 系列**（注意这是循环移位，不是无符号移位）

**实战建议：** 如果面试题涉及无符号右移，通常输入是 `uint32` 或 `uint64`，直接移位即可；如果是负数需要特别注意，先明确是否需要转 `uint`。
