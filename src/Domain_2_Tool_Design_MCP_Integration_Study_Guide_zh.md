# Domain 2：工具设计与 MCP 集成 —— 完整学习指南（中文）

**Claude Certified Architect (Foundations) | 考试权重：18%**

> 本中文版保留原教程结构、考点逻辑与代码示例；术语以中文为主，关键技术名词保留英文（如 `tool_choice`、`is_error`、MCP）。

---

## 如何使用本指南

本指南采用“先讲解、再检验”的结构。针对 5 个任务点，每个部分都包含：
1. **概念讲解**（含生产场景示例）
2. **考试陷阱**（高频误区与干扰项）
3. **练习场景**（附标准答案）
4. **与下一任务点的衔接**

完成全部 5 个任务点后，你会做一套 **7 题综合模拟题**，用于定位薄弱项并回补。

---

## 自评：你从哪里开始？

| 评级 | 描述 | 建议学习方式 |
|---|---|---|
| **Novice** | 没有 MCP 或工具设计经验 | 逐字阅读全篇；先独立做题再看答案。 |
| **Intermediate** | 在 IDE 或 Agent 中使用过 MCP 工具 | 重点看“考试陷阱”和练习场景；概念可略读。 |
| **Advanced** | 构建过 MCP Server 或设计过 Agent 工具 API | 先做练习和模拟题，再针对薄弱点补读讲解。 |

---

## 域概览

- **考试权重：18%**
- **及格线：720 / 1000**
- **题型：场景化单选（1 正确 + 3 干扰）**
- **高频场景：**客户支持解决 Agent · 多 Agent 研究系统 · 开发效率工具

> **Domain 2 核心考试哲学**
>
> 1. **先做低成本高收益修复**：先改工具描述，再上路由分类器
> 2. **先做权限收敛再放开**：先用受限工具，再给通用能力
> 3. **先用社区 Server 再自建**：优先复用 MCP 社区生态
> 4. **始终修根因**：根因是描述差，就改描述，不要绕系统补丁

---

## Task 2.1：工具接口设计

### 工具描述是“选工具”的核心机制

这是 Domain 2 最重要的概念。

当 Claude 决定调用哪个工具时，主要依据是你在 API 请求中提供的 **tool description（工具描述）**。它不是给人看的文档，而是给模型做选择的指令。

描述过于简略时，模型无法区分相似工具，就会误路由、猜测、选错。

这不是“可优化项”，而是**核心机制本身**。

---

### 好的工具描述应包含什么

生产级工具描述要回答 5 个问题：

| 组件 | 作用 | 示例 |
|---|---|---|
| **核心目的** | 一句话说明工具做什么 | “给定订单 ID，返回当前状态、承运商和预计送达日期。” |
| **输入规范** | 参数类型、格式、约束 | “`order_id`: string，格式 `ORD-XXXXX`（X 为数字），必填。” |
| **示例查询** | 该工具适用的自然语言请求 | “当用户问：‘我的订单到哪了？’‘订单 #12345 何时送达？’时使用。” |
| **边界与限制** | 工具不能做什么 | “不返回订单明细或价格；如需明细请用 `get_order_details`。” |
| **显式分工** | 与相似工具如何区分 | “仅用于订单状态查询；客户资料请用 `get_customer_profile`。” |

**❌ 差的描述（过于简略）：**
```json
{
  "name": "get_customer",
  "description": "Retrieves customer information."
}
```

```json
{
  "name": "lookup_order",
  "description": "Retrieves order information."
}
```

这两条几乎不可区分，模型只能“猜”。

**✅ 好的描述（可区分、可执行）：**
```json
{
  "name": "get_customer",
  "description": "Retrieves a customer's profile information including name, email address, phone number, and account status. Input: customer_id (string, format 'CUST-XXXXX') OR email (string, valid email format). Use this tool when the customer asks about their account details, contact information, or membership status. Do NOT use this tool for order tracking, billing, or purchase history — use lookup_order or get_billing_history instead."
}
```

```json
{
  "name": "lookup_order",
  "description": "Retrieves the current status, shipping carrier, tracking number, and estimated delivery date for a specific order. Input: order_id (string, format 'ORD-XXXXX'). Use this tool when the customer asks about order status, shipping progress, delivery timing, or package tracking. Handles queries like 'Where is my order?', 'Has order #12345 shipped?', 'When will my package arrive?'. Do NOT use this tool for customer profile information — use get_customer instead."
}
```

