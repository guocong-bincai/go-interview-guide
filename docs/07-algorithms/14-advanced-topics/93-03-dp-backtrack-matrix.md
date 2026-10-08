# 进阶高频题（三）：Kadane / 网格回溯 / 字符串 DP / 矩阵原地 / 链表模拟

> 考察频率：★★★★☆  优先级：P2  语言：Go
> 题型识别：线性 DP（最大子段）、网格 DFS 回溯、字典序构造、双状态 DP、栈消括号、矩阵原地旋转、字符串 DP、网格路径 DP、链表大数模拟、哈希映射复制
> 说明：本篇补齐 07 模块此前未覆盖的 10 道大厂高频题。LFU/Trie/最短路见 [91-01-advanced-topics.md](91-01-advanced-topics.md)，中位数二分/设计类/区间 DP 见 [92-02-design-and-interval-dp.md](92-02-design-and-interval-dp.md)。

---

## 【1】🔥🔥🔥🔥🔥 最大子数组和（LeetCode 53）

### 面试官考察意图

Kadane 算法是「线性 DP 压缩状态」最经典的样板题。面试官想看两件事：你能不能把 `dp[i] = 以 i 结尾的最大子段和` 一句话说清，以及能不能**把它从数组压缩成一个滚动变量**（很多候选人只会写 O(n) 数组版的 DP）。

### 核心答案（30 秒版）

- 定义 `dp[i]` = **以 `nums[i]` 结尾**的最大子数组和（不是「前 i 个里的最大值」，这是最常见的理解错误）。
- 转移：`dp[i] = max(nums[i], dp[i-1] + nums[i])`——要么另起炉灶从自己开始，要么接上前一段。
- 答案是 `max(dp[0..n-1])`，不是 `dp[n-1]`。
- 因为 `dp[i]` 只依赖 `dp[i-1]`，用一个变量 `cur` 滚动即可，空间 O(1)。
- 直觉版本：`cur` 一旦变成负数，继续往后接只会拖累，直接丢弃从当前元素重开。

### Go 代码

```go
func maxSubArray(nums []int) int {
    best, cur := nums[0], nums[0]
    for i := 1; i < len(nums); i++ {
        // 前缀和为负 → 丢弃，从当前元素重新起头
        if cur < 0 {
            cur = nums[i]
        } else {
            cur += nums[i]
        }
        if cur > best {
            best = cur
        }
    }
    return best
}

// 等价的 max 写法（Go 1.21+ 内置 max）
func maxSubArray2(nums []int) int {
    best, cur := nums[0], nums[0]
    for i := 1; i < len(nums); i++ {
        cur = max(nums[i], cur+nums[i])
        best = max(best, cur)
    }
    return best
}
```

### 高频追问

**Q：如果要求返回子数组的起止下标怎么办？**
> 多维护 `start`、`bestStart`、`bestEnd`：当 `cur < 0` 决定重开时 `start = i`；当 `cur > best` 更新 `bestStart = start; bestEnd = i`。

**Q：全负数数组会不会出错？**
> 不会。初始化 `best = cur = nums[0]`，即使全是负数也只选「最大的那一个负数」，符合"子数组非空"的定义。若允许空子数组答案是 0，则初始化 `best = 0`。

**Q：这题和「买卖股票的最佳时机（122）」什么关系？**
> 把相邻价格差转成「涨跌数组」，最大子数组和就是最大利润——这是同一套 Kadane 模板的应用。

### 面试话术

> "Kadane：`dp[i]` 是以 i 结尾的最大和，`dp[i]=max(nums[i], dp[i-1]+nums[i])`；因为只依赖前一项，一个滚动变量就能 O(1) 空间。前缀和变负就果断重开。"

---

## 【2】🔥🔥🔥🔥🔥 单词搜索（LeetCode 79）

### 面试官考察意图

网格回溯的入门必考题。考察点：DFS 状态设计（当前坐标 + 已匹配长度）、**原地标记避免额外 visited 数组**、以及回溯时把状态改回来。

### 核心答案（30 秒版）

