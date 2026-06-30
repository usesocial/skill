# Account setup and administration

Use this reference for the `social` skill's account/admin surface: install,
login, logout, billing, usage, logs, updates, troubleshooting, and feedback.
Use `/linkedin` for LinkedIn connection and LinkedIn operations.

## Install the CLI

Run the hosted setup command in an interactive terminal:

```bash
curl -fsSL https://usesocial.dev/install.sh | bash
```

It prefers Bun, then falls back to npm. It installs the public plugin and starts
`social account login`. The package publishes the `social` binary (ESM, Node >=
22.5). If the binary is missing after install, surface the install log; it is
usually a permissions error on the global prefix.

Homebrew is also supported:

```bash
brew install usesocial/tap/cli
```

Homebrew installs the same binary; plugin install and login are manual.

## Staying current

Two moving parts can drift: the `social` binary and the installed plugin. Check
without auth, provider calls, or usage cost:

```bash
social update
```

It prints `{ "cli": ..., "skill": ... }` JSON with `status` and
`updateCommand` fields. Never update silently; surface the suggested command and
ask first.

Update the CLI with the package manager the user used:

| Installed via | Update command |
| --- | --- |
| Bun | `bun install -g @usesocial/cli@latest` |
| npm | `npm install -g @usesocial/cli@latest` |
| Homebrew | `brew upgrade usesocial/tap/cli` |

If the plugin was installed in the project, update it with:

```bash
bunx --bun skills update social --project --yes
```

or, for npm-only environments:

```bash
npx --yes skills update social --project --yes
```

Use `--global` instead of `--project` for a global install. Updating the plugin
markdown does not change the current session; the refresh applies the next time
skills load.

## `social account login`

`account login` runs the Better Auth device-authorization flow.

Interactive terminal: guided login. Ask the user to run it directly; do not pipe
`yes` into it.

Agent / non-TTY shell: non-blocking state machine. Each call advances or checks
one step:

```bash
social account login
```

| `.status` | Meaning | Next step |
| --- | --- | --- |
| `logged_in` | Signed in. | Continue. |
| `already_logged_in` | Existing valid session. | Continue. |
| `pending_approval` | Device flow awaiting approval. | Surface `verificationURL`; poll gently. |
| `expired` | Code expired before approval. | Re-run `login`. |
| `error` | Login failed. | Surface `.message`; stop. |

After success, credentials live in the OS keyring (`social-cli`) with a fallback
at `~/.social/credentials.json` (mode `0600`). Sessions last two years unless
revoked. `social account logout` clears the active credentials.

Use bare `social account` to inspect auth state and connected accounts. It
prints compact JSON with `status`, credential namespace/path, verified session
data when available, connected account rows, and seat counts when online.

## Billing and usage

Use account commands for account and spend audits:

```bash
social account billing
social account billing portal
social account usage
social account logs --limit 20
social account logs --platform linkedin --limit 20
```

`account billing` prints seats, subscription, and aggregate usage billing.
`account billing portal` prints a hosted portal URL as JSON. `account logs`
returns `{ items, meta: { cursor } }` and caps `--limit` at 100 rows per call.
For longer windows, page with `.meta.cursor`; for totals, prefer
`social account usage`.

## Logout

```bash
social account logout
```

Logout clears only the active API URL's credentials. Re-run `social account` to
verify status.

## Environment variables

Defaults point at production; override only for local dev or staging:

| Variable | Default | Purpose |
| --- | --- | --- |
| `SOCIAL_API_URL` | `https://api.usesocial.dev/v1` | Versioned API base. |
| `SOCIAL_WEB_URL` | `https://usesocial.dev` | Web app for device approval and browser handoffs. |
| `WSL_DISTRO_NAME`, `WSL_INTEROP` | - | Auto-detected. Do not set. |

For local dev against the monorepo `just dev`:

```bash
export SOCIAL_API_URL=http://localhost:8787/v1
export SOCIAL_WEB_URL=http://localhost:3000
```

## Error catalog

Errors arrive on stderr with a non-zero exit. JSON-only command surfaces print
`{ "error": "...", "type": "..." }`, with `status`, `body`, `issues`, or
`retryAfterSeconds` when available. Surface messages verbatim.

| Code | Meaning | Fix |
| --- | --- | --- |
| `unauthenticated` / `Not signed in` | No bearer or expired. | `social account login`. |
| `scope_missing` | Token has `read`, command needs `write`. | `social account logout`, then `social account login` and choose Read + Write. |
| `account_not_found` | `--account` value did not match. | Run `social account`, reuse the printed username/id. |
| `rate_limited` | Upstream throttle hit. | Back off; use retry JSON when present. |
| `billing_seat_timed_out` | Seat bump/payment action did not complete. | Finish the printed billing URL, then retry the operation. |
| `command not found: social` | Not installed or `$PATH` missing the global bin. | Re-run install; check `bun pm bin -g` / `npm bin -g`. |

## `social schema`

Use the schema command when uncertain about CLI structure:

```bash
social schema | jq '.subCommands | keys'
social schema "<command path>"
```

The schema is especially useful before writing feedback or debugging why a
command is unavailable.
