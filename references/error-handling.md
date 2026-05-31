# Error Handling, Limits, and Hard Rules

## Hard Rules (must never break)

1. Never call `pptalker_create_video` until the user explicitly confirms
   `final_notes`.
2. `final_notes.length` must equal `total_pages` and at least one entry
   must be non-empty.
3. After Step 3, always use the canonical `request_id` and `file_name`
   returned by `pptalker_parse_ppt` (HTML may have changed `file_name`
   to `.pdf`).
4. Match `language` to the language of `final_notes` and the chosen voice.
5. For `.pdf` / `.html` / `.htm` inputs, always ask the user to choose the
   narration source (user-provided vs. auto-generate). Never silently use
   `notes_source="html_aside"` or any other embedded notes for these
   formats.
6. Never call `pptalker_generate_notes`. Generate notes locally per
   Step 5 (see `note-generation-prompt.md`).
7. Before local note generation, always ask the user for narration
   preferences (language / words-per-slide / tone / extra context).
   One round-trip with "default" is fine — but never skip the question.
8. Retry any failing MCP tool **at most 3 times total**. After 3 failed
   attempts, stop, surface the last error, and ask the user how to proceed.
   Exception: a `pptalker_create_video` TIMEOUT is not a failure (task is
   already submitted) — never re-call it; poll `pptalker_get_video_status`
   with the same `request_id` instead.

## Error Codes

| Code  | Meaning                | Action                                                                 |
|-------|------------------------|------------------------------------------------------------------------|
| 1003  | INSUFFICIENT_CREDITS   | Stop. Link the user to https://www.pptalker.com/pricing.               |
| 1004  | Pages/chars exceed plan| Reduce slides or upgrade plan.                                          |
| 1005  | Corrupted file         | Ask the user to re-export the deck.                                     |
| 1002  | Generic task fail      | Retry once; if persistent, simplify the deck or notes.                  |
| 1010  | Voice clone needs Max  | Switch to a standard voice and inform the user.                         |

## Timeouts

| Where                        | Meaning                                       | Action                                                                                                  |
|------------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `create_video` TIMEOUT       | Long-running render task; already submitted    | **Do NOT retry.** Go to Step 8 and poll `pptalker_get_video_status` every 30s with the same `request_id`. |
| `upload_ppt` TIMEOUT         | Slow network / large file                      | Retry once; ensure file < 50 MB.                                                                          |
| `get_video_status` TIMEOUT   | Server slow                                    | Retry; or fall back to the 30s polling loop.                                                              |

If `pptalker_create_video` returns an error about missing notes, re-run
Step 4 — notes were not actually confirmed.

## Cost & Plan Limits

| Resource         | Free  | Pro    | Max    |
|------------------|-------|--------|--------|
| Credits / month  | 5 min | 30 min | 120 min |
| Max slides       | 10    | 50     | 200    |
| Watermark        | Yes   | No     | No     |
| Voice cloning    | No    | No     | Yes    |

1 credit = 1 minute of output. Upload + parse + local note generation
are free.

A 10-slide deck typically renders to a 3–5 minute video → 3–5 credits.
Verify `left_credits >= estimated_minutes` in Step 1 before proceeding.
