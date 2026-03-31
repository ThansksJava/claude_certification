# Domain 1：Agentic 架构与编排 —— 完整学习指南（中文）

**Claude Certified Architect (Foundations) | 考试权重：27%（单一最高权重域）**

> 说明：本中文版在不改变含义的前提下，对原文做了适度简化与本地化，重点保留考试要点、关键术语和代码示例。英文技术字段（如 `stop_reason`、`Task`、`fork_session`）保持不译，以便与官方文档对照。

---

## 如何使用本指南

本指南采用“先讲解、再检验”的结构。每个任务点一般包含：
1. **概念讲解** + 生产场景示例
2. **考试陷阱说明**（常见误区与干扰项）
3. **练习场景** + 标准解析或答案要点
4. **与下一任务点的衔接提示**

在 7 个任务点之后，提供 **10 题综合模拟题**，用于域内查漏补缺。你可以对照答案和解析，将错题映射回相应任务点再复习。

---

## 先做自评：你从哪里开始？

| 评级 | 描述 | 建议学习方式 |
|---|---|---|
| **Novice** | 几乎没有 agentic / LLM 系统经验 | 全文精读；先做练习再看答案 |
| **Intermediate** | 做过简单单 Agent 系统 | 重点看“考试陷阱”和练习题，概念部分可略读 |
| **Advanced** | 做过生产级多 Agent 系统 | 先刷练习与模拟题，再回读自己不熟的任务点 |

---

## 域概览

- **考试权重：27%**（五大域中最高）
- **通过分数：720 / 1000**
- **题型：场景化单选（1 正确 + 3 干扰）**
- **高频场景：**客服问题解决 Agent、多 Agent 研究系统、开发者效率工具

> **核心考试哲学（很重要）**
> 1. 高风险场景优先 **确定性（deterministic）** 方案，而非纯概率性（probabilistic）
> 2. 修复方案要 **与问题规模匹配**（不要为格式问题端出“核武器”）
> 3. 故障归因看 **根因组件**，不是看“最终暴露点”（即问题是在哪个组件第一次被引入的）

---

## Task 1.1：Agentic Loop

### Agentic Loop 的完整生命周期

以“客服处理账单争议”为例：

```text
1. 把用户消息通过 Messages API 发给 Claude
2. Claude 返回后，检查响应里的 stop_reason
3. 若 stop_reason == "tool_use":
      → 执行 Claude 请求的工具
      → 将工具结果作为新消息追加到会话历史
      → 把更新后的会话再次发给 Claude
      → 回到步骤 2
4. 若 stop_reason == "end_turn":
      → Claude 推理完成
      → 将最终答复返回给用户
```

关键点：**工具结果必须写回对话历史**。模型每次 API 调用都不保留内存，唯一上下文就是你传入的 `messages`。

```python
import anthropic

client = anthropic.Anthropic()

def run_agentic_loop(initial_messages, tools, system_prompt):
    messages = initial_messages.copy()

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )

        # 正确：以 stop_reason 作为终止依据
        if response.stop_reason == "end_turn":
            final_text = next(
                (block.text for block in response.content if hasattr(block, "text")),
                ""
            )
            return final_text

        elif response.stop_reason == "tool_use":
            # 1) 先把 assistant 的 tool_use 响应写入历史
            messages.append({"role": "assistant", "content": response.content})

            # 2) 执行工具并收集结果
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # 3) 把工具结果作为 user 消息追加，继续下一轮
            messages.append({"role": "user", "content": tool_results})
```

### 三大反模式（高频考点）

| 反模式 | 为什么错 | 正确做法 |
|---|---|---|
| 解析自然语言（如“我做完了”）判断结束 | 自然语言有歧义，模型中途也可能说“先做完第一部分” | 用 `stop_reason == "end_turn"` |
| 迭代上限当主终止条件（如 10 轮强停） | 可能提前截断或空转；应由模型显式信号决定 | 上限只做保险，不做主逻辑 |
| 用内容块类型判断结束（首块是 text 就结束） | `text` 与 `tool_use` 可同时出现，易提前退出 | 永远看 `stop_reason` |

### Model-driven vs Pre-configured

- **Model-driven：**模型按上下文自主选工具，灵活性高
- **Pre-configured：**固定流程或决策树，审计性强

考试规则：一般场景偏向 model-driven；涉及关键业务约束时，用程序化手段做硬约束（见 Task 1.4）。

### 练习 1.1：过早终止 Bug

**场景：**

