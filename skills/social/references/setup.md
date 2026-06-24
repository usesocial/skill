# Setup & Auth

How to get `social` working from a fresh machine and how to recover from auth failures. Shared across LinkedIn and X — only the per-platform `connect` handshake differs.

## Install the CLI

Run the hosted setup command in an interactive terminal:

```bash
curl -fsSL https://usesocial.dev/install.sh | bash
```

It prefers Bun, then falls back to npm. It installs the public skill with `bunx skills add usesocial/skill` or `npx skills add usesocial/skill`, then starts `social account login`. The package publishes the `social` binary (ESM, Node >= 22.5). If the binary is missing after install, surface the install log — usually a permissions error on the global prefix.

Homebrew is also supported: `brew install usesocial/tap/cli` installs the same binary (skill and login are then manual).

## Staying current

Two moving parts can drift: the `social` binary and this skill. Keep both fresh, but never update silently — let the user decide before changing installed tools.

Update the CLI with the package manager the user actually used:

| Installed via | Update command                      |
| ------------- | ----------------------------------- |
| Bun           | `bun add -g @usesocial/cli@latest`  |
| npm           | `npm install -g @usesocial/cli@latest` |
| Homebrew      | `brew upgrade usesocial/tap/cli`    |

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

`account login` runs the better-auth **device-authorization** flow. Its behavior
depends on the shell:

**Interactive terminal.** Fully guided: choose Read or Read + Write, enter email
and phone, click the emailed magic link, approve the browser request, and confirm
billing checkout if a seat is needed. The CLI sends the magic link with the
device approval screen attached, then blocks until the web session approves. Ask
the user to run it directly; do not pipe `yes` into it.

**Agent / non-TTY shell.** A non-blocking **state machine** that advances one step
per call - a skill can poll it. It never prompts and never blocks waiting for the
human. The states (read `.status`, not the exit code):

| `.status`           | Meaning                                            | Next step |
| ------------------- | -------------------------------------------------- | --------- |
| `pending_approval`  | Device flow started; awaiting browser approval.    | Surface `verificationURL` to the user; call `login` again to poll. |
| `logged_in`         | Approved; credentials stored.                      | Continue to connect. |
| `already_logged_in` | A valid session already exists.                    | Continue. |
| `expired`           | The device code lapsed before approval (exit `4`). | Re-run `login` to issue a fresh code. |
| `error`             | Surface `.message`; stop.                          | |

The first call returns `pending_approval` with `verificationURL`, `userCode`,
`expiresAt`, and `scope`. The human opens `verificationURL`, where the browser
collects their email and approves the session - the CLI never asks the agent for
an email, phone, magic link, or bearer. Default scope is `read,write`; pass
`--scope read` for read-only. The in-flight device code is persisted to
`~/.social/device-login.json` (mode `0600`) between calls and removed once the
flow resolves. Phone capture is interactive-only and best-effort. Billing-seat
acquisition is a separate concern (surfaced by `social account` seats), not part
of the non-TTY login states. **Do not background `login`, pipe `yes` into it, or
poll it without a cap.** The full onboarding walk-through is in
`references/get-started.md`.

After success, credentials live in the OS keyring (service `social-cli`) with a fallback at `~/.social/credentials.json` (mode `0600`). Sessions last two years unless revoked. `social account logout` clears both.

Use bare `social account` to inspect auth state and connected accounts. It always prints compact JSON with `status`, credential namespace/path, verified session data when available, connected account rows, and seat counts when the session is online.

Use `social account billing` for the current seat, subscription, and usage-billing snapshot. Use `social account billing portal` to print the hosted billing portal URL; it prints `{ "url": "...", "opened": false }`, so agents can hand the URL to the user.

## Connecting a platform account

`account login` only authenticates the user against the social API. Each platform needs its own connection handshake:

```bash
social account connect linkedin    # LinkedIn connection URL
social account connect x           # X OAuth handshake
```

Like login, agent/non-TTY connect is a non-blocking **state machine** - one step
per call, no waiting:

| `.status`          | Meaning                              | Next step |
| ------------------ | ------------------------------------ | --------- |
| `pending_approval` | No account linked yet.               | Surface `connectURL`; call `connect` again to poll. |
| `connected`        | Account is linked (`.account`).      | Done. |

