# X - `social x`

Shared rules live in `SKILL.md`: `sync` pulls own data into the local mirror, `sql` reads it for free, named reads hit the live network and spend credits, writes act.

`social x <command>`. Live X list reads use `--limit` and `--cursor`; the next cursor is `.meta.cursor`. List output is `.items[]`; single-resource output is `.data`.

## Account lifecycle

| Command | Purpose |
| --- | --- |
| `social account connect x` | OAuth handshake. Opens the web app on a TTY; prints the URL on non-TTY. |
| `social account reconnect x <account>` | Re-auth after token revoke. |
| `social account disconnect x <account>` | Disconnect an account. |
| `social account` | Inspect authenticated user and connected accounts. |

## Profiles and posts

| Command | Args | Notes |
| --- | --- | --- |
| `profile [target]` | `--user-fields`, `--tweet-fields`, `--expansions`, `--account` | Authenticated profile by default. Target accepts `@handle`, profile URL, or `profile_id:<id>`. |
| `tweets <target>` | `--limit 5-100`, `--cursor`, `--since-id`, `--until-id`, `--start-time`, `--end-time`, `--exclude replies\|retweets`, field flags, `--account`, `-H/--header` | Live, metered, target required. |
| `liked [target]` | `--limit`, `--cursor`, field flags, `--account`, `-H/--header` | Posts a user liked; omit target for the selected account. |
| `mentions [target]` | `--limit`, `--cursor`, field flags, `--account`, `-H/--header` | Posts mentioning a user; omit target for the selected account. |
| `followers [target]` | `--limit 1-1000`, `--cursor`, `--user-fields`, `--expansions`, `--tweet-fields`, `--account`, `-H/--header` | A user's followers, live and metered; omit target for the selected account. |
| `following [target]` | `--limit 1-1000`, `--cursor`, `--user-fields`, `--expansions`, `--tweet-fields`, `--account`, `-H/--header` | Accounts a user follows, live and metered; omit target for the selected account. |

Fresh data or someone else's graph: the live `followers`/`following` commands above. Your own graph for free: sync, then query `x_followers` or `x_following` with SQL.

## Tweets and engagement

| Command | Args | Notes |
| --- | --- | --- |
| `tweet <target>` | field flags, `--account`, `-H/--header` | Single post by `post_id:<id>` or URL. |
| `timeline` | `--limit 1-100`, `--cursor`, time/id bounds, field flags, `--account` | Home timeline. |
| `replies <target>` | `--limit 10-100`, `--cursor`, field flags, `--account` | Replies to `post_id:<id>`, `conversation_id:<id>`, or a post URL. |
| `quotes <target>` | `--limit`, `--cursor`, field flags, `--account`, `-H/--header` | Quote posts of a post. |
| `likers <target>` | `--limit`, `--cursor`, `--user-fields`, `--account`, `-H/--header` | Users who liked a post. |
| `reposters <target>` | `--limit`, `--cursor`, `--user-fields`, `--account`, `-H/--header` | Users who reposted a post. |

Field flags extend default comma-separated presets and de-dupe repeated fields. Timeline, liked, mentions, quotes, replies, tweet, and tweets use safe tweet/user/media/poll/place defaults. If a returned post has only `author_id`, budget a `profile profile_id:<id>` follow-up when the author name matters.

## Writes

Confirm with the user before every write.

| Command | Args | Notes |
| --- | --- | --- |
| `post` | body from stdin | Public post. |
| `message <recipients>` | body from stdin | DM to chat URL, `chat_id:<id>`, `@handle`, profile URL, or `profile_id:<id>`; comma-separate profile targets for a group. |
| `like <target>` / `unlike <target>` | `--account` | Like or unlike a post. |
| `repost <target>` / `unrepost <target>` | `--account` | Repost or undo repost. |
| `bookmark <target>` / `unbookmark <target>` | `--account` | Add or remove a saved post. |
| `follow <target>` / `unfollow <target>` | `--account` | Follow or unfollow a user. |
| `delete <target>` | `--account` | Delete one of your own posts. |

## Local mirror (SQL)

