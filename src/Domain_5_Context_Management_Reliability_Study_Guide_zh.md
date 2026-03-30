# Domain 5：上下文管理与可靠性
## Claude Certified Architect (Foundations) —— 学习指南（中文）
**考试权重：15% | 主要场景：客服 Agent、多 Agent 研究、结构化数据抽取**

---

## 概览

Domain 5 权重最小，但它是其他域的“承重层”。上下文管理做不好，会连带破坏：
- Domain 1 的多 Agent 编排
- Domain 4 的结构化抽取
- Domain 3 的 CI/CD 工作流

所以这不只是“15% 的一部分分值”，而是跨域失分源。

考试中通常不会出现标题叫“Context Management”的题。你会看到一个失败系统，然后需要识别根因是：上下文退化、错误被吞、或来源追踪缺失。

---

## Task Statement 5.1：上下文保真（Context Preservation）

### 渐进式摘要陷阱（Progressive Summarisation Trap）

**问题：**为了控制上下文长度而不断总结历史，会丢失交易级精确信息。

| 摘要前 | 摘要后 |
|---|---|
| "Customer wants a refund of £247.83 for order #8891 placed on 3rd March" | "Customer wants a refund for a recent order" |
| "Failure occurred at 14:32 UTC on API call to `/payments/v2/refund` with 503 response" | "There was a payment error" |
| "Customer has escalated twice, reference IDs #4421 and #4423" | "Customer has contacted support before" |

金额、日期、订单号、引用 ID 这类关键事实会被压平为模糊 prose。下游 Agent 因此无法给出精确处理。

**修复：建立持久 `case facts` 块，并逐轮原样携带。**

```
CASE FACTS (do not summarise — include verbatim in every prompt):
- Customer: Sarah Chen, account #88234
- Order: #8891, placed 2026-03-03
- Claimed amount: £247.83
- Product: Wireless headphones, SKU HW-7712
- Prior contact: ref #4421 (2026-03-10), ref #4423 (2026-03-12)
- Stated issue: Item arrived without charging cable
```

这个块绝不摘要；可压缩的是周边对话，不是事实块。

### “中段遗失”（Lost in the Middle）

**问题：**长输入中，模型通常更可靠处理开头和结尾，中间部分更容易遗漏。

```
[Beginning — HIGH ATTENTION]
  ... 关键发现 ...
[Middle — REDUCED ATTENTION]
  ... agents 2-8 的结果 ... <- 容易丢
[End — HIGH ATTENTION]
  ... 综合指令 ...
```

**修复策略：**
1. 关键结论放在输入开头
2. 长内容使用显式分节标题
3. 多 Agent 结果先放关键结论，再放证据细节

### 工具结果裁剪（Tool Result Trimming）

**问题：**工具常返回超量字段，持续累积会快速耗尽 token。

```python
order = lookup_order("8891")

relevant = {
    "order_id": order["id"],
    "placed_date": order["created_at"],
    "total": order["total_amount"],
    "status": order["fulfillment_status"],
    "items": order["line_items"]
}
# 将 relevant 写入上下文，而不是完整 order
```

这不是丢信息，而是过滤无关噪声（仓库 ID、内部路由码、A/B 标记等）。

### 全历史要求

每次 API 调用都必须带上从第一轮开始的完整会话历史。模型调用是无状态的。

```python
# WRONG
messages = [{"role": "user", "content": current_message}]

# CORRECT
messages = conversation_history + [{"role": "user", "content": current_message}]
```

### 上游 Agent 输出优化

下游上下文预算有限时，上游应输出结构化摘要，而非冗长原文+推理链。

| 上游输出 | 下游接收 |
|---|---|
| 论文全文 + 长推理 | ~8000 tokens 噪声 |
| `{ "key_claim": "...", "source": "...", "relevance_score": 0.91, "excerpt": "..." }` | ~150 tokens 高信号 |

在 Hub-and-Spoke 架构（Domain 1）尤为关键：协调器常要同时持有 4–6 个子 Agent 结果。

### 检查题

1. 12 轮客服对话里，结果只说“最近订单”，未给金额与订单号。根因是什么？
2. 协调器整合 6 个研究子 Agent，最终漏掉 Agent 3、4 结果。应做什么结构修复？

---

## Task Statement 5.2：升级策略与歧义消解

### 三个有效升级触发条件

