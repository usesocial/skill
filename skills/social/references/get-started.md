# Get started with /social

The guided first-run flow: install check → sign in → connect a platform → first
sync. This is the home of the **skill-owns-consent** pattern — the CLI never
prompts and never blocks on approval, so *you* (the agent) estimate cost, state
it, and get the human's explicit yes before anything estimated at $5 or more.
Cheaper metered steps just run, with the spend reported afterward.
The policy core lives in `SKILL.md`; this reference applies it to first-run
onboarding.

Run it when the user says "let's get started with /social", "set me up", "log me
in", or asks to connect LinkedIn/X for the first time. The install/account
checks, login polling, and connect polling below are free, non-blocking checks.
A metered live read used to size the first sync (a profile lookup) costs cents —
well under the $5 line — so run it when it helps, and mention the spend.

## The shape of onboarding

`social account setup` and `social account connect <platform>` are **pollable
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

## Step 2 — Set up the Social account (non-TTY poll loop)

In an agent shell, `social account setup` runs the device-authorization flow as a
poll. The states:

| `.status`          | Meaning                                   | What you do |
| ------------------ | ----------------------------------------- | ----------- |
| `pending_approval` | Device flow started; awaiting approval.   | Surface `verificationURL` to the human; poll again. |
| `pending_billing`  | Authentication is complete; billing is required. | Surface `checkoutURL`; poll again after checkout. |
| `ready`            | Authentication and base billing are ready. | Go to Step 3. |
| `expired`          | The code lapsed before approval.          | Re-run `setup` to restart cleanly. |
| `error`            | Surface `.message`; stop.                 | |

First call starts the flow:

```bash
social account setup
```

A `pending_approval` response looks like:

```json
{
  "status": "pending_approval",
  "verificationURL": "https://usesocial.dev/device?user_code=WXYZ-1234",
  "userCode": "WXYZ-1234",
  "expiresAt": "2026-06-12T00:30:00.000Z",
  "scope": "read,write",
  "nextCommands": ["social account setup"]
}
```

**Surface `verificationURL` to the human verbatim** and ask them to open it and
approve. The browser collects their email and confirms the session — you never
ask for or store their email, phone, magic link, or bearer token.

Then poll. Re-run the same command on a gentle interval (every ~5s, capped) until
the status changes:

```bash
social account setup   # call again; same JSON contract, advanced one step
```

If it returns `pending_billing`, surface `checkoutURL` and ask the user to finish
checkout, then continue polling the same command. Continue only after `ready`.
If it returns `expired`, tell the user the code lapsed and re-run `setup` to
issue a fresh `verificationURL`.

The setup grant defaults to `read,write`, which is required for onboarding.
Do **not** background `setup`, pipe `yes` into it, or loop it without a cap.

## Step 3 — Connect a platform

`setup` prepares the Social account; each platform still needs its own
connection. Like setup, connect is pollable in a non-TTY shell:

```bash
social account connect linkedin   # or: social account connect x
```

| `.status`          | Meaning                                  | What you do |
| ------------------ | ---------------------------------------- | ----------- |
| `connected`        | Account is linked.                       | Done; show `.account.username`. |
| `pending_billing`  | A billing seat must be activated first.  | Surface `paymentURL`; poll again after checkout. |
| `pending_approval` | Awaiting browser approval.               | Surface `connectURL`; poll again. |

A `pending_billing` response carries a `paymentURL` when checkout is needed.
Surface it to the human, have them complete checkout, then call connect again.
A `pending_approval` response carries a `connectURL` — surface it to the human,
ask them to approve in the browser/profile they want to use, then call connect
again to advance. When it returns `connected`, confirm the linked `@username`.

Re-run bare `social account` any time to confirm a connected-account row exists
for the platform.

## Step 4 — First sync, with the consent pattern

This is the one step that spends real usage. The pattern is always:
**estimate → state → confirm (at $5+) → run → verify.** A full-graph first sync
usually clears $5 easily; a small targeted sync (say, just DMs) often does not,
and can run once you have stated the estimate.

### 1. Estimate

`sync` cost scales with how many objects it pulls. Read the per-collection rate
from the schema, then use counts the user or previous local sync/status output
already gave you. If you do not already know the count, say the estimate is
rough until the first sync reports the exact spend.

```bash
# Rate: what the sync costs and how it is metered.
social schema "x sync" | jq '.cost'
```

For LinkedIn use `social schema "linkedin sync"`. If you need a fresher count
than the user or local metadata can provide, `social x profile --account
<@username>` and `social linkedin profile --account <@username>` are metered
live reads costing cents; run one to improve the estimate and mention the spend.

### 2. State the estimate

Tell the human, in plain language, what the sync will pull and roughly what it
will cost — e.g. "Syncing your ~4,200 X followers reads each one upstream and is
metered in usage dollars; at $0.015/item, that's roughly $63 before cache effects.
Want me to run it?" Be honest that it is an estimate; the exact spend appears in
`.meta.cost` when the response spends usage. If the estimate will likely run
past the included usage, say so: nothing blocks — the account auto-tops-up $15
of usage credits at a time, and that is the actual charge the human is agreeing
to.

### 3. Confirm

If the estimate reaches $5 — or you cannot bound it below $5 — get an explicit
yes. **Do not run a $5+ sync on assumption.** Under $5, the stated estimate is
enough; just run it. If the user only wanted, say, their DMs, sync just that
collection (`social x sync messages`) instead of the full graph. Smaller,
targeted syncs cost less and often duck under the threshold entirely.

### 4. Run

```bash
social x sync followers      # or the specific collection the user approved
```

Where the collection supports it, `--since <ISO date>` pulls only newer items and
spends less usage than a full re-pull.

### 5. Verify

```bash
# Actual spend for the call.
social x sync followers | jq '.meta.cost // 0'

# Or audit after the fact.
social account usage
```

Report the real cost back to the user, and flag if it crossed a usage threshold
(25% / 50% / 75% / 100% of included usage).

## After onboarding

The user is now set up. Hand off to the platform references for day-to-day work:

- `references/x.md` — X command catalog and recipes.
- `references/linkedin.md` — LinkedIn command catalog and recipes.
- `references/setup.md` — auth state machine details, scopes/billing, cache,
  error catalog, troubleshooting.

The consent pattern from Step 4 applies beyond the first sync: for any
usage-spending command, estimate and state the cost, and get a yes when the
estimate — for one command, or the task's metered commands taken together —
reaches $5. `destructive` and `outbound_write` hazards are always confirmed,
regardless of cost. See the hazard vocabulary in `SKILL.md`; hazards are
advisory signals that *you* act on — the CLI itself never gates.