- 从每个格子出发，向上下左右四个方向 DFS，逐字符匹配 `word`。
- 关键优化：**用原地修改代替 visited**——访问前把 `board[i][j]` 临时改成一个不可能匹配的字符（如 `'#'`），回溯时再恢复。
- 剪枝：越界、字符不匹配直接返回 false；匹配长度等于 `len(word)` 返回 true。
- 复杂度：最坏 O(m·n·4^L)（L 为单词长度），空间 O(L) 递归栈。

### Go 代码

```go
func exist(board [][]byte, word string) bool {
    m, n := len(board), len(board[0])
    var dfs func(i, j, k int) bool
    dfs = func(i, j, k int) bool {
        if k == len(word) {
            return true // 每个字符都匹配上了
        }
        if i < 0 || i >= m || j < 0 || j >= n || board[i][j] != word[k] {
            return false
        }
        saved := board[i][j]
        board[i][j] = '#' // 就地标记已访问，省掉 visited 数组
        found := dfs(i+1, j, k+1) ||
            dfs(i-1, j, k+1) ||
            dfs(i, j+1, k+1) ||
            dfs(i, j-1, k+1)
        board[i][j] = saved // 回溯：还原现场
        return found
    }
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if dfs(i, j, 0) {
                return true
            }
        }
    }
    return false
}
```

### 高频追问

**Q：为什么 `board[i][j] = saved` 在 `dfs` 返回后一定执行？**
> 哪怕四个方向里有一个提前 return true，因为我们把返回值汇总到 `found` 再还原，保证回溯一定发生。若直接 `return dfs(...)` 就会跳过还原，是常见 bug。

**Q：能进一步剪枝吗？**
> 可以：统计 `board` 各字符频次与 `word` 比对，缺字符直接返回 false；比较 `word[0]` 与 `word[len-1]` 在棋盘上的出现次数，从更少的那个方向搜索（LeetCode 212 单词搜索 II 里更关键的是 Trie + 提前终止）。

**Q：和「岛屿数量」的区别？**
> 岛屿是**连通块计数**（标记已访问后不回溯，因为整块都算一个）；单词搜索是**路径搜索**，不同分支要还原现场，否则一条错误路径会污染其他分支。

### 面试话术

> "网格 DFS 回溯，方向四连，用改棋盘字符代替 visited，递归返回后必须还原。这道题的核心坑就是'回溯要还原状态'。"

---

## 【3】🔥🔥🔥🔥 下一个排列（LeetCode 31）

### 面试官考察意图

考察你**对排列字典序规律的推导能力**，而不是背诵。这是一道"想得出就 10 行、想不出就写不出"的题，面字节/腾讯出现频率很高。

### 核心答案（30 秒版）

要找「字典序紧挨着的下一个更大排列」：

1. **从右往左找第一个「升序对」`nums[i] < nums[i+1]`**：`i` 就是要变大的位置；若不存在说明已是降序（最大排列），直接整体反转成升序即可。
2. 在 `i` 右侧（必为降序区间）**从右往左找第一个 `> nums[i]` 的数 `nums[j]`**，交换 `nums[i]`、`nums[j]`。
3. 此时 `i+1..n-1` 仍是降序，**反转成升序**即得最小后缀。

### Go 代码

```go
func nextPermutation(nums []int) {
    n := len(nums)
    // 1) 从右往左找第一个升序对
    i := n - 2
    for i >= 0 && nums[i] >= nums[i+1] {
        i--
    }
    if i >= 0 {
        // 2) 在 i 右侧降序区间里从右往左找第一个比 nums[i] 大的数
        j := n - 1
        for nums[j] <= nums[i] {
            j--
        }
        nums[i], nums[j] = nums[j], nums[i]
    }
    // 3) 反转 i+1..n-1，使其变为升序（i<0 时等价于整体反转）
    for l, r := i+1, n-1; l < r; l, r = l+1, r-1 {
        nums[l], nums[r] = nums[r], nums[l]
    }
}
```

### 高频追问

**Q：为什么交换后反转就够，不用排序？**
> 因为 `i` 右侧原本是降序（步骤 1 的循环保证），交换后依然是非递增的；把它反转成升序，就得到"在原位置固定 `nums[i]` 变大的数后，后缀的最小排列"，即字典序的紧邻下一个。

**Q：`nums[i] >= nums[i+1]` 用 `>=` 而不是 `>` 会怎样？**
> 处理含重复元素的排列必须用 `>=`，否则相同元素会被误判成升序对，产生非法排列。

