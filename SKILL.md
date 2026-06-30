# 私域 AI 技能：方法论指南

---

## 1. 提示词工程模式

### 1.1 提示词架构

每条有效提示词遵循分层结构：

```
第一层：角色定义
第二层：约束条件
第三层：输出格式
第四层：示例/参考
```

**角色定义**

用精确的专业身份开头。不要只说"你是AI助手"——要定义领域专长和语气。

```
模式：  "你是一个专业的{领域}专家，擅长{具体技能}。"
示例：  "你是一个专业的营销文案撰写专家，擅长为儿童教育产品撰写吸引人的推广文案。"
```

为什么有效：LLM 对角色框架的响应会激活相关知识簇。"儿童教育营销文案专家"产出的内容与通用"助手"截然不同。

**约束条件**

约束是最强大的杠杆。要数值化、可验证：

- 字数约束："每个卖点10-14个字"
- 行数约束："必须恰好3行"
- 内容规则："不要出现价格信息"
- 结构规则："第一行是主题引入，第二行是核心卖点，第三行是行动号召"

为什么要数值化约束：模糊指令（"简短一些"）产出不稳定。精确数字（"10-14个字"）在提示词和输出之间建立可验证的契约。

**输出格式**

用代码可解析的标记定义格式：

```
创意内容：使用分段标记（===群发===, ===朋友圈===）
验证结果：使用严格JSON（{"pass": true/false, "reason": "..."}）
结构化数据：使用带固定前缀的编号列表
```

为什么要可解析格式：AI 输出必须流入下游代码。如果流水线需要提取"群发文案"，模型必须以可预测、可机器解析的结构产出。

### 1.2 温度策略

温度不是万能的。要匹配任务性质：

| 任务类型 | 温度 | 理由 |
|----------|------|------|
| 创意生成 | 0.8–0.9 | 最大化多样性，避免重复模式 |
| 事实验证 | 0.1 | 确定性判断，不需要创意 |
| 格式调整 | 0.3 | 保持语义同时重组，轻微灵活性 |

**关键洞察**：当同一内容经过多个 AI 阶段（生成 → 验证 → 调整）时，每个阶段应有独立温度。创意阶段需要高温保证多样性；验证阶段需要低温保证可靠性。

### 1.3 提示词锚定

图片生成中，用优先级标记控制模型严格遵循哪些指令：

```
【最高优先级】文字元素必须精准呈现，不允许任何文字错误
【次高优先级】整体构图和色彩方案
【一般优先级】背景细节和装饰元素
```

为什么要优先级标记：图像模型的"注意力预算"有限。没有明确优先级，可能渲染了精美背景却拼错文字。优先级标记告诉模型在哪里投入精确度。

### 1.4 基于参考的生成

不要从零描述图片，而是提供参考图 + 修改指令：

```
结构：
  reference_image: [基础图片路径]
  prompt: "基于参考图，将{元素A}改为{元素B}，保持{元素C}不变"
```

为什么基于参考：纯文生图方差大。参考图锚定构图、风格和布局——AI只需修改特定元素，大幅减少不可控变化。

---

## 2. 工作流设计

### 2.1 流水线架构

```
[生成] → [验证] → [去重] → [格式化] → [部署]
  AI       AI      代码     AI/代码     代码
```

**为什么顺序重要**：

1. **先生成**：自由创作，不受唯一性或格式约束
2. **再验证**：在错误进一步传播前捕获事实错误
3. **再去重**：与历史语料库比对去除相似内容（基于代码，确定性）
4. **再格式化**：将幸存内容适配为渠道特定要求
5. **最后部署**：通过确定性调度推送最终内容

### 2.2 为什么验证与生成分离

单一提示词说"生成准确的营销文案"不可靠。替代方案：

- **生成器（模型A，高温）**：针对创意和吸引力优化
- **验证器（模型B或同模型，低温）**：针对事实数据库的准确性优化

这是"生成器-评论家"模式。生成器不需要为准确性自我审查（这会降低创意）。验证器不需要有创意（这会降低可靠性）。

**验证输出必须结构化**：
```json
{"pass": true, "reason": "卖点与已验证素材库一致"}
{"pass": false, "reason": "提到了该主题不存在的功能X"}
```

