# PPTalker — 演示文稿 → 配音讲解视频

> 把一份 **PPT / PDF / HTML 幻灯片**，一键变成带 **AI 配音 + 多语种字幕**的专业讲解视频。逐页讲稿可由 Agent 自动生成，**确认后再渲染**。

一个适配 Claude Code / Codex / CodeBuddy 等 Agent 环境的开源 **Skill**。它编排 [PPTalker](https://www.pptalker.com) 的 MCP 工具，走一条「上传 → 解析 → 逐页讲稿（抽取或本地生成）→ 用户确认 → 渲染 → 轮询出片」的稳健流程，把静态幻灯片变成可直接分享的讲解视频。

[English](./README.en.md) · [贡献指南](./CONTRIBUTING.md) · [LICENSE](./LICENSE)

---

## 它解决什么问题

幻灯片做完了，真正费时的是**讲出来**——录屏要反复重录，配音要对口型，字幕要逐句敲，换一种语言又得从头来。

这个 skill 把「幻灯片 → 讲解视频」这条最枯燥的链路自动化：

- **不用录屏、不用对口型**：逐页讲稿交给 AI 配音，自动生成与朗读同步的字幕。
- **多语种一键切换**：中 / 英 / 日 / 西等，50+ 真人级音色任选，换语言只需改一个参数。
- **讲稿可控、先审后渲**：原生备注自动抽取，没有备注则按专业口吻本地生成，**你确认后才会消耗额度渲染**。

一句话：**让「做完的 PPT」自己开口讲，变成能存档、能分享、能投放的讲解视频。**

---

## 和 content-to-slides 是绝配 🔗

[**content-to-slides**](https://github.com/YoungXu06/content-to-slides) 负责**从 0 生产幻灯片**：把一篇文章 / 一个视频链接 / 一个 GitHub 仓库，做成横屏 4:3 的 5–10 页讲解 slides + PDF + 逐页口播讲稿。

**PPTalker 负责把这些幻灯片讲出来**：接住 content-to-slides 产出的 `presentation.html` / `output.pdf`，配上 AI 语音和字幕，渲染成成片视频。

两者天然衔接，组成一条完整流水线：

```
   一个链接 / 一篇文章 / 一个仓库
              │
   content-to-slides   ──►  HTML / PDF 幻灯片  +  逐页讲稿 (script.md)
              │
   PPTalker (本 skill)  ──►  带 AI 配音 + 多语种字幕的讲解视频
              │
        发抖音 / B 站 / YouTube / 小红书 / 知识库存档
```

> content-to-slides 生成的 `script.md` 讲稿可直接作为本 skill 的「逐页讲稿」输入——无需重写，端到端从「一个链接」到「一条成片」。

---

## 它能产出什么

| 产出 | 说明 |
|------|------|
| 讲解视频 | 带 AI 配音 + 同步字幕的 MP4，按幻灯片顺序逐页讲解 |
| `share_url` | 永久公开播放链接（`pptalker.com/video/public/...`），可直接分享 / 嵌入 |
| 逐页讲稿 | 渲染前确认的 `final_notes`，专业口吻、TTS 友好，可复用 |

---

## 30 秒上手

装好 skill 并配置 MCP 后，直接对 Agent 说：

```
把这个 PPT 做成配音讲解视频：/abs/path/to/deck.pptx
把这份 PDF 幻灯片转成中文讲解视频：/abs/path/to/slides.pdf
讲解一下这个 HTML slides，输出英文配音：/abs/path/to/presentation.html
```

Agent 会自动：上传 → 解析逐页内容 → 抽取或生成逐页讲稿 → **给你确认/修改** → 选音色 → 渲染 → 轮询出片，最后只给你一个永久 `share_url`。

---

## 支持的输入

| 格式 | 讲稿来源 |
|------|---------|
| `.pptx` / `.ppt` | 优先用幻灯片原生备注；无备注则本地按专业口吻生成 |
| `.pdf` / `.html` / `.htm` | 始终先问你：自己提供讲稿，还是从幻灯片内容自动生成 |

> 配合 content-to-slides：它产出的 HTML/PDF 直接作为这里的输入，讲稿也可沿用它生成的 `script.md`。

---

## 适合 / 不适合

**适合：**
- 把已经做好的 PPT / PDF / HTML 幻灯片快速转成配音讲解视频
- 同一套幻灯片要做多语种版本（中 / 英 / 日 / 西）
- 内容创作者把 content-to-slides 生成的 deck 一键成片，投放短视频平台

**不适合：**
- 需要复杂转场动效、镜头运镜的影视级视频
- 没有幻灯片、纯口播的播客式音频（本 skill 以幻灯片逐页为骨架）

---

## 工作流

```
1  核额度    pptalker_get_account        确认剩余分钟数
2  上传      pptalker_upload_ppt         本地文件 → request_id
3  解析      pptalker_parse_ppt          → 逐页图、原生备注、页数
4  定讲稿     抽取 / 用户提供 / 本地生成    →  final_notes
5  本地生成   （可选）按语言/字数/语气生成    专业 TTS 讲稿
6  选音色     pptalker_list_voices        50+ 音色（可选）
7  渲染      pptalker_create_video       ⚠️ 必须先经用户确认讲稿
8  轮询      pptalker_get_video_status   出片 → share_url
```

> **核心安全约束**：未经用户明确确认 `final_notes`，绝不调用 `pptalker_create_video`。完整流程见 [SKILL.md](./SKILL.md)。

---

## 安装

本 skill 是纯文件结构（`SKILL.md` + `references/`），把整个目录放进 Agent 的 skills 搜索路径即可。

```bash
# Claude Code
git clone https://github.com/YoungXu06/pptalker-skill.git \
  ~/.claude/skills/pptalker

# CodeBuddy
git clone https://github.com/YoungXu06/pptalker-skill.git \
  ~/.codebuddy/skills/pptalker
```

---

## 关于 PPTalker

[**PPTalker**](https://www.pptalker.com) 是一款「演示文稿一键转配音视频」的在线产品：上传 PPT / PDF / HTML，自动配 **AI 真人级语音 + 多语种字幕**，几分钟出片。支持 **20+ 语言、50+ 音色与声音克隆**，无需录屏、无需对口型、无需剪辑——特别适合课程讲解、产品介绍、知识科普与短视频创作者。本 skill 正是把 PPTalker 的能力接入 Agent，让你一句话完成「幻灯片 → 成片」。

> 🌐 官网 https://www.pptalker.com ｜ 🤖 Agent 入口 https://www.pptalker.com/agents

---

## 依赖：配置 PPTalker MCP

本 skill 编排 PPTalker 的 MCP 工具，需先在你的 MCP 客户端（Claude Code / Codex / CodeBuddy / Cursor 等）中注册。最简配置只需一个 API Key：

```json
{
  "mcpServers": {
    "pptalker": {
      "command": "npx",
      "args": ["-y", "@pptalker/mcp-server"],
      "env": { "PPTALKER_API_KEY": "pptk_live_..." }
    }
  }
}
```

### 可配置的系统参数（env）

除 `PPTALKER_API_KEY` 外其余均为**可选的默认值**——渲染时若未在对话里单独指定，就套用这里的配置。下面是一份带全部参数的完整示例：

```json
{
  "mcpServers": {
    "pptalker": {
      "command": "npx",
      "args": ["-y", "@pptalker/mcp-server"],
      "env": {
        "PPTALKER_API_KEY": "pptk_live_xxxx",
        "PPTALKER_LANGUAGE": "Chinese",
        "PPTALKER_VOICE": "智斌",
        "PPTALKER_SPEED": "100",
        "PPTALKER_SUBTITLES": "true",
        "PPTALKER_SUBTITLE_SIZE": "medium",
        "PPTALKER_SUBTITLE_COLOR": "white",
        "PPTALKER_SUBTITLE_BG": "semi-transparent",
        "PPTALKER_RESOLUTION": "1.5"
      }
    }
  }
}
```

| 环境变量 | 必填 | 默认值 | 可选值 / 说明 |
|---------|:---:|--------|--------------|
| `PPTALKER_API_KEY` | ✅ | — | 以 `pptk_live_` 开头的 API Key |
| `PPTALKER_API_BASE` | | `https://api.pptalker.com` | 自建 / 代理时覆盖 API 地址 |
| `PPTALKER_LANGUAGE` | | `Chinese`（zh-CN） | 友好名（`Chinese`/`English`/`Japanese`…）、短码（`zh`/`en`/`ja`…）或 BCP-47（`zh-CN`/`en-US`/`pt-BR`…），大小写不敏感 |
| `PPTALKER_VOICE` | | 随语言的默认音色 | 音色名，如 `Matthew`、`智斌`，或你的克隆音色名 |
| `PPTALKER_SPEED` | | `100` | 语速，`80`–`150` |
| `PPTALKER_SUBTITLES` | | `true` | 是否显示字幕：`true` / `false` |
| `PPTALKER_SUBTITLE_SIZE` | | `medium` | 字幕字号：`small` / `medium` / `large` |
| `PPTALKER_SUBTITLE_COLOR` | | `white` | 字幕字体颜色名，如 `white`、`yellow` |
| `PPTALKER_SUBTITLE_BG` | | `semi-transparent` | 字幕背景：`semi-transparent` / `none` / `solid` |
| `PPTALKER_RESOLUTION` | | `1.5` | 分辨率缩放，`1.0`–`2.0` |

> 这些只是**默认值**：同一套配置下，仍可在对话里临时改语言 / 音色 / 语速等，渲染时传入的参数会覆盖默认。语言与音色的完整对照见 [SKILL.md](./SKILL.md)。

- API Key 申请：https://www.pptalker.com/profile
- 额度与定价：https://www.pptalker.com/pricing （1 额度 = 1 分钟成片；上传、解析、本地生成讲稿均免费）

---

## 目录结构

```
pptalker-skill/
├── SKILL.md                          # Skill 主文件：触发词、工具图、8 步工作流、硬规则
├── README.md                         # 中文说明（本文件）
├── README.en.md                      # 英文说明
├── CONTRIBUTING.md                   # 贡献指南
├── LICENSE                           # AGPL-3.0
├── .gitignore
└── references/                       # 分层知识库（Agent 按需读取）
    ├── note-generation-prompt.md     # 讲稿生成 prompt 模板 + TTS 安全规则 + 语气块
    ├── error-handling.md             # 硬规则、错误码、超时策略、套餐限制
    └── examples.md                   # 两个完整范例 + 各格式取文recipe
```

---

## 关键设计原则

1. **先审后渲** —— 讲稿未经用户确认，绝不渲染（避免浪费额度、避免错误成片）。
2. **渲染超时不是失败** —— `create_video` 超时说明任务已提交，**绝不重试**，改去轮询，避免重复扣费。
3. **重试至多 3 次** —— 任一 MCP 工具失败最多重试 3 次，仍失败就停下报错、问用户怎么办。
4. **讲稿 TTS 友好** —— 纯口语文本，禁 Markdown / 列表符号，型号版本号去连字符（"GPT-5" → "GPT 5"），中日文 CJK 与 ASCII 间不留空格。
5. **PDF/HTML 必问讲稿来源** —— 不静默使用嵌入备注，始终让用户选「自己提供 / 自动生成」。
6. **只暴露 `share_url`** —— 永久公开链接给用户；内部临时直链仅供调试，绝不外露。

---

## 常见问题

**Q：渲染卡很久 / MCP 调用超时了？**
A：`create_video` 超时是正常的（长任务已在服务端提交），**不要重试**，去轮询 `pptalker_get_video_status`（复用同一个 `request_id`），重复调用可能重复扣费。

**Q：额度怎么算？**
A：1 额度 = 1 分钟成片。10 页幻灯片大约 3–5 分钟 → 3–5 额度。上传、解析、本地生成讲稿都免费。

**Q：能换语言 / 换音色吗？**
A：可以。`pptalker_list_voices` 浏览 50+ 音色，渲染时传 `language` + `voice` 即可；讲稿语言要和音色语言一致。

更多错误码与超时处理见 [references/error-handling.md](./references/error-handling.md)。

---

## 贡献

欢迎改进讲稿 prompt、补充取文策略、完善错误处理。请先读 [CONTRIBUTING.md](./CONTRIBUTING.md)。

核心约定：改流程同步更新 SKILL.md 对应步骤；讲稿规则改动同步 `note-generation-prompt.md`；不削弱「先审后渲、超时不重试、至多重试 3 次」三条硬规则。

---

## 相关项目

- [**content-to-slides**](https://github.com/YoungXu06/content-to-slides) —— 把文章 / 视频 / GitHub 仓库做成 5–10 页讲解 slides + PDF + 逐页讲稿。它产出的幻灯片正是本 skill 的理想输入，两者组成「内容 → 幻灯片 → 讲解视频」完整流水线。

---

## License

[AGPL-3.0](./LICENSE) © PPTalker Skill contributors
