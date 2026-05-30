# Social agent skills

Agent skills for the [`@usesocial/cli`](https://www.npmjs.com/package/@usesocial/cli) CLI. They teach supported agents when and how to call `social` to read and write LinkedIn and X on the user's behalf — search people, fetch posts, list bookmarks, read timelines, inspect billing — and how to handle setup, scopes, and output formats.

## What ships

Two platform-scoped skills, each with its own progressive-disclosure references. Each auto-triggers on natural intent for its platform and is also invokable as a slash command.

- **`skills/linkedin/SKILL.md`** → `/social:linkedin` — search people/posts, fetch profiles, drill into post comments/reactions, look up companies/employees/jobs, manage LinkedIn account lifecycle, audit usage.
- **`skills/x/SKILL.md`** → `/social:x` — search recent tweets, fetch tweets by ID, list a user's tweets, read the home timeline, manage bookmarks, manage X account lifecycle, audit usage.

Each skill carries the same shape of references, loaded only when needed:

- `references/setup.md` — install, `social login`, scope/billing, env vars, troubleshooting.
- `references/commands.md` — full platform command catalog with arguments and output shapes.
- `references/output.md` — `--json` / `--pretty`, `jq` recipes, error catalog.
- `references/examples.md` — end-to-end platform-specific recipes.

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

For local development against `just dev`, point both at localhost — see `apps/cli/README.md`.
