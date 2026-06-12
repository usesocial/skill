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
| `profile [target]` | `--user-fields`, `--tweet-fields`, `--expansions`, `--account` | Authenticated profile by default. Target accepts `@username`, profile URL, or `profile_id:<id>`. |
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

Field flags extend default comma-separated presets and de-dupe repeated fields. Timeline, liked, mentions, quotes, replies, tweet, and tweets use safe tweet/user/media/poll/place defaults. When the response includes expansion users, the CLI joins them into each post's `.author`; if a post still has only `author_id`, budget a `profile profile_id:<id>` follow-up when the author name matters.

## Writes

Confirm with the user before every write.

| Command | Args | Notes |
| --- | --- | --- |
| `post` | body from stdin | Public post. |
| `message <recipients>` | body from stdin | DM to chat URL, `chat_id:<id>`, `@username`, profile URL, or `profile_id:<id>`; comma-separate profile targets for a group. |
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
social x sync tweets --since 2026-05-04 --timeout 900
social x sql
```

`--since` limits a sync to newer items using an ISO date like `2026-05-04` or datetime like `2026-05-04T00:00:00Z` on collections whose bare-sync row shows `supportsSince: true`. Prefer it over full re-pulls; it spends fewer credits. `--reset` deletes the collection's local rows and sync state; the next plain sync rebuilds from scratch.

`--timeout <seconds>` is accepted for sync command parity and validates as a positive integer. X does not add in-process rate-limit retries; on a sync 429, use the JSON `resumeAt`, `retryCommand`, `hint`, and `syncResume` fields when present.

`sync` output is always `{ data, meta }`; bare sync listings are `.data[]`, and collection summaries or `--reset` results are `.data`.

Bare `sql` prints compact schema metadata under `.data`. Query output is `{ account, items, meta }`; project rows with `.items[]`. `.meta.cost.credits` is `0` on every SQL read.

Never-synced tables fail with the sync command:

```text
No synced x_messages yet — run `social x sync messages` first.
```

`x_messages` is a view. It includes `conversation_type` (`1to1` or `group`), `has_attachments` (0/1), `sender_username`, `sender_name`, `sender_avatar_url`, and `sender_headline` from `x_profiles`. For `1to1` rows it also includes `counterpart_id`, `counterpart_username`, `counterpart_name`, `counterpart_avatar_url`, and `counterpart_headline` for the other participant; counterpart columns are `null` on group rows. `x_profiles.receives_your_dm` is `1` for profiles that accept your DMs — useful before drafting outreach.

`x_conversation_participants` is a derived view with `conversation_id`, `participant_id`, `participant_username`, `participant_name`, and `participant_avatar_url`. It uses observed senders, dash-form 1:1 conversation IDs, and `ParticipantsJoin`/`ParticipantsLeave` `participant_ids`.

Triage caveats: `text` is `null` on attachment-only DMs — not an error; the message carried media or a card, and `has_attachments` will be `1`. `x_conversation_participants` cannot see lurkers who never sent a message and never triggered a join/leave event. A dash-free conversation with only one observed inbound message is classified as `group`.

### Recipes

```bash
# Inbox triage.
social x sync messages
social x sql "SELECT conversation_type, COALESCE(counterpart_username, sender_username) AS visible_user, sender_username, has_attachments, text, datetime(created_at/1000,'unixepoch') AS at FROM x_messages ORDER BY created_at DESC LIMIT 20" \
  | jq '.items[]'

# Conversation with one person; 1:1 rows expose counterpart_username.
social x sql "SELECT sender_username, counterpart_username, text, has_attachments, datetime(created_at/1000,'unixepoch') AS at FROM x_messages WHERE conversation_id = (SELECT conversation_id FROM x_messages WHERE sender_username = 'username' OR counterpart_username = 'username' LIMIT 1) ORDER BY created_at ASC" \
  | jq '.items[]'

# Who is in each thread, using the derived participants view.
social x sql "SELECT m.conversation_id, m.conversation_type, GROUP_CONCAT(DISTINCT COALESCE(p.participant_username, p.participant_id)) AS participants, datetime(MAX(m.created_at)/1000,'unixepoch') AS last_at FROM x_messages m LEFT JOIN x_conversation_participants p ON p.conversation_id = m.conversation_id GROUP BY m.conversation_id, m.conversation_type ORDER BY MAX(m.created_at) DESC LIMIT 20" \
  | jq '.items[]'

# Top synced followers.
social x sync followers
social x sql "SELECT username, name, followers_count FROM x_followers ORDER BY followers_count DESC LIMIT 100" \
  | jq '.items[]'

# Saved-post export.
social x sync bookmarks
social x sql "SELECT text, url FROM x_bookmarks ORDER BY created_at DESC LIMIT 1000" \
  | jq '.items[]'

# Own-content audit, free after sync. Metric columns are flat:
# like_count, retweet_count, reply_count, quote_count, bookmark_count, impression_count.
social x sync tweets
social x sql "SELECT text, url, like_count, retweet_count, reply_count, impression_count FROM x_tweets ORDER BY like_count DESC LIMIT 10" \
  | jq '.items[]'
```

Timestamps are epoch millis. Use `datetime(col/1000,'unixepoch')` in SQLite.

## Example live reads

```bash
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
social x tweets @username --limit 100 > /tmp/tweets.json
jq '.items | length' /tmp/tweets.json
jq '.meta.cursor // empty' /tmp/tweets.json
```

## End-to-end sketches

### Content audit

For your own account, prefer the free path: `social x sync tweets`, then the `x_tweets` SQL recipe above. Use the live read below for someone else's account:

```bash
SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")

social x tweets @username \
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
