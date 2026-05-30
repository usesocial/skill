# Setup & Auth

How to get `social` working from a fresh machine and how to recover from auth failures. Shared across LinkedIn and X — only the per-platform `connect` handshake differs.

## Install the CLI

Three supported routes — use whichever the user prefers; Homebrew is convenient on macOS/Linux, Bun/npm for Node-based setups:

```bash
brew install social                  # Homebrew (formula/tap as published)
# or
bun install -g @usesocial/cli@latest # Bun
# or
npm install -g @usesocial/cli        # npm
```

The package publishes the `social` binary (ESM, Node 24). If the binary is missing after install, surface the install log — usually a permissions error on the global prefix.

## Staying current

Two moving parts can drift: the `social` binary and this skill. Keep both fresh, but never update silently — surface the notice and let the user decide.

### The CLI

`social` checks for a newer release at most once a day and prints a one-line notice to **stderr** when one exists:

```
update available: 0.4.1 → 0.5.0 — run `social upgrade`
```

The first-use probe in `SKILL.md` already runs the CLI every session and captures stderr (`2>&1`), so this notice rides along for free — no extra round-trip. When you see it, surface it once and offer to update.

`social upgrade` is the canonical path. The CLI knows how it was installed (Homebrew, Bun, or npm) and dispatches the right command itself — you do **not** need to detect the package manager:

```bash
social upgrade            # detect install method, update in place
social upgrade --check    # report installed vs latest; exit non-zero if stale; change nothing
```

If `social upgrade` is unavailable (older build prints "unknown command"), fall back to the install command for the manager actually in use — ask the user how they installed it rather than guessing:

| Installed via | Update command                      |
| ------------- | ----------------------------------- |
| Homebrew      | `brew upgrade social`               |
| Bun           | `bun add -g @usesocial/cli@latest`  |
| npm           | `npm install -g @usesocial/cli@latest` |

Do **not** infer the manager by resolving `which social` symlinks — it is brittle across Homebrew prefixes, `~/.bun/bin`, and npm prefixes. Updating the CLI mid-session is safe: each call is a fresh process, so the new binary takes effect on the next command.

### The skill

If the skill was installed with `npx skills`, update it with:

```bash
npx skills update social
```

If it was installed by the CLI (`social login --skill-target …`), `social upgrade` refreshes it in place. Either way, updating the skill markdown does **not** change the current session — the old text is already loaded in context; the refresh takes effect the next time the skill loads.

Verify:

```bash
social --version
social --help
```

## `social login`

`login` runs the better-auth **device-authorization** flow. It is interactive — the CLI prints a verification URL and a user code, opens the browser to `${SOCIAL_WEB_URL}/device`, and polls until the web session approves the request. **Do not background it; do not pipe `yes` into it.** Either run it in a separate terminal the user controls, or invoke it as a foreground Bash command and surface the verification URL/code so the user can approve.

Flags:

| Flag                                  | Purpose                                                                                                                     |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--email <addr>`                      | Skip the email prompt.                                                                                                      |
| `--scope read` \| `read,write`        | Pick capability bundle. `read` is safer; `read,write` allows write endpoints. Default `read`.                               |
| `--accept-pricing`                    | Pre-accept the seat checkout if needed.                                                                                     |
| `--skill-target none\|manual\|<path>` | Where to install the agent skill. `social login` installs `usesocial/skill` by default; from inside this plugin use `none`. |
| `--no-open`                           | Print the URL instead of opening a browser (use under WSL/SSH/inside-agent).                                                |
| `--non-interactive`                   | Disable prompts; emit `{ status: "needs_input", missing: [...] }` JSON if anything is missing.                              |
| `--json` / `--pretty`                 | Machine output. Pair with `--non-interactive` for scripted flows.                                                           |

After success, credentials live in the OS keyring (service `social-cli`) with a fallback at `~/.social/credentials.json` (mode `0600`). `social logout` clears both.

## Connecting a platform account

`login` only authenticates the user against the social API. Each platform needs its own connection handshake:

```bash
social linkedin connect --no-open    # Unipile hosted auth — prints the URL to share
social x connect --no-open           # X OAuth handshake — prints the URL to share
```

The CLI polls for up to 5 minutes until the connection appears in `social <platform> list`, then prints the connected handle. For X, the bearer is requested with full scopes; the bearer-session `cliGrant` decides usage scope at request time.

To swap accounts:

```bash
social linkedin list
social linkedin disconnect <username-or-public-id>
social linkedin reconnect <username-or-public-id>

