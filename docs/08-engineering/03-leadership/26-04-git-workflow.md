# Git 分支策略与协作规范：从 GitFlow 到 Trunk-Based

> 考察频率：★★★★☆  优先级：P1
> 关键词：分支策略、Trunk-Based Development、GitFlow、rebase vs merge、squash、PR 规范、冲突治理

## 面试官考察意图

"你们团队 Git 怎么用的？"——这题看似简单，但能立刻分辨出候选人待过的是**作坊**还是**工程化团队**。面试官在意的不是你会不会 `git rebase`，而是：

1. 你能不能**讲清楚分支策略背后的权衡**（隔离性 vs 集成频率）；
2. 你知不知道**为什么现代团队在放弃 GitFlow**（长分支 = 集成地狱）；
3. 你有没有**处理过真实的协作灾难**：冲突风暴、误合、历史污染、revert 不干净。

**核心立场：没有"最好的分支策略"，只有"匹配发布节奏和团队规模的策略"。高频发布的团队用 Trunk-Based，需要维护多版本的团队才用 release branch。**

---

## 核心答案（30 秒版）

三种主流策略，按集成频率从低到高：

| 策略 | 结构 | 适合 | 代价 |
|------|------|------|------|
| **GitFlow** | main / develop / feature / release / hotfix | 多版本并行发布、有 QA 阶段 | 分支多、合并复杂、集成晚 |
| **GitHub Flow** | main + 短命 feature 分支 + PR | 单版本、持续部署 | 需要强 CI 与自动化测试 |
| **Trunk-Based** | 只有 main（trunk）+ 极短分支/直接提交 | 高频发布、强 CI、特性开关 | 要求极高的测试与开关纪律 |

**关键结论**：

- **集成分支越短命越好**：feature 分支存活时间应该以**小时/天**计，超过 3 天就该怀疑设计（拆小或上开关）；
- **合并方式**：主干保持线性用 **squash merge**（一个 PR 一个 commit）；需要保留开发过程用 **merge commit**；**禁止**在共享分支上 `rebase`（会改写他人历史）；
- **本地整理历史**用 `rebase -i`，**同步主干**用 `rebase`（本地分支独享时）或 `merge`（分支已推送共享时）。

**一句话选型**：持续部署团队 → Trunk-Based + Feature Flag；有发布列车/多版本 → GitHub Flow + release 分支；传统企业有 stage 环境 → GitFlow。

---

## 深度展开

### 一、为什么 GitFlow 在现代团队"过气"了

GitFlow 诞生于 2010 年，为**盒装软件 / 定期发版**设计：`develop` 集成、`release/*` 冻结测试、`main` 只放已发布版本、`hotfix/*` 紧急修复。它的问题：

1. **`develop` 与 `main` 长期分叉**：release 分支上修的 bug 要 cherry-pick 回 develop 和 main，漏一次就产生回归；
2. **集成推迟**：feature 分支活几周，最后合并时冲突集中爆发，叫做"集成地狱"（Integration Hell）；
3. **分支数量爆炸**：一个 10 人团队能同时存在 20+ 分支，`git branch -a` 看不过来。

**核心矛盾：分支越长，集成成本越高（近似二次增长）。** 假设每天主干有 N 个提交，一个存活 d 天的分支要合并时，冲突面大致与 `d × N` 成正比。

所以现代高频发布团队转向 **Trunk-Based Development（TBD）**：所有人往 `main` 提交（或极短分支），**每天至少合并一次**，未完成的功能用 **Feature Flag** 藏起来。

### 二、三种策略的详细对比

**GitFlow**

```
main    ──●──────────────●────────●  (只有发布版本 + tag)
           \            /          \
develop ────●──●──●──●────●──●──●──●  (集成主线)
             \    \      /
feature/a ────●────●────●
release/1.2 ─────────────●──●  (冻结测试)
hotfix/1.2.1 ──────────────────●
```

- 优点：发布隔离清晰，适合"发版前需要冻结回归"的场景；
- 缺点：合并路径多（feature→develop→release→main 还要回流），容易漏同步。

**GitHub Flow**

```
main ──●──●──●──────●──●──●──●  (始终可部署)
        \        /
feature ─●──●──●  (PR + CI + Review → squash 合并)
```

- 优点：简单、线性历史、可读；缺点：单版本假设，多环境/多版本吃力。

**Trunk-Based**

```
main ──●─●─●─●─●─●─●─●─●  (每天多次提交，始终绿)
        （未完成功能用 Feature Flag 关闭）
```

- 优点：集成频率最高、冲突最少、CI 反馈最快；缺点：**依赖强 CI + 开关体系 + 自动化测试**，没有这些基础设施就是灾难。

### 三、rebase vs merge：面经最高频追问

| 场景 | 正确操作 |
|------|----------|
| 本地分支整理提交（合并琐碎 commit） | `git rebase -i HEAD~n`（**只在未推送的本地分支**） |
| 把主干最新改动同步到**未推送**的本地分支 | `git rebase main` |
| 把主干最新改动同步到**已推送共享**的分支 | `git merge main`（**不要 rebase**） |
| 把一个 feature 分支合入主干 | `squash merge`（推荐）或 `merge --no-ff` |
| 撤销已推送的提交 | `git revert`（**禁止** `push --force` 到共享分支） |