```python
response = client.messages.create(...)

if response.content and response.content[0].type == "text":
    return response.content[0].text
```

1) Bug 是什么？
2) 正确修复是什么？

**答案要点：**
- Bug：响应中可能同时有 `text` + `tool_use`，代码会在执行工具前提前返回
- 修复：改为 `stop_reason` 判断

```python
if response.stop_reason == "end_turn":
    return next((b.text for b in response.content if hasattr(b, "text")), "")
elif response.stop_reason == "tool_use":
    ...
```

---

## Task 1.2：多 Agent 编排（Multi-Agent Orchestration）

### Hub-and-Spoke（中心辐射）架构

```text
                    ┌─────────────────┐
                    │   COORDINATOR   │
                    │     AGENT       │
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  Subagent A │  │  Subagent B │  │  Subagent C │
    │ (Web Search)│  │  (Doc Anal.)│  │ (Synthesis) │
    └─────────────┘  └─────────────┘  └─────────────┘

    ✅ 所有通信都经 Coordinator
    ❌ Subagent 之间不能直接通信
```

### Coordinator 职责

1. 任务拆解（decomposition）
2. 动态选择子 Agent（不是每次都全用）
3. 范围划分，避免重复劳动
4. 显式传递上下文
5. 聚合结果
6. 错误处理和恢复
7. 迭代补洞（refinement）直到达标

### 关键隔离原则（考试高频）

- 子 Agent **不会自动继承** coordinator 历史
- 子 Agent **不共享内存**
- 子 Agent **看不到其他子 Agent 结果**，除非 coordinator 显式传递

> 结论：子 Agent 需要的每条信息，都必须由 coordinator 写进 prompt。

### 典型失败：拆解过窄

如果任务是“AI 对创意产业影响”，coordinator 却只分配“绘画/摄影/插画”，导致音乐、写作、影视、戏剧全部缺失。

- **根因在 coordinator 的拆解**，不是子 Agent，也不是汇总 Agent。
- 考试干扰项会诱导你怪“搜索不全”或“汇总不全”，都不是根因。

### 练习 1.2：研究报告不完整

题目：可再生能源市场分析只覆盖了 solar/wind，没有 geothermal/tidal/biomass/fusion。根因是哪个组件？

- **正确答案：C（Coordinator）**

---

## Task 1.3：子 Agent 调用与上下文传递

### `Task` 工具

- coordinator 通过 `Task` 生成子 Agent 任务
- coordinator 的 `allowedTools` 必须包含 `"Task"`
- 如果多 Agent 系统“起不来子 Agent”，先检查这里

### 上下文传递：错误 vs 正确

**错误示例（缺上下文）：**

```python
subagent_prompt = """
请综合可再生能源研究结果，形成报告。
"""
```

问题：没有给具体 findings，子 Agent 只能空泛输出，甚至幻觉。

**正确示例（带结构化上下文）：**

```python
import json

subagent_prompt = f"""
请基于以下研究结果进行综合。

## WEB SEARCH FINDINGS
{web_search_results}

## ACADEMIC DOCUMENT FINDINGS
{document_analysis_results}

## SOURCE METADATA
{json.dumps(source_metadata, indent=2)}

## Research Goal
{original_research_goal}

## Quality Criteria
- 覆盖主要技术类别
- 每条事实可溯源引用
- 标记证据薄弱区域
"""
```

### 核心原则

1. 把上游结果完整传入下游
2. 内容与元数据分离（source URL、文档名、页码）
3. 给目标和质量标准，不要写死步骤
4. 显式传递 claim-source 映射

### 并行生成子任务

在 **同一轮 coordinator 响应** 里发多个 `Task` 调用，即可并行执行：

```python
[
    Task(agent="web_search_agent", prompt="...solar..."),
    Task(agent="web_search_agent", prompt="...wind..."),
    Task(agent="academic_agent", prompt="...geothermal..."),
    Task(agent="academic_agent", prompt="...tidal/wave..."),
]
```

考试规则：独立子任务 + 低延迟需求 -> 并行。

### `fork_session`

- 从共享基线分叉，后续独立推进
- 适合“同起点、不同策略并行评估”
- 区别于 Task 并行：`fork_session` 共享完整基线；Task 子 Agent 只拿到你传入的上下文

### 练习 1.3：报告无引用

根因：coordinator 没把结构化 source metadata 传给 synthesis 子 Agent。

修复：要求上游输出 claim-source 映射，并原样传入 synthesis。

