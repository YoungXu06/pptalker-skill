# Error Handling, Limits, and Hard Rules

## Hard Rules (must never break)

1. Never call `pptalker_render` with `notes_confirmed=true` until the user explicitly approves the
   narration text.
2. `notes.length` must equal `pages`, and at least one entry must be non-empty.
3. Always reuse the canonical `file_name` / `request_id` returned by the prepared result.
   HTML uploads are converted, so the canonical name becomes `.pdf`.
4. Match `language` to the language of the narration and the chosen voice.
5. Never re-upload a file to work around an error that happened after upload — that creates a
   second chargeable task.

## Error responses

Every error carries `error`, `retryable`, and `action`. Follow `action`; do not guess.

| Action | Meaning | What the agent should do |
|---|---|---|
| `add_credits` | Not enough minutes | Tell the user the balance; do not retry the same call |
| `reduce_input` | Over page or character limit | Ask the user to split the deck or shorten narration |
| `re_export_file` | File cannot be parsed | Ask for a `.pptx` or PDF export; `.ppt` is unsupported |
| `fix_notes` | Narration count ≠ page count | Show the page count, ask the user to add or remove entries |
| `choose_voice` | Voice id not in the catalog | Call `pptalker_account`, list voices, let the user pick |
| `confirm_notes` | Narration not approved yet | Show the narration and get explicit approval |
| `retry_later` | Backend unavailable or rate limited | Wait a few minutes; check status meanwhile |
| `fix_input` | Invalid parameter or filename | Fix the argument; do not retry unchanged |
| `check_request_id` | Task not found for this account | Confirm the id came from this account's own task |
| `retry_once` | Render failed | Retry the same id once, after confirming with the user |
| `contact_support` | Unclassified failure | Report the id and time to the maintainer |

## Limits

| Limit | Value |
|---|---|
| File size | 50 MiB |
| Slides per deck | 80 |
| Narration total | 100,000 characters |
| One narration entry | 20,000 characters |
| Speed | 80–150 |
| Concurrent renders per account | 2 |

## Guard rails

- `retryable: true` → safe to try again, but check `action` first.
- `retryable: false` → retrying will not help; the input or account state must change.
- `unknown` status or a timeout → **query `pptalker_status` before doing anything else.**
- Do not treat `failed` as proof that nothing was charged. Check `credits_charged` and the account
  balance before telling the user nothing happened.
- A temporary `BACKEND_UNAVAILABLE` is not a bad file. Do not ask the user to re-export it.
- If the same error repeats twice, stop and report it instead of retrying a third time.
