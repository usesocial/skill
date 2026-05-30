---
name: social
description: Use when the user wants to interact with their social networks — LinkedIn or X (Twitter). Covers searching people and posts, reading profiles, timelines, comments, reactions, companies, jobs, and bookmarks, managing connected accounts, and (where enabled) posting on their behalf. Triggers include "search LinkedIn", "find <name> on LinkedIn", "look up this tweet", "my X bookmarks", "show my timeline", "from:<handle>", and explicit `/social`. Operates the `social` CLI (npm `@usesocial/cli`); never call LinkedIn's or X's HTTP APIs directly.
argument-hint: [task — e.g. "search LinkedIn for AI founders", "list my X bookmarks", "show my home timeline"]
---

# social

Operate the user's LinkedIn and X accounts through the `social` CLI. The agent runs the calls; the user stays the decision-maker. Treat `social` as the **only** correct way to reach LinkedIn or X data here — never call their HTTP APIs directly.

`social` wraps two platform subtrees plus shared global commands:

```
social schema | usage | linkedin | x | login | logout
```

- **`social linkedin …`** — profiles, posts, comments, reactions, companies, jobs, people/post search, account lifecycle. Load `references/linkedin.md` for the full catalog and recipes.
- **`social x …`** — tweets, timelines, bookmarks, recent search, user posts, account lifecycle. Load `references/x.md` for the full catalog and recipes.
- **`social usage`** — recent proxy calls or a billing summary (`--summary`), optionally `--platform linkedin|x`.
- **`social schema [path] --json`** — authoritative machine-readable command tree. Cheaper than guessing.
- **`social login` / `social logout`** — session and scope. See `references/setup.md`.

If the user says "Twitter", route to X. If they ask for something a platform exposes but the catalog doesn't list, run `social <platform> --help` or `social schema --json` — do not invent endpoints.

## First-use setup

Before the first call to a platform in a session, confirm the CLI is installed, the user is signed in, and that platform is connected. Run the probe for the platform you need — one call doubles as the connectivity check, so an authenticated user pays no extra round-trip:

```bash
social linkedin users me --json 2>&1 | head -c 400    # for LinkedIn work
social x users me --json 2>&1 | head -c 400            # for X work
```

Interpret the output:

- **Exit 0 with a JSON profile** → installed, signed in, connected. Proceed. (For X, capture `.data.id` — most X list commands need it as a positional.)
- **`command not found: social`** → install: `bun install -g @usesocial/cli@latest` (fall back to `npm install -g @usesocial/cli`). Re-probe.
- **`unauthenticated`, `401`, `Not signed in`** → run `social login`. Interactive device flow that opens `${SOCIAL_WEB_URL}/device`; it cannot complete headlessly. Surface the verification URL/code and wait for the user to approve. `--no-open` prints the URL inline.
- **`platform_not_connected`** → run `social linkedin connect` or `social x connect` (`--no-open` to print the URL). The user approves the handshake in their browser.
- **An `update available …` notice on stderr** (the CLI self-checks at most once a day) → mention it once and offer to run `social upgrade`; update only with consent, never silently. See `references/setup.md` → "Staying current".

Do **not** background `social login` or `social <platform> connect` — both wait on a foreground poll loop.

Full install, scope, billing, and troubleshooting detail lives in `references/setup.md`.

## Invocation conventions

Shared across both platforms:

- `--json` / `--pretty` — pick `--json` whenever output feeds analysis (parsing, filtering, summarising, saving); pipe through `jq`. Use `--pretty` only for small payloads the user reads inline. No flag prints a human-formatted view — don't parse it.
- `--account <handle-or-id>` — disambiguate when multiple accounts of that platform are connected. Resolves against `social <platform> list`.
- `--no-cache` — bypass the cache. Avoid unless verifying freshly-published content; it costs more.
- `--help` — authoritative per-command flag list. Run `social <platform> <subtree> --help` when unsure.

