# LinkedIn command catalog

`social linkedin <subtree> <command>`. All commands return JSON over `--json`, pretty-printed JSON over `--pretty`, and pass through `--account <handle>` plus `--no-cache`.

## Account lifecycle

| Command | Purpose |
|---|---|
| `social linkedin connect [--no-open] [--json\|--pretty]` | Hosted-auth handshake (Unipile). Opens the web app. |
| `social linkedin reconnect <username-or-id> [--no-open]` | Re-auth an existing account. |
| `social linkedin disconnect [username-or-id]` | Disconnect. Omit the id when only one account is connected. |
| `social linkedin list [--include-disconnected]` | List connected accounts. |

## `users`

| Command | Args | Notes |
|---|---|---|
| `users me` | — | Connected profile. Smoke test. |
| `users get <identifier>` | `--linkedin-sections <csv>`, `--linkedin-api recruiter\|sales_navigator`, `--notify` | `<identifier>` is a public ID (`john-smith-1a2b`), a provider ID starting `ACo…`/`ADo…`, or `me`. `--notify` is rare — leave off. |
| `users connections <identifier>` | `--limit 1-1000`, `--cursor`, `--filter <name>` | Higher limit than the standard 100. |
| `users posts <identifier>` | `--limit 1-100`, `--cursor`, `--is-company` | Pass `--is-company` when the identifier is a numeric company ID. |

`me` works for `users get me`, `users posts me`, `users connections me`.

## `posts`

| Command | Args | Notes |
|---|---|---|
| `posts get <post-id>` | — | `<post-id>` is the LinkedIn numeric ID, `urn:li:ugcPost:<id>`, or `urn:li:share:<id>`. For activity URLs (`activity-1234-…`), pass just the trailing numeric ID. |
| `posts comments <post-id>` | `--limit 1-100`, `--cursor`, `--sort-by MOST_RECENT\|MOST_RELEVANT`, `--comment-id <id>` | `--comment-id` fetches replies to a specific comment. |
| `posts reactions <post-id>` | `--limit 1-100`, `--cursor`, `--comment-id <id>` | `--comment-id` switches to reactions on that comment. |

`<post-id>` for `comments` and `reactions` is the `social_id` returned in a post payload — usually the numeric ID after `activity-`.

## `search`

| Command | Args | Notes |
|---|---|---|
| `search people <keywords>` | `--limit 1-100`, `--cursor`, `--api recruiter\|sales_navigator` | Default API is `classic` (omit `--api`). Quote the keywords. |
| `search posts <keywords>` | `--limit 1-100`, `--cursor`, `--api recruiter\|sales_navigator` | Same flags as people. |

## `companies`

| Command | Args | Notes |
|---|---|---|
| `companies get <identifier>` | — | `<identifier>` is the company slug (`anthropic`), numeric ID, or URN. |
| `companies employees <identifier>` | `--limit 1-100`, `--cursor`, `--keywords <q>` | `--keywords` filters by role/name. |
| `companies jobs <identifier>` | `--limit 1-100`, `--cursor`, `--keywords <q>` | Same. |

## Example invocations

```bash
# Smoke test.
social linkedin users me --pretty

# Find founders.
social linkedin search people "founder ai" --limit 25 --json > /tmp/founders.json
jq -r '.items[] | [.name, .headline, .public_identifier] | @tsv' /tmp/founders.json

# Drill into a post's reactions.
social linkedin posts reactions 7286419083240247296 --limit 100 --json \
  | jq '.items[].reactor.public_identifier'

# Walk a company's recent posts.
social linkedin users posts anthropic --is-company --limit 20 --pretty

# Paginate.
PAGE1=$(social linkedin search people "AI safety" --limit 50 --json)
CURSOR=$(echo "$PAGE1" | jq -r '.cursor // empty')
[ -n "$CURSOR" ] && social linkedin search people "AI safety" --limit 50 --cursor "$CURSOR" --json
```

## Output shape (typical)

List endpoints return `{ items: [...], cursor?: string }`. Single-object endpoints return the object directly. Always inspect with `jq keys` if uncertain — the upstream Unipile payload bleeds through.

## Caching

`--no-cache` forces the proxy to bypass cached responses. Use sparingly: it costs more (no cache hit credit) and tends to be rate-limited harder upstream. Default behavior is correct for almost every query.