现在模型拿到的是“明确分工信号”：订单状态 -> `lookup_order`；资料查询 -> `get_customer`。

---

### 误路由问题（考试高频）

> **⚠️ 考试重点（样题 Q2）**

**场景：**“check the status of order #12345” 被错误路由到 `get_customer`，而不是 `lookup_order`。两者描述都很简短。

| 方案 | 结论 |
|---|---|
| **A) 在 system prompt 增加 few-shot** | ❌ 治标不治本，且 token 成本高。 |
| **B) 构建路由分类器** | ❌ 对“描述问题”来说过度工程化。 |
| **C) 扩展工具描述，写清用例与边界** | ✅ **正确答案**，低成本高收益，直击根因。 |
| **D) 合并成一个工具** | ❌ 成本更高且降低接口清晰度。 |

**考试原则：**先选“最简单且能修根因”的方案。

---

### 工具拆分（Tool Splitting）

当工具过于泛化时，应拆成“单一职责”的工具，输入输出契约明确。

**❌ 过泛工具：**
```json
{
  "name": "analyze_document",
  "description": "Analyses a document."
}
```

**✅ 拆分后：**
```json
{
  "name": "extract_data_points",
  "description": "Extracts structured numerical data points and statistics from a document. Input: document_id (string), data_categories (array of strings, e.g., ['revenue', 'headcount', 'dates']). Returns a JSON array of {category, value, unit, page_number, confidence_score} objects. Use when you need specific facts or figures from a document. Do NOT use for summaries or verification — use summarise_content or verify_claim_against_source."
}
```

```json
{
  "name": "summarise_content",
  "description": "Generates a structured summary of a document's key arguments and conclusions. Input: document_id (string), max_length (integer, default 500 words), focus_areas (optional array of strings). Returns {executive_summary, key_arguments[], conclusions[], limitations_noted[]}. Use when you need an overview of what a document says. Do NOT use for data extraction or claim verification."
}
```

```json
{
  "name": "verify_claim_against_source",
  "description": "Checks whether a specific factual claim is supported, contradicted, or unaddressed by a given source document. Input: claim (string, the assertion to verify), document_id (string). Returns {verdict: 'supported'|'contradicted'|'not_addressed', relevant_excerpts[], confidence_score}. Use when you need to fact-check a claim against a source. Do NOT use for summarisation or data extraction."
}
```

拆分后的收益：
- 单一职责明确
- 输入约束清晰
- 输出结构可预测
- 工具边界可执行

---

### 与系统提示词（system prompt）的相互作用

> **⚠️ 常见考试陷阱**

系统提示词中的关键词规则，可能覆盖你精心写好的工具描述。

例如 system prompt 写着：
```text
When the user mentions an "order", always look up the customer first to verify identity.
```

这会导致只要出现 “order”，模型就先调用 `get_customer`，即使用户只要查物流。

**修复顺序：**
1. 先改好工具描述
2. 再排查 system prompt 冲突（尤其是 `always` / `never` / `first` 这类硬规则）

---

### 🧪 练习场景 2.1：订单查询被误路由

**场景：**

- `get_customer`："Retrieves customer information."
- `lookup_order`："Retrieves order information."

用户说：“Check the status of order #12345.”，模型却稳定调用 `get_customer`。

备选方案：
- **A)** 加 few-shot
- **B)** 上路由分类器
- **C)** 扩展两者描述（用例+输入格式+显式边界）
- **D)** 合并为 `get_info`

应先做哪一个？

<details>
<summary>▶ 查看答案</summary>

**正确答案：C**

根因是描述不可区分。改描述是：
- **低成本**（只改两段字符串）
- **高杠杆**（直接提升选工具准确率）
- **无新增基础设施**（不引入分类器）

若 C 后仍误路由，再查 system prompt 是否有冲突规则。

</details>

---

*衔接 Task 2.2：工具选对之后，要处理“调用失败时如何让 Agent 正确恢复”。*

---

## Task 2.2：结构化错误响应

### MCP 的 `is_error` 标志

