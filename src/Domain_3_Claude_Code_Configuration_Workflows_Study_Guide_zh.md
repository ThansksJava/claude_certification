# Domain 3：Claude Code 配置与工作流
## Claude Certified Architect (Foundations) —— 学习指南（中文）
**考试权重：20% | 主要场景：代码生成、开发者效率、CI/CD**

---

## 概览

Domain 3 是考试里配置密度最高的域。你要么清楚文件该放哪、选项代表什么；要么就会失分。单靠“推理能力”不够。本指南覆盖 6 个任务点，并标出高频考试陷阱。

---

## Task Statement 3.1：`CLAUDE.md` 层级

### 三个层级

| 层级 | 位置 | 受版本控制？ | 作用范围 |
|---|---|---|---|
| 用户级 | `~/.claude/CLAUDE.md` | ❌ 否 | 仅你本人，不通过 git 共享 |
| 项目级 | `.claude/CLAUDE.md` 或仓库根目录 `CLAUDE.md` | ✅ 是 | 仓库所有克隆者共享 |
| 目录级 | `<subdirectory>/CLAUDE.md` | ✅ 是 | 仅在该目录下工作时生效 |

### 加载方式

- Claude Code 启动时会同时加载所有适用层级。
- 若规则冲突，更具体的层级（目录级）覆盖更宽泛层级（项目级）。
- 用户级配置会与项目级合并，但不会被推送到仓库。

### 考试最爱陷阱

> **“新同事没有拿到团队指令。”**
>
> 根因通常是：指令写在 `~/.claude/CLAUDE.md`（用户级），而不是 `.claude/CLAUDE.md`（项目级）。
> 新同事克隆仓库时不会获得你的用户级配置。

### 模块化组织

**`@import` 语法** —— 在 `CLAUDE.md` 中引用外部规则文件：
```markdown
@import .claude/rules/testing.md
@import .claude/rules/api-conventions.md
```

**`.claude/rules/` 目录** —— 按主题拆分规则：
```
.claude/
  rules/
    testing.md
    api-conventions.md
    deployment.md
    security.md
```

这样主 `CLAUDE.md` 保持精简，规则可独立更新。

### 调试：`/memory` 命令

- 在 Claude Code 会话中运行 `/memory`，可查看当前实际加载了哪些 memory 文件。
- **适用场景：**不同会话或不同成员行为不一致时。
- 这是首选诊断手段，不是手动反复读配置文件。

### 练习场景答案

开发者 A 遵守规范，开发者 B（新同事）不遵守。
- ❌ 错误选项：B）项目 `CLAUDE.md` 有语法错误；C）B 需要重装；D）分支差异
- ✅ 正确：规范放在 A 的 `~/.claude/CLAUDE.md`（用户级），不在仓库项目级。修复：迁移到项目级。

---

## Task Statement 3.2：自定义 Slash Commands 与 Skills

### 目录结构

| 位置 | 共享？ | 用途 |
|---|---|---|
| `.claude/commands/` | ✅ 是（版本控制） | 团队共享 slash 命令 |
| `~/.claude/commands/` | ❌ 否（个人） | 个人 slash 命令 |
| `.claude/skills/`（含 `SKILL.md`） | ✅ 是 | 按需触发的任务型工作流 |
| `~/.claude/skills/` | ❌ 否（个人） | 个人技能变体 |

### Skill Frontmatter 选项

```yaml
---
name: review
description: Performs a comprehensive code review
context: fork
allowed-tools: [Read, Grep, Glob]
argument-hint: "Enter the file or directory path to review"
---
```

| 选项 | 作用 | 何时使用 |
|---|---|---|
| `context: fork` | 在隔离子代理上下文运行，冗长输出不污染主会话 | 代码库分析、头脑风暴等高噪声任务 |
| `allowed-tools` | 限制技能可用工具 | 防止破坏性操作（如分析任务禁用 Write/Edit） |
| `argument-hint` | 未传参调用时提示用户输入参数 | 需要目标路径或参数的技能 |

### 关键区别（考试陷阱）

|  | Skills | `CLAUDE.md` |
|---|---|---|
| **何时生效** | 按需触发（必须调用） | 自动持续加载 |
| **适合放什么** | 任务特定流程 | 团队通用标准 |
| **常见误区** | 不要把通用标准放到 skill | 不要把任务流程塞进 `CLAUDE.md` |

### 个性化不影响团队

在 `~/.claude/skills/` 建立个人变体，并使用**不同名字**（如 `my-review`）。
不要覆盖共享 skill，以免影响团队成员。

### 练习场景答案

