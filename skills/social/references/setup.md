# Setup & Auth

How to get `social` working from a fresh machine and how to recover from auth failures. Shared across LinkedIn and X — only the per-platform `connect` handshake differs.

## Install the CLI

Run the hosted setup command in an interactive terminal:

```bash
curl -fsSL https://usesocial.dev/install.sh | bash
```

It prefers Bun, then falls back to npm. It installs the public skill with `bunx skills add usesocial/skill` or `npx skills add usesocial/skill`, then starts `social account login`. The package publishes the `social` binary (ESM, Node 24). If the binary is missing after install, surface the install log — usually a permissions error on the global prefix.

## Staying current

Two moving parts can drift: the `social` binary and this skill. Keep both fresh, but never update silently — let the user decide before changing installed tools.

Update the CLI with the package manager the user actually used:

| Installed via | Update command                      |
| ------------- | ----------------------------------- |
| Bun           | `bun add -g @usesocial/cli@latest`  |
| npm           | `npm install -g @usesocial/cli@latest` |

Do **not** infer the manager by resolving `which social` symlinks — it is brittle across `~/.bun/bin` and npm prefixes. Updating the CLI mid-session is safe: each call is a fresh process, so the new binary takes effect on the next command.

If the skill was installed with `npx skills`, update it with:

```bash
npx skills update social
```

Updating the skill markdown does **not** change the current session — the old text is already loaded in context; the refresh takes effect the next time the skill loads.

Verify:

```bash
social --version
social --help
```

## `social account login`

`account login` runs the better-auth **device-authorization** flow. It is fully interactive: choose Read or Read + Write, enter email, click the emailed magic link, approve the browser request, and confirm billing checkout if a seat is needed. The CLI sends the magic link with the device approval screen attached, then polls until the web session approves the request. **Do not background it; do not pipe `yes` into it; do not run it from an agent-mediated setup flow.** Ask the user to run it directly in an interactive terminal.

After success, credentials live in the OS keyring (service `social-cli`) with a fallback at `~/.social/credentials.json` (mode `0600`). `social account logout` clears both.

Use bare `social account` to inspect auth state and connected accounts. It always prints compact JSON with `status`, credential namespace/path, verified session data when available, connected account rows, and seat counts when the session is online.

## Connecting a platform account

`account login` only authenticates the user against the social API. Each platform needs its own connection handshake:

```bash
social account connect linkedin    # LinkedIn browser connection
social account connect x           # X OAuth handshake
```

The CLI opens the browser when run from an interactive terminal. In an agent/non-TTY run, it prints the connection URL to stderr so the user can open it themselves. It polls until the connection appears in bare `social account`, then prints the connected handle. LinkedIn connect/reconnect times out after 2 minutes; X connect/reconnect times out after 5 minutes. For X, the bearer is requested with full scopes; the bearer-session `cliGrant` decides usage scope at request time.

To swap accounts:

```bash
social account
social account disconnect linkedin <@handle|profile_id:<id>>
social account reconnect linkedin <@handle|profile_id:<id>>

social account
social account disconnect x <@handle|profile_id:<id>>
social account reconnect x <@handle|profile_id:<id>>
```

## Scopes

The bearer token carries one of:

- `read` — list/get only.
- `read,write` — adds POST/PUT/DELETE proxy capabilities.

Mismatch surfaces as `scope_missing` (HTTP 403). Fix:

```bash
social account logout
social account login
```

Choose Read + Write in the login prompt.

## Per-call account selection

Every command accepts `--account <@handle|profile_id:<id>>`. Without it the CLI uses the default account. Use it to disambiguate when multiple accounts of the same platform are connected. Resolves against bare `social account`.

## Caching

Allowlisted GET reads use the proxy cache by default. Cache hits are free because
they skip the upstream provider call. Fresh upstream reads and writes are still
metered.

The default cache TTL is 15 minutes. Configure the local default in seconds:

```bash
social account config cache ttl {total_in_seconds}
```

Useful presets:

