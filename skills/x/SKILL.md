---
name: x
description: This skill should be used when the user wants to do anything on X (formerly Twitter) from inside an agent — search recent posts, fetch a tweet by ID or URL, list a user's tweets, read the home timeline, list saved bookmarks, look up the authenticated profile, connect/disconnect an X account, or check `social` CLI usage and billing. Triggers include phrases like "search X", "search Twitter", "look up tweet", "my X bookmarks", "my Twitter timeline", "from:<handle>", "connect my X account", and explicit invocation via `/social:x`. Wraps the `social x` subtree of the `social` binary (npm `@usesocial/cli`).
argument-hint: [task — e.g. "search Twitter for 'from:elonmusk'", "list my bookmarks", "show my home timeline"]
---

# social x

Use the `social x` subtree to read (and where supported, write) X data on the user's behalf. Authenticates with the user's connected X token and routes through `/v1/x/*` on the API. Treat it as the only correct way to fetch X data in this skill — never call X's HTTP APIs directly.

## When this skill applies

Match any of:

- Users — `users me`, `users tweets`.
- Tweets — `tweets get`, `tweets list`.
- Timelines — `timelines home`.
- Bookmarks — `bookmarks list`.
- Search — `search recent`.
- Account lifecycle — `connect`, `reconnect`, `disconnect`, `list`.
- Usage / billing — `social usage` (recent calls or `--summary`).

If the user says "Twitter", route to X. If the user asks for something X exposes but the catalog below does not list, run `social x --help` or `social schema --json` — do not guess endpoints.

## First-use setup

Before the first X call in a session, confirm the CLI is installed, the user is signed in, and an X account is connected. Use a single combined probe so an authenticated user pays no extra round-trip:

```bash
social x users me --json 2>&1 | head -c 400
```

Interpret the output:

- **Exit 0 with `{"data": {"id": ..., "username": ...}}`** → installed, signed in, X connected. Capture `.data.id` — most other X commands need it as a positional. Proceed.
- **`command not found: social`** → install: `bun install -g @usesocial/cli@latest` (fall back to `npm install -g @usesocial/cli`). Re-probe.
- **`unauthenticated`, `401`, `Not signed in`** → run `social login`. This is an interactive device flow that opens `${SOCIAL_WEB_URL}/device`; it cannot complete headlessly inside the agent. Surface the verification URL/code to the user and wait for them to approve. Use `--no-open` to print the URL inline.
- **`platform_not_connected`** → run `social x connect` (or `--no-open` to print the URL). The user approves the OAuth handshake in their browser.

Do **not** background `social login` or `social x connect` — both wait on a foreground poll loop.

For full setup, scope, and troubleshooting details, load `references/setup.md`.

## Invocation pattern

X's API diverges from LinkedIn's. Memorise the surface differences:

| LinkedIn (`social linkedin …`) | X (`social x …`) |
|---|---|
| `--limit` | `--max-results` |
| `--cursor` | `--pagination-token` (or `--next-token` for `search recent`) |
| 1–100 (or 1–1000 for connections) | varies — 1–100 for most, 5–100 for `users tweets`, 10–100 for `search recent` |

Other shared flags:

- `--json` / `--pretty` — pick `--json` for analysis; `--pretty` only for inline review of small payloads.
- `--account <handle>` — disambiguate when multiple X accounts are connected.
- `--no-cache` — bypass cache. Avoid unless verifying freshly-published content.
- `--help` — authoritative per-command flag list.

Most X list commands need **your X user ID as a positional** (bookmarks, home timeline, users tweets). Resolve once:

```bash
MY_X_ID=$(social x users me --json | jq -r '.data.id')
```

Reuse `$MY_X_ID` for the session.

For the complete flag reference and parsing patterns, load `references/output.md`.

## Command surface

