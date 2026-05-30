# X command catalog

`social x <subtree> <command>`. All commands return JSON over `--json`, pretty-printed JSON over `--pretty`, and pass through `--account <handle>` plus `--no-cache`.

X diverges from LinkedIn: list endpoints use `--max-results` (not `--limit`), pagination uses `--pagination-token` or `--next-token` (not `--cursor`), and many list commands need your own X user ID as a positional. Resolve once:

```bash
MY_X_ID=$(social x users me --json | jq -r '.data.id')
```

## Account lifecycle

| Command | Purpose |
|---|---|
| `social x connect [--no-open]` | OAuth handshake. Opens the web app at `/connect/x`. |
| `social x reconnect <handle-or-id> [--no-open]` | Re-auth after token revoke. |
| `social x disconnect [handle-or-id]` | Disconnect. Omit id when only one is connected. |
| `social x list [--include-disconnected]` | List connected accounts. |

## `users`

| Command | Args | Notes |
|---|---|---|
| `users me` | `--user-fields`, `--tweet-fields`, `--expansions` | Authenticated profile. Returns `.data.id` — capture and reuse. |
| `users tweets <id>` | `--max-results 5-100`, `--pagination-token`, `--since-id`, `--until-id`, `--start-time`, `--end-time`, `--exclude replies\|retweets`, `--tweet-fields`, `--expansions`, `--media-fields`, `--poll-fields`, `--user-fields`, `--place-fields` | List a user's tweets. `<id>` is the numeric X user ID, not the handle. Resolve via `users me` or a search hit. |

## `tweets`

| Command | Args | Notes |
|---|---|---|
| `tweets get <id>` | `--tweet-fields`, `--expansions`, `--media-fields`, `--poll-fields`, `--user-fields`, `--place-fields` | Single tweet by numeric ID. |
| `tweets list <id> [<id>...]` | Same field/expansion flags | Up to 100 IDs. Space-separated positional. |

## `timelines`

| Command | Args | Notes |
|---|---|---|
| `timelines home <id>` | `--max-results 1-100`, `--pagination-token`, `--since-id`, `--until-id`, `--start-time`, `--end-time`, `--exclude replies\|retweets`, `--tweet-fields`, `--expansions`, `--user-fields`, `--media-fields`, `--poll-fields`, `--place-fields` | Reverse-chronological home timeline. `<id>` is the authenticated X user ID. |

## `bookmarks`

| Command | Args | Notes |
|---|---|---|
| `bookmarks list <id>` | `--max-results 1-100`, `--pagination-token`, `--tweet-fields`, `--expansions`, `--user-fields`, `--media-fields`, `--poll-fields`, `--place-fields` | The user's saved bookmarks. Use your own X user ID. |

## `search`

| Command | Args | Notes |
|---|---|---|
| `search recent <query>` | `--max-results 10-100`, `--next-token` \| `--pagination-token`, `--start-time` (ISO 8601 Z), `--end-time` (ISO 8601 Z), `--since-id`, `--until-id`, `--sort-order recency\|relevancy`, `--tweet-fields`, `--expansions`, `--user-fields`, `--media-fields`, `--poll-fields`, `--place-fields` | 7-day recent search. `<query>` follows X's [query syntax](https://docs.x.com/x-api/posts/search/integrate/build-a-query) — quote it. |

`--start-time` / `--end-time` use second granularity. `--start-time` is inclusive; `--end-time` is exclusive. ID-based windows (`--since-id` / `--until-id`) take precedence over time-based ones when both are set.

## Example invocations

```bash
# Smoke test.
social x users me --pretty

# My ID — capture once.
ID=$(social x users me --json | jq -r '.data.id')

# Last 50 bookmarks.
social x bookmarks list "$ID" --max-results 50 --json | jq '.data[].text'

# Home timeline, excluding replies.
social x timelines home "$ID" --max-results 25 --exclude replies --json

# A specific user's recent tweets (resolve their ID via search or a known list).
social x users tweets 44196397 --max-results 30 --exclude retweets --pretty

# Recent search.
social x search recent "from:elonmusk" --max-results 100 --sort-order recency --json

# Fetch by ID — multiple.
social x tweets list 1843123456789012345 1843234567890123456 --pretty

# Paginate.
PAGE1=$(social x bookmarks list "$ID" --max-results 100 --json)
NEXT=$(echo "$PAGE1" | jq -r '.meta.next_token // empty')
[ -n "$NEXT" ] && social x bookmarks list "$ID" --max-results 100 --pagination-token "$NEXT" --json
```

## Output shape (typical)

X endpoints follow the X v2 envelope:

```json
{
  "data": [...],
  "includes": { "users": [...], "media": [...], "tweets": [...] },
  "meta": { "next_token": "...", "result_count": 25 }
}
```

`tweets get` returns the single tweet at `.data`. `users me` returns the profile at `.data`. Joins (author, media) live in `.includes` — populate them with `--expansions author_id,attachments.media_keys`.

## Useful field/expansion presets

The default response is sparse:

- **Engagement metrics**: `--tweet-fields public_metrics,created_at`
- **Author info on a tweet**: `--expansions author_id --user-fields username,name,verified,public_metrics`
- **Media + alt text**: `--expansions attachments.media_keys --media-fields url,preview_image_url,alt_text`
- **Conversation thread**: `--tweet-fields conversation_id,in_reply_to_user_id,referenced_tweets`