### 面试话术

> "找下一个排列三步：从右找第一个升序对 i，从右找第一个大于 nums[i] 的 j 交换，再把 i 后半段反转成升序。已是最大排列时整体反转。"

---

## 【4】🔥🔥🔥🔥 乘积最大子数组（LeetCode 152）

### 面试官考察意图

Kadane 的"带负数"变体，考你**为什么不能只维护最大值**——负负得正，最小值乘以负数会翻身变最大。这是区分"背模板"和"真理解 DP"的题。

### 核心答案（30 秒版）

- 同时维护**以 i 结尾的最大乘积 `maxF[i]` 和最小乘积 `minF[i]`**。
- 因为 `nums[i]` 可能是负数：当前最大乘积可能来自 `minF[i-1] * nums[i]`，当前最小乘积可能来自 `maxF[i-1] * nums[i]`。
- 转移（三个候选：自己单干 / 最大乘自己 / 最小乘自己）：
  - `maxF[i] = max(nums[i], maxF[i-1]*nums[i], minF[i-1]*nums[i])`
  - `minF[i] = min(nums[i], maxF[i-1]*nums[i], minF[i-1]*nums[i])`
- **注意转移必须用上一轮的 `maxF/minF` 快照**，不能在同轮里被覆盖。
- 答案 = `max(maxF[0..n-1])`。

### Go 代码

```go
func maxProduct(nums []int) int {
    maxF, minF, ans := nums[0], nums[0], nums[0]
    for i := 1; i < len(nums); i++ {
        mx, mn := maxF, minF // 快照上一轮的值，避免被本轮覆盖
        x := nums[i]
        maxF = max(x, max(mx*x, mn*x))
        minF = min(x, min(mx*x, mn*x))
        ans = max(ans, maxF)
    }
    return ans
}
```

### 高频追问

**Q：出现 0 会发生什么？**
> `maxF`、`minF` 都会被重置为 0（因为候选里含 `nums[i]=0`），相当于以 0 为分界把数组切成若干段独立求解，正好符合直觉。

**Q：为什么乘积题的"最小值"这么重要？**
> 因为负号会翻转大小关系。`[-2, 3, -4]` 里最小值 `-6` 乘 `-4` 得 `24` 反而是全局最大。只存最大值会漏掉这种"两负相乘"的答案。

**Q：溢出怎么办？**
> Go `int` 是 64 位（在 64 位平台），乘积题一般数值范围受题目约束；若题目给出可能超过 int32 的数据量，显式用 `int64` 承接中间乘积。

### 面试话术

> "乘积题要同时维护最大和最小乘积，因为负数会让最小翻身变最大。转移时记得先用快照，别被本轮覆盖；一次性把 max/min 两个候选都算出来。"

---

## 【5】🔥🔥🔥🔥 最长有效括号（LeetCode 32）

### 面试官考察意图

栈的典型应用，比"有效的括号"难一档。核心技巧是**栈底始终放一个"最后一个未匹配的右括号下标"作为分界哨兵**。

### 核心答案（30 秒版）

- 栈里存**下标**，初始压入 `-1` 作为哨兵（表示"基准位置"）。
- 遇到 `'('`：压入其下标。
- 遇到 `')'`：先弹出栈顶。
  - 若弹完后栈为空 → 说明当前这个 `')'` 是多余的，把它自己的下标压入，作为新的分界哨兵。
  - 否则，`i - stack[top]` 就是当前有效括号长度，更新答案。
- 复杂度：时间 O(n)，空间 O(n)。

### Go 代码

```go
func longestValidParentheses(s string) int {
    res := 0
    stack := []int{-1} // 栈底哨兵：最后一个未匹配右括号的下标
    for i := 0; i < len(s); i++ {
        if s[i] == '(' {
            stack = append(stack, i)
        } else {
            stack = stack[:len(stack)-1] // 弹出栈顶（配对 '(' 或哨兵）
            if len(stack) == 0 {
                // 当前 ')' 没有可配对的 '('，它成为新的分界
                stack = append(stack, i)
            } else {
                if l := i - stack[len(stack)-1]; l > res {
                    res = l
                }
            }
        }
    }
    return res
}
```

