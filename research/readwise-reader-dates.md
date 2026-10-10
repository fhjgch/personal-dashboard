# What start and finish dates can the Readwise Reader API provide for books?

Research for [#2](https://github.com/fhjgch/personal-dashboard/issues/2) (part of map #1, "Life in Weeks prototype with Readings").
Sources checked 2026-10-10. Only primary sources were used:

- [R] Readwise Reader API docs: https://readwise.io/reader_api
- [W] Readwise (highlights) API docs: https://readwise.io/api_deets

## Short answer

- The Reader API returns one **snapshot** per document. It has **no event history**, so you cannot see reading sessions or past location changes [R].
- **Start candidates:** `first_opened_at` (best), then `saved_at` / `created_at` as a fallback ("added to library", not "started") [R].
- **Finish candidates:** `last_moved_at` when `location == "archive"` (the move into the archive), or `last_opened_at`. `reading_progress` tells you *whether* a book was finished (close to 1.0), not *when* [R].
- None of these fields is documented semantically. The docs give no definition of `first_opened_at`, `last_opened_at`, `last_moved_at`, `saved_at`, `reading_progress` or `updated_at`. The open timestamps can also be overwritten: the `seen` flag in UPDATE populates or clears them [R]. Treat every Reader-derived date as a heuristic, and keep the Logseq journal as the source of truth where the two disagree.
- **Books = `category == "epub"`**, filtered server-side with `GET /api/v3/list/?category=epub` [R].

## Endpoint

`GET https://readwise.io/api/v3/list/` [R]

| Parameter | Use for us |
|---|---|
| `category=epub` | Only books. Valid values: `article`, `email`, `rss`, `highlight`, `note`, `pdf`, `epub`, `tweet`, `video`. |
| `location` | `new`, `later`, `shortlist`, `archive`, `feed`. Use `archive` for finished candidates, or omit it and filter client-side. |
| `updatedAfter` | ISO 8601. Incremental sync. |
| `id` | Fetch a single document. |
| `limit` | 1 to 100, default 100. |
| `pageCursor` | Pagination (see below). |
| `tag` | Up to 5 values (AND). `?tag=` finds untagged documents. |

The response is `{count, nextPageCursor, results}`. Keep passing `nextPageCursor` as `pageCursor` until it is absent [R].

## Date-related fields per document [R]

All timestamps are ISO 8601 UTC, for example `"2022-12-08T02:53:29.639650+00:00"`.

| Field | What it plausibly means | Start/finish value | Reliability |
|---|---|---|---|
| `saved_at` | When the doc was saved to Reader | Weak start fallback | Says when the book was added, not when reading began. EPUBs are often uploaded long before reading. |
| `created_at` | When the Reader record was created | Weak start fallback | Usually about the same as `saved_at`. Not defined in the docs. |
| `first_opened_at` | First time the doc was opened | **Best start signal** | Undefined in the docs. Can be set or cleared through UPDATE `seen` ("Setting `true` will populate `first_opened_at` / `last_opened_at`; setting `false` will clear them."). Null if never opened. |
| `last_opened_at` | Most recent open | Finish fallback | Moves forward if you reopen a finished book later, for example to look something up. Only an upper bound on the finish date. |
| `last_moved_at` | Last change of `location` | **Finish signal when `location == "archive"`** | Only the *last* move is kept. If the book is moved again later, or re-archived, the finish date is lost or shifted. Undefined in the docs. |
| `updated_at` | Any change to the doc (tags, notes, progress...) | Not usable | Changes for unrelated edits. Only useful for incremental sync via `updatedAfter`. |
| `reading_progress` | Fraction read, for example `0`, `0.15`, `0.5` in the examples | Finished? yes/no (for example ≥ 0.95) | The docs give no range or definition. It holds no time information. |
| `location` | `new`/`later`/`shortlist`/`archive`/`feed` | `archive` ≈ "done" | The archive can also mean "abandoned". Combine with `reading_progress`. |
| `published_date` | The book's publication date | None | Not a reading date. |

Other useful fields: `id`, `title`, `author`, `source`, `category`, `tags`, `word_count`, `reading_time`, `url`, `source_url`, `parent_id` [R].

### Highlights as an extra signal

Highlights and notes made in Reader are documents too, with `parent_id` set to the book's id [R]. So you can derive per-book activity bounds from the highlight documents' own `created_at`: earliest highlight ≈ reading under way, latest highlight ≈ near finish. The classic Readwise API also exposes `highlighted_at` per highlight and `last_highlight_at` per book via `GET /api/v2/export/` and `/api/v2/books/`. Reader books have `source == "reader"` there, and `external_id` "is currently available only when `source` is `reader`" [W]. This is only useful for books where you actually highlight.

## Suggested derivation for a Reading (heuristic, to validate on real data)

- **Book:** `category == "epub"`. Skip documents with `parent_id` set (highlights and notes).
- **Start:** `first_opened_at`, else the earliest child-highlight `created_at`, else `saved_at`.
- **Finished?** `location == "archive"` and `reading_progress` ≥ about 0.9 (threshold to tune).
- **Finish:** `last_moved_at` (when archived), else `last_opened_at`, else the latest child-highlight `created_at`.
- **In progress:** `first_opened_at` set, not archived. Draw it from start to "now".
- Flag cases where finish < start, or a gap that is implausibly short or long, for manual override from Logseq.

## Auth

- Header: `Authorization: Token <token>`. Get the token at https://readwise.io/access_token [R][W].
- Check a token with `GET https://readwise.io/api/v2/auth/`. A `204` means the token is valid [R][W].

## Rate limits [R]

- LIST (`/api/v3/list/`): **20 requests/minute per token**. At 100 docs per page that is plenty for a personal library.
- SAVE and UPDATE: 50/min. Bulk update, delete and tags: 20/min.
- On `429 Too Many Requests`, wait for the number of seconds in `Retry-After`.
- Classic Readwise API [W]: default 240/min. Highlight LIST and Book LIST: 20/min.

## Open questions (not answerable from the docs)

- The exact triggers for `first_opened_at`, `last_moved_at` and `reading_progress`, for example whether `reading_progress` is per-scroll or per-page for EPUBs. Validate against a few known books in the real library.
- Whether `last_moved_at` is set for documents saved directly into `archive`.
- Whether deleted documents ever appear in LIST. The docs do not say.
- Webhooks exist for document and highlight changes, but they are documented elsewhere and are not needed for a local prototype [R].
