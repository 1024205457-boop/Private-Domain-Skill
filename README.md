# Private Domain AI Skill / 私域 AI 技能

A methodology guide for building AI-powered content generation and distribution workflows.

一份关于构建 AI 驱动内容生成与分发工作流的方法论指南。

## What This Document Covers / 文档内容

This is **not** an operational runbook. It is a methodology document covering:

这**不是**操作手册。它是一份方法论文档，涵盖：

- **Prompt Engineering Patterns** — How to structure prompts for reliable, high-quality output  
  **提示词工程模式** — 如何构造提示词以获得可靠、高质量的输出

- **Workflow Architecture** — Why each stage exists and how they connect  
  **工作流架构** — 每个阶段为什么存在、如何连接

- **Component Roles** — What each piece of the system is responsible for  
  **组件角色** — 系统每个部分负责什么

- **AI Boundary Definition** — What AI should do vs. what code/humans must handle  
  **AI 边界定义** — AI 应该做什么 vs 代码/人工必须处理什么

## Read the Skill / 阅读技能文档

See [SKILL.md](./SKILL.md) for the full methodology guide (bilingual EN/CN).

详见 [SKILL.md](./SKILL.md) 获取完整方法论指南（中英双语）。

## Architecture Overview / 架构概览

```
[Generation (AI)] → [Validation (AI)] → [Dedup (Code)] → [Format (AI)] → [Deploy (Code)]
     ↑                    ↑                                    ↑
  High temp (0.8-0.9)  Low temp (0.1)                    Med temp (0.3)
```

## Key Principles / 核心原则

1. **AI handles divergent tasks; Code handles convergent tasks**  
   AI 处理发散性任务；代码处理收敛性任务

2. **Never deploy AI output without validation**  
   永远不要未经验证就部署 AI 输出

3. **Each pipeline stage gets its own temperature**  
   每个流水线阶段有独立的温度设置

4. **Structured output enables reliable automation**  
   结构化输出支撑可靠的自动化

## Related / 相关链接

- [Private-Domain-Operations](https://github.com/1024205457-boop/Private-Domain-Operations) — The implementation code / 实现代码