---

## Task 1.4：工作流强制与 Handoff

### 强制手段光谱

| 类型 | 机制 | 可靠性 | 适用场景 |
|---|---|---|---|
| Prompt 引导 | 系统提示词（如“必须先核验再退款”） | 概率性（约 92–99%） | 低风险偏好（语气、格式） |
| 程序化强制 | 前置门禁、Hook 硬阻断 | 100% 确定性 | 金融/安全/合规 |

### 考试决策规则（必须记住）

> 单次失败就会造成财损、合规违规或安全事故 -> **必须程序化强制**。

所以在高风险题里，看到“加强 prompt / few-shot”通常是干扰项。

### 练习 1.4：8% 未核验退款

**场景：** 客服 Agent 中有 8% 的退款在未核验账户归属的情况下被执行，已有错误退款发生。工程团队提出四个方案：

- **A)** 实施程序化前置门禁：退款工具必须在会话状态中存在已核验的 account token 才能执行
- **B)** 重写系统提示词，加强措辞："你必须在处理退款前核验账户归属"
- **C)** 添加 10 个 few-shot 示例，展示正确的"先核验再退款"流程
- **D)** 增加路由分类器，将退款请求发送到专用退款子流水线

**正确答案：A**

| 选项 | 原因 |
|---|---|
| **A ✅** | 门禁是确定性的。退款工具在没有已核验 token 的情况下字面上无法执行，8% 失败率降为 0%。 |
| **B ❌** | 系统提示词指令已经存在，更强的措辞仍是概率性的，金融安全要求 0% 失败率。 |
| **C ❌** | Few-shot 示例可提高一致性，但仍是概率性的，失败率降低但不会归零。 |
| **D ❌** | 路由分类器只是把请求送到另一条流水线，并未在该流水线内强制核验，同样的失败仍会发生。 |

### 多诉求请求处理

一个用户请求可能包含多个 concern（账单 + 中断 + 账号问题）：
1. 先拆分
2. 并行调查
3. 统一合成答复

### 结构化人工交接（handoff）

人工升级时要产出自包含摘要：

```
ESCALATION HANDOFF SUMMARY（升级交接摘要）
==========================================
客户 ID：          [ID]
账户已核验：       [是 / 否]
对话摘要：         [2–3 句话描述发生了什么]
根因分析：         [出了什么问题及原因]
退款金额：         [金额（如适用）]
已完成动作：       [已执行步骤列表]
建议下一步：       [具体可执行的建议]
升级原因：         [为何需要人工介入]
```

> **关键：** 人工坐席通常看不到完整聊天记录，handoff 摘要必须完全自解释——所有相关信息都必须包含在摘要本身中。

---

## Task 1.5：Agent SDK Hooks

### PostToolUse Hook（工具执行后、模型看到前）

```text
Tool 执行 -> [PostToolUse Hook 归一化] -> Model 接收标准化结果
```

主要用途：统一不同 MCP 工具输出格式。

```python
from datetime import datetime

def post_tool_use_hook(tool_name: str, tool_result: dict) -> dict:
    if tool_name == "database_query":
        if "created_at" in tool_result:
            tool_result["created_at"] = datetime.fromtimestamp(
                tool_result["created_at"]
            ).isoformat()

        status_map = {1: "active", 2: "suspended", 3: "pending"}
        if "status" in tool_result:
            tool_result["status"] = status_map.get(tool_result["status"], "unknown")

    return tool_result
```

### Pre-Execution Hook（执行前拦截）

```text
Model 请求调用工具 -> [Pre Hook] -> 放行 / 阻断 / 重定向
```

用于：高价值交易阻断、合规前置校验、审批流。

```python
def pre_tool_use_hook(tool_name: str, tool_input: dict):
    if tool_name == "process_refund":
        amount = tool_input.get("amount", 0)
        if amount > 500:
            trigger_human_escalation(...)
            return None  # 阻断执行

    if tool_name == "international_transfer":
        if not compliance_check_passed(tool_input):
            raise ComplianceError("compliance check not completed")

    return tool_input
```

### Hook vs Prompt 决策框架

- 阻断高额退款：Hook
- 跨境转账合规检查：Hook
- 统一工具返回格式：Hook
- 回复语气正式：Prompt
- 回答前先总结：Prompt
- 重构前必须安全扫描：Hook

规则：单次失败有业务/法律风险 -> Hook。

### 练习 1.5：跨境转账合规缺口

