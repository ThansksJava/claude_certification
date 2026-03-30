# Domain 4：提示工程与结构化输出
## Claude Certified Architect (Foundations) —— 学习指南（中文）
**考试权重：20% | 主要场景：CI/CD 代码审查、结构化数据抽取**

---

## 概览

Domain 4 的题目“迷惑性”很强：错误选项常常听起来像“工程上也说得通”。真正得分点在于：**你是否知道哪种技术对应哪类问题**。只会背技巧名称，不够。

---

## Task Statement 4.1：显式判定标准（Explicit Criteria）

### 核心原则

**模糊的“置信度式指令”不稳定；具体可判定的分类标准才稳定。**

| ❌ 模糊（错误） | ✅ 分类式（正确） |
|---|---|
| “尽量保守。” | “仅在注释声明行为与实际代码行为冲突时标注。” |
| “只报告高置信度问题。” | “报告 bug 和安全漏洞；忽略样式偏好与局部命名问题。” |
| “尽量减少误报。” | “仅标记用户输入未参数化直接进入 SQL 的情况。” |

模糊指令的问题：模型对“保守”“高置信度”的理解会漂移。分类标准是可判定条件，稳定性更高。

### 误报对信任的破坏链

```
某类问题误报过高
  -> 开发者不再信任该类结果
  -> 逐渐不信任所有结果
  -> 真正关键问题被忽略
```

**修复步骤：**
1. 定位高误报类别
2. 临时关闭该类别
3. 保留其他类别，先恢复总体信任
4. 单独迭代被关闭类别的提示词
5. 达标后再重新启用

> 考试陷阱：把“提高置信度阈值”当万能解。通常不对，根因常是标准定义不清，而不是阈值不够高。

### 严重级别校准

**错误做法：**用 prose 描述严重级别。

> “Critical：非常严重，可能造成重大损失。”

**正确做法：**给每个级别配具体代码样例。

```python
# CRITICAL: SQL 注入，用户输入未参数化进入查询
query = f"SELECT * FROM users WHERE id = {user_input}"

# HIGH: 支付路径未处理异常
def process_payment(amount):
    result = payment_gateway.charge(amount)  # 没有 try/except

# MINOR: 命名风格不一致（一般可忽略）
userName = get_user()
```

代码样例是明确锚点，模型更容易稳定匹配。

### 检查题

1. 代码审查 Agent 40% 结果被标成高严重，团队开始忽略全部输出。第一步怎么做？
2. 为什么“只报告高置信度发现”不可靠？

---

## Task Statement 4.2：Few-Shot 提示

### 何时该上 Few-Shot

| 症状 | 根因 | 修复 |
|---|---|---|
| 输出格式多次调用不一致 | 缺少格式锚点 | 提供 2–4 个格式示例 |
| 边界案例判定不一致 | 仅靠分类标准不足 | 提供边界案例 + 判定理由 |
| 文档明明有信息却抽成 null | 不知道如何处理结构变体 | 提供多结构抽取示例 |

### 如何构建高质量示例

- 数量：**2–4 个**，重点覆盖关键变体即可
- 每个示例必须包含：
  1) 输入
  2) 输出
  3) **为什么这么判定**（reasoning）

没有 reasoning，模型容易只做表面模式匹配；有 reasoning 才更易泛化。

```text
Example 1:
Input: "Revenue: $2.4M (see Appendix B for breakdown)"
Output: { "revenue": 2400000, "revenue_source": "appendix_reference", "revenue_confidence": "indirect" }
Reasoning: 数值被提及，但明细在附录，标记为间接来源。

Example 2:
Input: "The company generated approximately two million in annual recurring revenue"
Output: { "revenue": 2000000, "revenue_source": "inline_approximate", "revenue_confidence": "approximate" }
Reasoning: "approximately" 表示近似值，应保留该不确定性。
```

### 降低幻觉的作用

Few-shot 覆盖多种文档结构后，抽取稳定性显著提升，尤其在：
- 行文引用 vs 参考文献
- 叙述段落 vs 表格
- 信息不完整 vs 完整记录
- 多处数据冲突

### 考试关键区分

| 问题 | 错误修复 | 正确修复 |
|---|---|---|
| 格式不一致 | 加更多文字说明 | 上 few-shot 格式示例 |
| 边界判定不一致 | 把规则写得更硬 | 用边界样例+理由 |
| 可提取信息变成空值 | 调整必填/可选乱试 | 用多格式抽取样例 |

### 检查题

1. 表格能抽，叙述文本抽不出同类财务字段，怎么修？
2. 为什么示例必须含 reasoning？

---

## Task Statement 4.3：使用 `tool_use` 做结构化输出

### 可靠性层级

```
tool_use + JSON schema
  -> 结构语法受约束

仅提示“请返回 JSON”
  -> 可能返回无效 JSON
```