- 团队共享 `/review`：放 `.claude/commands/review.md`
- 个人 `/brainstorm` 且输出很长：放 `~/.claude/skills/brainstorm.md` 并设置 `context: fork`

---

## Task Statement 3.3：路径特定规则（Path-Specific Rules）

### 语法

```yaml
# .claude/rules/test-conventions.md
---
paths: ["**/*.test.tsx", "**/*.spec.ts", "**/tests/**/*"]
---

# 这里写测试文件规范
- Use describe/it blocks, not test()
- Mock external services with jest.mock()
- Snapshot tests require approval before commit
```

### 相比目录级 `CLAUDE.md` 的核心优势

| 方案 | 覆盖范围 | 问题 |
|---|---|---|
| 目录级 `CLAUDE.md` | 仅该目录内文件 | 测试文件分散在 50+ 目录时需要大量重复配置 |
| 基于 glob 的路径规则 | 匹配模式的所有文件（全仓库） | 单文件即可全局覆盖 |

**考试规则：**若规则要应用到分散在多目录的同类文件（如与源码共置的测试文件），优先用 path-specific + glob，不要到处放目录级 `CLAUDE.md`。

### Token 效率

- 路径作用域规则仅在匹配文件时加载。
- 可减少无关上下文，降低 token 消耗。
- 根级常驻规则会在每次交互都占用 token，即使不相关。

### Glob 示例

```
terraform/**/*        -> 所有 Terraform 文件
**/*.test.tsx         -> 任意目录 React 测试文件
src/api/**/*.ts       -> API 目录所有 TypeScript
**/migrations/**/*    -> 所有数据库迁移文件
```

### 练习场景答案

测试文件分布于 50+ 目录时：
- ❌ B）每个目录都放 `CLAUDE.md`：维护成本高，新增目录易漏
- ❌ C）全放根 `CLAUDE.md`：对所有场景常驻加载，浪费上下文
- ❌ D）用 skill：按需触发，不是自动生效
- ✅ A）path-specific + `**/*.test.tsx`：一处维护，全局自动生效

---

## Task Statement 3.4：Plan Mode vs 直接执行

### 决策框架

**使用 Plan Mode 的场景：**
- 大规模复杂改动
- 存在多种可行路线，需要先评估
- 涉及架构决策
- 多文件联动（如库迁移影响 45+ 文件）
- 需先探索代码库再改动
- 对代码库不熟，先理解再动手

**使用直接执行（Direct Execution）的场景：**
- 变更范围清晰、理解充分
- 单文件 bug 且堆栈明确
- 增加简单日期校验之类的小改动
- 正确实现路径已确定

### Explore 子代理

- 将探索期冗长输出与主会话隔离。
- 仅返回摘要给主会话，不回灌原始噪声。
- 适用于多阶段任务，防止上下文窗口被探索日志占满。

### 组合模式（考试常考）

```
阶段 1：Plan Mode（调研、建模、设计）
阶段 2：Direct Execution（按方案实现）
```

这是生产实践常见范式，也是考试高频。

### 练习场景分类

| 任务 | 模式 | 理由 |
|---|---|---|
| 单体拆微服务 | Plan Mode | 架构级、多方案、全局影响 |
| 修复单函数空指针 | Direct Execution | 单点、范围清晰、方案明确 |
| 跨 30 文件迁移日志库 | Plan Mode | 多文件高影响，先评估再实施 |

---

## Task Statement 3.5：迭代式优化

### 技术优先级（从高到低）

1. **具体输入/输出示例（2–3 组）**：通常比纯文字描述更稳定。
2. **测试驱动迭代**：先写测试，再反馈失败信息，收敛更快。
3. **访谈式（interview pattern）**：先让 Claude 反问澄清，再实现，减少需求盲区。

### 反馈是“批量给”还是“分步给”

| 方式 | 适用场景 |
|---|---|
| 单次批量反馈 | 问题相互影响，改一个会牵动其他 |
| 顺序迭代反馈 | 各问题相对独立，逐个修复更高效 |

### 示例驱动沟通

当自然语言描述导致结果不一致时：

```
不要说："把时间格式改得更可读"

改为：
Input:  { "timestamp": 1711497600 }
Output: { "date": "2024-03-27", "time": "00:00:00 UTC" }

Input:  { "timestamp": 1711584000 }
Output: { "date": "2024-03-28", "time": "00:00:00 UTC" }
```

### 练习场景答案

开发者用纯文字描述转换规则，Claude 每次理解不同：
- ❌ 增加更多长文字
- ❌ 扩上下文窗口
- ✅ 提供 2–3 组 before/after 样例，消除歧义