**场景：** 处理国际电汇的 Agent 多次在未执行必要的 SWIFT 合规检查的情况下完成转账，每次都导致监管处罚。合规团队建议在系统提示词中添加更详细的合规说明。

应该使用 Hook 还是增强的提示词指令？为什么？

**答案：使用 Pre-Execution Hook。**

这是法律/监管风险场景。监管处罚使任何非零失败率都不可接受。在 `international_transfer` 工具执行前物理阻断——除非已通过合规检查结果存在——可提供 100% 确定性执行。

三次历史失败已证明现有的基于提示词的指导已经不足。更强的提示词指令会降低失败频率，但仍是概率性的。**考试规则：当题目告诉你基于提示词的方案已在生产中失败时，这正是在确认需要程序化强制。**

---

## Task 1.6：任务分解策略

### 考试主要考两类

#### 1) 固定顺序流水线（Prompt Chaining）

```text
输入 -> [分析] -> [交叉校验] -> [报告]
```

适合结构稳定、依赖明确的任务（代码评审、合规流程等）。

优点：可预测、可审计。
缺点：对意外发现不够自适应。

#### 2) 动态自适应分解（Dynamic Adaptive Decomposition）

```text
输入 -> 初始摸底 -> 发现结构 -> 按发现动态生成子任务
```

适合问题边界未知的探索型任务（如给遗留系统补测试）。

优点：能随证据动态调整。
缺点：可预测性较低，时延波动大。

### 辅助模式

- **并行 fan-out / fan-in：** 子任务相互独立且延迟敏感时使用
- **迭代优化（iterative refinement）：** 先出稿，再评估缺口，再定向补洞

### Attention Dilution（注意力稀释）

单轮评审过多文件常见症状：
- 前几文件很细，后面变浅
- 同类问题判定不一致
- 后半程漏明显缺陷

**原因：** 模型的注意力同时分散在过多本地细节上，无法对每个文件和每个跨文件交互保持相同的审查深度。

**正确修复：**
1. 先做逐文件本地分析（每个文件获得专注、一致的审查）
2. 再做跨文件集成分析（单独一遍检查数据流、依赖交互、重复逻辑和全系统模式）

### 分解策略决策表

| 场景 | 最佳策略 | 原因 |
|---|---|---|
| 各阶段有明确依赖关系的代码评审 | 固定顺序流水线 | 步骤可预测；后续阶段依赖前期输出 |
| 给结构未知的遗留代码库补测试 | 动态自适应分解 | 计划必须随依赖和热点的发现而调整 |
| 子主题相互独立的宽泛研究 | 并行 fan-out | 独立任务；延迟敏感 |
| 覆盖率关键的报告 | 迭代优化 | Coordinator 持续填补缺口直到达标 |
| 从同一基线比较两种方案 | `fork_session` | 共享起点，后续分叉独立分析 |

### ⚠️ Task 1.6 考试陷阱

1. **单轮处理太多文件** — 听起来高效，实际表现不一致。
2. **并行化有依赖的步骤** — 当后续分析需要前期输出时，这是错误的。
3. **对探索性工作使用固定流水线** — 当问题在调查过程中演变时，过于死板。
4. **对简单确定性流程使用动态自适应分解** — 对可预测任务过度复杂化。

### 练习 1.6：14 文件评审不一致

**场景：** 开发者效率 Agent 单轮评审 14 个文件。对部分文件的反馈很详细，但遗漏了其他文件中明显的 bug；同样的代码模式在一个文件中被标记为有问题，在另一个文件中却被通过。

**问题：** 根本问题是什么？正确的架构修复是什么？

**答案：**
- **根因：** 单轮评审文件过多导致的 attention dilution。
- **修复：** 改为多阶段架构：先运行专注的逐文件本地分析，再运行单独的跨文件集成检查，用于处理系统级问题和一致性校验。这样每个文件都能获得一致的深度，同时在后续阶段仍能发现跨文件问题。

---

## Task 1.7：会话状态与恢复

### 三种会话管理方式

| 选项 | 机制 | 作用 |
|---|---|---|
| `--resume <session-name>` | 恢复既有会话 | 延续原上下文和工具结果 |
| `fork_session` | 从基线分叉 | 共享起点、后续独立演进 |
| 新会话 + 摘要注入 | clean session + 结构化摘要 | 丢弃陈旧历史，保留有效结论 |

### 何时用哪个

#### `--resume`

适用：
- 数据和代码未变化
- 任务中断后继续
- 会话较新、上下文新鲜