生产环境做结构化抽取应优先 `tool_use`。

### `tool_use` 不能防什么（高频考点）

`tool_use` 保障的是**语法结构**，不保障**语义正确性**。

| 错误类型 | 示例 | `tool_use` 能防？ |
|---|---|---|
| JSON 语法错误 | `{"name": "Acme,}` | ✅ 能 |
| 语义错误 | 行项目合计 980，但 total=1000 | ❌ 不能 |
| 字段错位 | 税额写到 `discount` | ❌ 不能 |
| 幻觉补值 | 源文无值却编造必填字段 | ❌ 不能 |

最危险的是“必填缺失导致编造”，因此 schema 的 nullable/optional 设计很关键。

### `tool_choice` 选项

| 值 | 行为 | 适用场景 |
|---|---|---|
| `"auto"` | 模型可返回文本或调工具 | 普通会话 |
| `"any"` | 必须调用某个工具，但具体哪个由模型选 | 需要保证结构化输出，但存在多工具可选 |
| `{"type":"tool","name":"X"}` | 强制调用指定工具 X | 流程中第一步必须执行特定检查 |

### Schema 设计要点

**用 nullable 防编造：**
```json
{
  "company_name": { "type": "string" },
  "revenue": { "type": ["number", "null"], "description": "未给出则为 null" },
  "founded_year": { "type": ["integer", "null"], "description": "未给出则为 null" }
}
```

**用 enum + `unclear` 处理歧义：**
```json
{
  "sentiment": {
    "type": "string",
    "enum": ["positive", "negative", "neutral", "unclear"]
  }
}
```

**用 `other` + 详情字段做可扩展：**
```json
{
  "issue_type": {
    "type": "string",
    "enum": ["bug", "security", "performance", "style", "other"]
  },
  "issue_type_detail": {
    "type": ["string", "null"],
    "description": "当 issue_type=other 时填写"
  }
}
```

即使 schema 很严格，也应在提示词中写格式归一化规则（日期、货币、百分比等）。

### 检查题

1. `tool_use` 抽发票时总额与行项目不一致，这是什么错误？`tool_use` 能否避免？
2. 何时用 `tool_choice: "any"`，何时强制指定工具名？

---

## Task Statement 4.4：校验-重试闭环

### 带错误反馈的重试模式

重试不能“盲重试”。应回传：
1. 原始文档
2. 上次失败输出
3. 明确校验错误

```python
retry_prompt = f"""
Original document:
{original_document}

Your previous extraction:
{json.dumps(failed_extraction, indent=2)}

Validation error:
{validation_error}

Please re-extract, correcting the identified error.
"""
```

### 重试有效性的边界（关键考点）

| 场景 | 重试是否有效 | 原因 |
|---|---|---|
| 值填错字段 | ✅ 有效 | 结构性错误，可纠正 |
| 日期格式错误 | ✅ 有效 | 格式错误，可纠正 |
| 源文确实缺失必填字段 | ❌ 无效 | 信息不存在，重试造不出来 |
| 编造了看似合理值 | ❌ 对该字段无效 | 需改 schema（可空/可选） |

考试常设陷阱：对“源文缺失信息”提重试。正确是改 schema，而不是无限重试。

### `detected_pattern` 字段

在结构化结果里记录触发模式：

```json
{
  "finding": "SQL injection risk",
  "severity": "critical",
  "detected_pattern": "f-string interpolation in database query",
  "file": "app/db.py",
  "line": 47
}
```

用途：
- 可分析哪些模式导致被大量驳回
- 让提示词优化有数据依据

### 自校验字段

```json
{
  "line_items": [...],
  "stated_total": { "type": "number" },
  "calculated_total": { "type": "number" },
  "total_discrepancy_detected": { "type": "boolean" }
}
```

再配冲突检测字段：

```json
{
  "revenue_section_1": { "type": ["number", "null"] },
  "revenue_section_2": { "type": ["number", "null"] },
  "revenue_conflict_detected": { "type": "boolean" }
}
```

### 检查题

1. `founded_year` 源文没有但 schema 必填，重试后总被编造。怎么改？
2. 有效重试提示必须包含哪三项？

---

## Task Statement 4.5：批处理（Batch Processing）

### Message Batches API 关键事实

| 属性 | 值 |
|---|---|
| 成本 | 比同步 API 低约 50% |
| 处理窗口 | 最长 24 小时 |
| 延迟 SLA | 无实时保证 |
| 单请求内多轮工具调用 | ❌ 不支持 |
| 结果关联 | 用 `custom_id` |

### 匹配规则（4.5 最常考）

```
同步 API -> 阻塞型工作流
Batch API -> 可容忍延迟工作流
```