**The two platforms diverge — don't mix their flags:**

|               | LinkedIn (`social linkedin …`)                      | X (`social x …`)                                                                  |
| ------------- | --------------------------------------------------- | --------------------------------------------------------------------------------- |
| Page size     | `--limit` (1–100; 1–1000 for `users connections`)   | `--max-results` (1–100; 5–100 for `users tweets`; 10–100 for `search recent`)     |
| Pagination    | `--cursor` ← `.cursor`                              | `--pagination-token` (or `--next-token` for `search recent`) ← `.meta.next_token` |
| List shape    | `{ items: [...], cursor? }`                         | X v2 envelope: `{ data, includes, meta }`                                         |
| Positional ID | identifier passed inline; CLI resolves URLs/handles | most list commands need **your numeric X user ID** as a positional                |

For X, resolve your ID once and reuse it for the session:

```bash
MY_X_ID=$(social x users me --json | jq -r '.data.id')
```

## Choosing a command

1. Identify the **platform** (LinkedIn vs X — "Twitter" → X) and load that reference.
2. Identify the **noun**: profile/user, post/tweet, company, search, bookmark, timeline, comment, reaction.
3. Identify the **action**: fetch one, list many, search, drill into a sub-resource.
4. Construct `social <platform> <noun> <verb>` and add flags. Verify with `--help` if uncertain.

When the user gives a LinkedIn profile URL or handle, pass it through unchanged — the CLI resolves it. When they give a tweet URL like `https://x.com/handle/status/1843123456789012345`, extract the trailing numeric ID.

## Output handling

- Capture `--json` to a temp file when it might exceed a few thousand tokens, then `jq` over it: `social linkedin search people "query" --limit 100 --json > /tmp/people.json`. This also avoids re-billing the same query.
- Project only the fields you need with `jq` — full payloads are large and burn context fast.
- For user-facing summaries, build a short markdown table from `jq` output rather than dumping raw JSON.
- Surface errors verbatim — codes like `scope_missing`, `endpoint_not_available_in_v1`, `rate_limited`, `platform_not_connected` are precise. Full error catalog in `references/setup.md`.

## Scopes and billing

The bearer token carries one of `read` or `read,write`. Every read command works with `read`. Write endpoints (where enabled) need `read,write`; a mismatch surfaces as `scope_missing`. Fix:

```bash
social logout
social login --scope read,write
```

Every proxy call is metered. Prefer one `--limit 100` / `--max-results 100` over many small pages, but **cap pagination loops** with a safety bound (e.g. 20 pages × 100 = 2000 items) and surface the cap if it trips — bookmarks, timelines, and large company employee lists can run thousands deep. For high-fanout work, quote the cost back to the user before running it. Audit spend with `social usage --summary` (optionally `--platform linkedin|x`).

## Safety rules

- **Never** call LinkedIn or X HTTP APIs directly. Use `social`.
- **Never** echo or save the bearer shown during `login` — the CLI persists it to the OS keyring.
- **Never** retry a `rate_limited` error in a tight loop. Back off per the retry hint.
- **Confirm before write actions** (posting, connecting/disconnecting accounts). Reads are safe; writes and account-lifecycle changes are not.
- **Cap pagination loops** and tell the user when a cap trips.

## Additional resources

Loaded only when needed:

- **`references/setup.md`** — install, `social login`, `connect`, scopes/billing, env vars, error catalog, troubleshooting (both platforms).
- **`references/linkedin.md`** — full LinkedIn command catalog, flags, output shapes, jq recipes, and end-to-end playbooks (lead research, post engagement, paginated scrapes).
- **`references/x.md`** — full X command catalog, field/expansion presets, output shapes, jq recipes, and end-to-end playbooks (content audit, bookmarks → markdown, search analysis, thread reconstruction).

When uncertain about a flag or subtree, the authoritative source is `social <platform> <subtree> --help` and `social schema --json`. Both are cheap and always correct.
