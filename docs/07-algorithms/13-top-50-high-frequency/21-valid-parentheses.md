# 21. 有效括号（Valid Parentheses）

> 频率：★★★★☆  难度：简单  LeetCode 20

## 题目描述
给定一个只包括 `(`，`)`，`{`，`}`，`[`，`]` 的字符串，判断字符串是否有效。

## 小白先理解
遇到左括号，先记下来；遇到右括号，检查能不能和最近的左括号配对。
这就是典型的“后进先出”，所以要用栈。

## 解法一：反复替换匹配括号
- 不断把 `()`, `[]`, `{}` 替换成空
- 最后如果字符串为空，则有效

## 解法二：栈（推荐）
### Go
```go
func isValid(s string) bool {
    stack := []byte{}
    pairs := map[byte]byte{')': '(', ']': '[', '}': '{'}
    for i := 0; i < len(s); i++ {
        ch := s[i]
        if ch == '(' || ch == '[' || ch == '{' {
            stack = append(stack, ch)
        } else {
            if len(stack) == 0 || stack[len(stack)-1] != pairs[ch] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    return len(stack) == 0
}
```
### Python
```python
def is_valid(s):
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    for ch in s:
        if ch in '([{':
            stack.append(ch)
        else:
            if not stack or stack[-1] != pairs[ch]:
                return False
            stack.pop()
    return len(stack) == 0
```

## 一句话记忆
**括号匹配 = 栈。**
