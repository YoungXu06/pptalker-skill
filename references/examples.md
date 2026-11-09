# End-to-end Examples

Two complete walkthroughs, one per transport. Read this when unsure about call
order or how to branch after parsing finishes.

Call `pptalker_account()` first. `accepts_local_file_path` decides the mode:
`true` means the local package can read the file; `false` means the hosted server
needs a browser upload.

## Example 1: local/automatic mode

User: "Turn /Users/me/talk.pdf and its script into a Chinese narrated video."

```
1. pptalker_account()
     → plan Max, 162 min left
     → server_mode=local_mcp, upload_mode=direct_from_local_path
     → accepts_local_file_path=true, defaults={voice:"XY", speed:110, ...}

2. pptalker_render(file_path="/Users/me/talk.pdf",
                   notes_confirmed=false, wait=false)
     → request_id="agent-4f2a…", status=preparing
     # The local process uploaded the bytes; they did not enter the conversation.

3. pptalker_status(request_id="agent-4f2a…", wait=true, timeout_seconds=45)
     → status=preparing → poll again
     → status=needs_notes, pages=9

4. The user supplied a script → split it per slide, use it verbatim.
   Print all 9 entries numbered, ask "确认用这份讲稿生成吗？"
     → user replies "可以"

5. pptalker_render(request_id="agent-4f2a…",
                   notes=[…9 entries…], notes_confirmed=true)
     → status=submitted
     # Do not pass voice/speed unless the user asked to override the defaults.

6. pptalker_status(request_id="agent-4f2a…", wait=true, timeout_seconds=45)
     → status=rendering … → status=done, duration=520
     → video_url="https://www.pptalker.com/video/..."
```

Deliver the canonical page URL, not a storage URL.

## Example 2: hosted/browser mode

```
1. pptalker_account()
     → accepts_local_file_path=false, server_mode=hosted_browser_upload

2. pptalker_prepare_upload(file_name="pitch.pptx")
     → request_id="agent-8c1d…"
     → upload_url="https://www.pptalker.com/agent/upload#token=…"

3. Tell the user: open upload_url, sign in, and choose pitch.pptx.
   Do not ask for the token back.

4. pptalker_status(request_id="agent-8c1d…", wait=true, timeout_seconds=30)
     → status=awaiting_upload → keep polling
     → status=uploaded / preparing (upload landed)

5. pptalker_render(request_id="agent-8c1d…")     # no notes yet
     → status=preparing

6. pptalker_status(request_id="agent-8c1d…", wait=true)
     → status=needs_confirmation, pages=12, native_notes=[…12 entries…]

7. Show all 12 entries, ask for approval
     → user replies "改第 3 页：…，其余 OK"
     → apply the edit, show the revised narration, ask again
     → user replies "可以"

8. pptalker_render(request_id="agent-8c1d…",
                   notes=[…12 revised…], notes_confirmed=true)
     → status=submitted

9. pptalker_status(request_id="agent-8c1d…", wait=true)
     → status=done → deliver video_url
```

## Example 3: recovering after a client crash

```
1. pptalker_account()
     → last_tasks (local mode) or the saved hosted id

2. pptalker_status(request_id="agent-…")
     → status=done or rendering

3. Deliver or keep polling. Do NOT re-upload the source file.
```

## What NOT to do

- Do not use `pptalker_prepare_upload` when `accepts_local_file_path=true`.
- Do not pass `file_path` to the hosted server; it cannot read the disk.
- Do not ask the user for the one-time token from `upload_url`.
- Do not call render with notes before the hosted upload completes.
- Do not start over when parsing is slow; `preparing` is normal.
- Do not re-upload because a status call looked odd; poll first.
- Do not switch transport mid-task without telling the user that it creates a new
  task and may charge again.
