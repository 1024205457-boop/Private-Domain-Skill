# Private Domain Operations Skill / 私域推送运营技能

## Overview / 概述

This skill manages an end-to-end private domain content pipeline covering:
1. **Content Generation** — AI-generated promotional copy (text)
2. **Image Generation** — AI-generated promotional images
3. **Deployment** — Automated task configuration across multiple channels

本技能管理端到端的私域内容流水线：
1. **文案生成** — AI 生成推广文案
2. **图片生成** — AI 生成推广图片
3. **任务配置** — 自动化多渠道推送配置

---

## Pipeline Overview / 流水线总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    FULL DEPLOYMENT PIPELINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Phase 1: Content Generation (Python)                            │
│  ├─ Mini-course copy (14 themes × 3 channels)                   │
│  ├─ Member copy (11 categories × 3 channels)                    │
│  └─ Write to cloud spreadsheet                                   │
│                                                                   │
│  Phase 2: Image Generation (Node.js) [Parallel]                 │
│  ├─ Mini-course broadcast images (theme-specific)                │
│  ├─ Member broadcast images (scene-based)                        │
│  └─ Member moments images (composite)                            │
│                                                                   │
│  Phase 3: Task Deployment (Node.js)                              │
│  ├─ Update material library                                      │
│  ├─ Create broadcast tasks (daily)                               │
│  ├─ Create moments tasks (daily)                                 │
│  ├─ Create DM tasks (specific days only)                         │
│  └─ Send notification                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Content Generation / 文案生成

### Master Command / 主命令

```bash
python generate_all.py --start MMDD --days N [--format-prompt "..."]
```

### Products / 产品线

| Product | Channels | Schedule | AI Model |
|---------|----------|----------|----------|
| Mini-course (小课包) | Broadcast, Moments, DM | Daily / DM=Saturday only | glm-4-flash |
| Member (会员) | Broadcast, Moments, DM | Daily / DM=Tuesday only | glm-4-flash |

### Mini-Course Content Rules / 小课包文案规则

**Theme Rotation (14 themes cycling):**
蘑菇 → 火星 → 月球 → 钟表 → 建筑里的神兽 → 有毒生物 → 伪装大师 → 非洲动物 → 故宫 → 敦煌 → 钱币 → 四大发明 → 极地动物 → 长江黄河

**Template Structure:**
```
{theme_emoji} {price}小课包上新｜{AI_generated_title}
{link_1}
✨ {selling_point_1}  (10-14 chars)
🎯 {selling_point_2}  (10-14 chars)
🎬 {selling_point_3}  (10-14 chars)

🧧{upsell_text}
{link_2}
```

**Generation Rules:**
- Title: 8-14 characters, theme-specific, no generic phrases
- Selling points: Based on real course content (THEME_MATERIALS), fact-checked
- Emoji prefixes: Random unique selection from ✨🎯🎬📚🌟🔬💡🎪
- Optional decoration frame (30% probability, reduces to 2 selling points)
- Deduplication against historical titles and selling points

### Member Content Rules / 会员文案规则

**Category Rotation (11 categories):**
生物 → 天文 → 物理 → 化学 → 数学 → 人体 → 历史 → 文学 → 地理 → 科技 → 建筑

**Template Structure:**
```
📚【{category}】百科科普课堂
{opening}{AI_knowledge_point}

{brand}会员首月仅需5️⃣9️⃣元
{link}
🌟{guide_text}
```

**Generation Rules:**
- Knowledge point: ~34 characters, 2-3 sentences
- Opening phrases: Random from [你知道吗？/ 你注意过吗？/ 你发现了吗？/ 有个秘密，/ 涨知识啦！]
- Guide texts: Random from [带孩子探索更多奥秘 / 陪孩子发现世界的精彩 / ...]
- Semantic deduplication (50% similarity threshold)
- Fact-checked by LLM (glm-4-plus)

### Output / 输出

