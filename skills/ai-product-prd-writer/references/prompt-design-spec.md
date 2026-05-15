# Prompt Design Spec 模板

> 适用场景：PRD 中涉及 AI/LLM 功能时，产品经理用此模板定义 Prompt 的完整工程规格，交付给开发和算法团队。
>
> 三大厂商共识（Anthropic / OpenAI / Google）：结构化分隔、角色设定、静态内容前置、显式输出格式。

---

## 1. Prompt 元信息

| 字段 | 说明 | 示例 |
|------|------|------|
| Prompt ID | 唯一标识，便于版本追踪 | `prompt_summary_v2.1` |
| 目标模型 | 指定模型及 fallback | `claude-sonnet-4-20250514`，fallback: `gpt-4o` |
| Temperature | 创造性 vs 确定性 | `0.3`（事实性任务）/ `0.8`（创意生成） |
| Max Tokens | 输出上限 | `2048` |
| Top-p | 核采样参数 | `0.95` |
| 负责人 | Prompt Owner | `[姓名]` |
| 最后更新 | 版本日期 | `2026-05-15` |

---

## 2. System Prompt 结构

遵循 **静态内容前置** 原则（利于 Prompt Caching），按以下顺序组织：

```
<system>
<!-- 1. Role：角色设定 -->
你是[具体角色]，专长于[领域]，服务对象是[用户画像]。

<!-- 2. Global Instructions：全局指令 -->
- [核心行为准则 1]
- [核心行为准则 2]
- [禁止行为 1]

<!-- 3. Output Format：输出格式 -->
<output_format>
  返回 JSON，结构如下：
  {
    "summary": "string, 不超过200字",
    "confidence": "float, 0-1",
    "tags": ["string"]
  }
</output_format>

<!-- 4. Constraints：硬约束 -->
<constraints>
  - 不编造事实，无法确认时明确说明
  - 不涉及[合规红线内容]
  - 响应语言跟随用户输入语言
</constraints>
</system>
```

**设计要点：**
- Anthropic 推荐用 XML tags（`<context>` `<task>` `<examples>` `<output_format>`）做结构分隔
- OpenAI 推荐静态内容（system instructions、few-shot、tool definitions）放最前，动态内容放最后，最大化 cache hit
- Google PTCF 框架：Persona · Task · Context · Format，四要素缺一不可

---

## 3. Input 规范

### 3.1 User Input Schema

```
<user_input>
  {{user_query}}          <!-- 用户原始输入 -->
</user_input>
```

| 变量名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| `{{user_query}}` | string | 是 | 用户原始问题 | "帮我总结这篇文章" |
| `{{document}}` | string | 否 | 待处理文档内容 | 文章全文 |
| `{{user_preferences}}` | object | 否 | 用户偏好配置 | `{"language": "zh", "detail_level": "brief"}` |

### 3.2 Context Injection（上下文注入）

```
<context>
  <!-- 检索增强（RAG）注入位 -->
  {{retrieved_chunks}}

  <!-- 业务数据注入位 -->
  {{business_context}}
</context>
```

### 3.3 Prefill（预填充）

Anthropic 支持 assistant prefill 引导输出格式：

```
<assistant_prefill>
{"summary": "
</assistant_prefill>
```

---

## 4. Output 规范

### 4.1 Output Schema

```json
{
  "result": {
    "type": "[text | json | markdown | structured]",
    "schema": "[具体结构定义]",
    "max_length": "[字符/token 上限]"
  }
}
```

### 4.2 Success Criteria（成功标准）

| 维度 | 标准 | 度量方式 |
|------|------|----------|
| 准确性 | [具体指标，如事实正确率 ≥ 95%] | Golden dataset 对比 |
| 完整性 | [覆盖所有必填字段] | Schema 校验 |
| 格式合规 | [输出可被下游系统解析] | JSON Schema 验证 |
| 延迟 | [P99 < 3s] | 监控打点 |

### 4.3 Edge Case Handling

