# Setup & Auth

How to get `social` working from a fresh machine and how to recover from auth failures. Shared across LinkedIn, Instagram, and X — only the per-platform `connect` handshake differs.

## Install the CLI

Run the hosted setup command in an interactive terminal:

```bash
curl -fsSL https://usesocial.dev/install.sh | bash
```

It prefers Bun, then falls back to npm. It installs the public skill with `bunx skills add usesocial/skill` or `npx skills add usesocial/skill`, then starts `social account setup`. The package publishes the `social` binary (ESM, Node >= 22.5). If the binary is missing after install, surface the install log — usually a permissions error on the global prefix.

Homebrew is also supported: `brew install usesocial/tap/cli` installs the same binary (skill and login are then manual).

## Staying current

Two moving parts can drift: the `social` binary and this skill. Keep both fresh, but never update silently — let the user decide before changing installed tools.

Run a fresh check:

```bash
social update
```

It prints `{ "cli": ..., "skill": ... }` JSON with `status` and
`updateCommand` fields. The command is local-only: no auth, no provider calls,
and no usage cost. Ordinary `social` commands may also show a concise update
notice on stderr in a human terminal. CI, non-TTY shells, `DO_NOT_TRACK`, and
`SOCIAL_DO_NOT_TRACK` suppress those notices, and stdout remains JSON-only.

Update the CLI with the package manager the user actually used:

| Installed via | Update command                      |
| ------------- | ----------------------------------- |
| Bun           | `bun install -g @usesocial/cli@latest` |
| npm           | `npm install -g @usesocial/cli@latest` |
| Homebrew      | `brew upgrade usesocial/tap/cli`    |

If the skill was installed globally, update it with:

```bash
bunx --bun skills update social --global --yes
```

or, for npm-only environments:

```bash
npx --yes skills update social --global --yes
```

Use `--project` instead of `--global` only for a project-local skill. Updating the skill
markdown does **not** change the current session — the old text is already
loaded in context; the refresh takes effect the next time the skill loads.

Verify:

```bash
social --version
social --help
```

## `social account setup`

`account setup` completes Social authentication. Its
behavior depends on the shell:

**Interactive terminal.** Fully guided authentication followed by the same
readiness check used by non-interactive CLI and MCP clients.

**Agent / non-TTY shell.** A non-blocking **state machine** that advances one step
per call - a skill can poll it. It never prompts and never blocks waiting for the
human. The states (read `.status`, not the exit code):

| `.status`           | Meaning                                            | Next step |
| ------------------- | -------------------------------------------------- | --------- |
| `pending_approval`  | Device flow started; awaiting browser approval.    | Surface `verificationURL`; call `setup` again to poll. |
| `pending_billing`   | A reusable payment method is not ready.            | Surface `checkoutURL`; call `setup` again after card authorization. |
| `ready`             | Authentication and card setup are complete.        | Continue to provider connection. |
| `expired`           | The device code lapsed before approval (exit `4`). | Re-run `setup` to issue a fresh code. |
| `needs_input`       | Interactive input was required but unavailable.    | Surface `.reason`; rerun appropriately. |
| `error`             | Surface `.message`; stop.                          | |

The first call returns `pending_approval` with `verificationURL`, `userCode`,
`expiresAt`, and `scope`. The human opens `verificationURL`, where the browser
collects their email and approves the session - the CLI never asks the agent for
an email, phone, magic link, or bearer. The setup grant defaults to
`read,write`. The in-flight device code is persisted to
`~/.social/device-login.json` (mode `0600`) between calls and removed once the
flow resolves. Phone capture is interactive-only and best-effort. Once
authenticated, setup calls Autumn `setupPayment`. It returns `pending_billing`
with `checkoutURL` until the user authorizes a reusable card; the checkout never
selects, attaches, or purchases a provider plan. Once the payment method is
ready, setup returns the backend's safe user, normalized scope, exact
capabilities, and provider connection choices.
**Do not background `setup`, pipe `yes` into it, or poll it without a cap.**
The full onboarding walk-through is in
`references/get-started.md`.