| 触发条件 | 为什么有效 | 动作 |
|---|---|---|
| **客户明确要求人工** | 用户有自主选择权 | 立即升级，不要先“再试试” |
| **政策例外或政策缺口** | Agent 不能越权决策 | 带上下文升级，并标注触发的政策缺口 |
| **无法继续取得实质进展** | 继续只会浪费时间并加剧挫败 | 结构化交接：已尝试内容、失败点 |

### 两个不可靠触发（考试陷阱）

| 触发 | 为什么不可靠 |
|---|---|
| **情绪/愤怒程度** | 情绪强度与问题复杂度并不强相关 |
| **模型自报置信度** | 自校准不稳定，常在难题上“高自信” |

### “客户生气”细分逻辑（高频考点）

```
客户表达不满
  -> 问题是否仍可直接解决？
      YES -> 先安抚并给出解决方案，不立即升级
      -> 若客户再次明确要求人工 -> 立即升级
      -> 否则继续处理

但如果客户明确说："I want to speak to a human"
  -> 无条件立即升级
```

区别：**隐式不满 != 明确要求人工**。

### 客户匹配歧义

当查询匹配到多个同名客户（如两个 "Sarah Chen"）：

**错误做法：**按启发式“猜一个”（最近活跃/最可能）
- 有非零误判风险
- 可能导致错单处理与数据泄露

**正确做法：**请求额外标识信息
- 邮箱
- 电话
- 订单号
- 邮编

原则：不猜，必须确认。

### 检查题

1. 客户说“this is ridiculous”，Agent 立即升级，是否正确？
2. 用户询问竞品价格匹配，现行政策仅覆盖自家促销匹配，无竞品条款。应触发哪类升级？

---

## Task Statement 5.3：错误传播（Error Propagation）

### 结构化错误上下文

工具或子 Agent 失败时，不能只传状态码。下游协调器需要结构化语境：

```json
{
  "error_type": "transient",
  "what_was_attempted": "Journal database query for 'geothermal energy capacity 2023-2025'",
  "parameters_used": { "source": "academic_journals", "date_range": "2023-2025", "topic": "geothermal" },
  "partial_results": [
    { "title": "Global Geothermal Capacity Report 2022", "relevance": 0.78 }
  ],
  "potential_alternatives": ["web_search with same query", "retry after 30s"],
  "retry_recommended": true
}
```

### 四类错误分类

| 类型 | 描述 | 典型动作 |
|---|---|---|
| `transient` | 超时、限流、短时不可用 | 退避重试 |
| `validation` | 输入不符合工具约束 | 修输入后重试 |
| `business` | 业务规则限制（如退款超限） | 升级或给替代方案 |
| `permission` | 权限不足 | 升级，不盲重试 |

### 两个反模式

**反模式 1：静默吞错**
```python
try:
    result = fetch_journal_data(query)
except Exception:
    return []
```

看起来“成功但空结果”，协调器不知道数据源不可用，导致不可恢复的静默缺口。

**反模式 2：一错全停**
```python
try:
    result = fetch_journal_data(query)
except Exception:
    raise SystemExit("Pipeline failed")
```

单点失败毁掉所有已成功子任务结果。

**正确模式：结构化传播 + 保留部分结果**
```python
try:
    result = fetch_journal_data(query)
except TransientError as e:
    return ErrorResult(
        type="transient",
        attempted=query,
        partial_results=cache.get_recent(query),
        retry_recommended=True,
        error_detail=str(e)
    )
```

### 访问失败 vs 有效空结果

| 场景 | 实际含义 | 正确动作 |
|---|---|---|
| HTTP 503，无数据 | 访问失败，源不可达 | 重试并标注覆盖缺口 |
| HTTP 200，结果为空 | 有效空结果，确实无匹配 | 不重试，直接如实报告 |

混淆这两者是严重错误：
- 把失败当空结果 -> 静默漏源
- 把空结果当失败 -> 无意义重试

### 覆盖注记（Coverage Annotations）

综合 Agent 输出时必须标注“未覆盖部分及原因”，不能静默省略：

```
### Geothermal Energy ⚠️ (limited coverage)
Note: Academic journal access was unavailable during this session.
Coverage based on 2 web sources only.
```

### 检查题

1. 子 Agent 返回 HTTP 200 + 空数组，协调器重试 3 次，是否正确？
2. 6 子 Agent 流水线中，4 号超时，协调器保留其他结果并标注缺口，是否正确？

---

## Task Statement 5.4：代码库探索（Codebase Exploration）

### 长会话中的上下文退化

典型退化模式：

```
会话早期：
"AuthService.java line 847 使用 RS256 JWT"

会话后期：
"认证通常使用 Java 常见 token 方案"
```