social x list
social x disconnect <handle-or-id>
social x reconnect <handle-or-id>
```

## Scopes

The bearer token carries one of:

- `read` — list/get only.
- `read,write` — adds POST/PUT/DELETE proxy capabilities.

Mismatch surfaces as `scope_missing` (HTTP 403). Fix:

```bash
social logout
social login --scope read,write
```

## Per-call account selection

Every command accepts `--account <handle-or-id>`. Without it the CLI uses the default account. Use it to disambiguate when multiple accounts of the same platform are connected. Resolves against `social <platform> list`.

## Environment variables

Defaults point at production; override only for local dev or staging:

| Variable                         | Default                        | Purpose                                                                                           |
| -------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------- |
| `SOCIAL_API_URL`                 | `https://api.usesocial.dev/v1` | Versioned API base. ORPC at `${SOCIAL_API_URL}/rpc`, proxy at `${SOCIAL_API_URL}/{linkedin,x}/*`. |
| `SOCIAL_WEB_URL`                 | `https://usesocial.dev`        | Web app for device approval and OAuth landing.                                                    |
| `WSL_DISTRO_NAME`, `WSL_INTEROP` | —                              | **Auto-detected.** Do not set.                                                                    |

For local dev against the monorepo `just dev`:

```bash
export SOCIAL_API_URL=http://localhost:8787/v1
export SOCIAL_WEB_URL=http://localhost:3000
```

## Error catalog

Errors arrive on stderr with a non-zero exit. In `--json` mode they print as
`{ "error": "...", "type": "..." }`, with `status`, `body`, `issues`, or
`retryAfterSeconds` when available. Surface messages verbatim — they are precise enough
for the user to act on.

Exit codes are typed: `2` means fix command usage or local validation, `3` means not found,
`4` means login/scope/auth, `5` means API or unexpected failure, and `7` means rate
limited. On `7`, JSON errors may include `retryAfterSeconds`; back off before retrying.

| Code                                                 | Meaning                                           | Fix                                                                      |
| ---------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------ |
| `unauthenticated` / `Not signed in`                  | No bearer or expired.                             | `social login`.                                                          |
| `scope_missing`                                      | Token has `read`, command needs `write`.          | `social logout && social login --scope read,write`.                      |
| `platform_not_connected`                             | No connected account for that platform.           | `social linkedin connect` or `social x connect`.                         |
| `account_not_found`                                  | `--account` value did not match.                  | `social <platform> list`, reuse the printed handle/id.                   |
| `endpoint_not_available_in_v1`                       | Path not in the adapter's allowlist.              | Pick a different command; do not retry.                                  |
| `rate_limited`                                       | Upstream throttle hit (X, Unipile, or LinkedIn).  | Back off per the retry hint. X quotas are tight on free tiers.           |
| `invalid_argument`                                   | A flag failed parsing/validation.                 | Check `--help`; the ranges in the platform references are authoritative. |
| `no_available_seat`                                  | Org has no remaining billing seat at `connect`.   | User adds a seat in the dashboard or releases one with `disconnect`.     |
| `linkedin_connect_timed_out` / `x_connect_timed_out` | User did not approve in browser within 5 minutes. | Re-run `social <platform> connect`.                                      |
| `x_account_required`                                 | `disconnect` with multiple X accounts and no arg. | Add the handle/id.                                                       |

## Troubleshooting

| Symptom                             | Likely cause                                     | Fix                                                                          |
| ----------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------- |
| `command not found: social`         | Not installed or `$PATH` missing the global bin. | Re-run install; check `bun pm bin -g` / `npm bin -g`.                        |
| `Not signed in` / `unauthenticated` | No token or expired.                             | `social login`.                                                              |
| `scope_missing`                     | Token has `read`, command needs `write`.         | `social logout && social login --scope read,write`.                          |
| `platform_not_connected`            | Account for that platform not connected.         | `social linkedin connect` / `social x connect`.                              |
| Browser fails to open               | WSL or headless.                                 | Re-run with `--no-open`, surface the URL to the user.                        |
| Keyring write failure               | macOS Keychain locked, Linux missing libsecret.  | Falls back to `~/.social/credentials.json` automatically; check permissions. |

## `social schema`

The authoritative machine-readable command tree — use it when uncertain about a flag, a subtree, or whether something exists at all. Faster and cheaper than guessing:

```bash
social schema --json | jq '.subCommands | keys'
social schema --json | jq '.subCommands.linkedin.subCommands | keys'
social schema --json | jq '.subCommands.x.subCommands.tweets.subCommands'
```
