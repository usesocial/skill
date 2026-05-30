# Setup & Auth — LinkedIn

How to get `social linkedin` working from a fresh machine and how to recover from auth failures.

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

`login` runs the better-auth **device-authorization** flow. It is interactive — the CLI prints a verification URL and a user code, opens the browser to `${SOCIAL_WEB_URL}/device`, and polls until the web session approves the request. **Do not background it; do not pipe `yes` into it.** Either run it in a separate terminal the user controls, or invoke as a foreground Bash command and surface the verification URL/code so the user can approve.

Flags:

| Flag | Purpose |
|---|---|
| `--email <addr>` | Skip the email prompt. |
| `--scope read` \| `read,write` | Pick capability bundle. `read` is safer; `read,write` allows future write endpoints. Default `read`. |
| `--accept-pricing` | Pre-accept the seat checkout if needed. |
| `--skill-target none\|manual\|<path>` | Where to install the legacy agent skill (use `none` from this plugin — would conflict). |
| `--no-open` | Print the URL instead of opening a browser (use under WSL/SSH/inside-agent). |
| `--non-interactive` | Disable prompts; emit `{ status: "needs_input", missing: [...] }` JSON if anything is missing. |
| `--json` / `--pretty` | Machine output. Pair with `--non-interactive` for scripted flows. |

After success, credentials live in the OS keyring (service `social-cli`) with a fallback at `~/.social/credentials.json` (mode `0600`). `social logout` clears both.

## `social linkedin connect`

`login` only authenticates the user against the social API. LinkedIn needs a separate **Unipile hosted-auth** handshake:

```bash
social linkedin connect --no-open    # prints the URL to share with the user
```

The CLI polls for up to 5 minutes until the connection appears in `social linkedin list`, then prints the connected handle.

To swap accounts:

```bash
social linkedin list
social linkedin disconnect <username-or-public-id>
social linkedin reconnect <username-or-public-id>
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

Every command accepts `--account <handle-or-public-id>`. Without it the CLI uses the default account. Use it to disambiguate when multiple LinkedIn accounts are connected.

## Environment variables

Defaults point at production; override only for local dev or staging:

| Variable | Default | Purpose |
|---|---|---|
| `SOCIAL_API_URL` | `https://api.socialcli.dev/v1` | Versioned API base. ORPC at `${SOCIAL_API_URL}/rpc`, proxy at `${SOCIAL_API_URL}/linkedin/*`. |
| `SOCIAL_WEB_URL` | `https://socialcli.dev` | Web app for device approval and OAuth landing. |
| `WSL_DISTRO_NAME`, `WSL_INTEROP` | — | **Auto-detected.** Do not set. |

For local dev against `just dev`:

```bash
export SOCIAL_API_URL=http://localhost:8787/v1
export SOCIAL_WEB_URL=http://localhost:3000
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `command not found: social` | Not installed or `$PATH` missing the global bin. | Re-run install; check `bun pm bin -g` / `npm bin -g`. |
| `Not signed in` / `unauthenticated` | No token or expired. | `social login`. |
| `scope_missing` | Token has `read`, command needs `write`. | `social logout && social login --scope read,write`. |
| `platform_not_connected` | LinkedIn account not connected. | `social linkedin connect`. |
| `endpoint_not_available_in_v1` | Adapter does not route this Unipile path. | Pick a different command; check `social schema`. |
| `linkedin_connect_timed_out` | User did not approve within 5 minutes. | Re-run `social linkedin connect`. |
| Browser fails to open | WSL or headless. | Re-run with `--no-open`, surface URL to user. |
| Keyring write failure | macOS Keychain locked, Linux missing libsecret. | Falls back to `~/.social/credentials.json` automatically; check permissions. |