工具失败时，必须让 Agent 明确知道“这次调用失败”。MCP 工具结果里用 `is_error` 布尔值表达：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "content": "...",
  "is_error": true
}
```

当 `is_error: true` 时，Claude 会走恢复策略；若不标，模型可能把错误文本当成功数据继续推理。

---

### 四类错误（必考）

| 类别 | 示例 | 可重试？ | 正确恢复动作 |
|---|---|---|---|
| **Transient（瞬时）** | 超时、服务短暂不可用、限流、网络抖动 | ✅ | 指数退避重试 |
| **Validation（校验）** | 输入格式错误、缺字段、越界 | ✅（修输入后） | 修正输入再试；同输入重试无意义 |
| **Business（业务规则）** | 退款超政策阈值、账户冻结、商品下架 | ❌ | 走替代流程或升级处理 |
| **Permission（权限）** | 无权限、角色不足、凭证过期、IP 被阻断 | ❌（当前凭证下） | 升级权限或更换凭证 |

---

### 错误要结构化，不要只给字符串

**❌ 差示例：**
```json
{
  "is_error": true,
  "content": "Error: something went wrong"
}
```

**✅ 好示例：**
```json
{
  "is_error": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "Order service timed out after 5000ms.",
    "suggestedAction": "Retry after 2 seconds.",
    "attemptNumber": 1,
    "maxRetries": 3
  }
}
```

```json
{
  "is_error": true,
  "content": {
    "errorCategory": "business",
    "isRetryable": false,
    "message": "Refund of £450 exceeds the maximum automated refund limit of £200.",
    "customerFriendlyMessage": "I'm sorry, but refunds above £200 require manager approval. I'll escalate this to a manager who can process it within 24 hours.",
    "suggestedAction": "Escalate to manager queue with refund details.",
    "policyReference": "REFUND-POLICY-2024-v3, Section 4.2"
  }
}
```

业务错误建议提供 `customerFriendlyMessage`，方便 Agent 直接对用户解释。

---

### 🔴 关键区分：访问失败 vs 有效空结果

> **⚠️ 考试直接考这一点**

| 结果类型 | 实际含义 | Agent 的正确行为 |
|---|---|---|
| **访问失败** | 数据源没访问到（超时/鉴权失败等） | 判断是否重试，数据可能存在 |
| **有效空结果** | 查询成功执行，但匹配为 0 | 不要重试，答案就是“无结果” |

**访问失败：**
```json
{
  "is_error": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "Database connection timed out after 5000ms.",
    "resultCount": null
  }
}
```

**有效空结果：**
```json
{
  "is_error": false,
  "content": {
    "results": [],
    "resultCount": 0,
    "queryExecuted": true,
    "message": "No customer found matching email 'nonexistent@example.com'."
  }
}
```

混淆两者会导致严重后果：
- 把访问失败当空结果：误告“用户不存在”
- 把空结果当失败：重复重试并无谓升级

---

### 多 Agent 系统中的错误传播

规则如下：
1. **子 Agent 先本地恢复**（瞬时错误先重试）
2. **仅传播无法本地解决的错误**
3. **传播时附带部分成功结果**，便于协调器继续决策

**良好传播示例：**
```json
{
  "status": "partial_failure",
  "completedTasks": [
    {"query": "solar energy market 2024", "results": ["..."]},
    {"query": "wind energy capacity Europe", "results": ["..."]}
  ],
  "failedTasks": [
    {
      "query": "geothermal energy economics",
      "errorCategory": "transient",
      "attemptsExhausted": 3,
      "lastError": "Service unavailable (HTTP 503)"
    }
  ],
  "recommendation": "Partial results available. Re-assign failed query or proceed with available data."
}
```

---

### 🧪 练习场景 2.2：无限重试循环

**场景：**

```json
{
  "results": [],
  "message": ""
}
```

Agent 把它当潜在失败，重试 3 次后升级人工。最终发现是“用户确实不存在”。

1. 根因是什么？
2. 响应应该怎么改？

<details>
<summary>▶ 查看答案</summary>

**根因：**响应语义不完整，没区分“查询成功但 0 条”与“查询失败”。

**正确结构：**
```json
{
  "is_error": false,
  "content": {
    "results": [],
    "resultCount": 0,
    "queryExecuted": true,
    "message": "No customer account found matching email 'newuser@example.com'. This may be a new user without an existing account."
  }
}
```

关键是明确告诉模型：调用成功、结果为空、无需重试。

</details>

---

*衔接 Task 2.3：结果和错误都结构化之后，要进一步决定“哪些 Agent 拥有哪些工具”。*

---

## Task 2.3：工具分配与 `tool_choice`

### 工具过载问题（考试高频）

> **⚠️ 直接考点**

给单个 Agent 太多工具，会明显降低选工具可靠性。可用工具越多，模型越容易误选。

**建议范围：每个 Agent 约 4–5 个工具，并与角色强绑定。**

| Agent 角色 | 应给工具 | 不应给工具 |
|---|---|---|
| **Synthesis Agent** | `compile_report`, `format_citations`, `verify_fact`, `assess_coverage` | ~~`web_search`~~, ~~`fetch_url`~~, ~~`query_database`~~ |
| **Web Search Agent** | `web_search`, `extract_page_content`, `evaluate_source_credibility` | ~~`summarise_content`~~, ~~`compile_report`~~, ~~`format_citations`~~ |
| **Document Analysis Agent** | `read_document`, `extract_data_points`, `summarise_content`, `verify_claim_against_source` | ~~`web_search`~~, ~~`send_email`~~, ~~`create_ticket`~~ |

原则：角色做角色该做的事，工具按职责最小集分配。

---

### `tool_choice` 三种配置

#### 1) `{"type": "auto"}`（默认）

```json
{
  "tool_choice": {"type": "auto"}
}
```

由模型决定是否调用工具，适合绝大多数常规请求。

#### 2) `{"type": "any"}`（必须调用某个工具）

```json
{
  "tool_choice": {"type": "any"}
}
```

强制调用工具，但由模型选“哪个工具”。

适用：你需要保证输出来自某个结构化工具，但允许模型自主选具体格式。

#### 3) `{"type": "tool", "name": "..."}`（强制指定工具）

```json
{
  "tool_choice": {"type": "tool", "name": "extract_metadata"}
}
```

必须调用指定工具，适合“流程第一步必须执行某动作”。

```python
# 第一步：强制抽取元数据
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_metadata"},
    messages=messages
)