**为什么不能在共享分支 rebase**：rebase 会**改写提交哈希**，其他人的本地分支还指向旧哈希，一 push 就是冲突/重复历史，团队要集体"reset 止血"。**铁律：已推送的分支只允许 fast-forward 或 revert，不允许 force push（除非 `--force-with-lease` 且确认独占）。**

**`squash merge` 为什么是大多数团队的默认**：一个 PR = 一个提交，主干历史 = 一条清晰的"功能流"，`git log` 可读，`git revert <sha>` 能整体回滚一个功能。代价是丢失 PR 内的过程中间提交（但 PR 页面里还在，不影响追溯）。

### 四、提交规范与 PR 规范

**Conventional Commits**（让 changelog / 语义化版本自动化）：

```
<type>(<scope>): <subject>

feat(order): 支持订单超时自动关单
fix(pay): 修复支付回调重复入账
perf(gc): 减少 30% 临时对象分配
refactor/code-style-only commit
```

好处：`feat` → minor、`fix` → patch 可自动生成版本号与 CHANGELOG；`revert` 也有据可依。

**PR 规范**（Code Review 效率的关键）：

- **小 PR**：理想 < 400 行改动（研究普遍认为超过这个量级 review 质量断崖下降）；
- **单一职责**：一个 PR 只做一件事，混入格式化会让 diff 噪声淹没真实改动；
- **自测证据**：贴关键日志/截图/单测结果；
- **合并前置条件**：CI 绿 + ≥1 approve + 无冲突 + 分支已同步主干。

### 五、冲突治理：把冲突消灭在发生之前

1. **短命分支 + 频繁同步**：每天 `rebase/merge` 一次主干，冲突小到可以手改；
2. **按模块/文件划分 owner**：`CODEOWNERS` 让结构改动有明确负责人；
3. **格式化提交独立**：`gofmt`/`goimports` 的大范围格式化**单独一个 PR**，不要混在功能 PR 里（这是冲突的最大来源之一）；
4. **锁定生成文件**：`wire_gen.go`、`*.pb.go` 等生成物在 `.gitattributes` 标记 `-diff` 或不参与冲突合并，由生成命令统一重跑；
5. **大重构用"绞杀者模式"**：分多次小步替换，而不是一次改 200 文件。

### 六、真实事故复盘（面试故事素材）

**事故：一次 `push --force` 抹掉了同事 3 天的工作**

- 原因：同事在共享分支上 `rebase` 后 `push --force`，另一人未同步的本地提交被覆盖；
- 应急：`git reflog` 找到被覆盖的提交哈希，`git cherry-pick` 抢救回来；
- 根治：分支保护规则禁止 force push，启用 `--force-with-lease` 作为唯一例外（它会在远端有他人提交时拒绝）。

**教训清单**：

- 分支保护（protected branch）：禁止直接 push main、禁止 force push、要求 PR + CI；
- 关键分支开启 "require linear history"（只允许 squash/rebase merge）；
- 危险命令别名化：把 `git push -f` 从 muscle memory 里拿掉。

---

## 高频追问

**Q1：多人同时改一个文件，怎么减少冲突？**
拆分文件/模块（大文件是冲突温床）、明确 owner、短命分支高频同步、格式化提交独立。**根本解法是降低"同一文件被多人同时改"的概率**。

**Q2：如何回滚一个已上线的功能？**
如果合并是 squash → `git revert <squash-sha>` 一键回滚整个功能；如果是多个 merge commit → 需要逐个 revert，容易漏。这也是推荐 squash 的原因之一。

**Q3：cherry-pick 什么时候用？**
只用于"把某个修复搬到另一个不相关的分支"（如 hotfix 回移到 develop）。**它会产生重复提交（不同哈希）**，滥用会让历史出现两个内容相同但哈希不同的提交，后续合并冲突不断。

**Q4：Monorepo 下分支策略有什么不同？**
Monorepo 更依赖**影响范围检测**（只对改动目录跑对应 CI）和 **CODEOWNERS**，分支策略本身更倾向 Trunk-Based（因为跨仓同步成本没了，但跨模块冲突更频繁）。

**Q5：怎么保证 main 始终可发布？**
"main 永远绿"三条：① 合并前 CI 必须全绿（含集成测试）；② 未完成功能用 Feature Flag 关；③ 每次合并后自动部署到预发并跑冒烟。**main 变红是最高优先级事件，先 revert 修复，不允许"带着红主干继续开发"。**

---

## 面试话术

> "我们团队是**Trunk-Based**：主干 `main` 始终可发布，功能走**短命分支**（存活 < 1 天）+ PR + squash merge，每个 PR 一个 commit，回滚就是 `git revert`。未完成的功能用 **Feature Flag** 暗发布，所以半成品也能安全合主干。历史保持线性，本地用 `rebase -i` 整理、共享分支只允许 merge/revert，**分支保护禁止 force push**。提交用 Conventional Commits 自动生成 CHANGELOG，PR 控制在 400 行以内、格式化改动单独提。这套的前提是强 CI 和自动化测试——**没有这些基础设施，TBD 反而是灾难。**"

---

## 延伸阅读

- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [A successful Git branching model（GitFlow 原文）](https://nvie.com/posts/a-successful-git-branching-model/)
- 本仓库：[CI/CD 流水线设计：GitLab CI/Jenkins + 质量门禁 + 灰度发布](25-01-ci-cd-pipeline.md)
