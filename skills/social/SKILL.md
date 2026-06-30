---
name: social
description: |
  Use when the user wants account/admin help for the social CLI: install,
  login, logout, billing, usage, account logs, update checks, CLI feedback,
  bug reports, feature requests, or setup troubleshooting. Triggers include
  "install social", "log me in", "check billing", "show usage", "show logs",
  "update social", "report a bug", "request a feature", "send feedback", and
  explicit `/social`. For LinkedIn connection or LinkedIn work, use `/linkedin`.
  Operates the `social` CLI (npm `@usesocial/cli`); never call provider HTTP
  APIs directly.
argument-hint: 'task - e.g. "install", "log me in", "check billing", "show usage", "send feedback"'
---

# social

Use `social` for CLI account administration. The agent runs commands; the human
stays the principal. For LinkedIn connection, reads, writes, sync, SQL, outreach,
posting, inbox triage, or account research, load the `linkedin` skill.

```
social account | feedback | update
```

- `social account` - inspect install/login state, connected accounts, credential storage, and seat state.
- `social account login` - start or poll device login.
- `social account logout` - clear the active credentials.
- `social account billing` - inspect subscription, seats, and usage summary.
- `social account billing portal` - print a hosted billing portal URL.
- `social account usage` - inspect current usage and included/overage status.
- `social account logs` - audit recent calls and costs.
- `social update` - check installed CLI and skill freshness.
- `social feedback bug|feature` - send useful product feedback through the CLI.

## First checks

For any account/admin task, start with the free readiness check:

```bash
social account 2>&1 | head -c 600
```

Interpret the JSON status, not just the exit code:

- `command not found: social` - ask the user to install with `curl -fsSL https://usesocial.dev/install.sh | bash` in an interactive terminal.
- `"status": "logged_out"` or `"expired"` - run `social account login` and poll as described below.
- `"status": "logged_in"` - account/admin commands can run.
- Connected-account rows are informational here; use `/linkedin` to connect or operate LinkedIn.

Do not run metered provider reads as setup probes.

## Login state machine

In an agent shell, `social account login` is a non-blocking poll. The first call
starts device authorization and later calls observe progress.

```bash
social account login
```

Handle statuses:

| `.status` | Meaning | Action |
| --- | --- | --- |
| `logged_in` / `already_logged_in` | Session is ready. | Continue. |
| `pending_approval` | Browser approval is waiting. | Surface `verificationURL`; poll gently. |
| `expired` | Device code expired. | Re-run `social account login`. |
| `error` | Login failed. | Surface `.message`; stop. |

Never ask for magic links, bearer tokens, cookies, or credentials. Never
background login or pipe `yes` into it.

## Billing, usage, and logs

Use account commands for audits and billing handoff:

```bash
social account billing
social account billing portal
social account usage
social account logs --limit 20
social account logs --platform linkedin --limit 20
```

`account billing portal` prints `{ "url": "...", "opened": false }` in
non-interactive shells; hand that URL to the user.

Use `account logs` after provider errors, repeated rate limits, or large tasks.
Use `account usage` for totals.

## Updates

Check both CLI and skill freshness without auth or provider calls:

```bash
social update
```

Never update installed tools silently. Surface the returned `updateCommand` and
ask before changing the user's machine.

## Feedback

Use feedback mode for product bugs and feature requests. Gather safe context,
draft a concise report, show it unless the user has already approved sending,
then pipe the report:

```bash
echo "..." | social feedback bug
echo "..." | social feedback feature
```

Do not include bearer tokens, magic links, cookies, private message dumps, or
unrelated personal data.

## Exit codes

| Code | Meaning | What to do |
| ---: | --- | --- |
| `0` | Success | Continue. |
| `2` | Usage or validation error | Fix command, flags, IDs, JSON body, or local input. |
| `3` | Not found | Check the ID or select a different resource. |
| `4` | Auth or scope error | Run `social account login`, or log out and choose the needed scope. |
| `5` | API, proxy, or unexpected error | Retry later or surface the server error. |
| `7` | Rate limited | Back off; use `retryAfterSeconds`, `resumeAt`, and `retryCommand` when present. |

## Safety rules

- Never call provider HTTP APIs directly.
- Never echo or save bearer tokens.
- Never retry rate limits in a tight loop.
- Treat account logs as sensitive operational data.
- Use `/linkedin` for LinkedIn connection and LinkedIn work.

## Additional resources

- `references/setup.md` - account/admin setup, login, billing, logs, updates, errors, and troubleshooting.