避免：
- 文件已修改（会有 stale tool results）
- 会话过久、上下文漂移

#### `fork_session`

适用：
- 同一基线下比较不同方案
- 需要互不污染的并行评估

避免：
- 只是继续同一任务（该用 resume）
- 需要清空陈旧上下文（该用 fresh start）

#### 新会话 + 摘要注入——清空重来，保留知识

适用：
- 工具结果**已过期**——底层文件、数据或 API 已发生变更
- 超长会话后上下文**已退化**（数千 token 的历史导致推理混乱）
- 希望将复杂的旧调查压缩为清晰、结构化的起点
- 文件自上次会话后**已被修改**，需要在全新基础上重新分析

**正确做法示例：**

```python
# 第一步：将旧会话提炼为结构化摘要
prior_summary = """
## 历史分析摘要（截至 2026-03-26）

### 仓库结构
- 入口文件：app/main.py, workers/processor.py
- 已分析文件数：14
- 语言：Python 3.11

### 关键发现
1. auth/jwt_handler.py 存在时序侧信道漏洞
2. db/pool.py 在高并发下非线程安全
3. 自上次会话以来有三个文件被修改：auth/jwt_handler.py、db/pool.py、tests/test_auth.py

### 待完成工作
- jwt_handler.py 的安全修复尚未审查
- db/pool.py 并发行为缺少测试覆盖
- tests/test_auth.py 已重写——旧分析现已失效

### 已运行工具
- 静态分析：完成（已修改文件除外）
- 依赖审计：完成，无未解决 CVE
- 集成评审：尚未开始
"""

# 第二步：在全新会话中注入摘要
messages = [
    {
        "role": "user",
        "content": f"""
你正在继续一个 Python 代码库的安全和质量评审。
以下是已有发现的结构化摘要，用于引导你的上下文：

{prior_summary}

自上次分析以来，有三个文件已发生变更。请先仅重新分析这三个文件，
然后继续尚未开始的跨文件集成评审。
"""
    }
]
```

**关键原则：** 注入的摘要用准确、精炼的事实替换了陈旧的对话历史。Agent 重新开始，但并非从零开始。

### Stale Context——出错原因与危害（高频考点）

**生产中 stale context 的表现：**

1. 开发者在代码库上运行代码评审 Agent，Agent 生成详细报告。
2. 开发者修复了几个 bug，修改了 3 个文件。
3. 开发者恢复同一会话并提问。
4. Agent 给出矛盾建议——它"知道"（来自旧工具结果）`db/pool.py` 有某个 bug，但开发者已经修复了它。Agent 推荐的修复与新代码冲突。

**根因：** Agent 正在从**缓存的工具结果**推理，这些结果反映的是文件的旧状态，而非当前状态。

**考试规则：**

> **🔴 旧工具结果 = 新会话 + 摘要注入**
>
> 会话在文件被修改后恢复时：
> - **不要**直接 `--resume`（旧工具结果会污染推理）
> - **不要**让 Agent 从头全部重探（浪费时间；丢失有效的旧发现）
> - **应该**带结构化摘要新开会话，标明哪些发现仍有效、哪些文件需要重新分析

### 定向复查写法

**❌ 低效（重新探索一切）：**
```
"代码可能变了，请从头全部重查。"
```

**✅ 正确（定向重分析）：**
```
"自上次分析以来，有三个文件已发生变更：
  1. auth/jwt_handler.py — 已应用时序漏洞修复
  2. db/pool.py — 已应用线程安全补丁
  3. tests/test_auth.py — 已用新测试用例完全重写

所有其他旧发现仍然有效。请只重新分析这三个文件，
然后继续尚未开始的跨文件集成评审。"
```

### 会话管理决策表

| 场景 | 正确选项 | 原因 |
|---|---|---|
| 任务执行中途中断，无文件变更 | `--resume` | 旧上下文仍有效；直接继续 |
| 从共享基线评估两种架构方案 | `fork_session` | 共享起点的独立分支 |
| 文件自上次会话后已修改；旧发现部分有效 | 新会话 + 摘要注入 | 旧工具结果已过期；保留有效发现，丢弃旧的 |
| 极长会话后上下文已退化 | 新会话 + 摘要注入 | 清空重来防止推理混乱 |
| 两个独立评审者评估同一产物 | `fork_session` | 共享基线，独立的发散分析 |
| 长会话暂停隔夜，系统无变更 | `--resume` | 上下文足够新鲜；无过期问题 |