为什么验证用JSON：二元通过/失败加原因字符串支持自动重试逻辑。验证失败时，系统可自动重新生成（以失败原因作为额外上下文），无需人工干预。

### 2.3 去重：用代码，不用AI

去重使用 `difflib.SequenceMatcher` 配合 50% 相似度阈值——而非 AI 调用。

**为什么用代码去重**：
- 确定性：相同输入永远产出相同结果（AI 会引入随机性）
- 快速：每次比对毫秒级 vs 秒级
- 可审计：阈值是明确且可调的
- 低成本：与数百条历史记录比对无 API 费用

**阈值选择（50%）**：太低（30%）让准重复内容通过。太高（70%）拒绝仅共享常见短语的内容。50% 捕获结构性重复同时允许主题相似性。

### 2.4 格式调整作为独立阶段

生成后，内容可能需要渠道特定格式化（添加链接、调整长度、插入表情）。这是独立的 AI 调用因为：

1. 生成器不应担心最终格式（关注点分离）
2. 格式规则独立于内容规则变化
3. 格式调整可被验证（如正则检查链接在重新格式化后是否存活）

**安全网**：格式调整后，验证关键元素（链接、产品名）是否保留。验证失败则回退到调整前版本。永远不要因为格式化而丢失数据。

### 2.5 图片生成的场景池模式

不要让 AI 发明场景（高方差，常不合理）：

```
方法：维护一个30+场景描述的策划池
       ↓
     每次生成随机选择一个
       ↓
     AI 渲染固定文字 + 选中场景
```

**为什么用场景池**：
- 质量控制：每个场景都经过人工审核
- 一致性：保证品牌适配的美学
- 多样性：从大型池随机选择确保视觉多样性
- 可预测性：AI 的创意自由被限定在渲染而非构思

---

## 3. 组件角色

### 3.1 内容生成引擎

**角色**：大规模产出多样化、符合品牌调性的推广文案。

关键设计决策：
- 主题轮换（14个主题循环）确保话题多样性无需人工选择
- 每主题素材数据库提供事实基础
- 历史语料库支持自动去重
- 单次生成调用产出多渠道输出（群发、朋友圈、私信）

### 3.2 验证层

**角色**：防止事实错误到达终端用户。

设计：第二次 LLM 调用（或不同模型）将生成内容与验证素材数据库交叉参考。这捕获虚构的功能、不正确的描述或错误归属的内容。

### 3.3 调度引擎

**角色**：根据业务规则确定性地将内容映射到日期/渠道。

这是纯代码——无 AI 参与。业务规则（哪个渠道哪天发送、每个组什么时间发送）被编码为配置而非提示词。

### 3.4 部署编排器

**角色**：将最终确定的内容转化为平台 API 调用。

通过 RPA/API 集成处理任务创建、素材上传和调度。所有参数都是确定性的——AI 的工作在内容生成阶段已结束。

---

## 4. AI 边界定义

### 4.1 原则

**AI 处理发散性任务。代码处理收敛性任务。**

- 发散性：存在多个有效输出（创意写作、图像渲染）
- 收敛性：只有一个正确输出（调度、日期计算、API调用）

### 4.2 AI 应该做什么

| 任务 | 为什么用AI |
|------|------------|
| 生成推广文案 | 需要创意、文化细微差别、多样性 |
| 验证事实声明 | 需要关于语义的推理 |
| 调整文本格式 | 需要理解上下文来重构 |
| 渲染推广图片 | 需要视觉创意 |

### 4.3 AI 不应该做什么

| 任务 | 为什么用代码 |
|------|-------------|
| 安排哪天发送 | 确定性业务规则 |
| 计算主题轮换 | 简单模运算 |
| 内容去重 | 确定性字符串比对足够 |
| 上传素材到平台 | 固定参数的API调用 |
| 在文案中插入链接 | 模板替换，必须精确 |
| 确定发送时间 | 每渠道固定配置 |

### 4.4 灰色地带：人工审核

有些任务需要人工判断，AI 和代码都不能完全替代：

| 任务 | 为什么需要人工 |
|------|---------------|
| 部署前审批生成内容 | 品牌风险，AI无法判断的边缘情况 |
| 定义主题素材/卖点 | 需要产品知识和策略 |
| 设定业务规则（哪天、哪个渠道） | 战略决策 |
| 策划图片场景池 | 美学和品牌判断 |
| 处理验证边缘情况 | 当AI验证器不确定时 |

