# 数学类高频题（数论 + 模拟 + 随机）

> 考察频率：★★★☆☆  优先级：P1  Go + Python 双语

## TODO（待填写）

## 1. 质数与因数

### LeetCode 204 - 计数质数
- [ ] 埃拉托色尼筛法（Sieve of Eratosthenes），时间 O(N log log N)
- [ ] Go + Python 代码

### LeetCode 172 - 阶乘后的零
- [ ] 统计因子 5 的个数：`n/5 + n/25 + n/125 + ...`
- [ ] Go + Python 代码

## 2. 快速幂

### LeetCode 50 - Pow(x, n)
- [ ] 分治：`pow(x, n) = pow(x, n/2)^2`，时间 O(logN)
- [ ] 注意负指数和 INT_MIN 边界
- [ ] Go + Python 代码

### 矩阵快速幂
- [ ] 斐波那契数列第 N 项 O(logN) 求法
- [ ] Go 代码模板

## 3. 数字规律

### LeetCode 7 - 整数反转
- [ ] 溢出判断（不用 int64 的写法）
- [ ] Go + Python 代码

### LeetCode 9 - 回文数
- [ ] 反转一半数字
- [ ] Go + Python 代码

### LeetCode 292 - Nim 游戏
- [ ] 数学归纳：能整除 4 则必败
- [ ] Go + Python 代码

## 4. 随机算法

### LeetCode 382 - 链表随机节点（水塘抽样）
- [ ] 遍历到第 i 个节点时，以 1/i 的概率替换结果
- [ ] 证明每个节点等概率被选中
- [ ] Go + Python 代码

### LeetCode 384 - 打乱数组（Fisher-Yates）
- [ ] 从后往前，每个位置和随机前缀位置交换
- [ ] Go + Python 代码

## 5. 模拟题

### LeetCode 54 / 59 - 螺旋矩阵
- [ ] 四个方向循环 + 边界收缩
- [ ] Go + Python 代码

### LeetCode 48 - 旋转图像
- [ ] 先转置，再翻转
- [ ] Go + Python 代码

## 高频追问
- [ ] 水塘抽样解决什么问题？为什么能保证等概率？
- [ ] 快速幂的时间复杂度是多少？如何推导？