| Subtree | Commands | Notes |
|---|---|---|
| `users` | `me`, `tweets <id>` | `me` returns `.data.id` etc.; cache the ID. `users tweets <id>` accepts `--max-results 5-100`, `--since-id`, `--until-id`, `--start-time`, `--end-time`, `--exclude replies,retweets`, plus all `*-fields` and `--expansions` flags. |
| `tweets` | `get <id>`, `list <id> [<id>...]` | `get` is one tweet; `list` is up to 100 IDs space-separated. Both accept `*-fields` / `--expansions`. |
| `timelines` | `home <id>` | Reverse-chronological home timeline. `<id>` is the authenticated user's X ID. Same time/cursor/field flags as `users tweets`. |
| `bookmarks` | `list <id>` | `<id>` is the authenticated user's X ID. `--max-results 1-100`, `--pagination-token`. |
| `search` | `recent <query>` | 7-day search. `<query>` follows X's [query syntax](https://docs.x.com/x-api/posts/search/integrate/build-a-query) — quote it. `--max-results 10-100`, `--next-token` or `--pagination-token`, `--start-time` / `--end-time` (ISO 8601 Z), `--since-id` / `--until-id`, `--sort-order recency\|relevancy`. |
| Lifecycle | `connect [--no-open]`, `reconnect <id> [--no-open]`, `disconnect [<id>]`, `list [--include-disconnected]` | `disconnect` omits the id when only one account is connected. |

Load `references/commands.md` for full arguments, flag ranges, field/expansion presets, and example output shapes.

## Output shape

X commands return the v2 envelope:

```json
{
  "data": [...],
  "includes": { "users": [...], "media": [...], "tweets": [...] },
  "meta": { "next_token": "...", "result_count": 25 }
}
```

`tweets get` returns the single tweet at `.data`. Joins (author, media) live in `.includes` — populate them with `--expansions author_id,attachments.media_keys` and the matching `--*-fields` flags.

The default response is sparse. Common presets:

- **Engagement metrics:** `--tweet-fields public_metrics,created_at`
- **Author info on a tweet:** `--expansions author_id --user-fields username,name,verified,public_metrics`
- **Media + alt text:** `--expansions attachments.media_keys --media-fields url,preview_image_url,alt_text`
- **Conversation thread:** `--tweet-fields conversation_id,in_reply_to_user_id,referenced_tweets`

## Choosing a command

1. Identify the **noun**: tweet, user, bookmark, timeline, search.
2. Identify the **action**: fetch one, list many, search.
3. Construct `social x <noun> <verb>` and add flags. Verify with `--help` if uncertain.
4. If the command needs a user ID positional, ensure `$MY_X_ID` is set (or pass a specific ID).

When the user gives a tweet URL like `https://x.com/handle/status/1843123456789012345`, extract the trailing numeric ID. When they give a handle, you usually still need the numeric ID — resolve via search or a known mapping.

## Output handling

- Capture `--json` output to a temp file when it might exceed a few thousand tokens: `social x search recent "query" --max-results 100 --json > /tmp/x.json` then `jq` over it.
- Project only the fields you need with `jq` — X payloads can be deep, especially with expansions.
- For user-facing summaries, build a short markdown table from `jq` output rather than dumping raw JSON.
- Surface errors verbatim — codes like `scope_missing`, `endpoint_not_available_in_v1`, `rate_limited`, `platform_not_connected` are precise.

Load `references/output.md` for parsing recipes and the error catalog.

## Scopes and billing

The bearer token carries one of `read` or `read,write`. Reads work with `read`. If X write endpoints land and the token is `read`, the error is `scope_missing`. Fix:

```bash
social logout
social login --scope read,write
```

Every proxy call is metered. Prefer one `--max-results 100` over many small pages, but cap pagination loops — bookmarks and timelines can run thousands of items deep. Use `social usage --summary` to audit recent spend.

## End-to-end recipes

For multi-step playbooks (content audit, bookmarks → markdown, search → analysis), load `references/examples.md`.

## Quick safety rules

- **Never** call X HTTP APIs directly. Use `social x`.
- **Never** echo or save the bearer shown during `login` — the CLI persists it to the OS keyring.
- **Never** retry a `rate_limited` error in a tight loop. Back off per the retry hint.
- **Confirm before write actions** (when those land). Reads are safe; writes are not.
- **Cap pagination loops** with a safety bound (e.g. 20 pages × 100 results = 2000 items) and surface the cap to the user if it trips.

## Additional resources

Loaded only when needed:

- **`references/setup.md`** — install, `social login`, scope/billing, env vars, troubleshooting.
- **`references/commands.md`** — full X command catalog with arguments, field/expansion presets, and output shapes.
- **`references/output.md`** — `--json` / `--pretty`, `jq` recipes, error catalog.
- **`references/examples.md`** — end-to-end X recipes (content audit, bookmarks → markdown, search analysis).

When uncertain about a flag or subtree, the authoritative source is `social x <subtree> --help` and `social schema --json`. Both are cheap and always correct.