模型从“已发现的具体事实”退化为“似是而非的通用推断”。常见原因：
- 早期探索输出过长，填满上下文
- 关键事实被挤到中段低关注区
- 模型用通用模式补偿记忆缺口

### 缓解策略

**1) Scratchpad 文件**：把关键发现落地到文件

```
# findings.md
## Authentication
- JWT RS256 in AuthService.java:847
- Token expiry: 24h (config/auth.yml:12)
- Refresh logic: RefreshTokenService.java:203
```

依赖文件持久化，不依赖会话内短期记忆。

**2) 子代理分工**：每个调查任务开新上下文

```
Main: 调查支付链路
  -> Sub A: PaymentController 入口映射
  -> Sub B: 支付数据库写入追踪
  -> Sub C: 支付外部 API 调用盘点
```

**3) 阶段摘要注入**：每阶段结束先结构化总结，再进入下一阶段。

**4) `/compact`**：当会话“抓不住早期结论”时，用于压缩上下文占用。

### 崩溃恢复（Crash Recovery）

长流程要内建状态持久化：

```python
agent.export_manifest("./state/agent_3_manifest.json")
```

Manifest 示例：
```json
{
  "agent_id": "research_agent_3",
  "status": "completed",
  "completed_at": "2026-03-27T14:32:00Z",
  "findings": [...],
  "sources_searched": [...],
  "gaps": [...]
}
```

恢复时由协调器加载已有 manifest，只重跑未完成工作。

### 检查题

1. 会话后期回答开始泛化成“典型 Spring Boot 模式”，根因是什么？如何修复？
2. 6 Agent 流水线运行 3 小时后在第 5 个崩溃。没有 crash recovery 会怎样？加上后改变了什么？

---

## Task Statement 5.5：人工复核与置信度校准

### 聚合指标陷阱（Aggregate Metrics Trap）

危险标题：**“整体准确率 97%”**。

可能掩盖：

| 文档类型 | 准确率 |
|---|---|
| 标准发票 | 99.8% |
| 手写发票 | 58% |
| 多币种发票 | 71% |
| 德语发票 | 63% |

整体高分可能只是因为难样本占比低。

上线自动化前，至少按以下维度分层验证：
- 文档类型
- 字段类型（日期/金额/地址等）
- 语言与地区
- 文档年代（旧模板）

### 分层随机抽样（Stratified Random Sampling）

即使上线后也要持续抽检：

```
每月抽样：
- 高置信度 (>0.90) 随机 50 条
- 中置信度 (0.70-0.90) 随机 20 条
- 低置信度 (<0.70) 随机 10 条
```

为什么还要抽高置信度？因为模型置信度并非完美校准，长期漂移只靠持续抽样才能发现。

### 字段级置信度（而非文档级）

```json
{
  "vendor_name": { "value": "Acme Ltd", "confidence": 0.99 },
  "invoice_date": { "value": "2026-03-15", "confidence": 0.95 },
  "total_amount": { "value": 1250.00, "confidence": 0.88 },
  "vat_number": { "value": "GB123456789", "confidence": 0.61 },
  "payment_terms": { "value": null, "confidence": null }
}
```

路由策略：

```
confidence >= 0.90  -> 字段自动通过
0.70-0.89          -> 抽检（例如抽 10%）
< 0.70             -> 人工复核
```

校准步骤：
1. 构建 200+ 份有真值标注的数据集
2. 跑流水线
3. 按置信区间测实际 precision
4. 调阈值得到可接受 FN/FP 平衡
5. 每季度或重大提示变更后重校准

资源有限时优先人工处理“最高不确定性”样本，而非最旧或金额最高样本。

### 检查题

1. 流水线整体 96% 就全自动，上线后手写发票大量编造。验证流程哪里错了？
2. 文档整体置信度 0.94，但 `vat_number` 仅 0.52，是否应自动通过？

---

## Task Statement 5.6：信息溯源（Information Provenance）

### 结构化“结论-来源”映射

多 Agent 研究中，每条结论都应携带来源并贯穿全链路：

```json
{
  "claim": "Global geothermal capacity reached 15.6 GW in 2023",
  "source_url": "https://irena.org/publications/2024/geothermal-capacity",
  "document_name": "IRENA Geothermal Power Report 2024",
  "relevant_excerpt": "Total installed geothermal capacity reached 15,608 MW by end of 2023...",
  "publication_date": "2024-02",
  "source_type": "primary_report",
  "retrieved_by": "research_agent_2"
}
```

