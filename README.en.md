# PPTalker — Presentation → Narrated Video

> Turn a **PPT / PDF / HTML deck** into a professional narrated video with **AI voiceover + multilingual subtitles** in one shot. Per-slide notes can be auto-generated and are **confirmed before rendering**.

An open-source **Skill** for AI-agent environments (Claude Code / Codex / CodeBuddy). It orchestrates the [PPTalker](https://www.pptalker.com) MCP tools through a robust pipeline — upload → parse → per-slide notes (extracted or locally generated) → user confirms → render → poll — turning static slides into a ready-to-share narrated video.

[中文说明](./README.md) · [Contributing](./CONTRIBUTING.md) · [LICENSE](./LICENSE)

---

## What problem it solves

The deck is done — the time sink is **narrating it**: re-recording screen captures, matching voice to slides, typing subtitles line by line, then redoing it all for another language.

This skill automates the most tedious link, "slides → narrated video":

- **No screen recording, no lip-sync**: per-slide notes go to AI voiceover, with subtitles auto-synced to the audio.
- **One-flag multilingual**: zh / en / ja / es and more, 50+ lifelike voices — switching language is a single parameter.
- **Controllable notes, review-then-render**: native notes are auto-extracted; if absent, they're generated locally in a professional voice — **credits are only spent after you confirm**.

In one line: **let a finished deck speak for itself — into a video you can archive, share, and publish.**

---

## A perfect match with content-to-slides 🔗

[**content-to-slides**](https://github.com/YoungXu06/content-to-slides) **produces slides from scratch**: it turns an article, a video link, or a GitHub repo into a landscape 4:3 5–10 page explainer deck + PDF + per-slide speaker script.

**PPTalker makes those slides talk**: it picks up the `presentation.html` / `output.pdf` from content-to-slides, adds AI voice and subtitles, and renders a finished video.

Together they form one seamless pipeline:

```
   one link / one article / one repo
              │
   content-to-slides   ──►  HTML / PDF deck  +  per-slide script (script.md)
              │
   PPTalker (this skill) ──►  narrated video with AI voiceover + subtitles
              │
        publish to TikTok / YouTube / Bilibili / Xiaohongshu / archive
```

> The `script.md` produced by content-to-slides can be fed directly as this skill's per-slide notes — no rewriting, end-to-end from "one link" to "one finished video".

---

## What you get

| Output | Description |
|--------|-------------|
| Narrated video | MP4 with AI voiceover + synced subtitles, explained slide by slide |
| `share_url` | Permanent public player link (`pptalker.com/video/public/...`), share / embed directly |
| Per-slide notes | The `final_notes` confirmed before rendering — professional, TTS-safe, reusable |

---

## 30-second start

After installing the skill and configuring MCP, just ask your agent:

```
Make a narrated video from this PPT: /abs/path/to/deck.pptx
Turn this PDF deck into a Chinese narrated video: /abs/path/to/slides.pdf
Narrate this HTML deck with English voiceover: /abs/path/to/presentation.html
```

The agent will: upload → parse per-slide content → extract or generate notes → **let you confirm/edit** → pick a voice → render → poll, then hand you a permanent `share_url`.

---

## Supported inputs

| Format | Notes source |
|--------|--------------|
| `.pptx` / `.ppt` | Prefer native slide notes; if none, generate locally in a professional voice |
| `.pdf` / `.html` / `.htm` | Always ask first: provide your own script, or auto-generate from slide content |

> Pairs with content-to-slides: its HTML/PDF output is a direct input here, and its `script.md` can be reused as the notes.

---

## Good for / Not for

**Good for:**
- Quickly turning finished PPT / PDF / HTML decks into narrated videos
- Producing multilingual versions (zh / en / ja / es) of the same deck
- Creators turning content-to-slides decks into finished short-videos in one step

**Not for:**
- Cinematic videos needing complex transitions or camera moves
- Pure-voice podcast audio with no slides (this skill is slide-by-slide by design)

---

## Workflow

```
1  Credits    pptalker_get_account        check remaining minutes
2  Upload     pptalker_upload_ppt         local file → request_id
3  Parse      pptalker_parse_ppt          → per-slide images, native notes, page count
4  Resolve    extract / user-provided / local generation  →  final_notes
5  Generate   (optional) by language / words / tone        professional TTS notes
6  Voice      pptalker_list_voices        50+ voices (optional)
7  Render     pptalker_create_video       ⚠️ requires confirmed notes
8  Poll       pptalker_get_video_status   done → share_url
```

> **Core safety rule**: never call `pptalker_create_video` until the user explicitly confirms `final_notes`. Full flow in [SKILL.md](./SKILL.md).

---

## Install

This skill is a plain file structure (`SKILL.md` + `references/`). Drop the directory into your agent's skills search path.

```bash
# Claude Code
git clone https://github.com/YoungXu06/pptalker-skill.git \
  ~/.claude/skills/pptalker

# CodeBuddy
git clone https://github.com/YoungXu06/pptalker-skill.git \
  ~/.codebuddy/skills/pptalker
```

---

## About PPTalker

[**PPTalker**](https://www.pptalker.com) is an online "deck → narrated video in one click" product: upload PPT / PDF / HTML and it auto-adds **lifelike AI voiceover + multilingual subtitles**, rendering a finished video in minutes. It supports **20+ languages, 50+ voices, and voice cloning** — no screen recording, no lip-sync, no editing — ideal for course explainers, product walkthroughs, knowledge content, and short-video creators. This skill simply wires PPTalker's power into your agent, so a single sentence takes you from "slides" to "finished video".

> 🌐 Website https://www.pptalker.com ｜ 🤖 Agent entry https://www.pptalker.com/agents

---

## Dependency: configure the PPTalker MCP

This skill orchestrates PPTalker's MCP tools. Register the server in your MCP client (Claude Code / Codex / CodeBuddy / Cursor, etc.) first. The minimal config needs only an API key:

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

### Configurable system parameters (env)

Apart from `PPTALKER_API_KEY`, all variables are **optional defaults** — applied at render time whenever you don't override them in the conversation. A full example with every parameter:

```json
{
  "mcpServers": {
    "pptalker": {
      "command": "npx",
      "args": ["-y", "@pptalker/mcp-server"],
      "env": {
        "PPTALKER_API_KEY": "pptk_live_xxxx",
        "PPTALKER_LANGUAGE": "English",
        "PPTALKER_VOICE": "Matthew",
        "PPTALKER_SPEED": "100",
        "PPTALKER_SUBTITLES": "true",
        "PPTALKER_SUBTITLE_SIZE": "medium",
        "PPTALKER_SUBTITLE_COLOR": "white",
        "PPTALKER_SUBTITLE_BG": "semi-transparent"
      }
    }
  }
}
```

| Variable | Required | Default | Accepted values / notes |
|----------|:--------:|---------|-------------------------|
| `PPTALKER_API_KEY` | ✅ | — | API key starting with `pptk_live_` |
| `PPTALKER_API_BASE` | | `https://api.pptalker.com` | Override the API URL (self-host / proxy) |
| `PPTALKER_LANGUAGE` | | `Chinese` (zh-CN) | Friendly name (`Chinese`/`English`/`Japanese`…), short code (`zh`/`en`/`ja`…), or BCP-47 (`zh-CN`/`en-US`/`pt-BR`…), case-insensitive |
| `PPTALKER_VOICE` | | language's default voice | PPTalker Voice name, e.g. `Matthew`, `智斌`, or your cloned voice name |
| `PPTALKER_SPEED` | | `100` | Speech speed, `80`–`150` |
| `PPTALKER_SUBTITLES` | | `true` | Show subtitles: `true` / `false` |
| `PPTALKER_SUBTITLE_SIZE` | | `medium` | Subtitle size: `small` / `medium` / `large` |
| `PPTALKER_SUBTITLE_COLOR` | | `white` | Subtitle font color name, e.g. `white`, `yellow` |
| `PPTALKER_SUBTITLE_BG` | | `semi-transparent` | Subtitle background: `semi-transparent` / `none` / `solid` |

> These are just **defaults**: with the same config you can still switch language / voice / speed on the fly in chat — parameters passed at render time override the defaults. Full language & voice mapping lives in [SKILL.md](./SKILL.md).

- Get an API key: https://www.pptalker.com/profile
- Credits & pricing: https://www.pptalker.com/pricing (1 credit = 1 minute of output; upload, parse, and local note generation are free)

---

## Directory structure

```
pptalker-skill/
├── SKILL.md                          # Main skill file: triggers, tool map, 8-step workflow, hard rules
├── README.md                         # Chinese docs
├── README.en.md                      # English docs (this file)
├── CONTRIBUTING.md                   # Contribution guide
├── LICENSE                           # AGPL-3.0
├── .gitignore
└── references/                       # Layered knowledge base (read on demand)
    ├── note-generation-prompt.md     # Note-generation prompt template + TTS-safe rules + voice blocks
    ├── error-handling.md             # Hard rules, error codes, timeout policy, plan limits
    └── examples.md                   # Two full walkthroughs + per-format extraction recipes
```

---

## Key design principles

1. **Review then render** — never render before the user confirms notes (no wasted credits, no wrong output).
2. **Render timeout is not a failure** — a `create_video` timeout means the task was already submitted; **never retry**, switch to polling to avoid double-charging.
3. **Retry at most 3 times** — any MCP tool retries at most 3 times total; then stop, report, and ask the user.
4. **TTS-safe notes** — plain spoken text, no Markdown/list markers; de-hyphenate model versions ("GPT-5" → "GPT 5"); no space between CJK and ASCII for zh/ja.
5. **Always ask for PDF/HTML notes source** — never silently use embedded notes; let the user choose "provide / auto-generate".
6. **Expose only `share_url`** — the permanent public link to the user; the internal temporary link is debug-only and never surfaced.

---

## FAQ

**Q: Rendering hangs / the MCP call timed out?**
A: A `create_video` timeout is normal (the long task is already submitted server-side). **Do not retry** — poll `pptalker_get_video_status` with the same `request_id`. Re-calling may double-charge credits.

**Q: How are credits counted?**
A: 1 credit = 1 minute of output. A 10-slide deck ≈ 3–5 min → 3–5 credits. Upload, parse, and local note generation are free.

**Q: Can I change language / voice?**
A: Yes. Browse 50+ voices with `pptalker_list_voices`, then pass `language` + `voice` at render time; the notes language must match the voice language.

More error codes and timeout handling in [references/error-handling.md](./references/error-handling.md).

---

## Contributing

Improvements to the note prompt, extraction strategies, and error handling are welcome. Read [CONTRIBUTING.md](./CONTRIBUTING.md) first.

Core conventions: keep SKILL.md steps in sync with flow changes; mirror note-rule changes in `note-generation-prompt.md`; never weaken the three hard rules ("review then render", "timeout no retry", "retry ≤ 3").

---

## Related projects

- [**content-to-slides**](https://github.com/YoungXu06/content-to-slides) — turns articles / videos / GitHub repos into 5–10 page explainer slides + PDF + per-slide script. Its decks are the ideal input for this skill, together forming a complete "content → slides → narrated video" pipeline.

---

## License

[AGPL-3.0](./LICENSE) © PPTalker Skill contributors
