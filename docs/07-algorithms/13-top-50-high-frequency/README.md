# 13 Top 50 High Frequency 模块

> 📌 共 51 道高频面试题 ｜ ✅ 已按面试频率排序（★★★★★ → ★☆☆☆）

---

## 📋 题目索引（点击直接跳转阅读）

| 序号 | 📄 文件名 | 🔥 频率 | 💡 考点 & 跳转 |
|---|---|---|---|
| 01 | `21-01-two-sum.md` | `★★★★★` | [01. 两数之和（Two Sum）](./21-01-two-sum.md)
| 02 | `22-02-three-sum.md` | `★★★★★` | [02. 三数之和（3Sum）](./22-02-three-sum.md)
| 03 | `23-03-longest-substring.md` | `★★★★★` | [03. 最长不重复子串（Longest Substring Without Repeating Characters）](./23-03-longest-substring.md)
| 04 | `24-04-lru-cache.md` | `★★★★★` | [04. LRU 缓存（LRU Cache）](./24-04-lru-cache.md)
| 05 | `25-05-reverse-linked-list.md` | `★★★★★` | [05. 反转链表（Reverse Linked List）](./25-05-reverse-linked-list.md)
| 06 | `26-06-linked-list-cycle.md` | `★★★★★` | [06. 环形链表（Linked List Cycle）](./26-06-linked-list-cycle.md)
| 07 | `27-07-merge-two-lists.md` | `★★★★★` | [07. 合并两个有序链表（Merge Two Sorted Lists）](./27-07-merge-two-lists.md)
| 08 | `28-08-level-order.md` | `★★★★★` | [08. 二叉树层序遍历（Binary Tree Level Order Traversal）](./28-08-level-order.md)
| 09 | `29-09-lca.md` | `★★★★★` | [09. 二叉树最近公共祖先（Lowest Common Ancestor of a Binary Tree）](./29-09-lca.md)
| 10 | `30-10-number-of-islands.md` | `★★★★★` | [10. 岛屿数量（Number of Islands）](./30-10-number-of-islands.md)
| 11 | `31-11-course-schedule.md` | `★★★★★` | [11. 课程表（Course Schedule）](./31-11-course-schedule.md)
| 12 | `32-12-merge-intervals.md` | `★★★★★` | [12. 合并区间（Merge Intervals）](./32-12-merge-intervals.md)
| 13 | `33-13-non-overlapping-intervals.md` | `★★★★★` | [13. 无重叠区间（Non-overlapping Intervals）](./33-13-non-overlapping-intervals.md)
| 14 | `34-14-jump-game.md` | `★★★★★` | [14. 跳跃游戏（Jump Game）](./34-14-jump-game.md)
| 15 | `35-15-trapping-rain-water.md` | `★★★★★` | [15. 接雨水（Trapping Rain Water）](./35-15-trapping-rain-water.md)
| 16 | `36-16-top-k-frequent.md` | `★★★★★` | [16. Top K 高频元素（Top K Frequent Elements）](./36-16-top-k-frequent.md)
| 17 | `37-_template.md` | `★★★★★` | [题目标题](./37-_template.md)
| 18 | `52-17-binary-tree-max-path-sum.md` | `★★★★☆` | [17. 二叉树最大路径和（Binary Tree Maximum Path Sum）](./52-17-binary-tree-max-path-sum.md)
| 19 | `53-18-best-time-to-buy-sell-stock.md` | `★★★★☆` | [18. 买卖股票最佳时机（Best Time to Buy and Sell Stock）](./53-18-best-time-to-buy-sell-stock.md)
| 20 | `54-19-house-robber.md` | `★★★★☆` | [19. 打家劫舍（House Robber）](./54-19-house-robber.md)
| 21 | `55-20-longest-increasing-subsequence.md` | `★★★★☆` | [20. 最长递增子序列（Longest Increasing Subsequence）](./55-20-longest-increasing-subsequence.md)
| 22 | `56-21-valid-parentheses.md` | `★★★★☆` | [21. 有效括号（Valid Parentheses）](./56-21-valid-parentheses.md)
| 23 | `57-22-minimum-window-substring.md` | `★★★★☆` | [22. 最小覆盖子串（Minimum Window Substring）](./57-22-minimum-window-substring.md)
| 24 | `58-23-search-in-rotated-sorted-array.md` | `★★★★☆` | [23. 搜索旋转排序数组](./58-23-search-in-rotated-sorted-array.md)
| 25 | `59-24-spiral-matrix.md` | `★★★★☆` | [24. 螺旋矩阵（Spiral Matrix）](./59-24-spiral-matrix.md)
| 26 | `60-25-edit-distance.md` | `★★★★☆` | [25. 编辑距离（Edit Distance）](./60-25-edit-distance.md)
| 27 | `61-26-permutations.md` | `★★★★☆` | [26. 全排列（Permutations）](./61-26-permutations.md)
| 28 | `62-27-subsets.md` | `★★★★☆` | [27. 子集（Subsets）](./62-27-subsets.md)
| 29 | `63-28-word-break.md` | `★★★★☆` | [28. 单词拆分（Word Break）](./63-28-word-break.md)
| 30 | `64-29-next-greater-element.md` | `★★★★☆` | [29. 下一个更大元素（Next Greater Element）](./64-29-next-greater-element.md)
| 31 | `65-30-largest-rectangle-in-histogram.md` | `★★★★☆` | [30. 柱状图中最大的矩形（Largest Rectangle in Histogram）](./65-30-largest-rectangle-in-histogram.md)
| 32 | `66-31-single-number.md` | `★★★★☆` | [31. 只出现一次的数字（Single Number）](./66-31-single-number.md)
| 33 | `67-32-number-of-provinces.md` | `★★★★☆` | [32. 省份数量（Number of Provinces）](./67-32-number-of-provinces.md)
| 34 | `68-33-rotting-oranges.md` | `★★★★☆` | [33. 腐烂的橘子（Rotting Oranges）](./68-33-rotting-oranges.md)
| 35 | `69-34-merge-k-sorted-lists.md` | `★★★★☆` | [34. 合并 K 个升序链表（Merge K Sorted Lists）](./69-34-merge-k-sorted-lists.md)
| 36 | `70-35-longest-palindromic-substring.md` | `★★★★☆` | [35. 最长回文子串（Longest Palindromic Substring）](./70-35-longest-palindromic-substring.md)
| 37 | `71-36-container-with-most-water.md` | `★★★★☆` | [36. 盛最多水的容器（Container With Most Water）](./71-36-container-with-most-water.md)
| 38 | `72-37-search-a-2d-matrix.md` | `★★★★☆` | [37. 搜索二维矩阵（Search a 2D Matrix）](./72-37-search-a-2d-matrix.md)
| 39 | `73-41-sliding-window-maximum.md` | `★★★★☆` | [41. 滑动窗口最大值（Sliding Window Maximum）](./73-41-sliding-window-maximum.md)
| 40 | `74-42-reverse-nodes-in-k-group.md` | `★★★★☆` | [42. K 个一组翻转链表（Reverse Nodes in k-Group）](./74-42-reverse-nodes-in-k-group.md)
| 41 | `75-43-product-of-array-except-self.md` | `★★★★☆` | [43. 除自身以外数组的乘积（Product of Array Except Self）](./75-43-product-of-array-except-self.md)
| 42 | `76-44-sort-colors.md` | `★★★★☆` | [44. 颜色分类（Sort Colors）](./76-44-sort-colors.md)
| 43 | `77-45-coin-change.md` | `★★★★☆` | [45. 零钱兑换（Coin Change）](./77-45-coin-change.md)
| 44 | `83-38-binary-tree-zigzag-level-order.md` | `★★★☆☆` | [38. 二叉树锯齿形层序遍历](./83-38-binary-tree-zigzag-level-order.md)
| 45 | `84-39-path-sum-iii.md` | `★★★☆☆` | [39. 路径总和 III（Path Sum III）](./84-39-path-sum-iii.md)
| 46 | `85-40-longest-consecutive-sequence.md` | `★★★☆☆` | [40. 最长连续序列（Longest Consecutive Sequence）](./85-40-longest-consecutive-sequence.md)
| 47 | `86-46-symmetric-tree.md` | `★★★☆☆` | [46. 对称二叉树（Symmetric Tree）](./86-46-symmetric-tree.md)
| 48 | `87-47-flatten-binary-tree-to-linked-list.md` | `★★★☆☆` | [47. 二叉树展开为链表（Flatten Binary Tree to Linked List）](./87-47-flatten-binary-tree-to-linked-list.md)
| 49 | `88-48-combination-sum.md` | `★★★☆☆` | [48. 组合总和（Combination Sum）](./88-48-combination-sum.md)
| 50 | `89-49-powx-n.md` | `★★★☆☆` | [49. Pow(x, n)](./89-49-powx-n.md)
| 51 | `90-50-spiral-matrix-ii.md` | `★★★☆☆` | [50. 螺旋矩阵 II（Spiral Matrix II）](./90-50-spiral-matrix-ii.md)

---
_🔄 最后更新：2026-08-10 19:17_