Syncable collections: `tweets`, `followers`, `following`, `bookmarks`, `messages`.

```bash
social x sync
social x sync messages
social x sql
```

Bare `sql` prints schema, row counts, and freshness. Query output is `{ account, items, meta }`; project rows with `.items[]`. `.meta.cost.credits` is `0` on every SQL read.

Never-synced tables fail with the sync command:

```text
No synced x_messages yet — run `social x sync messages` first.
```

`x_messages` is a view. It includes `sender_handle`, `sender_name`, `sender_avatar_url`, and `sender_headline` from `x_profiles`; `x_raw_messages` is internal.

### Recipes

```bash
# Inbox triage.
social x sync messages
social x sql "SELECT sender_handle, text, datetime(created_at/1000,'unixepoch') AS at FROM x_messages ORDER BY created_at DESC LIMIT 20" \
  | jq '.items[]'

# Conversation with one person; the view resolves sender_handle.
social x sql "SELECT sender_handle, text, datetime(created_at/1000,'unixepoch') AS at FROM x_messages WHERE sender_handle = 'handle' OR dm_conversation_id = (SELECT dm_conversation_id FROM x_messages WHERE sender_handle = 'handle' LIMIT 1) ORDER BY created_at ASC" \
  | jq '.items[]'

# Top synced followers.
social x sync followers
social x sql "SELECT username, name, followers_count FROM x_followers ORDER BY followers_count DESC LIMIT 100" \
  | jq '.items[]'

# Saved-post export.
social x sync bookmarks
social x sql "SELECT text, url FROM x_bookmarks ORDER BY created_at DESC LIMIT 1000" \
  | jq '.items[]'
```

Timestamps are epoch millis. Use `datetime(col/1000,'unixepoch')` in SQLite.

## Example live reads

```bash
# Smoke test.
social x profile

# Home timeline, excluding replies.
social x timeline --limit 25 --exclude replies

# A user's recent posts.
social x tweets profile_id:<profile-id> --limit 30 --exclude retweets

# Fetch by ID.
social x tweet post_id:<post-id>

# Paginate a live cursor read.
PAGE1=$(social x timeline --limit 100)
NEXT=$(echo "$PAGE1" | jq -r '.meta.cursor // empty')
[ -n "$NEXT" ] && social x timeline --limit 100 --cursor "$NEXT"
```

## jq recipes

Run against live command JSON unless the recipe says SQL:

```bash
# Top posts by like count.
jq '.items | sort_by(-.public_metrics.like_count) | .[0:10] | .[] | {id, url, text, likes: .public_metrics.like_count}'

# Project author info when present.
jq '.items[] | . as $t | {url: $t.url, text: $t.text, author: ($t.author.username // $t.author_id), likes: $t.public_metrics.like_count}'

# Drop verbose fields.
jq '.items[] | {id, url, text, created_at, metrics: .public_metrics}'

# Inspect billing and paging metadata.
jq '{cost: .meta.cost, cursor: .meta.cursor, resolved: .meta.resolved}'
```

Save live outputs before chaining so you do not re-bill:

```bash
social x tweets @handle --limit 100 > /tmp/tweets.json
jq '.items | length' /tmp/tweets.json
jq '.meta.cursor // empty' /tmp/tweets.json
```

## End-to-end sketches

### Content audit

```bash
SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")

social x tweets @handle \
  --limit 100 \
  --start-time "$SINCE" \
  --exclude replies,retweets \
  --tweet-fields public_metrics,created_at \
  > /tmp/my-tweets.json

jq '
  .items | sort_by(-.public_metrics.like_count) | .[0:10] | .[]
  | {id, url, created_at,
     likes: .public_metrics.like_count,
     retweets: .public_metrics.retweet_count,
     replies: .public_metrics.reply_count,
     text: (.text[0:140])}
' /tmp/my-tweets.json
```

### Billing audit

```bash
social account billing
social account usage

SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")
social account logs --platform x --from "$SINCE" --limit 100 \
  | jq -r '.items[] | [.createdAt, .platform, .method, .path, .responseStatus, .cacheStatus, .credits] | @tsv'
```