### 4.5 边界设计启发式

问三个问题来决定某步骤应该是AI、代码还是人工：

1. **是否只有一个正确答案？** → 代码
2. **是否有多个可接受答案且任务可重复？** → AI
3. **错误答案是否带来重大业务/品牌风险？** → 人工审核关卡

---

## 5. 设计模式总结

| 模式 | 实现方式 | 收益 |
|------|----------|------|
| 生成器-评论家 | 分离生成和验证调用 | 更高创意性且更高准确性 |
| 温度分层 | 每流水线阶段不同温度 | 每阶段针对目标优化 |
| 优先级锚定 | 图片提示词中的【最高优先级】标记 | 关键元素正确渲染 |
| 场景池 | 预策划场景，随机选择 | 受控多样性不损失质量 |
| 结构化输出 | 验证用JSON，解析用标记 | 可靠的流水线集成 |
| 回退安全 | 格式调整后正则验证 | 永不丢失关键数据 |
| 基于代码去重 | difflib 50%阈值 | 快速、确定性、可审计 |
| 基于参考的图片生成 | 基础图 + 修改提示词 | 低方差、高一致性 |

---

## 6. 应避免的反模式

1. **单一巨型提示词** — 不要让一个提示词同时生成、验证、格式化和调度。每个关注点需要独立调用和独立温度。

2. **用AI做确定性逻辑** — 不要用AI计算"下周二"或"14个主题中第7个"。代码对此100%可靠；AI不是。

3. **非结构化AI输出** — 如果下游代码需要解析AI输出，你必须指定精确格式。"写一个好的回复"不可解析；"输出包含'pass'和'reason'键的JSON"可解析。

4. **无验证的AI** — 永远不要直接部署AI生成的内容。在生成和发布之间必须有验证关卡（自动或人工）。

5. **图片生成的无限创意自由** — 无约束的图片生成产出不一致的品牌美学。用场景池、参考图和优先级标记限定创意空间。

---

*本方法论源自自动化内容分发系统的生产实施模式。*

---
---

# Private Domain AI Skill: Methodology Guide

---

## 1. Prompt Engineering Patterns

### 1.1 Prompt Architecture

Every effective prompt follows a layered structure:

```
Layer 1: Role Definition
Layer 2: Constraints
Layer 3: Output Format
Layer 4: Examples or References
```

**Role Definition**

Begin with a precise professional identity. Don't just say "you are an AI assistant" — define domain expertise and tone.

```
Pattern:  "You are a professional {domain} expert, skilled at {specific_skill}."
Example:  "You are a professional marketing copywriter, skilled at writing engaging promotional copy for children's educational products."
```

Why this works: LLMs respond to role-framing by activating relevant knowledge clusters. A "marketing copywriter for children's education" produces different output than a generic "helpful assistant."

**Constraints**

Constraints are the most powerful lever. Be numerical and verifiable:

- Character counts: "Each selling point must be 10-14 characters"
- Line counts: "Must be exactly 3 lines"
- Content rules: "No pricing information"
- Structural rules: "Line 1 = theme intro, Line 2 = core selling point, Line 3 = CTA"

Why numerical constraints: Vague instructions ("keep it short") produce inconsistent results. Exact numbers ("10-14 characters") create a verifiable contract between prompt and output.

**Output Format**

Define format with markers that code can parse:

```
For creative content:   Use section markers (===broadcast===, ===moments===)
For validation:         Use strict JSON ({"pass": true/false, "reason": "..."})
For structured data:    Use numbered lists with fixed prefixes
```

Why parseable formats: AI output must flow into downstream code. If your pipeline needs to extract "the broadcast copy" from a generation, the model must produce it in a predictable, machine-parseable structure.

### 1.2 Temperature Strategy

Temperature is not one-size-fits-all. Match it to the task's nature:

| Task Type | Temperature | Rationale |
|-----------|-------------|-----------|
| Creative generation | 0.8–0.9 | Maximize diversity, avoid repetitive patterns |
| Fact validation | 0.1 | Deterministic yes/no judgment, no creativity needed |
| Format adjustment | 0.3 | Preserve meaning while restructuring, slight flexibility |

