# Setup & Auth

How to get `social` working from a fresh machine and how to recover from auth failures. Shared across LinkedIn and X — only the per-platform `connect` handshake differs.

## Install the CLI

Prefer Bun, fall back to npm:

```bash
bun install -g @usesocial/cli@latest
# or
npm install -g @usesocial/cli
```

The package publishes the `social` binary (ESM, Node 24). If the binary is missing after install, surface the install log — usually a permissions error on the global prefix.

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
| `SOCIAL_API_URL`                 | `https://api.socialcli.dev/v1` | Versioned API base. ORPC at `${SOCIAL_API_URL}/rpc`, proxy at `${SOCIAL_API_URL}/{linkedin,x}/*`. |
| `SOCIAL_WEB_URL`                 | `https://socialcli.dev`        | Web app for device approval and OAuth landing.                                                    |
| `WSL_DISTRO_NAME`, `WSL_INTEROP` | —                              | **Auto-detected.** Do not set.                                                                    |

For local dev against the monorepo `just dev`:

```bash
export SOCIAL_API_URL=http://localhost:8787/v1
export SOCIAL_WEB_URL=http://localhost:3000
```

## Error catalog

Errors arrive on stderr with a non-zero exit. In `--json` mode they print as `{ "status": "error", "code": "...", "message": "..." }`. Surface messages verbatim — they are precise enough for the user to act on.

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