# 后续步骤：恢复自动选择
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=tools,
    tool_choice={"type": "auto"},
    messages=messages
)
```

---

### 跨角色“受限能力”注入（Scoped Cross-Role Tools）

> **⚠️ 样题 Q9 模式**

当某子 Agent 高频需要“另一个角色”的能力时，完全经协调器转发会带来明显往返开销。

典型链路：
```
Synthesis Agent 要验事实
  -> 交回 Coordinator
  -> Coordinator 路由 Verification Agent
  -> Verification 完成返回 Coordinator
  -> Coordinator 再回传 Synthesis Agent
```

如果 85% 是“单步简单核验”，可以给 Synthesis Agent 一个**受限版** `verify_fact`，复杂 15% 再回协调器。

```json
{
  "name": "verify_fact",
  "description": "Performs a simple factual lookup to verify a single claim. Input: claim (string), source_hint (optional string). Returns {verified: boolean, source: string, confidence: float}. Use ONLY for straightforward factual claims that can be verified with a single lookup. For complex claims requiring multi-step reasoning, cross-referencing multiple sources, or domain expertise, return the claim to the coordinator for full verification."
}
```

这类工具必须满足：
- **Constrained**：只覆盖简单场景
- **Bounded**：写清何时不要用
- **降往返**：优化主流路径
- **保兜底**：复杂场景仍走全流程

---

### 用“受限替代”替换“通用强工具”

**❌ 风险大：**
```json
{
  "name": "fetch_url",
  "description": "Fetches content from any URL."
}
```

**✅ 更安全：**
```json
{
  "name": "load_document",
  "description": "Loads a document from the approved document store. Input: document_id (string, format 'DOC-XXXXX'). Only accepts document IDs from the internal document catalogue. Cannot fetch arbitrary URLs. Returns the document text content and metadata."
}
```

原则：给能力，不给过度权限。

---

### 🧪 练习场景 2.3：延迟升高 40%

**场景：**

- Coordinator Agent：统一编排
- Web Search Agent：`web_search` + `extract_page_content`
- Document Analysis Agent：`read_document` + `summarise_content` + `extract_data_points`
- Synthesis Agent：`compile_report` + `format_citations`

Synthesis 在写报告时频繁核验事实，全部经 Coordinator 中转，导致每次核验多 2–3 次往返，整体延迟 +40%。数据表明 85% 是单步核验。

方案：
- **A)** 把 Web Search 的全部工具给 Synthesis
- **B)** 给 Synthesis 增加受限 `verify_fact`，复杂核验仍走 Coordinator
- **C)** 预缓存所有可能事实
- **D)** 提升 Coordinator 处理速度

<details>
<summary>▶ 查看答案</summary>

**正确答案：B**

A 会破坏角色边界并增加误选风险；C 不现实；D 治标不治本。B 以最小改动优化高频路径，同时保留复杂场景的完整编排。

</details>

---

*衔接 Task 2.4：理解工具分配后，还要会配置和发现 MCP Server 能力。*

---

## Task 2.4：MCP Server 集成

### MCP 是什么？

**Model Context Protocol (MCP)** 是让 Agent 发现并使用外部 Server 工具的标准协议。

你不必把所有工具硬编码在 API 请求里；连接 MCP Server 后，Agent 在连接阶段即可发现其暴露的工具。

可把它理解为“给模型用的 USB 接口”：插上就能识别能力。

---

### 配置作用域层级（考试会考）

| 层级 | 文件 | 进版本库？ | 团队共享？ | 适用场景 |
|---|---|---|---|---|
| **项目级** | 项目根目录 `.mcp.json` | ✅ | ✅ | 团队统一集成（Jira、GitHub、共享库） |
| **用户级** | `~/.claude.json` | ❌ | ❌ | 个人工具、实验配置、个人偏好 |

关键行为：项目级和用户级已配置 Server 的工具会在连接时一起被发现，非懒加载。

**项目级 `.mcp.json` 示例：**
```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-jira"],
      "env": {
        "JIRA_URL": "https://mycompany.atlassian.net",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

---

### 环境变量展开

> **⚠️ 配置细节考点**

`.mcp.json` 支持 `${VAR_NAME}` 展开，用于避免密钥入库：

```json
{
  "env": {
    "GITHUB_TOKEN": "${GITHUB_TOKEN}",
    "DATABASE_URL": "${DATABASE_URL}"
  }
}
```

每个开发者在本地环境中设置变量；仓库里只保留变量引用。

---

### MCP Resources（资源）

MCP Server 除工具外，还可暴露**只读资源**，让 Agent 先理解可用数据全貌，再发起精准调用。

示例：
- Jira 工单摘要（状态、负责人、优先级）
- 文档层级目录
- 数据库 schema 信息

资源价值：减少“探索性盲查”调用，降低延迟和成本。

---

### 自建 vs 复用：如何决策

> **⚠️ 考试倾向非常明确：优先社区 Server**

| 选择 | 适用条件 | 示例 |
|---|---|---|
| **用社区 MCP Server** | 标准服务、已有成熟实现 | Jira / GitHub / Slack / PostgreSQL / Google Drive |
| **自建 MCP Server** | 社区方案覆盖不了团队专属需求 | 内部部署流水线、专有数据仓库、公司特定审批流 |

**考试默认答案：**先用社区。只有“明确缺口”才自建。

典型自建理由：
- 内部认证机制社区版不支持
- 需要工具层强制脱敏（如自动 PII Redaction）
- 需把多 API 操作封装为单原子动作
- 需做专有领域转换逻辑

---

### 提升 MCP 工具描述质量

MCP 工具会和内置工具竞争“被选中”概率。若 MCP 描述太弱，模型会优先内置工具。

例如 MCP 暴露了 `search_codebase`，描述只有 “Searches code.”，那模型可能更偏向内置 `Grep`。

应强化描述，突出差异能力：

```json
{
  "name": "search_codebase",
  "description": "Performs semantic code search across the entire repository with cross-reference resolution. Unlike text-based grep, this tool understands code structure: it can find implementations of an interface, callers of a function across modules, and usages of a type including aliased imports. Input: query (string, natural language or code pattern). Use this tool for structural code queries. Use the built-in Grep tool only for literal string matching."
}
```

---

### 🧪 练习场景 2.4：是否要自建 Jira Server？

**场景：**团队要把 Claude 开发助手接入 Jira。一位资深开发建议花两周自建 MCP Server。

1. 为什么应先评估社区 Server？
2. 什么情况下才值得自建？

<details>
<summary>▶ 查看答案</summary>

**先评估社区 Server 的原因：**
- Jira 社区 MCP Server 已存在且维护中
- 覆盖常见 CRUD、搜索、流转
- 被广泛使用，稳定性更高
- API 变更通常有持续跟进
- 两周自建成本高且不保证更优

**自建合理场景：**
- 需联动内部审批（如 P1 需同步触发内部告警）
- 需强制脱敏后再返回给 Agent
- 需做社区版本不支持的复杂字段映射
- 需与内部数据库写入组成原子事务

原则：先复用，发现明确缺口后“补缺口”而非“全量重造”。

</details>

---

*衔接 Task 2.5：除 MCP 工具外，Claude 内置文件工具也常考，尤其是 Grep/Glob 区分。*

---

## Task 2.5：内置工具（Built-In Tools）

### Grep vs Glob：本质区别（高频考点）

| 工具 | 搜索对象 | 适用任务 | 示例 |
|---|---|---|---|
| **Grep** | 文件**内容** | 找调用方、报错文本、import 语句、TODO | “找所有调用 `processPayment()` 的文件” |
| **Glob** | 文件**路径/文件名** | 按命名规则找文件 | “找所有 `*.test.tsx` 或 `tsconfig*.json`” |

口诀：**Grep 查文件里写了什么；Glob 查文件叫什么。**

**Grep 示例：**
```text
Grep "processPayment(" across codebase
-> src/checkout.ts:45, src/subscription.ts:112, src/refund.ts:23
```

**Glob 示例：**
```text
Glob "**/*.test.tsx"
-> src/checkout.test.tsx, src/subscription.test.tsx, src/components/Button.test.tsx
```

常见陷阱：题目问“谁 import 了 `legacyAuth`”，错误选项用 Glob（找名字），正确应是 Grep（搜内容）。

---

### Read / Write / Edit：修改文件的优先级

#### `Edit` 优先

- 基于唯一锚点做**精确局部修改**
- 快、准、token 成本低

示例：
```text
Edit src/config.ts
Find: "maxRetries: 3"
Replace: "maxRetries: 5"
```

#### `Edit` 失败场景：锚点不唯一

```python
result = None  # line 12
result = None  # line 45
result = None  # line 78
```

若要把 `result = None` 改成 `result = []`，会出现歧义。

#### 兜底：`Read + Write`

`Edit` 因不唯一失败时：
1. `Read` 全文件
2. 内存中修改
3. `Write` 覆盖回写

更重，但最可靠。

**推荐层级：**
```text
先尝试 Edit（快且精确）
  -> 若失败（匹配不唯一）
再用 Read + Write（重但稳）
```

---

### 增量式理解代码库（考试会考流程）

**❌ 错误做法：**先把很多文件全读一遍（上下文预算杀手）

**✅ 正确做法：**
1. 先用 Grep 找入口
2. Read 入口文件看导入关系
3. 只沿相关链路继续 Read
4. 必要时再用 Grep 追踪跨模块调用

示例：
```text
Grep "export function processPayment" -> src/payments/processor.ts
Read src/payments/processor.ts
  -> imports validateCard from './validation'
  -> imports chargeAccount from '../billing/charge'
Read src/payments/validation.ts
Read src/billing/charge.ts
Skip src/audit/logger.ts (与当前问题无关)
```

原则：Discover -> Read -> Discover -> Read，避免一次性装载无关内容。

---

### 跨包装模块（barrel exports）追踪

```typescript
// src/payments/index.ts
export { processPayment } from './processor';
export { validateCard } from './validation';
```

查 `processPayment` 调用方时，首次 Grep 可能只命中 re-export。

正确路径：
1. Grep `processPayment`（定位定义与转发）
2. Grep `from.*payments` / `require.*payments`（找消费方）
3. Read 消费文件确认是否真的调用该函数

---

### 🧪 练习场景 2.5：定位废弃函数调用

**任务：**
1. 找所有调用 `legacyEncrypt()` 的文件
2. 找这些调用方对应的测试文件

<details>
<summary>▶ 查看答案</summary>

**Step 1：Grep 找调用方（内容搜索）**
```text
Grep "legacyEncrypt(" across codebase
-> src/auth/login.ts:34
-> src/auth/token.ts:67
-> src/data/export.ts:112
-> src/crypto/legacy.ts:5 (定义，可跳过)
```

**Step 2：Glob 找测试文件（路径搜索）**
```text
Glob "**/*login*.test.*"  -> src/auth/login.test.ts
Glob "**/*token*.test.*"  -> src/auth/token.test.ts
Glob "**/*export*.test.*" -> src/data/export.test.ts, src/api/export.test.ts
```

若命中多个测试文件，再 Read 确认谁对应目标模块。

</details>

---

*以上完成 5 个任务点。下面用 7 题模拟题检验掌握程度。*

---

## Domain 2 综合模拟题（7 题）

### Question 1（Task 2.1：工具描述）

客服 Agent 有 3 个工具：
- `get_account_info` — "Gets account information"
- `get_billing_info` — "Gets billing information"
- `get_subscription_info` — "Gets subscription information"

用户问：“Why was I charged £49.99 last month?” 模型却调用了 `get_account_info`。最有效的第一步修复是？

- **A)** 增加前置分类器，把 billing 关键词路由到 `get_billing_info`
- **B)** 扩展三个工具描述（用例、示例查询、输入格式、显式边界）
- **C)** 合并为 `get_customer_data(type)`
- **D)** 在 system prompt 加 few-shot