### 高频追问

**Q：哨兵 `-1` 的作用到底是什么？**
> 它代表"当前有效段的起点前一位"。当栈弹出到只剩哨兵时，`i - (-1) = i+1` 就是从头开始的有效长度；遇到多余右括号时，把它自己压入当新哨兵，把之后的有效段和前面隔开。

**Q：还有别的解法吗？**
> 两种常见替代：① **左右计数法**（正反各扫一遍，`left == right` 时记录长度，`right > left` 时清零）；② **线性 DP**：`dp[i]` 表示以 i 结尾的最长有效长度，`s[i]==')'` 时看 `s[i-1]` 是 `'('` 还是 `')'` 分类转移。栈法最直观、最不易写错。

**Q：和"有效的括号（20）"的区别？**
> 20 题只要判断整体是否合法，栈里存字符即可；32 题要求**最长连续合法子串**，所以栈必须存下标才能算长度。

### 面试话术

> "栈存下标，栈底放 -1 当基准。每见到 ')' 先弹栈，弹空说明它多余就压回自己当新基准，否则用当前下标减新栈顶就是有效长度。O(n)。"

---

## 【6】🔥🔥🔥🔥 旋转图像（LeetCode 48）

### 面试官考察意图

矩阵原地操作的基础题。考察你能否把「旋转 90°」拆成两个更简单的原地操作（转置 + 水平翻转），而不是傻傻地四元组循环。

### 核心答案（30 秒版）

顺时针旋转 90° = **先主对角线转置，再每一行水平翻转**。

- 转置：只遍历上三角 `j > i`，交换 `matrix[i][j]` 与 `matrix[j][i]`（若全遍历会交换两次等于没动）。
- 水平翻转：每行用双指针从两端向中间交换。
- 复杂度：时间 O(n²)，空间 O(1)（满足原地要求）。

### Go 代码

```go
func rotate(matrix [][]int) {
    n := len(matrix)
    // 1) 沿主对角线转置：只处理上三角，避免交换两次
    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
        }
    }
    // 2) 每行水平翻转
    for i := 0; i < n; i++ {
        for l, r := 0, n-1; l < r; l, r = l+1, r-1 {
            matrix[i][l], matrix[i][r] = matrix[i][r], matrix[i][l]
        }
    }
}
```

### 高频追问

**Q：如果要求逆时针旋转 90° 呢？**
> 改成「先转置，再上下翻转（交换第 i 行和第 n-1-i 行）」即可。顺时针 = 转置 + 左右翻；逆时针 = 转置 + 上下翻。

**Q：能不能用"逐层四元组"一步到位？**
> 可以，对每层 `(i, j)` 做 `(i,j)→(j,n-1-i)→(n-1-i,n-1-j)→(n-1-j,i)` 四点轮换。但转置+翻转更好写、更不易下标算错，面试推荐。

### 面试话术

> "顺时针 90° 就两步：主对角线转置（只走上三角），再逐行水平翻转。全程原地 O(1) 空间，比四元组轮换好写。"

---

## 【7】🔥🔥🔥 正则表达式匹配（LeetCode 10）与通配符匹配（44）

### 面试官考察意图

字符串 DP 的天花板之一。难度在于**把 `*` 的"匹配 0 次 / 多次"拆成两个独立转移**，以及正确处理 `p` 以 `a*b*` 开头匹配空串的初始化。

### 核心答案（30 秒版）

`dp[i][j]` = `s` 的前 i 个字符能否被 `p` 的前 j 个字符匹配。

- 初始化：`dp[0][0] = true`；对 `p` 里形如 `x*` 的前缀，允许匹配空串：`p[j-1] == '*' && dp[0][j-2]`。
- 转移分两类：
  - `p[j-1]` 是普通字符或 `.`：`dp[i][j] = dp[i-1][j-1] && (p[j-1]=='.' || p[j-1]==s[i-1])`。
  - `p[j-1] == '*'`：`*` 修饰的是 `p[j-2]`：
    - **匹配 0 次**：`dp[i][j] = dp[i][j-2]`
    - **匹配 ≥1 次**（前提 `p[j-2]` 能匹配 `s[i-1]`）：`dp[i][j] = dp[i][j] || dp[i-1][j]`