The first call returns `{ status: "pending_approval", platform, connectURL }` and
prints `Open this URL: <url>`; surface `connectURL` to the user, have them
approve in the browser/profile they want to use, then call `connect` again.
Interactive connect prints the same URL and polls until the connection appears in
bare `social account`. Once the account appears it returns
`{ status: "connected", platform, account }`. Bare `social account` also shows the
connected-account row. `reconnect` remains the interactive blocking flow. For X,
the bearer is requested with full scopes; the bearer-session `cliGrant` decides
usage scope at request time.

To swap accounts:

```bash
social account
social account disconnect linkedin <@username|profile_id:<id>>
social account reconnect linkedin <@username|profile_id:<id>>

social account
social account disconnect x <@username|profile_id:<id>>
social account reconnect x <@username|profile_id:<id>>
```

## Scopes

The bearer token carries one of:

- `read` — list/get only.
- `read,write` — adds POST/PUT/DELETE proxy capabilities.

`read,write` already covers LinkedIn Page invites and raw proxy writes; there is no separate Page scope.

Mismatch surfaces as `scope_missing` (HTTP 403). Fix:

```bash
social account logout
social account login
```

Choose Read + Write in the login prompt.

## Per-call account selection

Every command accepts `--account <@username|profile_id:<id>>`. Without it the CLI uses the default account. Use it to disambiguate when multiple accounts of the same platform are connected. Resolves against bare `social account`.

## Caching

Allowlisted GET reads use the proxy cache by default. Cache hits are free because
they skip the upstream provider call. Fresh upstream reads and writes are still
metered.

The default cache TTL is 15 minutes. Configure the local default in seconds:

```bash
social account config cache ttl {total_in_seconds}
```

Use command-level cache headers only when command help lists `--header`:

```bash
social linkedin profile @username -H "Cache-Control: no-cache"   # bypass cache read, refresh cache
social linkedin profile @username -H "Cache-Control: no-store"   # bypass cache read and write
social linkedin profile @username -H "Cache-Control: max-age=60" # override TTL for this request
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

Exit codes are typed: `2` means fix command usage, flags, IDs, JSON body, or
local input. Server-side API/proxy failures exit `5`. `3` means not found, `4`
means login/scope/auth, and `7` means rate limited. LinkedIn proxy requests
retry short waits automatically, honoring `Retry-After` first and then
exponential fallback; when the wait exceeds the in-process budget (60 seconds,
or `--timeout <seconds>` on sync) the command exits `7` immediately instead of
sleeping. JSON may include `retryAfterSeconds`. Sync rate-limit errors may also
include `resumeAt`, `retryCommand`, `hint`, and `syncResume`. When
`syncResume.cursorPersisted` is true, re-run `retryCommand` after `resumeAt`;
already-synced pages are saved and the sync resumes from the saved cursor. To
size a rate-limit window, `social account logs --platform <platform> --limit 20`
shows recent upstream calls with status and usage — a run of `429`s marks the
window.

| Code                                                 | Meaning                                           | Fix                                                                      |
| ---------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------ |
| `unauthenticated` / `Not signed in`                  | No bearer or expired.                             | `social account login`.                                                  |
| `scope_missing`                                      | Token has `read`, command needs `write`.          | `social account logout`, then `social account login` and choose Read + Write. |
| `platform_not_connected`                             | No connected account for that platform.           | `social account connect linkedin` or `social account connect x`.         |
| `account_not_found`                                  | `--account` value did not match.                  | `social account`, reuse the printed username/id.                           |
| `endpoint_not_available_in_v1`                       | Path not in the adapter's allowlist.              | Pick a different command; do not retry.                                  |
| `rate_limited`                                       | Upstream throttle hit.                            | LinkedIn retries short waits automatically; long waits exit `7` with resume guidance — re-run `retryCommand` after `resumeAt`. X quotas are tight on free tiers. |
| `invalid_argument`                                   | A flag failed parsing/validation.                 | Check `--help`; the ranges in the platform references are authoritative. |
| `billing_seat_timed_out`                             | Seat bump/payment action did not complete.        | Finish the printed billing URL, then re-run `social account connect <platform>`. |
| `no_available_seat`                                  | Legacy/direct API path has no remaining seat.     | Re-run CLI `connect` or add a seat in the dashboard.                     |
| `linkedin_connect_timed_out` / `x_connect_timed_out` | User did not approve in browser within the platform timeout. | Re-run `social account connect <platform>`.                              |
| `Missing required positional argument: ACCOUNT`       | `disconnect` or `reconnect` is missing an account. | Add the username/id.                                                       |

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