<details>
<summary>▶ 查看答案</summary>

**正确答案：B**

根因是描述无法区分。B 最低成本、最高杠杆、直击根因。

</details>

---

### Question 2（Task 2.1：System Prompt 冲突）

改好工具描述后，模型仍在所有订单请求前先调 `get_customer_profile`。下一步该查什么？

- **A)** 模型能力不够，换更强模型
- **B)** system prompt 可能有覆盖规则（如“提到 order 一律先验身份”）
- **C)** 描述再写到 500 字
- **D)** 继续拆更多工具

<details>
<summary>▶ 查看答案</summary>

**正确答案：B**

关键词硬规则可能覆盖工具描述，需先排查提示词冲突。

</details>

---

### Question 3（Task 2.2：错误分类）

工具尝试处理 £450 退款，但自动退款上限是 £200。该错误类别与 `isRetryable` 应该是？

- **A)** Validation，`true`
- **B)** Business，`false`
- **C)** Transient，`true`
- **D)** Permission，`false`

<details>
<summary>▶ 查看答案</summary>

**正确答案：B**

这是业务规则限制，不是输入格式问题，也不是服务抖动或权限缺失。

</details>

---

### Question 4（Task 2.2：访问失败 vs 空结果）

客户查询返回空数组，Agent 重试 3 次后升级。后查明客户确实不存在。浪费重试的根因是什么？