---

## Task Statement 3.6：CI/CD 集成

### `-p` 参数（必须记住）

```bash
claude -p "Analyse this PR for security vulnerabilities"
```

- `-p` = print mode（非交互模式）
- 不加 `-p` 时，CI 任务会等待交互输入而卡住
- 这是 Domain 3 在 CI/CD 场景最常考知识点

### 结构化 CI 输出

```bash
claude -p "Review this PR" \
  --output-format json \
  --json-schema ./schemas/review-findings.json
```

- 输出机器可解析的结构化结果
- 自动系统可直接转为 PR 行内评论
- 若不加 `--output-format json`，通常得到自由文本，后处理脆弱

### 会话隔离（考试陷阱）

> **“用生成代码的同一会话做审查”** 是错误做法。
>
> 同一会话保留了生成阶段的思路，更不容易质疑自身决策。
>
> ✅ 正确：使用**独立审查实例**。

### 增量审查上下文

当新提交后重复审查：

```bash
claude -p "Review changes. Prior findings: [prior findings here].
Report ONLY new or still-unaddressed issues."
```

- 把历史问题作为上下文提供
- 明确只输出“新增或未修复”问题
- 避免重复评论，维护团队对 CI 工具的信任

### 用 `CLAUDE.md` 提升 CI 质量

建议在项目 `CLAUDE.md` 里明确：
- 测试标准
- 高价值测试判据
- 可用 fixtures 与测试工具

否则，CI 触发的 Claude 可能生成“看起来很多、价值很低”的样板测试。

### 练习场景答案

CI 脚本：`claude "Analyse this PR"` 一直卡住：
- ❌ B）加大超时
- ❌ C）换模型
- ❌ D）加 `--verbose`
- ✅ A）加 `-p`：`claude -p "Analyse this PR"`

---

## Domain 3 考试陷阱速记

| 陷阱 | 正确答案 |
|---|---|
| 新成员拿不到指令 | 从 `~/.claude/CLAUDE.md` 迁移到 `.claude/CLAUDE.md` |
| 测试规范分散 50+ 目录 | 用 glob 路径规则，不要每目录一个 `CLAUDE.md` |
| Skill 输出污染主会话 | `context: fork` |
| CI 卡住 | 加 `-p` |
| 用同会话审查自己代码 | 独立审查实例 |
| 通用标准写进 Skill | 应放 `CLAUDE.md` |
| 任务流程写进 `CLAUDE.md` | 应放 Skill |
| 大规模多文件改动 | 先 Plan 再执行 |
| 纯文字需求解释不稳定 | 改用具体输入/输出示例 |
| 会话行为不一致定位 | 用 `/memory` |

---

## 8 题模拟考试

*(建议先做题，再看答案)*

**Q1 (3.1)** — 团队在仓库里定义了 Claude Code 规范。三位资深工程师行为一致，五位新加入工程师行为不稳定，且都在同一分支。最可能根因是？

A) 新人 IDE 插件覆盖了 Claude 设置  
B) 规范其实放在资深工程师本机 `~/.claude/CLAUDE.md`，不在仓库里  
C) 项目级 `CLAUDE.md` 有语法错误，只影响新版本  
D) 新人需要执行 `claude --refresh` 才能读取最新设置

---

**Q2 (3.1)** — 某开发者说当前会话没有应用正确规则。最直接的诊断命令是？

A) `cat .claude/CLAUDE.md`  
B) `claude --debug`  
C) `/memory`  
D) `git diff HEAD~1 .claude/`

---

**Q3 (3.2)** — 团队要做 `/security-audit` skill 做深度分析，但冗长输出污染主会话。哪个 frontmatter 选项能解决？

A) `allowed-tools: [Read, Grep]`  
B) `context: fork`  
C) `argument-hint: "Enter audit scope"`  
D) `output: summary`

---

**Q4 (3.3)** — Terraform 文件散落在 `infrastructure/`、`modules/`、`environments/staging/`、`environments/production/`。希望编辑任意 Terraform 文件时自动应用规范，正确做法是？

A) 四个目录各放一个 `CLAUDE.md`  
B) 所有 Terraform 规范写进根 `CLAUDE.md`  
C) 建 `.claude/rules/terraform.md`，并设置 `paths` glob 覆盖这些目录  
D) 建 `/terraform-mode` skill 按需触发

---

**Q5 (3.4)** — 团队要在 47 个文件中把 `moment.js` 迁移到 `date-fns`（改 import、API 调用、格式化）。正确方式？