### ⚠️ Task 1.7 考试陷阱

1. **文件修改后使用 `--resume`** — 最常见陷阱。旧工具结果导致建议矛盾。
2. **部分变更后让 Agent 全部重探** — 浪费时间；丢失有效的旧分析。
3. **混淆 `fork_session` 与 `--resume`** — fork 创建独立分支；resume 继续单一线程。
4. **不带摘要的新会话** — 丢弃所有有效的旧发现；Agent 从零开始。
5. **不指定哪些文件发生了变更** — 导致 Agent 要么不必要地重查一切，要么遗漏变更文件。

### 练习 1.7：建议自相矛盾

**场景：** 开发者在 12 个 Python 文件的代码库上运行安全评审 Agent，Agent 识别出三个漏洞。开发者修复了其中两个，修改了 `auth/jwt_handler.py` 和 `db/pool.py`。开发者恢复同一会话并问："我修复的两个漏洞现在解决了吗？"

Agent 给出矛盾建议：同时说"jwt_handler.py 中的时序漏洞似乎已修复"和"你应该对 jwt_handler.py 应用补丁 X"——而补丁 X 正是开发者刚刚应用的修复。

以下哪个是正确的处理方式？

- **A)** 恢复会话并明确告诉 Agent："忽略两个已修改文件的旧工具结果，立即重读它们"
- **B)** 启动无上下文的全新会话——让 Agent 从头发现一切
- **C)** Fork 会话，让一个分支独立验证每个修复
- **D)** 启动新会话，注入旧有效发现的结构化摘要，并指定两个已修改文件做定向重分析

**正确答案：D**

| 选项 | 原因 |
|---|---|
| **A ❌** | 在恢复的会话中尝试选择性覆盖旧工具结果不可靠；旧结果仍留在上下文中并继续干扰推理。 |
| **B ❌** | 丢弃了所有有效的旧发现（第三个漏洞仍然存在且被正确识别）。Agent 会重新发现它，浪费时间且可能在新过程中遗漏。 |
| **C ❌** | `fork_session` 用于从共享基线探索不同方案。两个分支都会继承相同的旧工具结果——并不能解决 stale context 问题。 |
| **D ✅** | 新会话彻底消除了旧工具结果。结构化摘要保留了有效发现（第三个漏洞）和对代码库结构的了解。指定两个已修改文件将重分析精确导向需要的地方。 |

---

## Domain 1 综合模拟题（10 题）

> 使用方式：单选；建议每题 90 秒。

### Q1（Task 1.1）

```python
for block in response.content:
    if block.type == "text":
        return block.text
```

问题：这个终止逻辑的核心 bug 是？

- A) 应改为 `response.stop_reason == "end_turn"`
- B) 应检查 `tool_result`
- C) 应先 append tool result 再检查文本
- D) 应检查最后一个 block

### Q2（Task 1.2）

劳动力市场研究只覆盖白领，缺蓝领/制造/农业/零工。根因组件是？

- A) Web Search 子 Agent
- B) Academic 子 Agent
- C) Coordinator
- D) Synthesis 子 Agent

### Q3（Task 1.2）

A 子 Agent 发现监管矛盾，需要 B 子 Agent 交叉验证。Coordinator 应做什么？

- A) 什么也不做，B 会自动继承
- B) 在给 B 的 prompt 显式写入 A 的发现
- C) 让 B 直接查询 A
- D) 让 B 轮询共享状态

### Q4（Task 1.3）

4 个独立任务串行导致 40 秒延迟，如何降时延？

- A) 提高 `max_tokens`
- B) 单轮响应里发出全部 `Task` 调用并行执行
- C) 用 `fork_session`
- D) 复用第一个任务缓存

### Q5（Task 1.4）

> £10,000+ 提现有 4% 未经过反欺诈审核；prompt 已加强两次。

最佳修复：

- A) 加 few-shot
- B) 加路由分类器
- C) 加程序化门禁：无 fraud-review token 禁止提现工具执行
- D) 再强化系统 prompt

### Q6（Task 1.5）

三套 MCP 工具返回时间/状态格式不一致，模型经常误读，最佳方案？

- A) 在 prompt 写解释
- B) PostToolUse Hook 统一格式
- C) 让每个子 Agent 自己做归一化
- D) 重写全部 MCP 工具

### Q7（Task 1.5）

跨境转账多次绕过 SWIFT 合规检查，为什么只改 prompt 不够？正确方案是？