| 边界场景 | 期望行为 | 示例输出 |
|-----------|----------|----------|
| 输入为空 | 返回提示而非报错 | `{"error": "请提供待处理的内容"}` |
| 输入超长（> context window） | 截断 + 提示 | `{"warning": "内容已截断至前8000字"}` |
| 输入含对抗性内容 | 拒绝 + 说明原因 | `{"refused": true, "reason": "..."}` |
| 模型无法确定答案 | 明确标注不确定性 | `{"confidence": 0.3, "caveat": "..."}` |
| 多语言混合输入 | [指定处理策略] | 跟随主要语言 / 统一用中文 |

---

## 5. Few-shot Examples

设计 3-5 个示例，覆盖 **正常 / 边界 / 拒绝** 三类场景：

```
<examples>
  <!-- 示例1：正常场景 -->
  <example>
    <input>[典型用户输入]</input>
    <output>[期望输出]</output>
  </example>

  <!-- 示例2：边界场景 -->
  <example>
    <input>[模糊/不完整/超长输入]</input>
    <output>[期望的降级处理输出]</output>
  </example>

  <!-- 示例3：拒绝场景 -->
  <example>
    <input>[违规/超出能力范围的输入]</input>
    <output>[期望的拒绝输出，含原因说明]</output>
  </example>
</examples>
```

**设计原则：**
- 示例应来自真实用户数据或产品场景，不要编造过于理想化的例子
- 每个示例标注 **意图**（为什么选这个例子）
- Few-shot 属于静态内容，放在 system prompt 中以利用 caching

---

## 6. Guardrails（分层防护）

采用 **Input → Model → Output** 三层防护架构：

### 6.1 Input Layer（输入层）

| 检查项 | 规则 | 处理方式 |
|--------|------|----------|
| 长度校验 | `len(input) ≤ [上限]` | 截断或拒绝 |
| 注入检测 | 检测 prompt injection 模式 | 拒绝 + 告警 |
| PII 过滤 | 脱敏手机号/身份证/邮箱 | 替换为 `[REDACTED]` |
| 内容分类 | 暴力/色情/违法内容检测 | 拒绝 + 记录 |

### 6.2 Model Layer（模型层）

在 system prompt 的 `<constraints>` 中写入：

- 不输出[具体禁止内容类别]
- 不执行[具体禁止行为]
- 遇到不确定信息时，明确标注而非猜测

### 6.3 Output Layer（输出层）

| 检查项 | 规则 | 处理方式 |
|--------|------|----------|
| Schema 校验 | 输出符合定义的 JSON Schema | 重试 / fallback |
| 内容审核 | 二次过滤敏感内容 | 替换或阻断 |
| 长度校验 | 输出不超过 max_tokens | 截断 + 补充结尾标记 |
| 幻觉检测 | [如适用] 对比知识库验证事实 | 标注置信度 |

### 6.4 Compliance（合规）

- [ ] 符合《生成式人工智能服务管理暂行办法》
- [ ] 用户数据不用于模型训练（确认 API 条款）
- [ ] 日志中脱敏处理用户输入
- [ ] [其他行业特定合规要求]

---

## 7. Multi-turn 对话设计

> 仅多轮对话场景需要此节。单次调用可跳过。

### 7.1 State Management（状态管理）

```
对话状态机：
[初始] → [信息收集] → [处理中] → [结果确认] → [结束]

每轮需维护的状态变量：
- {{session_id}}: 会话标识
- {{turn_count}}: 当前轮次
- {{collected_fields}}: 已收集的字段列表
- {{current_stage}}: 当前阶段
```

### 7.2 Context Window Strategy

| 策略 | 适用场景 | 实现方式 |
|------|----------|----------|
| 全量保留 | 短对话（< 10轮） | 直接拼接历史 |
| 滑动窗口 | 中等对话 | 保留最近 N 轮 + 摘要 |
| 摘要压缩 | 长对话（> 20轮） | 每 K 轮生成摘要替换历史 |

### 7.3 Instruction Decay 对策

多轮对话中 system prompt 指令会随轮次增加而"衰减"，对策：

- 关键约束在每轮 user message 末尾追加 reminder
- 使用 `<reminder>` 标签包裹核心规则
- 监控第 N 轮后的格式合规率，设置告警阈值

---

## 8. Tool / Function Calling

> 仅涉及工具调用场景需要此节。

### 8.1 Tool Definitions