| 工作流 | 选型 | 原因 |
|---|---|---|
| PR 合并前代码审查 | 同步 | 开发者在等待 |
| 夜间 1 万文件安全巡检 | 批处理 | 无人实时等待 |
| 实时客服路由分类 | 同步 | 用户在等待 |
| 每周合规报告 | 批处理 | 定时任务 |
| 夜间自动生成测试 | 批处理 | 非实时 |

> 常见陷阱：因为省钱就“全量改批处理”。若流程是阻塞型，省钱也不应牺牲交互时延。

### 批处理失败重提策略

```
1. 提交批次
2. 按 custom_id 收集失败项
3. 只重提失败项，并按失败原因修正：
   - 文档过大 -> 分块
   - 格式问题 -> 预归一化
   - schema 校验失败 -> 针对该类型改提示
4. 不要全批重跑
```

### 检查题

1. pre-commit 钩子慢，提议改 Batch API，是否正确？
2. 500 份文档失败 47 份，应该如何重提？

---

## Task Statement 4.6：多实例审查（Multi-Instance Review）

### 自审限制

同一会话“先生成再自审”会出现：
- 保留生成时的推理偏置
- 不愿质疑自己刚做的决定
- 更易漏掉细微逻辑错误

**正确做法：独立审查实例。**

### 多通道（多阶段）审查架构

单次审查太多文件会发生注意力稀释：
- 有些文件审得很深
- 有些只扫一眼
- 同样模式在不同文件结论不一致

推荐两阶段：

```
Pass 1：逐文件局部分析
  -> 保证每个文件分析深度一致
  -> 抓本地问题

Pass 2：跨文件整合分析
  -> 基于 Pass 1 摘要
  -> 抓跨文件数据流/架构一致性问题
```

### 置信度路由

```json
{
  "finding": "Potential race condition in cache invalidation",
  "severity": "high",
  "confidence": 0.62,
  "confidence_rationale": "Pattern matches known race condition but depends on threading model"
}
```

```
confidence >= 0.85  -> 自动发布
0.60-0.84          -> 人工复核
< 0.60             -> 丢弃或回流优化提示
```

阈值不应拍脑袋，需要在标注集上做精度校准。

### Domain 4 全链路架构（元模式）

```
文档输入
  -> 加载 few-shot
  -> tool_use + schema 抽取
  -> 校验层（结构+语义+跨字段）
  -> 失败则带错误反馈重试
  -> 独立审查实例
  -> 置信度路由（自动/人工/丢弃）
  -> 记录 detected_pattern 并回流优化
```

### 检查题

1. 同样空指针模式在 `user.py` 被报、在 `order.py` 未报。根因和修复是什么？
2. 独立审查更能发现问题，有人提议为省调用合并会话，是否可行？

---

## Domain 4 考试陷阱速记

| 陷阱 | 正确答案 |
|---|---|
| 误报高就“提高置信度阈值” | 用显式分类标准定义可判定条件 |
| 格式不一致就“写更详细说明” | 用 few-shot 固定目标格式 |
| 源文缺信息靠重试解决 | 改为 nullable/optional 字段 |
| `tool_use` 可避免全部错误 | 仅避免语法错误，不避免语义/编造 |
| 为省钱全部改 Batch | 仅延迟容忍流程可用 Batch |
| 同会话自审 | 用独立审查实例 |
| 14 文件单通道审查 | 逐文件 + 跨文件两阶段 |
| 严重级别用 prose 定义 | 用具体代码样例定义 |
| few-shot 只给 I/O 不给理由 | 需加入 reasoning |
| 必须结构化输出仍用 `auto` | 用 `any` 或指定工具 |

---

## 8 题模拟考试

*(建议先做后看答案)*

**Q1 (4.1)** — CI 代码审查中，“安全漏洞”结果 35% 为误报，团队开始忽略全部结果。最有效的立即动作是？

A) 在系统提示中加“只报告高置信度”  
B) 临时关闭“安全漏洞”类别，单独迭代后再开，同时保留其他类别  
C) 把最低严重级别提升到 critical  
D) 增加更多误报样例

---

**Q2 (4.2)** — 抽取流水线在表格可正确提财务值，但在叙述文本里常返回 null。正确修复？

A) 给 schema 字段加“即使在 prose 也要提取”描述  
B) 增加 few-shot：覆盖叙述文本提取示例  
C) 把字段改必填强制提取  
D) 改回提示词 JSON

---

**Q3 (4.3)** — schema 中 `invoice_total` 必填，但部分发票无总额。怎么改可防编造？

A) 仅加“不要猜测”描述  
B) 改为 `{"type": ["number", "null"]}`，并注明缺失则 null  
C) 额外加状态 enum 字段  
D) 把 `tool_choice` 改成 `any`

---

**Q4 (4.3)** — 导出前必须先调用 `verify_compliance`。哪种 `tool_choice` 能硬性保证？