- A) prompt 不参与工具决策；改工具定义
- B) 生产失败已证明概率方案不足，应使用 Pre-Execution Hook 阻断
- C) prompt 无法引用外部系统；改 Post hook 审计
- D) 继续强化 prompt 即可

### Q8（Task 1.6）

遗留代码库未知结构，需补测试，最佳分解策略？

- A) 固定顺序流水线
- B) 动态自适应分解
- C) 按文件全并行
- D) 直接迭代补测

### Q9（Task 1.6）

18 文件单轮评审：前细后粗且漏安全问题。根因与修复？

- A) token 不够，增大上限
- B) attention dilution；改为 per-file + cross-file 两阶段
- C) 后半段没调用安全工具
- D) 打乱文件顺序

### Q10（Task 1.7）

重构 3 个服务后直接 resume 旧会话，建议自相矛盾，正确做法？

- A) 继续 resume 并让 Agent 忽略旧结果
- B) fork 出分支验证
- C) 新会话但完全无上下文
- D) 新会话 + 结构化摘要 + 定向重查 3 个改动服务

---

## 答案速查

- Q1: A
- Q2: C
- Q3: B
- Q4: B
- Q5: C
- Q6: B
- Q7: B
- Q8: B
- Q9: B
- Q10: D

---

## 评分建议

| 分数 | 准备度 | 建议 |
|---|---|---|
| 10/10 | 掌握优秀 | 进入 Domain 2 |
| 8-9/10 | 基本就绪 | 回看错题对应陷阱后进入 Domain 2 |
| 6-7/10 | 尚未稳定 | 重读相关 Task + 复做练习 |
| <=5/10 | 基础薄弱 | 全文重学，重点看“考试陷阱” |

错题映射：
- Q1 -> Task 1.1
- Q2/Q3 -> Task 1.2
- Q4 -> Task 1.3
- Q5 -> Task 1.4
- Q6/Q7 -> Task 1.5
- Q8/Q9 -> Task 1.6
- Q10 -> Task 1.7

---

## Domain 1 综合实战（Capstone）

### 任务目标

构建一个 coordinator + 两个子 Agent（web search / document analysis）的系统，要求：
- 正确 agentic loop
- 结构化上下文传递
- 程序化前置门禁
- PostToolUse 归一化 Hook
- 用多诉求查询做端到端验证

### 架构示意

```text
                    ┌──────────────────────────┐
                    │     COORDINATOR AGENT     │
                    └────────────┬─────────────┘
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │  Web Search      │  │  Document        │  │  Synthesis       │
   │  Subagent        │  │  Analysis        │  │  Subagent        │
   └──────────────────┘  └──────────────────┘  └──────────────────┘
          │                      │
   [PostToolUse Hook 归一化] [PostToolUse Hook 归一化]
          └──────────┬───────────┘
                     ▼
           [前置门禁：两个子任务都完成才允许 synthesis]
```

### 实现检查清单

1) **Loop 正确性（Task 1.1）**
- [ ] 以 `stop_reason == "end_turn"` 终止
- [ ] 工具结果追加回历史
- [ ] 仅把迭代上限作为保险（如 20）

2) **Coordinator + 两个子 Agent（Task 1.2）**
- [ ] coordinator 做拆解
- [ ] 子 Agent 各自 prompt / 工具权限分离
- [ ] `allowedTools` 包含 `"Task"`
- [ ] 两个 `Task` 同轮发出并行执行
- [ ] 子 Agent 不直接互通

3) **结构化上下文传递（Task 1.3）**
- [ ] 子 Agent prompt 包含所需全部上下文
- [ ] synthesis 接收 claim-source 元数据
- [ ] 元数据至少含 `claim`、`source_url` 或 `document_name`、`page`
- [ ] synthesis prompt 包含质量标准

4) **程序化门禁（Task 1.4）**
- [ ] synthesis 工具前置检查两个 completion token
- [ ] 用代码硬阻断，不依赖 prompt
- [ ] 阻断时报可理解错误

5) **PostToolUse Hook（Task 1.5）**
- [ ] 每次工具返回后触发
- [ ] Unix 时间戳 -> ISO 8601
- [ ] 数字状态码 -> 可读文本
- [ ] 模型只看到归一化结果