```json
{
  "name": "[tool_name]",
  "description": "[工具用途，写清楚让模型判断何时调用]",
  "parameters": {
    "type": "object",
    "properties": {
      "[param_name]": {
        "type": "[类型]",
        "description": "[参数说明]"
      }
    },
    "required": ["[必填参数]"]
  }
}
```

### 8.2 调用策略

| 配置项 | 值 | 说明 |
|--------|------|------|
| Tool Choice | `auto` / `required` / `none` | 模型自行决定 / 强制调用 / 禁止调用 |
| 并行调用 | 是 / 否 | 是否允许单轮调用多个工具 |
| 最大调用轮次 | `[N]` | 防止无限循环 |

### 8.3 Fallback

- 工具调用失败时：[重试 N 次 → 降级为纯文本回答 → 报错]
- 工具返回异常数据时：[校验 → 丢弃 → 用默认值]

---

## 9. 评估标准

### 9.1 Automated Metrics（自动化指标）

| 指标 | 计算方式 | 目标值 | 说明 |
|------|----------|--------|------|
| 格式合规率 | 输出通过 Schema 校验的比例 | ≥ 99% | 基础门槛 |
| 准确率 | Golden dataset 上的正确率 | ≥ [目标值] | 核心指标 |
| 拒绝准确率 | 应拒绝输入被正确拒绝的比例 | ≥ 95% | 安全指标 |
| 平均延迟 | TTFT / 总耗时 | P99 < [目标值] | 体验指标 |

### 9.2 Human Evaluation（人工评估）

| 维度 | 评分标准（1-5） | 评估人 |
|------|------------------|--------|
| 有用性 | 是否解决了用户问题 | [产品/运营] |
| 流畅度 | 表达是否自然 | [内容团队] |
| 安全性 | 是否存在风险内容 | [合规团队] |

### 9.3 Launch Gate（上线门槛）

- [ ] Golden dataset（≥ 100 条）评估通过
- [ ] LLM-as-Judge 评分 ≥ [阈值]
- [ ] 边界场景测试全部通过
- [ ] 安全红线场景 0 违规
- [ ] 延迟指标达标
- [ ] 产品负责人 Sign-off

---

## 10. 版本管理

### 10.1 版本规范

采用 **Prompt as Code** 原则——Prompt 变更走 Code Review 流程：

| 版本号 | 变更类型 | 说明 |
|--------|----------|------|
| v1.0.0 → v2.0.0 | Major | 角色/任务定义变更，需重新评估 |
| v1.0.0 → v1.1.0 | Minor | 新增 few-shot / 调整约束 |
| v1.0.0 → v1.0.1 | Patch | 措辞微调，不影响输出逻辑 |

### 10.2 环境链路

```
开发环境 → 测试环境 → 预发布（灰度 5%） → 全量发布
   ↓           ↓           ↓                  ↓
 本地调试    Golden set   A/B 对比         监控 + 告警
```

### 10.3 Rollback 策略

- 线上输出异常率 > [阈值]% 时自动回滚到上一版本
- 保留最近 3 个稳定版本的完整 prompt 快照

---

## 附录：常用框架速查

| 框架 | 要素 | 适用场景 |
|------|------|----------|
| **PTCF** | Persona · Task · Context · Format | Google 推荐，通用场景 |
| **COSTAR** | Context · Objective · Style · Tone · Audience · Response | 内容生成类任务 |
| **RTF** | Role · Task · Format | 简单任务快速起草 |
| **XML 结构化** | `<context>` `<task>` `<examples>` `<output_format>` | Anthropic 推荐，复杂任务 |
| **Chain of Thought** | 在 prompt 中加入 `<thinking>` 引导推理过程 | 推理/分析类任务 |

---

## 附录：Caching-Friendly Layout 检查清单

- [ ] System prompt 中静态内容（角色、指令、few-shot、tool definitions）在前
- [ ] 动态内容（用户输入、检索结果、上下文注入）在后
- [ ] 不同用户共享的内容集中在 prompt 前半部分
- [ ] Few-shot examples 不随请求变化
- [ ] 使用 cache breakpoints 标记（如 Anthropic 的 `cache_control`）