- **A)** 超时时间设置太短
- **B)** 响应未区分“有效空结果”与“访问失败”，导致 Agent 把空结果当潜在失败
- **C)** 应该重试超过 3 次
- **D)** 人工升级链路配置错误

<details>
<summary>▶ 查看答案</summary>

**正确答案：B**

响应语义不完整，模型只能保守重试。

</details>

---

### Question 5（Task 2.3：`tool_choice`）

文档处理流程要求“第一步必须抽元数据（作者、日期、类型）”，应如何配 `tool_choice`？

- **A)** `{"type":"auto"}`
- **B)** `{"type":"any"}`
- **C)** 首次调用 `{"type":"tool","name":"extract_metadata"}`，后续改 `{"type":"auto"}`
- **D)** 首轮只保留 `extract_metadata` 其余工具都移除

<details>
<summary>▶ 查看答案</summary>

**正确答案：C**

`tool_choice` 的命名强制最直接、最稳妥。

</details>

---

### Question 6（Task 2.4：MCP 配置）

团队要共享 Jira 集成，但每人用自己的 token。正确方案是？

- **A)** 把 token 直接写进 `.mcp.json` 并提交
- **B)** 在 `.mcp.json` 用 `${JIRA_API_TOKEN}`，每位开发者本地设置环境变量
- **C)** 每人在 `~/.claude.json` 手配一份
- **D)** 把共享 token 写进 system prompt

