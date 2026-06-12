# Social agent skill

An agent skill for the [`@usesocial/cli`](https://www.npmjs.com/package/@usesocial/cli) CLI. It teaches supported agents when and how to call `social` to interact with LinkedIn and X — outreach, posting, audience insights, message triage, account research, billing audits — plus how to send useful bug reports and feature requests through `social feedback`.

## What ships

One `social` skill spanning both platforms, with progressive-disclosure references loaded only when needed. It auto-triggers on natural intent ("search LinkedIn", "my saved X posts") and is also invokable as `/social`.

- **`skills/social/SKILL.md`** → `/social` — the shared spine: when to use it, first-use setup probe, invocation conventions, feedback mode, billing, safety, and how to pick a platform reference.
- **`skills/social/references/setup.md`** — install, `social account login`, account `connect`, scopes/billing, env vars, error catalog, troubleshooting (both platforms).
- **`skills/social/references/linkedin.md`** — full LinkedIn command catalog, flags, output shapes, `jq` recipes, and end-to-end playbooks.
- **`skills/social/references/x.md`** — full X command catalog, field/expansion presets, output shapes, `jq` recipes, and end-to-end playbooks.

## Text input

All freeform text for posts, comments, messages, message edits, and connection
request notes is sent via stdin. Do not pass text as a positional argument:

```sh
echo "Thanks for the note." | social x message @username
echo "Thanks for the note." | social linkedin message @username
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

The repo also carries Claude and Codex plugin manifests for agents that can consume plugin directories directly.

## Prerequisites

The skill expects the `social` binary on `PATH`. If it is missing, ask the user
before installing global tools, then have them run the setup command in an
interactive terminal:

```sh
curl -fsSL https://usesocial.dev/install.sh | bash
```

Keep the skill current with `npx skills update social`. Re-run the relevant install command when you need to update the CLI.

## Configuration

Env vars are optional — defaults point at the hosted API:

| Variable | Default | Purpose |
|---|---|---|
| `SOCIAL_API_URL` | `https://api.usesocial.dev/v1` | API base URL |
| `SOCIAL_WEB_URL` | `https://usesocial.dev` | Device-approval web URL |

For local development, point both at localhost. See `skills/social/references/setup.md` for the full setup, scope, and troubleshooting reference.