A) `tool_choice: "auto"`  
B) `tool_choice: "any"`  
C) `tool_choice: {"type":"tool","name":"verify_compliance"}`  
D) 在 system prompt 写“先调用 verify_compliance”

---

**Q5 (4.4)** — 发票抽取把 VAT 填到了 `discount`。无错误反馈重试总是同错。正确重试方式？

A) 提高 temperature 增加多样性  
B) 发送原文 + 失败输出 + 明确校验错误后重试  
C) 把 `vat` 与 `discount` 都改 nullable  
D) 改回提示词 JSON

---

**Q6 (4.5)** — 两个流程：(1) 实时客服路由；(2) 夜间批量生成测试。有人提议都改 Batch API 省 50%。正确做法？

A) 两个都改  
B) 只把夜间测试改 Batch；实时路由保持同步  
C) 只把实时路由改 Batch；测试保持同步  
D) 两个都不改

---

**Q7 (4.6)** — 单会话先重构后自审，漏掉细微逻辑错误。根因是？

A) 上下文不够长  
B) 同会话继承生成推理偏置，审查不够独立  
C) 代码审查必须用 plan mode  
D) 缺少 bug few-shot

---

**Q8 (4.6)** — 12 文件单次审查结果不均：4 个深审、6 个浅审、2 个几乎未分析，同样空指针模式在不同文件结论不一致。根因和修复？

A) few-shot 不足；补示例  
B) 单通道注意力稀释；改为逐文件分析 + 跨文件整合两阶段  
C) 置信度过低；加阈值路由  
D) 文件太大；直接截断

---

## 模拟题答案

| 题号 | 答案 | 关键原因 |
|---|---|---|
| 1 | **B** | 先关闭高误报类别恢复信任，再单独优化 |
| 2 | **B** | few-shot 可补齐不同文档结构的抽取模式 |
| 3 | **B** | nullable 防止缺失信息被强制编造 |
| 4 | **C** | 仅指定工具名能硬性保证先执行该工具 |
| 5 | **B** | 有效重试必须带错误反馈，盲重试只会复现错误 |
| 6 | **B** | 实时场景保持同步；夜间任务可批处理 |
| 7 | **B** | 同会话有生成偏置，独立实例更能发现问题 |
| 8 | **B** | 单通道注意力分散，需多阶段审查架构 |

**评分建议：**
- 8/8：Domain 4 已就绪
- 6–7/8：回看错题对应任务点
- <6：按任务点重学并补充实战场景

---

## 动手练习

构建一个完整抽取管线，覆盖 Domain 4 关键点：

### 1）Schema 设计（必填 + 可选 + 可空 + 枚举）
```json
{
  "name": "extract_invoice",
  "description": "Extract structured data from invoice documents",
  "input_schema": {
    "type": "object",
    "properties": {
      "vendor_name": { "type": "string" },
      "invoice_number": { "type": "string" },
      "invoice_date": {
        "type": ["string", "null"],
        "description": "ISO 8601 (YYYY-MM-DD)，缺失则 null"
      },
      "line_items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "description": { "type": "string" },
            "amount": { "type": "number" }
          }
        }
      },
      "stated_total": { "type": ["number", "null"], "description": "缺失则 null" },
      "calculated_total": { "type": "number", "description": "行项目求和" },
      "total_discrepancy_detected": { "type": "boolean" },
      "currency": { "type": "string", "enum": ["GBP", "USD", "EUR", "other"] },
      "currency_detail": { "type": ["string", "null"], "description": "currency=other 时填写" }
    },
    "required": ["vendor_name", "invoice_number", "line_items", "calculated_total", "total_discrepancy_detected", "currency"]
  }
}
```

### 2）Few-shot 示例

补 2–3 组覆盖：
- 表格发票
- 叙述式发票（如“咨询费 500 英镑”）
- 多币种且总额隐式给出

每组都写 reasoning。

### 3）校验-重试循环

```python
def extract_with_retry(document, schema, max_retries=2):
    result = extract(document, schema)
    for attempt in range(max_retries):
        errors = validate(result)
        if not errors:
            return result
        result = extract_with_feedback(document, result, errors)
    return result
```

### 4）对比实验

- 不加 few-shot 处理 10 份文档，记录空值率/格式错误率
- 加 few-shot 再处理同样 10 份
- 对比：空值率、格式合规率、编造率（抽样人工核验）

---

## 总结：Domain 4 必背 6 点

1. **分类标准优先于“高置信度”口号。**
2. **一致性问题先上 few-shot。**
3. **`tool_use` 只保语法，不保语义。**
4. **nullable 字段可显著降低编造。**
5. **Batch 仅用于延迟容忍流程。**
6. **审查要独立实例，不要同会话自审。**

