# End-to-end Examples

Two complete walkthroughs against the **hosted** MCP server. Read this when unsure about call order
or how to branch after parsing finishes.

The server never receives a local file path. The agent creates a task, the user uploads the file in
a browser, and only then can rendering start.

## Example 1: PDF with an existing script

User: "Turn output.pdf and its script into a video."

```
1. pptalker_account()
     → plan Max, 162 min left

2. pptalker_prepare_upload(file_name="output.pdf")
     → request_id="agent-4f2a…", upload_url="https://www.pptalker.com/agent/upload#token=…"

3. Tell the user: open upload_url, sign in, choose the file.
     → user confirms the page says "Upload complete"

4. pptalker_render(request_id="agent-4f2a…")         # no notes yet
     → status=preparing, action=check_status

5. pptalker_status(request_id="agent-4f2a…", wait=true, timeout_seconds=30)
     → status=preparing
     → poll again …
     → status=needs_notes, pages=9

6. The user supplied a script → split it per slide, use it verbatim.
   Print all 9 entries numbered, ask "确认用这份讲稿生成吗？"
     → user replies "可以"

7. pptalker_render(request_id="agent-4f2a…", notes=[…9 entries…],
                   notes_confirmed=true, language="zh-CN")
     → status=submitted

8. pptalker_status(request_id="agent-4f2a…", wait=true, timeout_seconds=30)
     → status=processing … → status=done, duration=520,
       video_url="https://www.pptalker.com/video/public/<user>/agent-4f2a…"
```

`video_url` is the canonical page, not a storage link. Give it to the user as-is.

## Example 2: deck with native speaker notes

```
1. pptalker_prepare_upload(file_name="pitch.pptx")
     → request_id + upload_url; user uploads in the browser

2. pptalker_render(request_id="agent-…")
     → status=preparing

3. pptalker_status(request_id="agent-…", wait=true)
     → status=needs_confirmation, pages=12, native_notes=[…12 entries…]

4. Show all 12 entries, ask for approval
     → user replies "改第 3 页：…，其余 OK"
     → apply the edit, show the revised narration, ask again
     → user replies "可以"

5. pptalker_render(request_id="agent-…", notes=[…12 revised…], notes_confirmed=true)
     → status=submitted
```

## What NOT to do

- Do not ask the user for the one-time token from `upload_url`; it stays in the browser.
- Do not call `pptalker_render` with notes before the upload completes — it returns
  `UPLOAD_REQUIRED` and wastes a round trip.
- Do not start over when parsing is slow. `preparing` is normal for large decks; keep polling the
  same `request_id`.
- Do not re-upload because a status call looked odd. Poll `pptalker_status` first.