- Excel files (versioned, per product)
- Cloud spreadsheet (Shimo) columns:
  - A: Date (MM-DD format)
  - B: Weekday
  - C: Member DM (Tuesday only)
  - D: Member Broadcast
  - E: Member Moments
  - F: Mini-course DM (Saturday only)
  - G: Mini-course Broadcast
  - H: Mini-course Moments

---

## Phase 2: Image Generation / 图片生成

### Mini-Course Images / 小课包图片

**Command:**
```bash
node batch.js --all -n 5              # All themes, 5 images each
node batch.js -t 故宫 -t 敦煌 -n 3    # Specific themes
```

**Specifications:**
- Dimensions: 2600×2080 (5:4 ratio)
- Model: doubao-seedream-4.5
- Style: Cartoon characters (boy + monkey) in theme-specific scene
- Text overlay: Theme title + subtitle (for some themes)
- Output: `output/{theme_id}/{theme}_v{batch}_{index}.jpg`

**Theme Configuration (per theme):**
```javascript
{
  id: "gugong",           // Output folder name
  name: "故宫",           // Chinese name
  title: "紫禁城探秘",    // Main title
  subtitle: "皇宫奥秘·千年传承",  // Subtitle
  scene: "...",           // Detailed AI scene description
  overlaySubtitle: true,  // Whether to add text overlay
  referenceImage: "..."   // Theme-specific reference
}
```

### Member Broadcast Images / 会员群发图

**Command:**
```bash
node generate.js -t qunfa -s 0701 -d 7 -n 3
```

**Specifications:**
- Dimensions: 2600×2080
- Style: Characters in diverse daily scenes (30 scene pool)
- Fixed text elements: "{promo_text_1}", "{promo_text_2}", "{cta_button}"
- Output: `output/qunfa/{MMDD}/{product_card}{MMDD}_{index}.jpg`

### Member Moments Images / 会员朋友圈图

**Two-step process:**

**Step 1: Generate background**
```bash
node generate.js -t pyq -s 0701 -d 7 -n 3
```
- Dimensions: 2048×3072 (2:3 vertical)
- Pure scenic background with top text only
- Output: `output/pyq/{MMDD}/{MMDD}朋友圈_{index}.jpg`

**Step 2: Composite final image**
```bash
node compose-pyq.js --dir output/pyq
```
- Layers: Background + Monkey film strip + Rights card + QR code
- Output: `output/pyq/pyq_composed/{MMDD}/`

---

## Phase 3: Deployment / 推送配置

### Master Command / 主命令

```bash
node api/配置推送.js START_DATE END_DATE [OPTIONS]
```

**Options:**
| Flag | Description |
|------|-------------|
| `--generate` | Run content generation first |
| `--小课包` | Mini-course only |
| `--会员` | Member only |
| `--dacu` | Large promotion mode |
| `--format-prompt "..."` | AI format adjustment |

### Task Types Created / 创建的任务类型

| Task | Channel | Schedule | Time |
|------|---------|----------|------|
| Mini-course Broadcast | WeChat Group | Daily | 19:00 |
| Mini-course Moments | Moments | Daily | 19:00 |
| Mini-course DM | Private Message | Saturday | 18:00/19:00/20:00 |
| Member Broadcast | WeChat Group | Daily | 12:00 |
| Member Moments | Moments | Daily | 12:00 |
| Member DM | Private Message | Tuesday | 12:00/13:00/14:00 |

### Material Library / 素材库

**Folders:**
| Folder | Content | Naming Pattern |
|--------|---------|----------------|
| 群发 | Member broadcast cards | `{product_card}-{MMDD}.jpg` |
| 朋友圈 | Member moments images | `{MMDD}朋友圈.jpg` |
| 私信 | Member DM cards | Same as 群发 |
| 小课包 | Mini-course broadcast cards | `{theme}-{MMDD}.png` |
| 小课包朋友圈 | Mini-course moments | `{theme}·百科朋友圈.jpg` |

