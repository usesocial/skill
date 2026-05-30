# X end-to-end recipes

Patterns that string several `social x` calls together. Save outputs to `/tmp` and re-read with `jq` rather than re-billing the same query.

## 1. Content audit

**Goal:** "Pull my last 30 days of tweets, rank by engagement."

```bash
MY_ID=$(social x users me --json | jq -r '.data.id')
SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")

social x users tweets "$MY_ID" \
  --max-results 100 \
  --start-time "$SINCE" \
  --exclude replies,retweets \
  --tweet-fields public_metrics,created_at \
  --json > /tmp/my-tweets.json

# Top 10 by likes.
jq '
  .data
  | sort_by(-.public_metrics.like_count)
  | .[0:10]
  | .[]
  | {
      id,
      created_at,
      likes: .public_metrics.like_count,
      retweets: .public_metrics.retweet_count,
      replies: .public_metrics.reply_count,
      text: (.text[0:140])
    }
' /tmp/my-tweets.json
```

## 2. Bookmarks → markdown reading list

**Goal:** "Export my X bookmarks as a markdown reading list."

```bash
MY_ID=$(social x users me --json | jq -r '.data.id')

PAGE_TOKEN=""
PAGE=1
> /tmp/bookmarks.ndjson
while :; do
  if [ -n "$PAGE_TOKEN" ]; then
    OUT=$(social x bookmarks list "$MY_ID" \
            --max-results 100 \
            --pagination-token "$PAGE_TOKEN" \
            --tweet-fields author_id,created_at,public_metrics \
            --expansions author_id \
            --user-fields username \
            --json)
  else
    OUT=$(social x bookmarks list "$MY_ID" \
            --max-results 100 \
            --tweet-fields author_id,created_at,public_metrics \
            --expansions author_id \
            --user-fields username \
            --json)
  fi
  echo "$OUT" >> /tmp/bookmarks.ndjson
  PAGE_TOKEN=$(echo "$OUT" | jq -r '.meta.next_token // empty')
  PAGE=$((PAGE + 1))
  [ -z "$PAGE_TOKEN" ] && break
  [ "$PAGE" -gt 20 ] && break    # safety cap: 2000 items
done

# Compose markdown.
jq -r '
  (.includes.users // []) as $users
  | .data[]?
  | . as $t
  | ($users[] | select(.id == $t.author_id) | .username) as $u
  | "- [@\($u)](https://x.com/\($u)/status/\($t.id)) — \($t.text | gsub("\n"; " ") | .[0:120])"
' /tmp/bookmarks.ndjson > /tmp/bookmarks.md
```

Surface the safety cap to the user if it trips.

## 3. Topic search → engagement digest

**Goal:** "Find the top recent tweets about `<topic>`."

```bash
TOPIC="ai safety -is:retweet lang:en"

social x search recent "$TOPIC" \
  --max-results 100 \
  --sort-order relevancy \
  --tweet-fields public_metrics,created_at \
  --expansions author_id \
  --user-fields username,name,verified \
  --json > /tmp/search.json

# Top 10 by engagement = likes + 2*retweets.
jq -r '
  (.includes.users // []) as $users
  | .data
  | map(
      . as $t
      | { id: $t.id,
          text: ($t.text | gsub("\n"; " ") | .[0:200]),
          likes: $t.public_metrics.like_count,
          retweets: $t.public_metrics.retweet_count,
          score: ($t.public_metrics.like_count + 2 * $t.public_metrics.retweet_count),
          author: ($users[] | select(.id == $t.author_id) | .username) }
    )
  | sort_by(-.score)
  | .[0:10]
  | .[]
  | "- @\(.author): \(.text) (\(.likes) ❤︎, \(.retweets) ↻)"
' /tmp/search.json
```

## 4. Conversation reconstruction

**Goal:** "Pull the full conversation thread for a tweet URL."

```bash
TWEET_ID="<numeric-id-from-the-URL>"

# 1. Get the root tweet's conversation_id.
ROOT=$(social x tweets get "$TWEET_ID" \
        --tweet-fields conversation_id \
        --json)
CONV_ID=$(echo "$ROOT" | jq -r '.data.conversation_id')

# 2. Search for all tweets in the conversation (recent window only).
social x search recent "conversation_id:$CONV_ID" \
  --max-results 100 \
  --tweet-fields in_reply_to_user_id,referenced_tweets,created_at,public_metrics \
  --expansions author_id,in_reply_to_user_id \
  --user-fields username \
  --json > /tmp/thread.json

# 3. Print as a flat timeline.
jq -r '
  (.includes.users // []) as $users
  | .data
  | sort_by(.created_at)
  | .[]
  | "[\(.created_at)] @\(($users[] | select(.id == .author_id) | .username) // .author_id): \(.text)"
' /tmp/thread.json
```

Only works for conversations in the last 7 days (X recent-search limit).

## 5. Connect a new account end-to-end

```bash
# 1. Make sure the user has a seat.
social usage --summary --json | jq '.seats // empty'

# 2. Connect.
social x connect --no-open
# (CLI prints a URL — surface it to the user, wait for them to approve.)

# 3. Confirm.
social x list
social x users me --pretty
```

Use `--no-open` from inside a Claude Code session so the URL lands in the chat — the user opens it themselves.

## 6. X billing & usage audit

**Goal:** "What have I been spending on X?"

```bash
# Aggregate first.
social usage --summary --platform x --pretty

# Then the recent firehose for the last 30 days.
SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")
social usage --platform x --from "$SINCE" --limit 100 --json | jq
```