- 答案 `dp[m][n]`，时间 O(m·n)。

### Go 代码

```go
// LeetCode 10：'.' 匹配任意单字符，'*' 匹配前面一个字符的 0 次或多次
func isMatch(s string, p string) bool {
    m, n := len(s), len(p)
    dp := make([][]bool, m+1)
    for i := range dp {
        dp[i] = make([]bool, n+1)
    }
    dp[0][0] = true
    // 处理 p 以 "a*b*c*" 形式匹配空串的情况
    for j := 2; j <= n; j++ {
        if p[j-1] == '*' {
            dp[0][j] = dp[0][j-2]
        }
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if p[j-1] == '*' {
                // 匹配 0 次：忽略 "x*" 这一整块
                dp[i][j] = dp[i][j-2]
                // 匹配 ≥1 次：p[j-2] 能匹配 s[i-1]，则消耗掉 s 的一个字符
                if p[j-2] == '.' || p[j-2] == s[i-1] {
                    dp[i][j] = dp[i][j] || dp[i-1][j]
                }
            } else if p[j-1] == '.' || p[j-1] == s[i-1] {
                dp[i][j] = dp[i-1][j-1]
            }
        }
    }
    return dp[m][n]
}
```

```go
// LeetCode 44 通配符匹配：'?' 匹配任意单字符，'*' 匹配任意长度（含空）
func isMatchWild(s string, p string) bool {
    m, n := len(s), len(p)
    dp := make([][]bool, m+1)
    for i := range dp {
        dp[i] = make([]bool, n+1)
    }
    dp[0][0] = true
    for j := 1; j <= n; j++ { // p 的前缀全为 '*' 时可匹配空串
        if p[j-1] == '*' {
            dp[0][j] = true
        } else {
            break
        }
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            switch p[j-1] {
            case '*':
                // '*' 匹配空(dp[i][j-1]) 或匹配掉 s 的一个字符(dp[i-1][j])
                dp[i][j] = dp[i][j-1] || dp[i-1][j]
            case '?':
                dp[i][j] = dp[i-1][j-1]
            default:
                dp[i][j] = dp[i-1][j-1] && p[j-1] == s[i-1]
            }
        }
    }
    return dp[m][n]
}
```

### 高频追问

**Q：第 10 题 `*` 的转移为什么是 `dp[i-1][j]` 而不是 `dp[i-1][j-1]`？**
> 因为 `*` 可以匹配**多个**字符。写成 `dp[i-1][j]` 表示"消耗掉 s 的当前字符后，`*` 还能继续匹配"，通过反复转移自然实现"匹配多次"；`j-1` 就退化成只能匹配一次了。

**Q：为什么要先处理 `dp[0][j]`？**
> 空串场景：`s=""`、`p="a*b*"` 是匹配的，这条初始化负责让 `a*`、`b*` 各自匹配 0 次，否则第一行永远是 false。

**Q：能用递归/记忆化写吗？**
> 可以，`dfs(i, j)` 加 memo，思路和递推一致；但迭代 DP 更容易保证每个状态只算一次，面试写递推更稳。

### 面试话术

> "字符串 DP：`dp[i][j]` 表示 s 前 i、p 前 j 是否匹配。遇到 `*` 分'匹配 0 次'和'匹配多次'两条转移；别忘了先初始化 `dp[0][j]` 处理空串匹配 `a*b*` 的情况。"

---

## 【8】🔥🔥🔥 最小路径和（LeetCode 64）

### 面试官考察意图

网格 DP 的入门题，考察**二维 DP 的边界初始化**和**原地复用输入节省空间**。与「不同路径（62）」是同一模板的两个变体，常一起出现。

### 核心答案（30 秒版）

- `dp[i][j]` = 走到 `(i,j)` 的最小路径和，只能从**上边**或**左边**过来。
- 转移：`dp[i][j] = min(dp[i-1][j], dp[i][j-1]) + grid[i][j]`。
- 边界：第一行只能从左来、第一列只能从上（先单独累加）。
- 直接在原 `grid` 上累加即可原地操作，空间 O(1)（不计输入）。
- 「不同路径（62）」把 `min` 换成 `+`（求方案数）即可复用同一套骨架。

### Go 代码

