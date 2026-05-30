# Social agent skill

An agent skill for the [`@usesocial/cli`](https://www.npmjs.com/package/@usesocial/cli) CLI. It teaches supported agents when and how to call `social` to interact with LinkedIn and X on the user's behalf — search people and posts, fetch profiles, read timelines, list bookmarks, drill into comments/reactions, look up companies, audit billing — and how to handle setup, scopes, and output formats.

## What ships

One `social` skill spanning both platforms, with progressive-disclosure references loaded only when needed. It auto-triggers on natural intent ("search LinkedIn", "my X bookmarks") and is also invokable as `/social`.

- **`skills/social/SKILL.md`** → `/social` — the shared spine: when to use it, first-use setup probe, invocation conventions, billing, safety, and how to pick a platform reference.
- **`skills/social/references/setup.md`** — install, `social login`, account `connect`, scopes/billing, env vars, error catalog, troubleshooting (both platforms).
- **`skills/social/references/linkedin.md`** — full LinkedIn command catalog, flags, output shapes, `jq` recipes, and end-to-end playbooks.
- **`skills/social/references/x.md`** — full X command catalog, field/expansion presets, output shapes, `jq` recipes, and end-to-end playbooks.

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

The skill expects the `social` binary on `PATH`. If it is missing, the skill will install it and walk the user through `social login` (device flow, requires browser approval).

```sh
bun install -g @usesocial/cli@latest    # preferred
npm install -g @usesocial/cli           # fallback
social login                          # device-authorization sign-in
```

## Configuration

Env vars are optional — defaults point at the hosted API:

| Variable | Default | Purpose |
|---|---|---|
| `SOCIAL_API_URL` | `https://api.socialcli.dev/v1` | API base URL |
| `SOCIAL_WEB_URL` | `https://socialcli.dev` | Device-approval web URL |

For local development, point both at localhost. See `skills/social/references/setup.md` for the full setup, scope, and troubleshooting reference.
