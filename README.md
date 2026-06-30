# Private Domain Skill / 私域运营技能

A Claude Code skill for automating private domain content generation, image creation, and multi-channel deployment.

这是一个 Claude Code 技能，用于自动化私域内容生成、图片制作和多渠道推送配置。

## What This Skill Does / 技能说明

This skill encapsulates the complete workflow for:
- AI-powered promotional copywriting generation (text)
- AI-powered promotional image generation (visual)
- One-click deployment to messaging platforms (broadcast, moments, DM)

本技能封装了完整的私域推送工作流：
- AI 文案生成（文字内容）
- AI 图片生成（视觉素材）
- 一键配置推送任务（群发、朋友圈、私信）

## Usage / 使用方式

```
/private-domain
```

See [SKILL.md](./SKILL.md) for the full skill specification.

详见 [SKILL.md](./SKILL.md) 获取完整技能规范。

## Architecture / 架构

```
Content Generation (Python/ZhipuAI)
        ↓
Image Generation (Node.js/ModAI)
        ↓
Spreadsheet Write (Shimo API)
        ↓
Task Deployment (Emon RPA API)
        ↓
Notification (WeChat Webhook)
```

## Related Repository / 相关仓库

- [Private-Domain-Operations](https://github.com/1024205457-boop/Private-Domain-Operations) — The implementation code
