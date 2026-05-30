# Output, flags, and parsing — X

How to shape and consume `social x` output.

## Output modes

Every command supports the same trio:

- **No flag** — pretty `console.log` intended for a human in a terminal. Do not parse it programmatically.
- **`--json`** — raw single-line JSON. Pipe to `jq` or save to disk.
- **`--pretty`** — pretty-printed JSON (2-space indent). Still valid JSON.

Pick `--json` for any downstream analysis. `--pretty` is better only when the user reads the result inline and the payload is small.

## Limits

X uses `--max-results` (LinkedIn uses `--limit` — don't mix them):

| Command | Flag | Range |
|---|---|---|
| `search recent` | `--max-results` | 10–100 |
| `users tweets` | `--max-results` | 5–100 |
| `bookmarks list`, `timelines home`, `tweets list` | `--max-results` | 1–100 |
| `social usage` | `--limit` | 1–100 (default 50) |

Always prefer one large-`max-results` call over many small ones — every call is metered.

## Pagination

X uses **`--pagination-token`** for most commands; `search recent` accepts either `--next-token` or `--pagination-token`. The token comes back as `.meta.next_token`. Stop when it is missing or empty.

```bash
NEXT=$(echo "$PAGE1" | jq -r '.meta.next_token // empty')
[ -n "$NEXT" ] && social x bookmarks list "$ID" --max-results 100 --pagination-token "$NEXT" --json
```

Always cap loops with a page bound — bookmarks and timelines can run thousands deep.

## Time windows

`search recent`, `timelines home`, and `users tweets` support time bounds:

- `--start-time YYYY-MM-DDTHH:mm:ssZ` (UTC, seconds, inclusive)
- `--end-time YYYY-MM-DDTHH:mm:ssZ` (UTC, seconds, exclusive)
- `--since-id`, `--until-id` — ID-based bounds. Take precedence over time bounds when both are set.

Build a 30-day window:

```bash
SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")
social x users tweets "$ID" --start-time "$SINCE" --max-results 100 --json
```

## Sort order

`search recent` accepts `--sort-order recency|relevancy`. Default is recency.

## Account selection

Every command accepts `--account <handle-or-id>`. Resolves against `social x list`. Default account is used when omitted.

## Cache

`--no-cache` forces a fresh upstream fetch. Default already serves fresh-enough data. Reach for it only when:

- Verifying a freshly-published tweet is visible.
- Debugging a stale-data report.
- Comparing pre- and post-action state.

## jq recipes

Run against `--json` output:

```bash
# Top tweets by like count.
jq '.data | sort_by(-.public_metrics.like_count) | .[0:10] | .[] | {id, text, likes: .public_metrics.like_count}'

# Join author info from .includes.users.
jq '
  (.includes.users // []) as $users
  | .data[]
  | . as $t
  | ($users[] | select(.id == $t.author_id)) as $u
  | { text: $t.text, author: $u.username, likes: $t.public_metrics.like_count }
'

# Drop verbose fields for an LLM-friendly summary.
jq '.data[] | {id, text, created_at, metrics: .public_metrics}'

# Aggregate usage by platform (inspect schema first).
social usage --summary --json | jq
```

When chaining over a saved file, write once:

```bash
social x search recent "ai safety" --max-results 100 --json > /tmp/search.json
jq '.data | length' /tmp/search.json
jq '.meta.next_token // empty' /tmp/search.json
```

This avoids re-billing the same query.

## Field/expansion combos

The default response is sparse. Combine flags to enrich:

```bash
# Engagement-rich tweet list with author info.
social x users tweets "$ID" \
  --max-results 100 \
  --tweet-fields public_metrics,created_at,conversation_id \
  --expansions author_id \
  --user-fields username,name,verified \
  --json
```

## Error catalog

Errors arrive on stderr with a non-zero exit. In `--json` mode they print as `{ "status": "error", "code": "...", "message": "..." }`. Common codes:

| Code | Meaning | Fix |
|---|---|---|
| `unauthenticated` | No bearer or expired. | `social login`. |
| `scope_missing` | Token lacks `x:write`. | `social logout && social login --scope read,write`. |
| `platform_not_connected` | No connected X account. | `social x connect`. |
| `account_not_found` | `--account` value did not match. | `social x list`, reuse the printed handle/id. |
| `endpoint_not_available_in_v1` | Path not in the adapter's allowlist. | Pick a different command; do not retry. |
| `rate_limited` | X throttle hit. | Back off per the retry hint. X's quotas are tight on free tiers. |
| `invalid_argument` | A flag failed parsing/validation. | Check `--help`; ranges in this doc are authoritative. |
| `no_available_seat` | Org has no remaining billing seat at `connect`. | User adds a seat in the dashboard or releases one with `disconnect`. |
| `x_connect_timed_out` | User did not approve in browser within 5 minutes. | Re-run `social x connect`. |
| `x_account_required` | `disconnect` with multiple accounts and no arg. | Add the handle/id. |

Surface error messages verbatim — they are precise enough for the user to act on.

## `social schema`

The authoritative machine-readable command tree:

```bash
social schema --json | jq '.subCommands.x.subCommands | keys'
social schema --json | jq '.subCommands.x.subCommands.tweets.subCommands'
```

Use this when uncertain about a flag, a sub-tree, or whether something exists at all. Faster and cheaper than guessing.
