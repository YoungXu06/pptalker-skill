# Error Handling, Limits, and Hard Rules

## Hard Rules (must never break)

1. Start with `pptalker_account` and route on `accepts_local_file_path`. Never send a local path to
   the hosted server and never ask for a browser link from the local package.
2. Never call `pptalker_render` with `notes_confirmed=true` until the user explicitly approves the
   narration text.
3. `notes.length` must equal `pages`, and at least one entry must be non-empty.
4. Reuse the same `request_id` for parsing, confirming and rendering. Never upload twice for one
   video — that creates a second chargeable task.
5. Match `language` to the language of the narration and the chosen voice. When the narration is
   CJK, the renderer picks a CJK font automatically, so a mismatched `language` will not break
   subtitles — but it still affects the voice.
6. In browser mode, review can take longer than the upload link. A completed upload is not tied to
   the link TTL, so there is no need to re-upload while editing narration.

## Error responses

Every error carries `error`, `retryable`, and `action`. Follow `action`; do not guess.

| Action | Meaning | What the agent should do |
|---|---|---|
| `wait_for_upload` | Hosted link is open but no file yet | Keep polling status; do not call render early |
| `finish_upload` | The browser upload has not completed | Ask the user to open the link and upload; issue a new link if it expired |
| `restart_upload_with_new_id` | Local read/PUT failed or slot is incomplete | Start a new local task from `file_path`; do not reuse a broken id |
| `submit_prepare` | Local PUT completed, prepare not submitted | Call render with the same id |
| `check_request_id` | The id is unknown for this account | Reuse the id from `pptalker_prepare_upload`; do not invent one |
| `add_credits` | Not enough minutes | Tell the user the balance; do not retry the same call |
| `reduce_input` | Over page or character limit | Ask the user to split the deck or shorten narration |
| `re_export_file` | File cannot be parsed | Ask for a `.pptx` or a PDF export; `.ppt` is unsupported |
| `fix_notes` | Narration count ≠ page count | Show the page count, ask the user to add or remove entries |
| `choose_voice` | Voice id not in the catalogue | Call `pptalker_account`, list voices, let the user pick |
| `confirm_notes` | Narration not approved yet | Show the narration and get explicit approval |
| `retry_later` | Transient backend issue | Retry the same call; nothing was charged |

## Upload and auth errors

| Error | Meaning | What to do |
|---|---|---|
| `UPLOAD_REQUIRED` | Browser mode: the file was never uploaded | Open `upload_url`, sign in, choose the file |
| `UPLOAD_MISSING` | Browser mode: the object is gone or empty | Issue a new link; do not reuse the old one |
| `UPLOAD_TOO_LARGE` | Over the 50 MiB limit | Ask for a smaller file or a PDF export |
| `LOCAL_UPLOAD_FAILED` | Local mode: read or PUT failed | Follow `restart_upload_with_new_id`; start from the same local file |
| `INSUFFICIENT_SCOPE` | The connection lacks a scope | The client re-opens consent; ask the user to approve |
| `INVALID_TOKEN` | Expired or revoked connection | The client refreshes automatically; if it persists, re-login |

A `403` with `insufficient_scope` is **not** a render failure. The client will offer to
re-authorize; let it.

In local mode, `restart_upload_with_new_id` and `check_status` come from the local
package, not from the browser upload service. Do not translate them into a browser
link unless the user explicitly agrees to switch transports.

## `preparing` is not an error

Parsing runs in the background. `status=preparing` means "still working", and large decks can take
several minutes. Keep polling the same `request_id`:

```
pptalker_status(request_id="agent-…", wait=true, timeout_seconds=30)   # repeat
```

Do not re-upload, do not create a second task, and do not tell the user it failed.

## Limits

- ≤80 slides per video
- ≤100,000 narration characters total
- ≤20,000 characters per slide
- ≤50 MiB per upload
- Upload link: 15 minutes (the completed task itself does not expire)
- In-call polling: ≤30 seconds per call
