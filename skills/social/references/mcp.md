# Hosted Social MCP

Use this reference when a user asks how to connect Social through MCP, how to
find their MCP URL, or how to get an authorization code.

## Connect

Install Social into detected MCP clients when the user asks:

```bash
$(social mcp install)
```

`social mcp install` only prints a fixed `bunx add-mcp` or `npx add-mcp`
command; command substitution executes it. The emitted command includes
`--global` and leaves client selection interactive.

Print the canonical production URL:

```bash
social mcp url
```

The result is intentionally the same secret-free URL for every user:

```text
https://mcp.usesocial.dev
```

Configure it as a remote Streamable HTTP server in an OAuth-capable MCP client.
Then click the client's **Connect** or **Authenticate** action, or invoke a
Social tool so the client starts authentication. A useful first request is:

```text
Use Social to show my connected accounts.
```

The MCP client should open Social in the browser. The user enters their email,
redeems the magic link, and approves the requested capabilities. Existing
emails sign in; new emails create a Social account.

## Authorization codes and tokens

Do not obtain, display, or paste an authorization code manually. The MCP client:

1. discovers Social's OAuth metadata;
2. dynamically registers itself;
3. starts an authorization-code flow with S256 PKCE;
4. receives the temporary code at its own redirect URI; and
5. exchanges the code for access and refresh tokens.

The MCP URL never contains an API key or token. Do not append credentials as a
query parameter or reuse the CLI bearer. Production MCP authentication is OAuth
only.

## Permissions

The default grant includes read and write scopes. The OAuth consent screen lists
the exact scopes before they are granted. Writes execute immediately; the MCP
client owns confirmation before calling them.

After OAuth authorization, call `account_setup` with no arguments. This tool is
a mutation because it uses Autumn `setupPayment` to authorize a reusable card.
If it returns `pending_billing`, surface `checkoutURL` and invoke the exact
`account_setup` entry in `nextTools` after approval. Setup never selects or
purchases a provider plan. Once ready, preserve `user`, `scope`, and exact
`capabilities`, then follow its provider-specific `nextTools`.

`account_connect` accepts `{ platform, attemptId? }` and can charge the stored
card for only that provider's plan or next seat, or the applicable legacy Social
Pro seat during transition. Omit `attemptId` on the first
call, then use the exact returned ID from every `nextTools` retry. Surface
`paymentURL` only when `pending_billing` requires customer action; after that,
surface the hosted `connectURL` from `pending_approval`. Replaying a completed
ID is stable. Omit the ID after completion only to connect another account.

## Troubleshooting

- **No browser opens:** click the client's connect/authenticate control or invoke
  a Social tool. Re-add the server as remote Streamable HTTP if necessary.
- **The client asks for an API key, bearer token, or pasted authorization code:**
  it likely does not support remote MCP OAuth. Do not invent or expose a secret;
  ask which client/version the user is configuring and check its OAuth support.
- **`social mcp` is unknown:** update the installed Social CLI, then rerun
  `social mcp url`.
- **Login succeeds but tools are missing:** reconnect and approve the required
  read or write scopes. Tool visibility follows the exact OAuth grant.

Discovery endpoints, useful only for client diagnostics:

```text
https://mcp.usesocial.dev/.well-known/oauth-protected-resource
https://usesocial.dev/.well-known/oauth-authorization-server/api/auth
```
