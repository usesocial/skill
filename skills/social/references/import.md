# Local imports

Use this reference when the user already has a complete local export of their own
social data and wants to seed the CLI mirror without a metered upstream sync.
This is a local SQLite operation, not a public `social` command.

Do not call LinkedIn or X HTTP APIs. Do not run a metered `social x sync` unless
the user explicitly asks for a fresh upstream refresh.

## When to use

Use a local import when all are true:

- The file is already on disk.
- The file belongs to the selected connected account.
- The user says the export is complete enough to stand in for the first sync.
- The target collection is a local mirror table such as X followers.

Do not mark partial exports as fully synced. If the file is only a sample,
preview, or partial Wrapped depth, import it into a separate working file or ask
before touching the CLI mirror.

## Locate the X cache DB

The sync engine stores one SQLite DB per selected account:

```bash
social account | jq '.accounts[] | select(.platform == "x") | {username, profileId, status}'
```

The default account selector is `@username` for the first connected X account.
For an explicit account, use the same selector you would pass to `--account`.

The cache filename is:

```text
~/.social/cache/x-<sanitized-selector>.db
```

Sanitization replaces every character outside `A-Z`, `a-z`, `0-9`, `.`, `_`,
and `-` with `_`. For example, `@cyrusnewday` becomes:

```text
~/.social/cache/x-_cyrusnewday.db
```

Before importing, open or create the DB by running the free local sync-status
command. Bare `sync` lists collections and initializes the local schema; it does
not fetch follower pages.

```bash
social x sync >/dev/null
```

If the account is not connected, connect it first with `social account connect x`.

## X followers import contract

Import complete X follower exports into `x_followers_raw`. `x_followers` is a
read-only view over that raw table.

Canonical CSV column order:

```text
id,created_at,username,name,description,location,url,is_identity_verified,protected,verified,verified_type,subscription_type,parody,most_recent_tweet_id,pinned_tweet_id,profile_banner_url,profile_image_url,affiliation_badge_url,affiliation_description,affiliation_url,subscription_subscribes_to_you,followers_count,following_count,tweet_count,listed_count,like_count,media_count,raw,synced_at
```

Required values:

- `id`: X user id.
- `raw`: JSON string for the original or reconstructed X user object.
- `synced_at`: import time as epoch milliseconds.

Recommended values:

- `created_at`: account creation time as epoch milliseconds.
- `username`, `name`, `description`, `location`, `url`.
- `profile_image_url`, `profile_banner_url`.
- `verified`, `verified_type`, `is_identity_verified`, `protected`.
- `followers_count`, `following_count`, `tweet_count`, `listed_count`,
  `like_count`, `media_count`.

Boolean columns are `1`, `0`, or empty. Unknown numeric/text fields can be empty.

If the source is raw Wrapped JSON, generate the canonical CSV from the JSON
objects. If the source is already CSV, map its headers into the canonical order.
Preserve the source row as `raw`; if the CSV does not include raw JSON,
reconstruct a JSON object from the mapped fields.

## Import the canonical CSV

Set the DB path and import timestamp:

```bash
ACCOUNT_SELECTOR="@username"
SANITIZED_SELECTOR="$(printf '%s' "$ACCOUNT_SELECTOR" | sed -E 's/[^A-Za-z0-9._-]/_/g')"
DB="$HOME/.social/cache/x-${SANITIZED_SELECTOR}.db"
CSV="/path/to/x-followers-import.csv"
NOW_MS="$(date +%s)000"
```

Import rows and mark the followers collection synced:

```bash
sqlite3 "$DB" <<SQL
DROP TABLE IF EXISTS _social_import_x_followers;
CREATE TABLE _social_import_x_followers AS SELECT * FROM x_followers_raw WHERE 0;
.mode csv
.import --skip 1 "$CSV" _social_import_x_followers
INSERT OR REPLACE INTO x_followers_raw SELECT * FROM _social_import_x_followers;
DROP TABLE _social_import_x_followers;
INSERT INTO sync_state (collection, cursor, last_synced, object_count)
VALUES ('x_followers', NULL, $NOW_MS, (SELECT count(*) FROM x_followers_raw))
ON CONFLICT(collection) DO UPDATE SET
  cursor = excluded.cursor,
  last_synced = excluded.last_synced,
  object_count = excluded.object_count;
SQL
```

Use `cursor = NULL` only for a complete follower export. A null cursor tells the
local mirror that the initial full walk is complete.

## Verify

Run free local reads:

```bash
social x sql "SELECT count(*) AS followers FROM x_followers"
social x sql "SELECT username, name, followers_count FROM x_followers ORDER BY followers_count DESC LIMIT 20"
```

Expected metadata:

- `meta.cost` is absent.
- `meta.cache.source` is `local`.

If verification fails with missing columns, regenerate the canonical CSV from
the current table schema:

```bash
sqlite3 "$DB" ".schema x_followers_raw"
```
