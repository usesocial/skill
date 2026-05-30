---
name: linkedin
description: This skill should be used when the user wants to do anything on LinkedIn from inside an agent — search people or posts, fetch a profile, list someone's posts/connections, drill into a post's comments or reactions, look up a company's employees or jobs, connect/disconnect a LinkedIn account, or check `social` CLI usage and billing. Triggers include phrases like "search LinkedIn", "find on LinkedIn", "look up <name> on LinkedIn", "LinkedIn profile", "LinkedIn comments", "LinkedIn company employees", "connect my LinkedIn", and explicit invocation via `/social:linkedin`. Wraps the `social linkedin` subtree of the `social` binary (npm `@usesocial/cli`).
argument-hint: [task — e.g. "search for AI founders", "fetch comments on <post-url>", "list Anthropic employees"]
---

# social linkedin

Use the `social linkedin` subtree to read (and where supported, write) LinkedIn data on the user's behalf. Routed through Unipile hosted auth at `/v1/linkedin/*` on the API. Treat it as the only correct way to fetch LinkedIn data in this skill — never call LinkedIn's HTTP APIs directly.

## When this skill applies

Match any of:

- Profiles — `users me`, `users get`, `users posts`, `users connections`.
- Posts — `posts get`, `posts comments`, `posts reactions`.
- Search — `search people`, `search posts`.
- Companies — `companies get`, `companies employees`, `companies jobs`.
- Account lifecycle — `connect`, `reconnect`, `disconnect`, `list`.
- Usage / billing — `social usage` (recent calls or `--summary`).

If the user asks for something LinkedIn exposes but the catalog below does not list, run `social linkedin --help` or `social schema --json` — do not guess endpoints.

## First-use setup

Before the first LinkedIn call in a session, confirm the CLI is installed, the user is signed in, and a LinkedIn account is connected. Use a single combined probe so an authenticated user pays no extra round-trip:

```bash
social linkedin users me --json 2>&1 | head -c 400
```

Interpret the output:

- **Exit 0 with JSON profile** → installed, signed in, LinkedIn connected. Proceed.
- **`command not found: social`** → install: `bun install -g @usesocial/cli@latest` (fall back to `npm install -g @usesocial/cli`). Re-probe.
- **`unauthenticated`, `401`, `Not signed in`** → run `social login`. This is an interactive device flow that opens `${SOCIAL_WEB_URL}/device`; it cannot complete headlessly inside the agent. Surface the verification URL/code to the user and wait for them to approve. Use `--no-open` to print the URL inline.
- **`platform_not_connected`** → run `social linkedin connect` (or `--no-open` to print the URL). The user approves Unipile in their browser.

Do **not** background `social login` or `social linkedin connect` — both wait on a foreground poll loop.

For full setup, scope, and troubleshooting details, load `references/setup.md`.

## Invocation pattern

Every command supports `--json` and `--pretty`:

- **Pass `--json`** when output feeds further analysis (parsing, filtering, summarising, saving). Pipe through `jq`.
- **Pass `--pretty`** only when the user reads the result directly in chat and it is small.

Default flags worth knowing:

- `--limit <n>` — almost every list command. Use 10 for exploratory, 25–50 for analysis. Max 100 (1000 for `users connections`).
- `--cursor <token>` — pagination cursor returned in the previous page's `.cursor`.
- `--account <handle>` — only needed when multiple LinkedIn accounts are connected; resolved by `social linkedin list`.
- `--no-cache` — bypass the cache layer. Avoid unless verifying freshly-published content; costs more.
- `--help` — authoritative per-command flag list. Run `social linkedin <subtree> --help` when uncertain.

For the complete flag reference and parsing patterns, load `references/output.md`.

## Command surface

| Subtree | Commands | Notes |
|---|---|---|
| `users` | `me`, `get <id>`, `posts <id>`, `connections <id>` | `<id>` is a public identifier (`john-smith-1a2b`), a provider ID starting `ACo…`/`ADo…`, or `me`. `users posts` supports `--is-company` for numeric company IDs. `users connections` allows `--limit` up to 1000. |
| `posts` | `get <post-id>`, `comments <post-id>`, `reactions <post-id>` | `<post-id>` is the numeric ID from an activity URL, `urn:li:ugcPost:<id>`, or `urn:li:share:<id>`. `comments` supports `--sort-by MOST_RECENT\|MOST_RELEVANT`; both `comments` and `reactions` accept `--comment-id` to scope into a specific comment thread. |
| `search` | `people <keywords>`, `posts <keywords>` | Quote the keywords. `--api recruiter\|sales_navigator` opts into LinkedIn API families (default classic). |
| `companies` | `get <identifier>`, `employees <identifier>`, `jobs <identifier>` | `<identifier>` is a slug (`anthropic`), numeric ID, or URN. `employees` and `jobs` accept `--keywords <q>` to filter. |
| Lifecycle | `connect [--no-open]`, `reconnect <id> [--no-open]`, `disconnect [<id>]`, `list [--include-disconnected]` | `disconnect` omits the id when only one account is connected. |

Load `references/commands.md` for full arguments, flag ranges, and example output shapes per command.

## Choosing a command

1. Identify the **noun**: user, post, company, search, comment, reaction.
2. Identify the **action**: fetch one, list, search, drill into a sub-resource.
3. Construct `social linkedin <noun> <verb>` and add flags. Verify with `--help` if uncertain.

When the user names a profile URL or handle, pass it through unchanged — do not invent an ID. The CLI resolves it.

## Output handling

- Capture `--json` output to a temp file when it might exceed a few thousand tokens: `social linkedin search people "query" --limit 100 --json > /tmp/people.json` then `jq` over it.
- Project only the fields you need with `jq` — full LinkedIn payloads are large and burn context fast.
- For user-facing summaries, build a short markdown table from `jq` output rather than dumping raw JSON.
- Surface errors verbatim — codes like `scope_missing`, `endpoint_not_available_in_v1`, `rate_limited`, `platform_not_connected` are precise.

Load `references/output.md` for parsing recipes and the error catalog.

## Scopes and billing

The bearer token carries one of `read` or `read,write`. Read commands (every LinkedIn op in the catalog above) work with `read`. If LinkedIn write endpoints get added and the token is `read`, the error is `scope_missing`. Fix:

```bash
social logout
social login --scope read,write
```

Every proxy call is metered. Prefer one `--limit 50` over fifty `--limit 1` calls. For high-fanout work (paginating a 1000-employee company), quote the cost back to the user before running it. Use `social usage --summary` to audit recent spend.

## End-to-end recipes

For multi-step playbooks (lead research, post engagement analysis, paginated scrapes), load `references/examples.md`.

## Quick safety rules

- **Never** call LinkedIn HTTP APIs directly. Use `social linkedin`.
- **Never** echo or save the bearer shown during `login` — the CLI persists it to the OS keyring.
- **Never** retry a `rate_limited` error in a tight loop. Back off per the retry hint.
- **Confirm before write actions** (when those land). Reads are safe; writes are not.

## Additional resources

Loaded only when needed:

- **`references/setup.md`** — install, `social login`, scope/billing, env vars, troubleshooting.
- **`references/commands.md`** — full LinkedIn command catalog with arguments and output shapes.
- **`references/output.md`** — `--json` / `--pretty`, `jq` recipes, error catalog.
- **`references/examples.md`** — end-to-end LinkedIn recipes (lead research, post engagement, paginated scrapes).

When uncertain about a flag or subtree, the authoritative source is `social linkedin <subtree> --help` and `social schema --json`. Both are cheap and always correct.