A) 直接执行，一次性让 Claude 全改  
B) 先 Plan Mode：盘点用法、设计迁移，再实施  
C) 前 10 个文件用 Plan，后 37 个直接执行  
D) 用只允许 `Edit` 的自定义 skill 完成

---

**Q6 (3.4)** — 大型多阶段重构中，主会话过长，Claude 开始遗忘早期结论。当前阶段需要大量代码探索。正确解法？

A) 开新会话并从头复述  
B) 使用 Explore 子代理隔离探索输出，只回传摘要  
C) 后续全部切到 Plan Mode  
D) 把对话拆成多个文件扩容上下文

---

**Q7 (3.5)** — 开发者要求“所有 API 响应的 key 从 snake_case 改 camelCase”，多次执行仍不一致。第一优先尝试哪种技术？

A) 写更详细的 prose 规则覆盖所有边界  
B) 提供 2–3 组具体 before/after 样例  
C) 用 interview pattern 先澄清  
D) 先用 plan mode 全图谱分析

---

**Q8 (3.6)** — CI 脚本如下：
```bash
claude "Review this pull request for code quality issues and output findings as JSON"
```
流水线 30 秒后无输出并卡住。正确修复？

A) 加 `--timeout 120`  
B) 改为 `claude -p "Review this pull request for code quality issues" --output-format json`  
C) 加 `--non-interactive`  
D) 重定向到文件：`claude "Review..." > findings.txt`

---

## 模拟题答案

| 题号 | 答案 | 关键原因 |
|---|---|---|
| 1 | **B** | 用户级 `~/.claude/CLAUDE.md` 不会被版本控制共享 |
| 2 | **C** | `/memory` 直接展示当前会话加载的 memory 文件 |
| 3 | **B** | `context: fork` 将技能运行在隔离子代理中 |
| 4 | **C** | glob 路径规则可跨目录统一覆盖 |
| 5 | **B** | 47 文件迁移属大规模多文件任务，应先规划 |
| 6 | **B** | Explore 子代理可隔离噪声并保持主会话清晰 |
| 7 | **B** | 具体样例比文字描述更稳定 |
| 8 | **B** | `-p` 启用非交互模式，避免 CI 卡住 |

**评分建议：**
- 8/8：Domain 3 备考就绪
- 6–7/8：回看薄弱任务点后再测
- <6：按错题任务点重学并补场景练习

---

## 动手练习

搭建一个覆盖 Domain 3 全要点的项目结构：

```
project-root/
├── CLAUDE.md                          <- 项目级（团队标准）
├── src/
│   └── CLAUDE.md                      <- 目录级（src 特定规则）
├── .claude/
│   ├── CLAUDE.md                      <- 项目级替代位置
│   ├── commands/
│   │   └── review.md                  <- 共享 /review 命令
│   ├── skills/
│   │   └── security-audit.md          <- 带 context: fork 的共享技能
│   └── rules/
│       ├── test-conventions.md        <- paths: ["**/*.test.ts"]
│       └── api-conventions.md         <- paths: ["src/api/**/*"]
└── .github/
    └── workflows/
        └── claude-review.yml          <- 使用 -p + JSON 输出
```

**CI 工作流（`claude-review.yml`）示例：**
```yaml
- name: Claude Code Review
  run: |
    claude -p "Review this PR for quality issues. Prior findings: ${{ env.PRIOR_FINDINGS }}.
    Report only new or unaddressed issues." \
    --output-format json \
    --json-schema ./schemas/review.json > findings.json
```

**Skill 示例（`.claude/skills/security-audit.md`）：**
```yaml
---
name: security-audit
description: Performs deep security analysis of the codebase
context: fork
allowed-tools: [Read, Grep, Glob]
argument-hint: "Enter the directory or file path to audit"
---

Perform a comprehensive security audit of $ARGUMENTS...
```

**测试规则示例（`.claude/rules/test-conventions.md`）：**
```yaml
---
paths: ["**/*.test.ts", "**/*.spec.ts", "**/tests/**/*"]
---

All test files must follow these conventions:
- Use describe/it blocks
- Mock all external dependencies
- Each test must have an assertion
```

---

## 总结：Domain 3 必背 5 点

1. **用户级不共享，项目级随仓库共享。**
2. **`context: fork` = 隔离上下文，主会话不被冗长输出污染。**
3. **`.claude/rules/` + glob 可跨代码库生效，目录级 `CLAUDE.md` 只能管一个目录。**
4. **CI 必须加 `-p`，否则会卡在交互等待。**
5. **代码审查要用独立实例，不要在生成同会话里自审。**