```go
func minPathSum(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    // 第一列只能从上方来
    for i := 1; i < m; i++ {
        grid[i][0] += grid[i-1][0]
    }
    // 第一行只能从左方来
    for j := 1; j < n; j++ {
        grid[0][j] += grid[0][j-1]
    }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            grid[i][j] += min(grid[i-1][j], grid[i][j-1])
        }
    }
    return grid[m-1][n-1]
}
```

### 高频追问

**Q：如果只能往右/往下，"不同路径"怎么算？**
> 求方案数：`dp[i][j] = dp[i-1][j] + dp[i][j-1]`，边界都是 1；还可进一步压缩成一维滚动数组，`dp[j] += dp[j-1]`。

**Q：面试官问"能不能压缩成 O(n) 空间"？**
> 能。因为 `dp[i][j]` 只依赖左和上，一维数组 `dp[j]` 从左到右滚动更新即可（更新前 `dp[j]` 是上一行、`dp[j-1]` 是当前行左边）。

### 面试话术

> "网格 DP：`dp[i][j] = min(上, 左) + grid[i][j]`，先单独处理第一行第一列，可以直接在原数组上累加，O(1) 额外空间。"

---

## 【9】🔥🔥🔥 两数相加（LeetCode 2）

### 面试官考察意图

链表模拟十进制加法的经典题，考察**进位处理**和**链表遍历 + 虚拟头节点**的基本功。几乎没有算法难度，但会因为边界（最后还有进位、两链长度不等）写挂。

### 核心答案（30 秒版）

- 用**虚拟头节点 `dummy`** 简化链表构造，`cur` 指针尾插。
- 循环条件：`l1 != nil || l2 != nil || carry != 0`——三个条件缺一不可，最后一个负责处理"两链都走完但仍有进位"的情况。
- 每轮：取 `x`（l1 有则 `l1.Val` 否则 0）、`y` 同理，`sum = x + y + carry`，`carry = sum / 10`，新节点值 `sum % 10`。
- 复杂度：O(max(m,n)) 时间，O(max(m,n)) 空间。

### Go 代码

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func addTwoNumbers(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    cur := dummy
    carry := 0
    for l1 != nil || l2 != nil || carry != 0 {
        x, y := 0, 0
        if l1 != nil {
            x = l1.Val
            l1 = l1.Next
        }
        if l2 != nil {
            y = l2.Val
            l2 = l2.Next
        }
        sum := x + y + carry
        carry = sum / 10
        cur.Next = &ListNode{Val: sum % 10}
        cur = cur.Next
    }
    return dummy.Next
}
```

### 高频追问

**Q：为什么循环条件里要有 `carry != 0`？**
> 处理 `5 + 5 = 10` 这种"最后一位还在进位"的情况。少了这个条件会丢掉最高位的 1（`[9,9] + [1] = [0,0,0,1]`）。

**Q：如果链表是正序存储（最高位在前）呢？**
> 两个办法：① 先反链表再套本题；② 用两个栈把数字压栈、从低位弹出相加。LeetCode 445 就是正序版本。

**Q：虚拟头节点 `dummy` 的价值？**
> 避免对头节点判空和分别处理，尾插逻辑统一；返回时给 `dummy.Next` 即可，是链表题的通用套路。

### 面试话术

> "链表模拟加法：虚拟头节点尾插，循环条件写 `l1!=nil || l2!=nil || carry!=0`，最后那个 carry 处理最高位进位。短板补 0，同长处理。"

---

## 【10】🔥🔥🔥 随机链表的深拷贝（LeetCode 138）

### 面试官考察意图

考察**哈希表建立"原节点 → 新节点"映射**的思维，以及能否进一步做到 **O(1) 额外空间**（节点交织法）。这是"看似简单但随机指针容易漏"的典型题。

### 核心答案（30 秒版）

**哈希法（优先写，最清晰）：**

1. 第一遍遍历：为每个原节点 `cur` 建一个克隆节点，存进 `map[*Node]*Node`。
2. 第二遍遍历：`m[cur].Next = m[cur.Next]`、`m[cur].Random = m[cur.Random]`。
3. 返回 `m[head]`。注意 Go map 对不存在的键返回零值 `nil`，所以 `m[cur.Next]`（当 `Next == nil`）天然是 `nil`，无需特判。
- 时间 O(n)、空间 O(n)。

**交织法（O(1) 额外空间）：** 把克隆节点插到原节点后面 → 设置 `random` 指向原节点 `random` 的 `next` → 拆开两条链。

### Go 代码

```go
type Node struct {
    Val    int
    Next   *Node
    Random *Node
}