```bash
social account config cache mode live         # 15 minutes, default
social account config cache mode analytical   # 24 hours
social account config cache mode historical   # 1 week
```

Use command-level cache headers only when command help lists `--header`:

```bash
social linkedin profile @handle -H "Cache-Control: no-cache"   # bypass cache read, refresh cache
social linkedin profile @handle -H "Cache-Control: no-store"   # bypass cache read and write
social linkedin profile @handle -H "Cache-Control: max-age=60" # override TTL for this request
```

`Cache-Control` is the functional request cache surface. Cached responses may
preserve validators such as `ETag` and `Last-Modified` when the upstream returns
them, but the CLI does not expose conditional revalidation semantics; use
`Cache-Control: no-cache` when you need a fresh upstream read.

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

Errors arrive on stderr with a non-zero exit. JSON-only command surfaces print
`{ "error": "...", "type": "..." }`, with `status`, `body`, `issues`, or
`retryAfterSeconds` when available. Surface messages verbatim — they are precise enough
for the user to act on.

Exit codes are typed: `2` means fix command usage or local validation, `3` means not found,
`4` means login/scope/auth, `5` means API or unexpected failure, and `7` means rate
limited. On `7`, JSON errors may include `retryAfterSeconds`; back off before retrying.

| Code                                                 | Meaning                                           | Fix                                                                      |
| ---------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------ |
| `unauthenticated` / `Not signed in`                  | No bearer or expired.                             | `social account login`.                                                  |
| `scope_missing`                                      | Token has `read`, command needs `write`.          | `social account logout`, then `social account login` and choose Read + Write. |
| `platform_not_connected`                             | No connected account for that platform.           | `social account connect linkedin` or `social account connect x`.         |
| `account_not_found`                                  | `--account` value did not match.                  | `social account`, reuse the printed handle/id.                           |
| `endpoint_not_available_in_v1`                       | Path not in the adapter's allowlist.              | Pick a different command; do not retry.                                  |
| `rate_limited`                                       | Upstream throttle hit.                            | Back off per the retry hint. X quotas are tight on free tiers.           |
| `invalid_argument`                                   | A flag failed parsing/validation.                 | Check `--help`; the ranges in the platform references are authoritative. |
| `billing_seat_timed_out`                             | Seat bump/payment action did not complete.        | Finish the opened billing URL, then re-run `social account connect <platform>`. |
| `no_available_seat`                                  | Legacy/direct API path has no remaining seat.     | Re-run CLI `connect` or add a seat in the dashboard.                     |
| `linkedin_connect_timed_out` / `x_connect_timed_out` | User did not approve in browser within the platform timeout. | Re-run `social account connect <platform>`.                              |
| `Missing required positional argument: ACCOUNT`       | `disconnect` or `reconnect` is missing an account. | Add the handle/id.                                                       |

## Troubleshooting

| Symptom                             | Likely cause                                     | Fix                                                                          |
| ----------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------- |
| `command not found: social`         | Not installed or `$PATH` missing the global bin. | Re-run install; check `bun pm bin -g` / `npm bin -g`.                        |
| `Not signed in` / `unauthenticated` | No token or expired.                             | `social account login`.                                                      |
| `scope_missing`                     | Token has `read`, command needs `write`.         | `social account logout`, then `social account login` and choose Read + Write. |
| `platform_not_connected`            | Account for that platform not connected.         | `social account connect linkedin` / `social account connect x`.              |
| Browser fails to open               | WSL or headless.                                 | Re-run from the agent/non-TTY context and surface the printed URL to the user. |
| Keyring write failure               | macOS Keychain locked, Linux missing libsecret.  | Falls back to `~/.social/credentials.json` automatically; check permissions. |

## `social schema`

The authoritative machine-readable command tree — use it when uncertain about a flag, a subtree, or whether something exists at all. Faster and cheaper than guessing:

```bash
social schema | jq '.subCommands | keys'
social schema | jq '.subCommands.linkedin.subCommands | keys'
social schema | jq '.subCommands.x.subCommands | keys'
```