6) **多诉求端到端测试（综合）**
- [ ] 用“EU AI 监管 + OpenAI/Google/Anthropic 合规情况”做测试输入
- [ ] 验证两个子 Agent 并行
- [ ] 人为禁用一个子 Agent，验证门禁能阻断
- [ ] 验证 hook 至少转换一个 timestamp 和一个 status code
- [ ] 最终报告引用可追溯到结构化元数据

---

## 参考实现骨架（中文注释版）

> 以下保留原始 Python 结构，便于你直接落地。你可以在本仓库新增 `src/domain1_capstone.py` 后填充 TODO。

```python
import anthropic
import json
from datetime import datetime

client = anthropic.Anthropic()

# PostToolUse Hook
# 用于统一工具输出，避免模型误读异构格式

def post_tool_use_hook(tool_name: str, tool_result: dict) -> dict:
    for field in ["created_at", "published_at", "last_modified"]:
        if field in tool_result and isinstance(tool_result[field], (int, float)):
            tool_result[field] = datetime.fromtimestamp(tool_result[field]).isoformat()

    status_map = {1: "active", 2: "suspended", 3: "pending", 4: "archived"}
    if "status" in tool_result and isinstance(tool_result["status"], int):
        tool_result["status"] = status_map.get(tool_result["status"], "unknown")

    return tool_result


class SessionState:
    def __init__(self):
        self.web_search_complete = False
        self.document_analysis_complete = False
        self.web_search_results = None
        self.document_analysis_results = None

    def mark_web_search_complete(self, results):
        self.web_search_complete = True
        self.web_search_results = results

    def mark_document_analysis_complete(self, results):
        self.document_analysis_complete = True
        self.document_analysis_results = results

    def synthesis_gate(self):
        if not self.web_search_complete:
            raise RuntimeError("Prerequisite gate: web search not completed")
        if not self.document_analysis_complete:
            raise RuntimeError("Prerequisite gate: document analysis not completed")
        return True


def build_synthesis_prompt(original_query: str, web_results: list, doc_results: list) -> str:
    return f"""
请综合以下研究结果并生成报告。

## 原始问题
{original_query}

## Web Search 结果
{json.dumps(web_results, indent=2, ensure_ascii=False)}

## Document Analysis 结果
{json.dumps(doc_results, indent=2, ensure_ascii=False)}

## 质量要求
- 覆盖主要结论
- 每条事实给出来源（URL 或文档+页码）
- 标记证据薄弱区域
- 输出结构化报告（摘要、详细发现、来源清单）
"""


def run_agentic_loop(messages, tools, system_prompt, session_state, max_iterations=20):
    iteration = 0

    while iteration < max_iterations:
        iteration += 1

        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            return next((b.text for b in response.content if hasattr(b, "text")), "")

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    raw_result = execute_tool(block.name, block.input, session_state)
                    normalised = post_tool_use_hook(block.name, raw_result)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(normalised, ensure_ascii=False),
                    })

            messages.append({"role": "user", "content": tool_results})
            continue

        raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")

    raise RuntimeError(f"Safety cap reached: {max_iterations}")


def execute_tool(tool_name, tool_input, session_state):
    # 这里放真实工具逻辑；以下为示例桩
    if tool_name == "web_search":
        results = [{"claim": "placeholder", "source_url": "https://example.com", "date": "2026-03-27", "document": None, "page": None}]
        session_state.mark_web_search_complete(results)
        return {"findings": results, "status": 1, "published_at": 1711497600}

    if tool_name == "document_analysis":
        results = [{"claim": "placeholder", "source_url": None, "date": None, "document": "example.pdf", "page": 1}]
        session_state.mark_document_analysis_complete(results)
        return {"findings": results, "status": 1, "created_at": 1711497600}

    if tool_name == "synthesise_report":
        session_state.synthesis_gate()
        prompt = build_synthesis_prompt(
            tool_input["original_query"],
            session_state.web_search_results,
            session_state.document_analysis_results,
        )
        return {"report": f"TODO: call synthesis subagent with prompt\n{prompt}"}

    raise ValueError(f"Unknown tool: {tool_name}")
```

---

## 最终检查（Passing 标准）

1. `text + tool_use` 同时出现时，loop 不会提前结束
2. 并行子任务启动时间几乎同时（例如 <1s）
3. 人为禁用一个子任务时，synthesis 被门禁明确阻断
4. 看到 hook 把 `1711497600` 转成 ISO 时间字符串
5. 最终报告至少有一条可追溯引用（来自结构化 metadata）

---

完成后，如果模拟题 >= 8/10 且 Capstone 清单全部打勾，再进入 Domain 2。