// 哈希映射法：O(n) 时间，O(n) 空间
func copyRandomList(head *Node) *Node {
    if head == nil {
        return nil
    }
    m := make(map[*Node]*Node)
    for cur := head; cur != nil; cur = cur.Next {
        m[cur] = &Node{Val: cur.Val}
    }
    for cur := head; cur != nil; cur = cur.Next {
        m[cur].Next = m[cur.Next]   // nil 的 map 取值是 nil，天然安全
        m[cur].Random = m[cur.Random]
    }
    return m[head]
}

// O(1) 空间：节点交织法
func copyRandomListO1(head *Node) *Node {
    if head == nil {
        return nil
    }
    // 1) 在每个原节点后插入克隆节点：A -> A' -> B -> B'
    for cur := head; cur != nil; cur = cur.Next.Next {
        clone := &Node{Val: cur.Val, Next: cur.Next}
        cur.Next = clone
    }
    // 2) 设置克隆节点的 Random
    for cur := head; cur != nil; cur = cur.Next.Next {
        if cur.Random != nil {
            cur.Next.Random = cur.Random.Next
        }
    }
    // 3) 拆开两条链
    dummy := &Node{}
    tail := dummy
    for cur := head; cur != nil; {
        clone := cur.Next
        cur.Next = clone.Next // 还原原链
        tail.Next = clone     // 接到克隆链
        tail = clone
        cur = cur.Next
    }
    return dummy.Next
}
```

### 高频追问

**Q：`m[cur.Next]` 当 `cur.Next == nil` 时为什么不会 panic？**
> Go 里从 map 读取不存在的键返回该类型的零值；`*Node` 的零值是 `nil`，正好对应"新链的末尾也是 nil"，省掉显式判空。

**Q：交织法第 2 步 `cur.Random.Next` 是什么意思？**
> `cur.Random` 是原节点的随机指针，它 `Next` 正是我们插在它后面的克隆节点，也就是克隆节点该指向的 `Random`。这是交织法的精髓。

**Q：为什么不能只复制 `Val` 和 `Next`？**
> `Random` 可能指向**前面或后面任意节点**，一次遍历时目标可能还没创建，所以必须两遍（或交织两/三遍）才能正确连线。

### 面试话术

> "深拷贝带随机指针的链表：第一遍建 map 原→新，第二遍连 Next 和 Random，靠 map 零值是 nil 免特判。要 O(1) 空间就用节点交织法：插克隆、设 random、拆链三步。"

---

## 小结

| 题目 | 核心技巧 | 复杂度 | 易错点 |
|------|----------|--------|--------|
| 最大子数组和 (53) | Kadane 线性 DP | O(n)/O(1) | 答案要取全局 max 而非 dp[n-1] |
| 单词搜索 (79) | 网格 DFS + 原地标记 | O(mn·4^L) | 回溯必须还原状态 |
| 下一个排列 (31) | 找升序对 + 反转后缀 | O(n)/O(1) | 含重复需用 `>=` |
| 乘积最大子数组 (152) | 同时维护 max/min | O(n)/O(1) | 转移要用快照防覆盖 |
| 最长有效括号 (32) | 栈存下标 + 哨兵 | O(n)/O(n) | 多余右括号当新哨兵 |
| 旋转图像 (48) | 转置 + 水平翻转 | O(n²)/O(1) | 转置只走上三角 |
| 正则/通配符匹配 (10/44) | 字符串 DP | O(mn) | `*` 的 0 次/多次转移 + 空串初始化 |
| 最小路径和 (64) | 网格 DP | O(mn)/O(1) | 第一行/列单独初始化 |
| 两数相加 (2) | 链表模拟 + 虚拟头 | O(max(m,n)) | 循环条件含 carry |
| 随机链表深拷贝 (138) | 哈希映射 / 交织法 | O(n) | Random 需两遍连线 |
