---
name: pptalker
description: >
  Convert presentations (.pptx/.ppt/.pdf/.html/.htm) into professional
  narrated videos with AI voiceover and multilingual subtitles. Orchestrates
  the PPTalker MCP tools with a confirm-before-render workflow: extract or
  locally generate per-slide speaker notes, let the user review/edit them,
  then render the video. Use this skill whenever the user wants to "convert
  PPT to video", "make a video from slides", "narrate this deck", "ppt to
  video", "slides to video", "html slides to video", "slidev/reveal.js to
  video", "把 PPT 做成视频", "讲解这个 PPT", or provides a presentation file
  with the goal of producing a narrated video.
---

# PPTalker — Presentation → Narrated Video

End-to-end pipeline that turns a deck into a narrated video. Speaker notes
**must be confirmed by the user before video rendering** — never call
`pptalker_create_video` on un-reviewed notes.

## Retry Policy (applies to every MCP tool)

Retry any failing PPTalker MCP tool **at most 3 times total**. If it still
fails after 3 attempts, stop, report the last error to the user, and ask how
to proceed — do not keep looping. Exception: `pptalker_create_video` TIMEOUT
is NOT a failure (the task is already running) — never re-call it; switch to
polling `pptalker_get_video_status` instead (see Step 8).

## Tool Map

```
pptalker_get_account        check credits (minutes)
pptalker_upload_ppt         upload local file → request_id, file_name
pptalker_parse_ppt          → per-slide images, native notes, has_notes
pptalker_list_voices        browse 50+ voices
pptalker_create_video       render (REQUIRES confirmed `notes`)
pptalker_get_video_status   poll until done
```

## Setup

If MCP tools are missing, instruct the user to add to their MCP client. Only
`PPTALKER_API_KEY` is required; everything else is an optional default applied
at render time when not overridden in the tool call:

```json
{
  "mcpServers": {
    "pptalker": {
      "command": "npx",
      "args": ["-y", "@pptalker/mcp-server"],
      "env": {
        "PPTALKER_API_KEY": "pptk_live_...",
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

Supported env vars (defaults override-able per `pptalker_create_video` call):

| Variable | Required | Default | Accepted values |
|----------|:--------:|---------|-----------------|
| `PPTALKER_API_KEY` | yes | — | key starting with `pptk_live_` |
| `PPTALKER_API_BASE` | no | `https://api.pptalker.com` | API URL override (self-host / proxy) |
| `PPTALKER_LANGUAGE` | no | `Chinese` (zh-CN) | friendly name / short code / BCP-47, case-insensitive |
| `PPTALKER_VOICE` | no | language default | voice name (`Matthew`, `智斌`, …) or cloned voice name |
| `PPTALKER_SPEED` | no | `100` | `80`–`150` |
| `PPTALKER_SUBTITLES` | no | `true` | `true` / `false` |
| `PPTALKER_SUBTITLE_SIZE` | no | `medium` | `small` / `medium` / `large` |
| `PPTALKER_SUBTITLE_COLOR` | no | `white` | color name (`white`, `yellow`, …) |
| `PPTALKER_SUBTITLE_BG` | no | `semi-transparent` | `semi-transparent` / `none` / `solid` |
| `PPTALKER_RESOLUTION` | no | `1.5` | `1.0`–`2.0` |

These set the **defaults**; a `pptalker_create_video` call may still override
`language` / `voice` / `speed` / `show_subtitles` per render (Step 7).

API keys: https://www.pptalker.com/profile.

## Workflow

### 1. Verify credits

Call `pptalker_get_account()`. A 10-slide deck ≈ 3–5 min video ≈ 3–5
credits. If `left_credits` is insufficient, stop and link the user to
https://www.pptalker.com/pricing.

### 2. Upload

```
pptalker_upload_ppt(file_path="/abs/path/to/deck.pptx")
```

Supported: `.pptx`, `.ppt`, `.pdf`, `.html`, `.htm`. Returns `request_id`
and `file_name` — keep both.

> **Upload priority**: When multiple formats exist for the same deck, choose
> in this order: `.pptx` / `.ppt` > `.pdf` > `.html` / `.htm`

### 3. Parse

```
pptalker_parse_ppt(request_id, file_name)
```

Returns:

- `request_id`, `file_name` — **canonical** ids; use these (not the
  upload's) for all subsequent calls. HTML uploads switch `file_name` to
  `*.pdf`.
- `total_pages`, `ppt_img_urls` — slide image URLs (4-hour signed).
- `ppt_notes`, `has_notes`, `notes_source` — handle per Step 4 below.

### 4. Resolve `final_notes`

Branch on the source file type. **Do not** auto-use embedded notes for
PDF/HTML — always ask the user.

#### 4A. `.pptx` / `.ppt`

- `has_notes=true` → show `ppt_notes` via the **Notes Display Format**
  below; ask the user to confirm or edit.
- `has_notes=false` → tell the user the deck has no native notes; ask
  whether to auto-generate locally (default Yes) or paste/provide notes.

#### 4B. `.pdf` / `.html` / `.htm`

Always pause and ask the user (translate to their language):

```
This is a {ext} deck with {total_pages} slides. Where should I get the
per-slide narration from?

  1. I'll provide the script.
     - Paste the per-slide notes in chat
     - or give a local file path (.txt/.md/.json/.srt — one block per slide)

  2. Auto-generate locally from slide content.
     - I'll extract slide text from the source file and write the
       narration myself using this skill's prompt template.
     - You'll be asked for a few preferences before generation.
```

Then:

- **User-provided** → split into exactly `total_pages` blocks (blank-line
  separators, `[1]`/`#1`/`Slide 1` headers, or `---`). If the count
  mismatches, show the parsed count and ask for a fix. Treat as draft
  `final_notes`, then go through the **Notes Display Format**.
- **Auto-generate** → go to Step 5.

### 5. Local note generation

Triggered by 4A's auto-generate branch or 4B's option 2. 

#### 5.1 Ask the user for preferences

In one short message (translate to user's language):

```
Quick narration setup — pick or just say "default":

  • Language        : zh-CN | en-US | ja-JP | es-ES | …  (default: deck language)
  • Words per slide : ~30 / ~60 / ~100                   (default: ~60, hard cap 100)
  • Tone / audience : professional | friendly | educational | sales | …
                                                         (default: professional)
  • Extra context   : one line — key message, audience  (optional)
```

#### 5.2 Extract per-slide source text

Pull text per slide, in slide order. The result is `slide_texts` of
length `total_pages` (empty strings allowed for visual-only slides).
Extraction recipes per file type are in `references/examples.md`.

#### 5.3 Run the note generation prompt

Render the template in **`references/note-generation-prompt.md`** with
the user's answers and `slide_texts`. Two of the placeholders are
inline blocks chosen by language and tone — see that file's "Voice
block selection" and "Banlist block selection" sections to pick the
right ones. The prompt enforces TTS-safe formatting and a
professional, grounded voice **as generation-time constraints**, so
expect to use post-processing only as a safety net. Ask the LLM
(yourself) to return strict JSON, parse it into `final_notes` in slide
order, then show via the **Notes Display Format** for confirmation.

### 6. Voice (optional)

Default voice comes from MCP env config. If the user wants to choose,
run `pptalker_list_voices(language="zh"|"en"|...)` and pass the chosen
voice **name** to `pptalker_create_video`.

### 7. Render

**Only after the user has explicitly confirmed `final_notes`**:

```
pptalker_create_video(
  request_id,                    # canonical, from Step 3
  file_name,                     # canonical, from Step 3
  notes=final_notes,             # confirmed; length == total_pages
  language="zh-CN" | "en-US" | "ja-JP" | "es-ES",
  voice=<optional name>,
  speed=100, show_subtitles=true,
  video_title=<optional>
)
```

> **A TIMEOUT here is normal — do NOT retry the call.** `create_video`
> kicks off a long-running render. If the MCP call times out, the task
> has almost certainly already been submitted server-side under the
> same `request_id`. Retrying may double-charge credits or be rejected
> as a duplicate. Go straight to Step 8 and poll.

### 8. Poll

Preferred (single call, server polls internally):

```
pptalker_get_video_status(request_id, wait=true, timeout_seconds=300)
```

**Fallback when Step 7 timed out, or `wait=true` itself times out:**
loop with `wait=false` every **30 seconds** until status is terminal
(`done` / `failed` / `not_found`). Always reuse the same `request_id`
— never re-create the task.

```
loop:
    resp = pptalker_get_video_status(request_id, wait=false)
    if resp.status in ("done", "failed", "not_found"): break
    sleep 30s
```

Suggested cap: ~30 minutes. If it stays `processing` beyond 2× the
deck's spoken minutes, report status and let the user decide whether
to keep waiting.

When done, surface **only `share_url`** to the user — the canonical
PPTalker link
(`https://www.pptalker.com/video/public/{user_id}/{request_id}`,
permanent, opens the in-browser player).

**Never show `video_url`.** It is an internal 2-hour temporary COS
direct link kept for debugging only.

## Notes Display Format

When showing notes for confirmation, use exactly this layout so the
user can reply with precise edits:

```
=== Speaker notes ({N} slides) ===
[1] <slide 1 narration>
[2] <slide 2 narration>
...

Reply options:
- "OK" / "确认"          → proceed to render
- "全部重写"              → re-run Step 5 (ask preferences again, regenerate)
- "改第 3 页：..."        → replace slide 3 only
- "第 5、7 页重写"        → regenerate those slides only via the Step 5 prompt
```

Apply edits in-place to `final_notes`, then re-confirm before Step 7.

## References

Load these only when needed:

- **`references/note-generation-prompt.md`** — full prompt template +
  placeholder list + post-processing checks. Read this in Step 5.3.
- **`references/error-handling.md`** — Hard Rules, error codes, timeout
  policy, plan limits. Read this when an error fires or before
  estimating credit usage.
- **`references/examples.md`** — two complete walkthroughs (PPTX with
  notes, HTML auto-generate) and per-file-type extraction recipes.
  Read this if uncertain about sequencing or extraction.