<details>
<summary>▶ 查看答案</summary>

**正确答案：B**

项目级共享配置 + 本地变量注入，是安全且可协作的标准方式。

</details>

---

### Question 7（Task 2.5：内置工具）

要找“所有 import 了 `legacyAuth` 的文件”，正确做法是？

- **A)** Glob `**/*legacyAuth*`
- **B)** Read 全部 `src/` 手动查
- **C)** Grep `import.*legacyAuth` 或 `require.*legacyAuth`
- **D)** Glob `**/auth/**`

<details>
<summary>▶ 查看答案</summary>

**正确答案：C**

问题是“内容匹配”，应使用 Grep。

</details>

---

## 评分建议

| 分数 | 评估 | 下一步 |
|---|---|---|
| **7/7** | Domain 2 已具备考试水平 | 进入 Domain 3 |
| **6/7** | 接近就绪 | 回看失分任务点并重做对应练习 |
| **5/7** | 需定向补强 | 重读失分对应的 2 个任务点 |
| **<5** | 需系统复习 | 全文重读 + 全部练习重做 |

---

## 动手练习

### 练习：MCP 工具设计、错误处理与配置

**Part 1：制造并修复工具描述歧义**

设计 3 个 MCP 工具：
1. `search_knowledge_base`
2. `search_support_tickets`
3. `search_help_articles`

