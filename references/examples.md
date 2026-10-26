# End-to-end Examples

Two complete walkthroughs. Read this when unsure about call order or how to branch after
the prepared result comes back.

## Example 1: `.pptx` with native speaker notes

User: "Convert /tmp/pitch.pptx to a video with English narration."

```
1. pptalker_account()
     → plan Free, 28 min left, voices include "Matthew" (en-US)

2. pptalker_render(file_path="/tmp/pitch.pptx", wait=false, notes_confirmed=false)
     → status=needs_confirmation, pages=12,
       canonicalfile="pitch.pdf", native_notes=[...12 entries...]

3. Show all 12 entries numbered, then ask "确认用这份讲稿生成吗？"
     → user replies "改第 3 页：…，其余 OK"
     → apply the edit, show the revised narration again, ask again
     → user replies "可以"

4. pptalker_render(request_id="agent-…", notes=[...12 revised...],
                   notes_confirmed=true, language="en-US", voice="Matthew")
     → status=submitted

5. pptalker_status(request_id="agent-…")      (repeat every 10–20s)
     → status=processing → … → status=done, duration=612, video_url=https://…
```

## Example 2: HTML deck, no native notes

User: "把这个 html 演示做成中文视频：/Users/me/deck/index.html"

```
1. pptalker_render(file_path="/Users/me/deck/index.html", notes_confirmed=false)
     → status=needs_notes, pages=8, canonicalfile="index.pdf", native_notes=["", …]

2. Draft 8 entries from the slide content (15–80 chars each, no invented numbers)
   Show them numbered, ask for approval
     → user says "第 5 页补充一下我们上季度的数据"
     → cannot invent data; ask the user for the exact figure
     → user provides "环比增长 18%"
     → show the updated narration again, get approval

3. pptalker_render(request_id="agent-…", notes=[...], notes_confirmed=true,
                   language="zh-CN", style="clean", speed=100)
     → status=submitted

4. pptalker_status(request_id="agent-…") → status=done → deliver video_url
```

## Example 3: recovering after a client crash

User: "刚才那个视频做好了吗？"

```
1. pptalker_account()
     → last_tasks=[{request_id:"agent-…", status:"processing", submission:"accepted"}]

2. pptalker_status(request_id="agent-…")
     → status=done, video_url=https://…

3. Deliver the link. Do NOT re-upload the source file.
```

## Example 4: user hands over narration up front

User: "用这份讲稿做视频，第一页讲市场背景…第二页讲产品…第三页讲财务…"

```
1. pptalker_render(file_path="/abs/deck.pptx", notes_confirmed=false)
     → status=needs_notes, pages=3

2. Use the user's own wording verbatim as notes (3 entries).
   Do not polish or reorder.
   Show all 3 and ask: "确认用这份讲稿生成吗？"

3. pptalker_render(request_id="agent-…", notes=[user's 3 entries],
                   notes_confirmed=true)
```

If the user's list length ≠ page count, stop and say "你给了 N 条，稿子有 M 页" — never pad or trim.
