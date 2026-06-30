# Social agent plugin

Agent skills for the [`@usesocial/cli`](https://www.npmjs.com/package/@usesocial/cli) CLI.
The plugin keeps `/social` for account administration and makes `/linkedin` the
primary skill for LinkedIn work: connecting an account, researching, posting,
replying, messaging, managing requests, syncing local data, and auditing spend.

## What ships

Two skills ship in one plugin:

- **`skills/social/SKILL.md`** -> `/social` - install, login, logout, billing,
  usage, logs, update checks, setup troubleshooting, and feedback.
- **`skills/social/references/setup.md`** - account/admin setup, login, billing,
  logs, updates, errors, and troubleshooting.
- **`skills/linkedin/SKILL.md`** -> `/linkedin` - LinkedIn connection and
  LinkedIn operation through `social linkedin ...`.
- **`skills/linkedin/references/get-started.md`** - guided LinkedIn onboarding:
  install check, login, connect LinkedIn, and first sync with explicit cost
  consent.
- **`skills/linkedin/references/linkedin.md`** - full LinkedIn command catalog,
  flags, output shapes, `jq` recipes, and end-to-end playbooks.

The plugin remains named `social` so existing install and update commands keep
working. LinkedIn is the primary operator skill.

## Text input

All freeform text for LinkedIn posts, comments, messages, message edits, and
connection request notes is sent via stdin. Do not pass text as a positional
argument:

```sh
echo "Thanks for the note." | social linkedin message @username
social linkedin post < draft.md
```

Targets, typed IDs, URLs, and usernames still go on argv. Pipe a JSON object via
stdin for advanced structured payloads.

Feedback reports are stdin-only too:

```sh
echo "..." | social feedback bug
echo "..." | social feedback feature
```

For bugs, agents should gather safe reproduction context before submitting. For
features, agents should understand the user's job to be done, current workaround,
intent, constraints, and examples so the report is useful founder feedback.

## Install

Install with `npx skills`:

```sh
npx skills add usesocial/skill
```

For local development:

```sh
npx skills add ./skill
```

The repo also carries Claude and Codex plugin manifests for agents that can
consume plugin directories directly.

## Prerequisites

The plugin expects the `social` binary on `PATH`. If it is missing, ask the user
before installing global tools, then have them run the setup command in an
interactive terminal:

```sh
curl -fsSL https://usesocial.dev/install.sh | bash
```

Keep the plugin current with `npx skills update social`. Re-run the relevant
install command when the CLI needs an update.

## Configuration

Env vars are optional; defaults point at the hosted API:

| Variable | Default | Purpose |
|---|---|---|
| `SOCIAL_API_URL` | `https://api.usesocial.dev/v1` | API base URL |
| `SOCIAL_WEB_URL` | `https://usesocial.dev` | Device-approval web URL |

For local development, point both at localhost. See
`skills/social/references/setup.md` for account/admin setup and
`skills/linkedin/references/get-started.md` for LinkedIn connection.