先让 `search_knowledge_base` 与 `search_help_articles` 描述故意模糊（近似），再改成“明确用例 + 输入约束 + 显式边界”。

**Part 2：结构化错误响应**

分别写出 4 类错误响应：
- Transient：知识库超时
- Validation：工单 ID 格式错误
- Business：免费用户请求企业版内容
- Permission：无权访问内部工单

每个响应至少包含：`errorCategory`、`isRetryable`、`message`；Business 还要有 `customerFriendlyMessage`。

**Part 3：MCP 配置文件**

创建 `.mcp.json`，要求：
- 配置知识库 Server（`${KB_API_KEY}`）
- 配置工单 Server（`${TICKET_SERVICE_URL}` + `${TICKET_API_TOKEN}`）
- 文件可安全提交（不含明文密钥）

**Part 4：强制首步工具调用**

写代码：首轮强制 `search_knowledge_base`，后续切回 `auto`。并说明适用场景：**总是先查内部文档，再考虑外部来源**。

---

## 速记卡（Quick Reference）

| 概念 | 关键点 |
|---|---|
| 工具描述 | 是工具选择的核心机制，不是附属文档 |
| 误路由首修 | 先改描述，再考虑分类器 |
| 工具拆分 | 泛化工具拆成单一职责 + 明确契约 |
| system prompt 冲突 | 关键词硬规则可覆盖工具描述 |
| `is_error` | 错误必须显式标注 |
| 四类错误 | Transient 重试；Validation 修输入；Business/Permission 升级 |
| 访问失败 vs 空结果 | 前者可考虑重试；后者不要重试 |
| 错误传播 | 本地恢复优先；仅传播不可恢复错误；附带部分结果 |
| 工具过载 | 每个 Agent 控制在 4–5 个，按角色收敛 |
| `tool_choice:auto` | 默认，模型自主决策 |
| `tool_choice:any` | 必须调用某个工具，但模型可选具体工具 |
| `tool_choice:tool` | 强制调用指定工具（适合强制首步） |
| 跨角色受限工具 | 用受限能力消除高频中转往返 |
| `.mcp.json` | 项目级、可版本化、团队共享 |
| `~/.claude.json` | 用户级、不共享 |
| `${VAR}` 展开 | 防止密钥入库 |
| MCP Resources | 先看目录/结构，减少探索性调用 |
| 社区 vs 自建 | 先社区，缺口再自建 |
| Grep | 查内容（调用、import、报错） |
| Glob | 查路径（文件名、后缀、命名模式） |
| Edit | 先用，局部精准修改 |
| Read + Write | Edit 锚点不唯一时的可靠兜底 |
| 增量探索 | Grep -> Read -> 跟导入 -> 再 Grep，避免全量读文件 |