After success, credentials live in the OS keyring (service `social-cli`) with a fallback at `~/.social/credentials.json` (mode `0600`). Sessions last two years unless revoked. `social account logout` clears both.

Use bare `social account` to inspect auth state and connected accounts. It always prints compact JSON with `status`, credential namespace/path, verified session data when available, connected account rows, and seat counts when the session is online.

Use `social account billing` for the Social LinkedIn and Social X subscription,
connected-account, and X credit snapshot. Use `social account billing portal`
to print the hosted billing portal URL; it prints `{ "url": "...", "opened":
false }`, so agents can hand the URL to the user.

## Connecting a platform account

`account setup` authenticates the user and authorizes a reusable payment method.
Each platform connect command then creates or resumes its own partial
`connected_accounts` row:

```bash
social account connect linkedin    # LinkedIn connection URL
social account connect x           # X OAuth handshake
```

Like setup, agent/non-TTY connect is a non-blocking **state machine** - one step
per call, no waiting:

| `.status`          | Meaning                              | Next step |
| ------------------ | ------------------------------------ | --------- |
| `pending_billing`  | The selected provider seat needs billing. | Surface `paymentURL` only when present; retry with the returned `accountId`. |
| `pending_approval` | No account linked yet.               | Surface `connectURL`; retry with the returned `accountId`. |
| `connected`        | Account is linked (`.account`).      | Done; exact-ID replay returns this result. |

Terminal connection responses include a machine-readable `.code` and `.retryable`;
surface the code and never silently start another purchase.

The first call without `--account-id` resumes an active unexpired partial
account or creates one, purchasing only the selected provider's plan or next
seat, or the applicable legacy Social Pro seat during transition. It normally
charges the payment method established during setup and returns
`{ status: "pending_billing", accountId, platform, paymentURL }` only when the
bank requires customer action. Capture `accountId`, surface `paymentURL`, have
the user approve the payment, then call
`social account connect <platform> --account-id <accountId>`. Social LinkedIn costs $20
per connected LinkedIn account per month. Every LinkedIn read, sync, and write is
covered by that seat, with no credits, top-ups, or per-action charges. Social X
costs $20 per connected X account per month, includes 15,000
credits with one-month rollover, and offers $15 top-ups for another 15,000
credits. Both subscriptions can coexist. Once billing is ready, connect returns
`{ status: "pending_approval", accountId, platform, connectURL }` and prints
`Open this URL: <url>`; surface `connectURL` to the user, have them approve in
the browser/profile they want to use, then call `connect` again with the same
ID. Replaying a connected ID is stable; omit `--account-id` after completion only
to connect another account. Interactive
connect prints the same URL and polls until the connection appears in bare
`social account`. Once the partial row is complete it returns
`{ status: "connected", accountId, platform, account }`. Bare `social account`
also shows the connected-account row. `reconnect` remains the interactive
blocking flow, carries no new-account ID, and does not change billing.
LinkedIn uses Unipile hosted auth; web connect lives at `/connect/linkedin`. For
X, the bearer is requested with full scopes; the bearer-session `cliGrant`
decides usage scope at request time.

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

`read,write` already covers LinkedIn Page invites and LinkedIn raw proxy writes;
there is no separate Page scope.

Mismatch surfaces as `scope_missing` (HTTP 403). Fix:

```bash
social account logout
social account setup
```

Choose Read + Write in the login prompt.

## Per-call account selection

Every command accepts `--account <@username|profile_id:<id>>`. Without it the CLI uses that platform's shared server default. Use `social account default get <linkedin|x>` to inspect it and `social account default set <linkedin|x> <selector>` to change it across CLI and MCP. Use `--account` to override it for one call. Selectors resolve against bare `social account`.

## Caching

Allowlisted GET reads use the proxy cache by default. X cache hits spend no
credits because they skip the upstream provider call. LinkedIn reads have no
per-action charge whether they hit the cache or the upstream provider.

The default cache TTL is 15 minutes. Configure the local default in seconds:

```bash
social account config cache ttl {total_in_seconds}
```

Use command-level cache headers only when command help lists `--header`:

```bash
social linkedin profile @username -H "Cache-Control: no-cache"   # bypass cache read, refresh cache
social linkedin profile @username -H "Cache-Control: no-store"   # bypass cache read and write
social linkedin profile @username -H "Cache-Control: max-age=60" # override TTL for this request
social instagram profile @username -H "Cache-Control: no-cache"
```

`Cache-Control` is the functional request cache surface. Cached responses may
preserve validators such as `ETag` and `Last-Modified` when the upstream returns
them, but the CLI does not expose conditional revalidation semantics; use
`Cache-Control: no-cache` when you need a fresh upstream read.

## Environment variables

Defaults point at production; override only for local dev or staging:

| Variable                         | Default                        | Purpose                                                                                           |
| -------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------- |
| `SOCIAL_API_URL`                 | `https://api.usesocial.dev/v1` | Versioned API base. ORPC at `${SOCIAL_API_URL}/rpc`, proxy at `${SOCIAL_API_URL}/{linkedin,instagram,x}/*`. |
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
| `unauthenticated` / `Not signed in`                  | No bearer or expired.                             | `social account setup`.                                                  |
| `scope_missing`                                      | Token has `read`, command needs `write`.          | `social account logout`, then `social account setup` and choose Read + Write. |
| `platform_not_connected`                             | No connected account for that platform.           | `social account connect linkedin` or `social account connect x`. |
| `account_not_found`                                  | `--account` value did not match.                  | `social account`, reuse the printed username/id.                           |
| `endpoint_not_available_in_v1`                       | Path not in the adapter's allowlist.              | Pick a different command; do not retry.                                  |
| `rate_limited`                                       | Upstream throttle hit.                            | LinkedIn retries short waits automatically; long waits exit `7` with resume guidance — re-run `retryCommand` after `resumeAt`. X quotas are tight on free tiers. |
| `invalid_argument`                                   | A flag failed parsing/validation.                 | Check `--help`; the ranges in the platform references are authoritative. |
| `billing_seat_timed_out`                             | Seat purchase/payment action did not complete.    | Finish the printed billing URL, then resume with `social account connect <platform> --account-id <accountId>`. |
| `no_available_seat`                                  | Legacy/direct API path has no remaining seat.     | Re-run CLI `connect` or add a seat in the dashboard.                     |
| `linkedin_connect_timed_out` / `x_connect_timed_out` | User did not approve in browser within the platform timeout. | Follow the terminal connection's `retryable` guidance; do not silently start another purchase. |
| `Missing required positional argument: ACCOUNT`       | `disconnect` or `reconnect` is missing an account. | Add the username/id.                                                       |

## Troubleshooting

| Symptom                             | Likely cause                                     | Fix                                                                          |
| ----------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------- |
| `command not found: social`         | Not installed or `$PATH` missing the global bin. | Re-run install; check `bun pm bin -g` / `npm bin -g`.                        |
| `Not signed in` / `unauthenticated` | No token or expired.                             | `social account setup`.                                                      |
| `scope_missing`                     | Token has `read`, command needs `write`.         | `social account logout`, then `social account setup` and choose Read + Write. |
| `platform_not_connected`            | Account for that platform not connected.         | `social account connect linkedin` / `social account connect x`. |
| Browser fails to open               | WSL or headless.                                 | Re-run from the agent/non-TTY context and surface the printed URL to the user. |
| Keyring write failure               | macOS Keychain locked, Linux missing libsecret.  | Falls back to `~/.social/credentials.json` automatically; check permissions. |

## `social schema`

The authoritative machine-readable command tree — use it when uncertain about a flag, a subtree, or whether something exists at all. Faster and cheaper than guessing:

```bash
social schema | jq '.subCommands | keys'
social schema | jq '.subCommands.linkedin.subCommands | keys'
social schema | jq '.subCommands.instagram.subCommands | keys'
social schema | jq '.subCommands.x.subCommands | keys'
```