**Material Library Notes:**
- Broadcast/DM tasks use mini-program cards from material library (needs update before task creation)
- Moments tasks upload images directly (no material library needed)
- Material library update command: `node 素材库更新-api.js --qunfa/--sixin/--dacu`

### Deployment Web Panel / 部署面板

A web-based UI (Flask + SocketIO) at `localhost:5006` providing:
- One-click execution
- Real-time log streaming
- Template editing with variable highlighting
- Material file management
- Link configuration
- Scheduled execution

---

## Key Configuration / 关键配置

### Environment Variables / 环境变量

```bash
ZHIPUAI_API_KEY=...     # ZhipuAI API key for text generation
MODAI_API_KEY=...       # ModAI API key for image generation
COOKIE=...              # Session cookie for internal services
WEBHOOK_URL=...         # WeChat Work bot webhook
SHIMO_SHEET_GUID=...    # Cloud spreadsheet document ID
SHIMO_USER_ID=...       # Spreadsheet user ID
SECRET_KEY=...          # Flask secret key
PORT=5006               # Server port
```

### Data Files / 数据文件

| File | Purpose |
|------|---------|
| `data/链接.xlsx` | All promotional links (2 sheets: mini-course, member) |
| `data/推送进度.json` | Theme/category rotation position |
| `data/小课包文案历史.json` | Title/selling-point dedup history |
| `data/会员知识点历史.json` | Knowledge-point dedup history |

---

## Common Operations / 常用操作

### Weekly Routine / 每周例行

```bash
# 1. Generate content for next week (Mon-Sun)
python scripts/generate/generate_all.py --start 0707 --days 7

# 2. Generate images (if new themes or dates needed)
cd image-generation/小课包群发图 && node batch.js -t 故宫 -n 5
cd image-generation/会员素材 && node generate.js -t qunfa -s 0707 -d 7

# 3. Deploy all tasks
node emon/api/配置推送.js 0707 0713 --generate
```

### One-Command Full Pipeline / 一键全流程

```bash
node emon/api/配置推送.js 0707 0713 --generate
```

This single command will:
1. Generate all content (calls Python internally)
2. Write to spreadsheet
3. Update material library
4. Create all broadcast/moments/DM tasks
5. Send webhook notification

### Emergency Operations / 紧急操作

```bash
# Stop a running task
node emon/停止任务.js "任务名关键词"

# Check account status
node emon/账号离线检查.js

# Update material library only
node emon/素材库更新-api.js --qunfa 0707
```

---

## Tech Stack / 技术栈

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Text AI | ZhipuAI (glm-4-flash/plus) | Copywriting & fact-checking |
| Image AI | ModAI (doubao-seedream-4.5) | Image-to-image generation |
| Backend | Python (Flask + SocketIO) | Web panel & content gen |
| Automation | Node.js | RPA API calls & image gen |
| Data | Excel (openpyxl), Shimo API | Config & content storage |
| Messaging | Emon RPA Platform | Task execution |
| Notification | WeChat Work Webhook | Deployment alerts |

---

## Constraints & Rules / 约束与规则

1. **DM Schedule**: Mini-course DM = Saturday only; Member DM = Tuesday only
2. **Theme Cycle**: 14 themes rotate in fixed order, tracked in progress file
3. **Deduplication**: AI content is checked against full history before acceptance
4. **Fact Checking**: All knowledge claims verified by secondary LLM call
5. **Material Order**: Material library MUST be updated BEFORE creating tasks
6. **Moments Independence**: Moments tasks upload directly, never touch material library
7. **Date Format**: Spreadsheet dates written as `MM-DD` strings (not integers, not YYYY-MM-DD)
8. **Image Naming**: Strict conventions per folder (see Material Library table)
9. **No Stopping Moments Tasks**: Once configured, moments tasks should not be stopped
10. **Creator Filter**: When searching tasks, always filter by creator to avoid conflicts