### 来源丢失死亡链（Provenance Death Chain）

```
研究子 Agent 找到: "15.6 GW in 2023 (IRENA 2024)"
  -> 以 prose 传给综合 Agent
综合写成: "approximately 15 GW"
  -> 传给报告格式化
最终报告: "capacity is significant"
  -> 审核问: 来源在哪?
  -> 无法追溯
```

结论：结构化映射能保真，纯 prose 传递会丢源。

### 冲突处理

两个可信来源给出不同数字时，不能随意二选一。

**错误：**静默丢弃一方。

**正确：**保留两者并标注冲突及解释。

```json
{
  "claim": "Global geothermal installed capacity",
  "values": [
    { "value": "15.6 GW", "source": "IRENA", "date": "2024-02", "scope": "end of 2023" },
    { "value": "14.9 GW", "source": "IEA", "date": "2023-11", "scope": "mid-2023" }
  ],
  "conflict_detected": true,
  "conflict_note": "Difference likely reflects different measurement dates (mid-year vs end-year). Both values preserved."
}
```

Agent 的职责是保留证据与时间语境，不是擅自仲裁。

### 时间维度意识（Temporal Awareness）

不同年份数据不同，常是时间序列变化而非冲突。必须在结构化输出中携带发布日期/采集时间。

### 按内容选择呈现形式

不要把所有内容硬塞同一种格式。应让格式服务内容：

| 内容类型 | 推荐格式 | 原因 |
|---|---|---|
| 财务与统计数据 | 表格 | 便于横向比较 |
| 新闻/叙事发现 | prose | 保留语境 |
| 技术问题/代码发现 | 结构化列表 | 可扫描、可执行 |
| 冲突来源 | 并排对照 | 双方信息清晰 |
| 时间序列 | 时间线/图表 | 展示趋势 |

考试陷阱：
“所有输出统一 JSON”听起来工程化，但在人类阅读报告场景下通常不是最佳答案。

### 检查题

1. 综合 Agent 产出 3000 字报告，但无法将任何结论追溯到来源。缺了什么架构要素？
2. 同一国家可再生占比，2022 与 2024 两个来源不同，应如何处理？

---

## Domain 5 考试陷阱速记

| 陷阱 | 正确答案 |
|---|---|
| 为省 token 摘要整个历史 | 把交易事实抽成持久 case facts 块并逐轮原样保留 |
| 客户生气就升级人工 | 仅三触发：明确要人工、政策缺口、无法继续推进 |
| 以模型自报置信度作为升级条件 | 置信度常失准，不可作为核心触发 |
| 200 + 空结果重试 | 这是有效空结果，不应重试 |
| 吞掉工具失败 | 结构化错误传播并保留部分结果 |
| 单个子 Agent 失败就终止管线 | 继续已有结果并标注覆盖缺口 |
| 96% 总准确率就可全自动 | 必须按文档类型/字段分层验证 |
| 同一会话既生成又审查 | 用独立审查实例（Domain 3/4 同样原则） |
| 冲突来源二选一 | 保留双方并标 `conflict_detected` |
| 同名客户按“最可能”选一个 | 追加标识信息确认，绝不猜选 |

---

## 6 题模拟考试

*(建议先答题，再看答案)*

**Q1 (5.1)** — 15 轮客服对话里，关于订单 #7734 的 £312.50 争议在第 8 轮后被摘要。到第 15 轮只剩“billing issue”这类模糊措辞。根因和修复是？

A) 上下文窗口太小，换更大模型  
B) 交易事实被压缩进渐进摘要，需抽出持久 case facts 块并禁止摘要  
C) 第 15 轮应再查一次工具恢复细节  
D) 增加 few-shot 教模型后文引用细节

---

**Q2 (5.2)** — 客户说“等了 20 分钟，太离谱了，赶紧处理”。Agent 立即升级人工。正确吗？

A) 对，客户情绪强烈可直接升级  
B) 对，两轮没解决即属于无法推进  
C) 不对，情绪本身不是触发；应先安抚并给方案，若再次明确要人工再升级  
D) 不对，应该第一轮就升级避免情绪恶化

---

**Q3 (5.3)** — 研究子 Agent 查询数据库得到 HTTP 200 + 空数组，协调器重试 4 次。正确吗？

A) 对，空结果可能是瞬时问题  
B) 对，学术库常需多次查询才出结果  
C) 不对，200+空数组是有效空结果，应直接报告  
D) 不对，应终止整条管线并换查询

---