**Key insight**: When the same content passes through multiple AI stages (generate → validate → adjust), each stage should have its own temperature. The creative stage needs high temperature for variety; the validation stage needs near-zero for reliability.

### 1.3 Prompt Anchoring

For image generation, use priority markers to control which instructions the model follows most strictly:

```
[HIGHEST PRIORITY] Text elements must be rendered precisely, no text errors allowed
[HIGH PRIORITY] Overall composition and color scheme
[NORMAL PRIORITY] Background details and decorative elements
```

Why priority markers: Image models have limited "attention budget." Without explicit priority, they may render a beautiful background but misspell the text. Priority markers tell the model where to spend its accuracy budget.

### 1.4 Reference-Based Generation

Instead of describing an image from scratch, provide a reference image + modification instructions:

```
Structure:
  reference_image: [base image URL/path]
  prompt: "Based on the reference, change {element_A} to {element_B}, keep {element_C} unchanged"
```

Why reference-based: Pure text-to-image has high variance. A reference image anchors the composition, style, and layout — the AI only needs to modify specific elements, dramatically reducing unwanted variation.

---

## 2. Workflow Design

### 2.1 Pipeline Architecture

```
[Generation] → [Validation] → [Deduplication] → [Formatting] → [Deployment]
     AI             AI            Code              AI/Code          Code
```

**Why this order matters**:

1. **Generation first**: Create content freely without constraints on uniqueness or formatting
2. **Validation second**: Catch factual errors before they propagate further
3. **Deduplication third**: Remove similar content against historical corpus (code-based, deterministic)
4. **Formatting fourth**: Adapt surviving content to channel-specific requirements
5. **Deployment last**: Push finalized content through deterministic scheduling

### 2.2 Why Separate Validation from Generation

A single prompt saying "generate accurate marketing copy" is unreliable. Instead:

- **Generator (Model A, high temperature)**: Optimizes for creativity and engagement
- **Validator (Model B or same model, low temperature)**: Optimizes for accuracy against a fact database

This is the "generator-critic" pattern. The generator doesn't need to self-censor for accuracy (which reduces creativity). The validator doesn't need to be creative (which reduces reliability).

**Validation output must be structured**:
```json
{"pass": true, "reason": "selling points match verified material database"}
{"pass": false, "reason": "mentioned feature X which does not exist in this theme"}
```

Why JSON for validation: Binary pass/fail with a reason string enables automated retry logic. If validation fails, the system can automatically regenerate (with the failure reason as additional context) without human intervention.

### 2.3 Deduplication: Code, Not AI

Deduplication uses `difflib.SequenceMatcher` with a 50% similarity threshold — NOT an AI call.

**Why code-based dedup**:
- Deterministic: Same inputs always produce same result (AI would introduce randomness)
- Fast: Milliseconds vs seconds per comparison
- Auditable: The threshold is explicit and tunable
- Cheap: No API cost for comparing against hundreds of historical entries

**Threshold choice (50%)**: Too low (30%) lets near-duplicates through. Too high (70%) rejects content that merely shares common phrases. 50% catches structural repetition while allowing topical similarity.

### 2.4 Format Adjustment as a Separate Stage

After generation, content may need channel-specific formatting (adding links, adjusting length, inserting emojis). This is a separate AI call because:

1. The generator shouldn't worry about final formatting (separation of concerns)
2. Format rules change independently of content rules
3. Format adjustment can be validated (e.g., regex-check that links survived the reformatting)

**Safety net**: After format adjustment, validate that critical elements (links, product names) are preserved. If validation fails, fall back to the pre-adjustment version. Never lose data to a formatting pass.

### 2.5 Scene Pool Pattern for Image Generation

Instead of asking AI to invent scenes (high variance, often nonsensical):

```
Approach: Maintain a curated pool of 30+ scene descriptions
           ↓
         Randomly select one per generation
           ↓
         AI renders the fixed text + selected scene
```

**Why a scene pool**:
- Quality control: Every scene has been human-reviewed
- Consistency: Brand-appropriate aesthetics guaranteed
- Variety: Random selection from a large pool ensures visual diversity
- Predictability: The AI's creative freedom is bounded to rendering, not conception

---

## 3. Component Roles

### 3.1 Content Generation Engine

**Role**: Produce diverse, on-brand promotional text at scale.

