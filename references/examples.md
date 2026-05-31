# End-to-end Examples

Two complete walkthroughs. Read this file when uncertain about the
sequencing of tool calls or the branching at Step 4.

## Example 1: `.pptx` with native speaker notes

User: "Convert /tmp/pitch.pptx to a video with English narration."

```
1. pptalker_get_account()                                     → 28 min left
2. pptalker_upload_ppt(file_path="/tmp/pitch.pptx")           → req=abc
3. pptalker_parse_ppt(req=abc, file_name="pitch.pptx")
     → has_notes=true, ppt_notes=[...], total_pages=12
4. Show notes via Notes Display Format
   → user replies "改第 3 页：…，其余 OK"
   → apply edit to ppt_notes → final_notes locked
5. pptalker_list_voices(language="en")                        → "Matthew"
6. pptalker_create_video(req=abc, file_name="pitch.pptx",
       notes=final_notes, language="en-US", voice="Matthew")  → submitted
7. pptalker_get_video_status(req=abc, wait=true)              → done, share_url=...
```

Final reply to the user:

> Your video is ready (4:12, 12 slides, 4 credits). Watch / share:
> {share_url}

## Example 2: `.html` deck (no notes, local generation)

User: "Make a video from /tmp/deck.html"

```
1. pptalker_get_account()                                     → 28 min left
2. pptalker_upload_ppt(file_path="/tmp/deck.html")            → req=def
3. pptalker_parse_ppt(req=def, file_name="deck.html")
     → file_name="deck.pdf", total_pages=8, ppt_img_urls=[...]

4. Ask the user (do NOT auto-use embedded notes):
     "8-slide HTML deck. Where should I get the narration from?
       1) You provide the script (paste here, or give a file path)
       2) Auto-generate locally from slide content"
   → user picks (2)

5. Ask preferences (Step 5.1):
     "Language? Words per slide? Tone? Extra context?"
   → user: "中文，每页约 60 字，专业风格，面向开发者"

   Extract slide source text (Step 5.2):
     - read /tmp/deck.html directly
     - split by .slide / <section> elements → slide_texts (length 8)

   Run note generation prompt (Step 5.3):
     - render the prompt in references/note-generation-prompt.md with
       {language}=zh-CN, {words_per_slide}=60, {tone}=professional,
       {background_info}="面向开发者", {total_pages}=8,
       {slide_texts}=[...]
     - LLM returns JSON → parse → final_notes (length 8)
     - run post-processing checks

   Show via Notes Display Format → user confirms → final_notes locked

6. (optional) pptalker_list_voices                             → choose voice
7. pptalker_create_video(req=def, file_name="deck.pdf",
       notes=final_notes, language="zh-CN")                   → submitted
   ↳ if this call returns TIMEOUT, do NOT retry — go to step 8.
8. pptalker_get_video_status(req=def, wait=true)              → done, share_url=...
   (or 30s polling loop with wait=false as fallback)
```

## Source-text extraction by file type

| Source            | Recommended extraction                                                      |
|-------------------|------------------------------------------------------------------------------|
| `.pptx` / `.ppt`  | `python-pptx` per-slide shape text; or fall back to `ppt_img_urls` (visual)  |
| `.pdf`            | The `pdf` skill (page-level text); or pdftotext / PyMuPDF                    |
| `.html` / `.htm`  | Read the file directly; split by `<section>` / `.slide` / page-break markers |
| Last resort       | Inspect `ppt_img_urls` from `pptalker_parse_ppt` and OCR / vision-describe   |

Always produce an array of length exactly `total_pages`, in slide order.
Empty strings are acceptable for visual-only slides.
