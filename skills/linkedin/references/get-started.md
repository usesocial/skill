# Get started with /linkedin

The guided first-run flow for LinkedIn: install check -> login -> connect
LinkedIn -> first useful sync. The CLI never prompts in an agent shell, so the
skill owns consent for metered reads and writes.

## Step 1 - Readiness check

Start with the free account check:

```bash
social account 2>&1 | head -c 600
```

- `command not found: social` -> ask the user to install:
  `curl -fsSL https://usesocial.dev/install.sh | bash`.
- `"status": "logged_out"` or `"expired"` -> run `social account login`.
- `"status": "logged_in"` -> continue.
- Logged in with no connected LinkedIn account -> run `social account connect linkedin`.
- Logged in with a connected LinkedIn account -> ready for LinkedIn work.

Do not probe setup with metered live reads.

## Step 2 - Login

In an agent shell, `social account login` is a pollable device flow:

```bash
social account login
```

Handle `.status`:

| `.status` | Meaning | Action |
| --- | --- | --- |
| `logged_in` / `already_logged_in` | User is signed in. | Continue. |
| `pending_approval` | Browser approval is waiting. | Surface `verificationURL`; poll gently. |
| `expired` | Code expired. | Re-run login. |
| `error` | Login failed. | Surface `.message`; stop. |

Never ask for or store magic links, cookies, bearer tokens, or credentials.

## Step 3 - Connect LinkedIn

Run:

```bash
social account connect linkedin
```

In a non-TTY shell this is also pollable:

| `.status` | Meaning | Action |
| --- | --- | --- |
| `connected` | LinkedIn account is ready. | Continue. |
| `pending_approval` | Hosted auth is waiting in the browser. | Surface `connectURL`; poll gently. |
| `billing_required` / `no_available_seat` | A seat or payment action is needed. | Surface the returned billing URL/action. |
| `timed_out` / `linkedin_connect_timed_out` | Approval window expired. | Re-run connect. |
| `error` | Connection failed. | Surface `.message`; stop. |

If LinkedIn shows a credential-sharing warning during hosted auth, explain that
the account should be real and established, and the user should approve only if
they understand the account-risk tradeoff.

## Step 4 - First useful sync

After connect, choose the smallest sync that matches the user's immediate job.
Do not run a broad first sync by default.

Common starts:

```bash
social linkedin sync messages
social linkedin sync requests
social linkedin sync connections
social linkedin sync posts
```

Before any metered sync, inspect the schema/cost when useful, state what will be
read, and get explicit consent:

```bash
social schema "linkedin sync" | jq '.cost'
```

Then run only the approved collection. Afterward, use SQL for free local reads:

```bash
social linkedin sql "SELECT sender_name, text FROM li_messages ORDER BY created_at DESC LIMIT 20"
social linkedin sql "SELECT user_name, message FROM li_requests WHERE type = 'received' ORDER BY created_at DESC LIMIT 50"
```

## Step 5 - Confirm readiness

Use:

```bash
social account
social linkedin sync
```

The account check confirms the connected LinkedIn row. Bare LinkedIn sync lists
available collections, freshness, and row counts without pulling upstream pages.