Key design decisions:
- Theme rotation (14 themes cycling) ensures topic variety without human selection
- Per-theme material databases provide factual grounding
- Historical corpus enables automated deduplication
- Multi-channel output (broadcast, moments, DM) from single generation call

### 3.2 Validation Layer

**Role**: Prevent factual errors from reaching end users.

Design: A second LLM call (or different model) cross-references generated content against verified material databases. This catches hallucinated features, incorrect descriptions, or misattributed content.

### 3.3 Scheduling Engine

**Role**: Deterministically map content to dates/channels based on business rules.

This is pure code — no AI involved. Business rules (which channel sends on which day, what time each group sends) are encoded as configuration, not prompts.

### 3.4 Deployment Orchestrator

**Role**: Translate finalized content into platform API calls.

Handles task creation, material upload, and scheduling through RPA/API integration. All parameters are deterministic — AI's job ended at content generation.

---

## 4. AI Boundary Definition

### 4.1 The Principle

**AI handles divergent tasks. Code handles convergent tasks.**

- Divergent: Multiple valid outputs exist (creative writing, image rendering)
- Convergent: Only one correct output exists (scheduling, date calculation, API calls)

### 4.2 What AI Should Do

| Task | Why AI |
|------|--------|
| Generate promotional copy | Requires creativity, cultural nuance, variety |
| Validate factual claims | Requires reasoning about semantics |
| Adjust text formatting | Requires understanding context to restructure |
| Render promotional images | Requires visual creativity |

### 4.3 What AI Should NOT Do

| Task | Why Code |
|------|----------|
| Schedule which day to send | Deterministic business rule |
| Calculate theme rotation | Simple modular arithmetic |
| Deduplicate content | Deterministic string comparison is sufficient |
| Upload materials to platforms | API call with fixed parameters |
| Insert links into copy | Template substitution, must be exact |
| Determine send time | Fixed configuration per channel |

### 4.4 The Gray Zone: Human Review

Some tasks need human judgment that neither AI nor code can fully replace:

| Task | Why Human |
|------|-----------|
| Approve generated content before deployment | Brand risk, edge cases AI can't judge |
| Define theme materials/selling points | Requires product knowledge and strategy |
| Set business rules (which day, which channel) | Strategic decisions |
| Curate image scene pool | Aesthetic and brand judgment |
| Handle validation edge cases | When AI validator is uncertain |

### 4.5 Boundary Design Heuristic

Ask three questions to decide if a step should be AI, code, or human:

1. **Is there exactly one correct answer?** → Code
2. **Are there multiple acceptable answers and the task is repeatable?** → AI
3. **Does a wrong answer carry significant business/brand risk?** → Human review gate

---

## 5. Design Patterns Summary

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| Generator-Critic | Separate generation and validation calls | Higher creativity AND accuracy |
| Temperature Stratification | Different temp per pipeline stage | Each stage optimized for its goal |
| Priority Anchoring | Explicit priority markers in image prompts | Critical elements rendered correctly |
| Scene Pool | Pre-curated scenes, random selection | Controlled variety without quality loss |
| Structured Output | JSON for validation, markers for parsing | Reliable pipeline integration |
| Fallback Safety | Regex-validate after format adjustment | Never lose critical data |
| Code-Based Dedup | difflib at 50% threshold | Fast, deterministic, auditable |
| Reference-Based Image Gen | Base image + modification prompt | Low variance, high consistency |

---

## 6. Anti-Patterns to Avoid

1. **Single monolithic prompt** — Don't ask one prompt to generate, validate, format, and schedule. Each concern needs its own call with its own temperature.

2. **AI for deterministic logic** — Don't use AI to calculate "next Tuesday" or "theme index 7 of 14." Code is 100% reliable for this; AI is not.

3. **Unstructured AI output** — If downstream code needs to parse AI output, you MUST specify the exact format. "Write a nice response" is unparseable; "Output JSON with keys 'pass' and 'reason'" is parseable.

4. **AI without validation** — Never deploy AI-generated content directly. Always have a validation gate (automated or human) between generation and publication.

5. **Unlimited creative freedom in images** — Unconstrained image generation produces inconsistent brand aesthetics. Use scene pools, reference images, and priority markers to bound the creative space.

---

*This methodology is derived from production implementation patterns in automated content distribution systems.*
