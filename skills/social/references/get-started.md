# Get started with /social

The guided first-run flow: install check → sign in → connect a platform → first
sync. This is the home of the **skill-owns-consent** pattern — the CLI never
prompts and never blocks on approval, so *you* (the agent) estimate cost, state
it, and get the human's explicit yes before anything that spends credits.

Run it when the user says "let's get started with /social", "set me up", "log me
in", or asks to connect LinkedIn/X for the first time. Each step below is a
free, non-blocking check; only the final sync spends credits, and only after a
confirmed yes.

## The shape of onboarding

`social account login` and `social account connect <platform>` are **pollable
state machines** in a non-TTY (agent) shell. Each call advances one step and
returns a tagged `status`; you call again to advance. Nothing blocks waiting for
the human — the human approves in their browser on their own time, and your next
call observes the result.

Read `.status` from the JSON, never the exit code alone. Drive the whole flow
without backgrounding anything.

## Step 1 — Install check

Bare `social account` answers install + login + connection in one free call. Do
not probe with metered live reads.

```bash
social account 2>&1 | head -c 600
```

- `command not found: social` → ask the user to install:
  `curl -fsSL https://usesocial.dev/install.sh | bash` (or
  `brew install usesocial/tap/cli`). Then re-run the check.
- Otherwise read `.status` and continue to Step 2.

## Step 2 — Sign in (non-TTY poll loop)

In an agent shell, `social account login` runs the device-authorization flow as a
poll. The states:

| `.status`          | Meaning                                   | What you do |
| ------------------ | ----------------------------------------- | ----------- |
| `logged_in`        | Already signed in.                        | Go to Step 3. |
| `already_logged_in`| Existing valid session.                   | Go to Step 3. |
| `pending_approval` | Device flow started; awaiting approval.   | Surface `verificationURL` to the human; poll again. |
| `expired`          | The code lapsed before approval.          | Re-run `login` to restart cleanly. |
| `error`            | Surface `.message`; stop.                 | |

First call starts the flow:

```bash
social account login
```

A `pending_approval` response looks like:

```json
{
  "status": "pending_approval",
  "verificationURL": "https://usesocial.dev/device?user_code=WXYZ-1234",
  "userCode": "WXYZ-1234",
  "expiresAt": "2026-06-12T00:30:00.000Z",
  "scope": "read,write",
  "nextCommands": ["social account login"]
}
```

**Surface `verificationURL` to the human verbatim** and ask them to open it and
approve. The browser collects their email and confirms the session — you never
ask for or store their email, phone, magic link, or bearer token.

Then poll. Re-run the same command on a gentle interval (every ~5s, capped) until
the status changes:

```bash
social account login   # call again; same JSON contract, advanced one step
```

When it returns `logged_in`, sign-in is done. If it returns `expired`, tell the
user the code lapsed and re-run `login` to issue a fresh `verificationURL`.

Default scope is `read,write` (needed to connect and post). Pass
`--scope read` for a read-only session.

Do **not** background `login`, pipe `yes` into it, or loop it without a cap.

## Step 3 — Connect a platform

`login` authenticates the user against the social API; each platform needs its
own connection. Like login, connect is pollable in a non-TTY shell:

```bash
social account connect linkedin   # or: social account connect x
```

| `.status`          | Meaning                                  | What you do |
| ------------------ | ---------------------------------------- | ----------- |
| `connected`        | Account is linked.                       | Done; show `.account.username`. |
| `pending_approval` | Awaiting browser approval.               | Surface `connectURL`; poll again. |
| `error`            | Surface `.message`.                      | |

A `pending_approval` response carries a `connectURL` — surface it to the human,
ask them to approve in the browser, then call connect again to advance. When it
returns `connected`, confirm the linked `@username`. Connecting a platform may
need a billing seat; if a seat URL is surfaced, hand it to the human the same
way.

Re-run bare `social account` any time to confirm a connected-account row exists
for the platform.

## Step 4 — First sync, with the consent pattern

This is the one step that spends credits, so it is the one step that needs an
explicit yes. The pattern is always: **estimate → state → confirm → run →
verify.**

### 1. Estimate

`sync` cost scales with how many objects it pulls. Read the per-collection rate
from the schema and the object count from a cheap profile read, then multiply.

```bash
# Rate: what the sync costs and how it is metered.
social schema "x sync" | jq '.cost'

# Count: the user's own follower/following counts from a profile read.
social x profile --account <@username> | jq '.data | {followers_count, following_count}'
```

For LinkedIn use `social schema "linkedin sync"` and `social linkedin profile`
(connection count). Bare `social <platform> sync` also lists each collection's
`objectCount` once something has been synced before.

### 2. State the estimate

Tell the human, in plain language, what the sync will pull and roughly what it
will cost — e.g. "Syncing your ~4,200 X followers reads each one upstream and is
metered in usage credits; that's roughly N credits. Want me to run it?" Be
honest that it is an estimate; the exact spend appears in `.meta.cost`.

### 3. Confirm

Get an explicit yes. **Do not sync on assumption.** If the user only wanted, say,
their DMs, sync just that collection (`social x sync messages`) instead of the
full graph. Smaller, targeted syncs cost less.

### 4. Run

```bash
social x sync followers      # or the specific collection the user approved
```

Where the collection supports it, `--since <ISO date>` pulls only newer items and
spends fewer credits than a full re-pull.

### 5. Verify

```bash
# Actual spend for the call.
social x sync followers | jq '.meta.cost'

# Or audit after the fact.
social account usage
```

Report the real cost back to the user, and flag if it crossed a usage threshold
(25% / 50% / 75% / 100% of included credits).

## After onboarding

The user is now set up. Hand off to the platform references for day-to-day work:

- `references/x.md` — X command catalog and recipes.
- `references/linkedin.md` — LinkedIn command catalog and recipes.
- `references/setup.md` — auth state machine details, scopes/billing, cache,
  error catalog, troubleshooting.

The consent pattern from Step 4 applies to every credit-spending or
account-mutating command, not just the first sync: estimate, state, confirm,
then act. See the hazard vocabulary in `SKILL.md` — `spends_credits`,
`destructive`, and `outbound_write` hazards are advisory signals that *you*
confirm with the human; the CLI itself never gates.
```