**Q4 (5.4)** — 2 小时代码探索中，早期能定位到 `AuthService.java` 的 RS256，后期只会说“典型 Java JWT 模式”。最可能原因？

A) 会话中途模型升级丢记忆  
B) 上下文退化：冗长探索输出挤压早期具体事实，模型转向泛化推断  
C) 提问不同导致答复不同  
D) 安全策略屏蔽了认证细节

---

**Q5 (5.5)** — 发票抽取整体准确率 98%，上线后多币种发票系统性错误。验证集 500 份仅含 8 份多币种。问题在哪？

A) 500 份总体太少  
B) 需要更多 few-shot  
C) 只看聚合准确率，类别样本不足掩盖了分层失败  
D) 置信度阈值过低

---

**Q6 (5.6)** — 能源报告中来源 A（2022）写 28%，来源 B（2024）写 34%。综合 Agent 仅保留 34%。正确做法？

A) 对，总是保留最新值  
B) 不对，应保留两者并附日期与 `conflict_detected`，通常是时间增长而非矛盾  
C) 不对，两者都删除  
D) 不对，必须找第三来源后才能写入

---

## 模拟题答案

| 题号 | 答案 | 关键原因 |
|---|---|---|
| 1 | **B** | 渐进摘要压缩交易事实；应用持久 case facts 原样携带 |
| 2 | **C** | 情绪不等于升级触发；明确要求人工才是立即触发 |
| 3 | **C** | HTTP 200 + 空数组是有效结果，不应重试 |
| 4 | **B** | 上下文退化导致从具体事实滑向泛化推理 |
| 5 | **C** | 聚合指标掩盖分层类别失败，样本分布失衡 |
| 6 | **B** | 两来源不同年份反映时间变化，应保留并标注来源与日期 |

**评分建议：**
- 6/6：Domain 5 就绪
- 5/6：复习错题任务点并补做一题
- <5：系统复习 5.2 与 5.3 后再测

---

## 动手练习

搭建一个 1 个协调器 + 2 个子 Agent 的演示系统，覆盖 Domain 5 核心点：

### 架构

```
Coordinator
├── SubAgent A: Web Search
├── SubAgent B: Document Analysis
├── 持久 case facts 块（绝不摘要）
├── 结构化错误传播（模拟 A 超时）
├── 综合输出覆盖注记
└── claim-source 映射保留
```

### 1）Case Facts 块

```python
CASE_FACTS = """
CASE FACTS [DO NOT SUMMARISE — INCLUDE VERBATIM]:
- Customer: James Okafor, account #44821
- Order: #9923, placed 2026-03-01
- Item: Standing desk, SKU FN-8841, £649.00
- Reported issue: Arrived with missing bolts (bags B and C)
- Prior contacts: ref #7731 (2026-03-05), ref #7799 (2026-03-08)
- Pending action: Replacement parts dispatched 2026-03-09, not yet confirmed received
"""
```

### 2）模拟超时 + 结构化错误传播

```python
def web_search_agent(query):
    try:
        result = search(query, timeout=10)
        return {"status": "success", "results": result}
    except TimeoutError:
        return {
            "status": "error",
            "error_type": "transient",
            "what_was_attempted": query,
            "partial_results": cache.get(query, []),
            "retry_recommended": True,
            "coverage_note": "Web search unavailable. Findings are limited to cached results."
        }
```

### 3）冲突来源保留

```python
def synthesise(findings):
    conflicts = detect_conflicts(findings)
    for conflict in conflicts:
        conflict["resolution"] = "both_preserved"
        conflict["conflict_detected"] = True
        conflict["note"] = "Consumer should select based on context and date"
    return findings
```

### 测试用例

提交一个查询，使其同时满足：
1. 两个来源返回不同年份统计值
2. 触发 Web Search 子 Agent 超时
3. 会话持续 8+ 轮，验证 case facts 精度不丢

验收标准：
- 最终输出同时保留冲突值与来源
- 输出包含覆盖缺口注记
- 第 8 轮仍能准确引用 case facts

---

## 总结：Domain 5 必背 6 点

1. **交易事实绝不摘要，必须用持久 case facts 原样传递。**
2. **升级三触发：明确要人工、政策缺口、无法推进。**
3. **HTTP 200 + 空数组是有效空结果，不要重试。**
4. **错误要结构化传播，保留部分结果并标注缺口。**
5. **聚合准确率会骗人，必须做分层验证与持续抽样。**
6. **冲突来源一律保留并附时间与归因，不做武断二选一。**

