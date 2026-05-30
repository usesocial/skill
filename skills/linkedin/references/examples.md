# LinkedIn end-to-end recipes

Patterns that string several `social linkedin` calls together. Save outputs to `/tmp` and re-read with `jq` rather than re-billing the same query.

## 1. Lead research

**Goal:** "Find AI safety founders in NYC and give me their headlines and profile URLs."

```bash
# 1. Search — capture once, paginate if needed.
social linkedin search people "AI safety founder New York" --limit 50 --json > /tmp/leads.json

# 2. Project the fields we care about.
jq -r '
  .items[]
  | [
      .name,
      .headline,
      .location.default,
      "https://www.linkedin.com/in/" + .public_identifier
    ]
  | @tsv
' /tmp/leads.json | column -t -s $'\t'

# 3. (Optional) Drill into the top three — full profiles, including experience.
jq -r '.items[0:3][].public_identifier' /tmp/leads.json | while read -r ID; do
  social linkedin users get "$ID" --linkedin-sections experience,education --json \
    > "/tmp/profile-$ID.json"
done
```

Confirm with the user before running step 3 — three profile fetches at $cost each.

## 2. Post engagement analysis

**Goal:** "Pull the comments and reactions on `<post-url>` and summarise the sentiment."

```bash
POST_ID="<numeric-id-from-the-URL>"

# Comments (most relevant first).
social linkedin posts comments "$POST_ID" --limit 100 --sort-by MOST_RELEVANT --json \
  > /tmp/comments.json

# Reactions (paginate until empty).
social linkedin posts reactions "$POST_ID" --limit 100 --json > /tmp/reactions-1.json
CURSOR=$(jq -r '.cursor // empty' /tmp/reactions-1.json)
if [ -n "$CURSOR" ]; then
  social linkedin posts reactions "$POST_ID" --limit 100 --cursor "$CURSOR" --json \
    > /tmp/reactions-2.json
fi

# Slim down for an LLM-friendly digest.
jq -r '.items[] | {text, author: .author.name, reactions: .reactions.total}' /tmp/comments.json
jq -r '.items[] | {type, name: .reactor.name}' /tmp/reactions-1.json
```

Summarise the projected JSON in chat. Do not paste raw payloads — they bloat context fast.

## 3. Company employee scrape

**Goal:** "Pull the first 200 employees at Anthropic with their roles."

```bash
COMPANY="anthropic"
> /tmp/employees.ndjson
CURSOR=""
PAGE=1
while :; do
  if [ -n "$CURSOR" ]; then
    OUT=$(social linkedin companies employees "$COMPANY" --limit 100 --cursor "$CURSOR" --json)
  else
    OUT=$(social linkedin companies employees "$COMPANY" --limit 100 --json)
  fi
  echo "$OUT" >> /tmp/employees.ndjson
  CURSOR=$(echo "$OUT" | jq -r '.cursor // empty')
  PAGE=$((PAGE + 1))
  [ -z "$CURSOR" ] && break
  [ "$PAGE" -gt 2 ] && break    # cap at 200
done

# Project.
jq -r '.items[] | [.name, .occupation, .public_identifier] | @tsv' /tmp/employees.ndjson \
  | column -t -s $'\t'
```

The safety cap (`PAGE -gt 2`) prevents an unintended 10,000-employee scrape — surface the limit to the user if they want more.

## 4. Connect a new account end-to-end

```bash
# 1. Make sure the user has a seat.
social usage --summary --json | jq '.seats // empty'

# 2. Connect.
social linkedin connect --no-open
# (CLI prints a URL — surface it to the user, wait for them to approve.)

# 3. Confirm.
social linkedin list
social linkedin users me --pretty
```

Use `--no-open` from inside a Claude Code session so the URL lands in the chat — the user opens it themselves.

## 5. Account billing & usage audit

**Goal:** "What have I been spending on LinkedIn?"

```bash
# Aggregate first.
social usage --summary --platform linkedin --pretty

# Then the recent firehose for the last 30 days.
SINCE=$(date -u -v-30d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
        || date -u -d '30 days ago' +"%Y-%m-%dT%H:%M:%SZ")
social usage --platform linkedin --from "$SINCE" --limit 100 --json | jq
```

Use this whenever the user asks "where did the bill go" or wants to spot unexpected hotspots.
