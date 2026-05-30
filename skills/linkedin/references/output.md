# Output, flags, and parsing — LinkedIn

How to shape and consume `social linkedin` output.

## Output modes

Every command supports the same trio:

- **No flag** — pretty `console.log` intended for a human in a terminal. Do not parse it programmatically.
- **`--json`** — raw single-line JSON. Pipe to `jq` or save to disk.
- **`--pretty`** — pretty-printed JSON (2-space indent). Still valid JSON.

Pick `--json` for any downstream analysis. `--pretty` is better only when the user reads the result inline and the payload is small.

## Limits

| Command | Flag | Range |
|---|---|---|
| Most LinkedIn list endpoints | `--limit` | 1–100 |
| `users connections` | `--limit` | 1–1000 |
| `social usage` | `--limit` | 1–100 (default 50) |

Always prefer one large-limit call over many small ones — every call is metered.

## Pagination

LinkedIn uses **`--cursor`** for every paginated endpoint. The next-page token comes back as `.cursor` in the response. Stop when it is missing or empty.

```bash
NEXT=$(jq -r '.cursor // empty' /tmp/page1.json)
[ -n "$NEXT" ] && social linkedin search people "$Q" --limit 100 --cursor "$NEXT" --json
```

## Time windows

LinkedIn endpoints have **no native time-window flags**. Filter after the fact in `jq` on `.created_at` or whichever date field the payload exposes.

## Account selection

Every command accepts `--account <handle-or-public-id>`. Resolves against `social linkedin list`. Default account is used when omitted.

## Cache

`--no-cache` forces a fresh upstream fetch. Default already serves fresh-enough data. Reach for it only when:

- Verifying a freshly-published post is visible to the proxy.
- Debugging a stale-data report.
- Comparing pre- and post-action state.

## jq recipes

Run against `--json` output:

```bash
# Pull name, headline, profile URL from a search.
jq -r '
  .items[]
  | [
      .name,
      .headline,
      "https://www.linkedin.com/in/" + .public_identifier
    ]
  | @tsv'

# Connections as CSV.
jq -r '.items[] | [.name, .occupation, .public_identifier] | @csv' > connections.csv

# Drop verbose fields for an LLM-friendly summary.
jq '.items[] | {name, headline, location: .location.default}'

# Aggregate usage by platform (inspect schema first).
social usage --summary --json | jq
```

When chaining over a saved file, write once:

```bash
social linkedin search people "AI infra" --limit 100 --json > /tmp/people.json
jq '.items | length' /tmp/people.json
jq -r '.items[].public_identifier' /tmp/people.json | sort -u
```

This avoids re-billing the same query.

## Error catalog

Errors arrive on stderr with a non-zero exit. In `--json` mode they print as `{ "status": "error", "code": "...", "message": "..." }`. Common codes:

| Code | Meaning | Fix |
|---|---|---|
| `unauthenticated` | No bearer or expired. | `social login`. |
| `scope_missing` | Token lacks `linkedin:write`. | `social logout && social login --scope read,write`. |
| `platform_not_connected` | No connected LinkedIn account. | `social linkedin connect`. |
| `account_not_found` | `--account` value did not match. | `social linkedin list`, reuse the printed handle/id. |
| `endpoint_not_available_in_v1` | Path not in the adapter's allowlist. | Pick a different command; do not retry. |
| `rate_limited` | Upstream throttle hit (Unipile or LinkedIn). | Back off per the retry hint. |
| `invalid_argument` | A flag failed parsing/validation. | Check `--help`; ranges in this doc are authoritative. |
| `no_available_seat` | Org has no remaining billing seat at `connect`. | User adds a seat in the dashboard or releases one with `disconnect`. |
| `linkedin_connect_timed_out` | User did not approve in browser within 5 minutes. | Re-run `social linkedin connect`. |

Surface error messages verbatim — they are precise enough for the user to act on.

## `social schema`

The authoritative machine-readable command tree:

```bash
social schema --json | jq '.subCommands.linkedin.subCommands | keys'
social schema --json | jq '.subCommands.linkedin.subCommands.posts.subCommands'
```

Use this when uncertain about a flag, a sub-tree, or whether something exists at all. Faster and cheaper than guessing.
