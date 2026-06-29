# powersync-maglink

Dedicated [PowerSync](https://powersync.com) instance for the **maglink** app
(offline-first kiosk for the Ativações / trade-show module). Deployed on the
**surtr** node via Coolify.

Unlike the generic `powersync-selfhost` spike (isolated `lists`/`todos` demo),
this instance replicates from maglink's **real project database** (`db_maglink`
on the shared `hauldr-db` cluster) and authenticates clients with maglink's own
**GoTrue (Supabase) JWTs** — no separate token minting.

```
maglink kiosk (tablet)  ──ws sync (GoTrue JWT)──►  powersync engine  ──logical repl──►  db_maglink
        │                                                                                  ▲
        └── writes ── /api/powersync/upload (cookie auth) ── pgrest (user JWT) ── PostgREST ┘
```

## Services

| Service      | Role                                                              | Public |
|--------------|-------------------------------------------------------------------|--------|
| `pg-storage` | Postgres bucket storage for the PowerSync engine (isolated)       | no     |
| `powersync`  | PowerSync Service engine, port 8080                               | **yes** (`powersync-maglink.coldcodelabs.com`) |

The replication source is **not** in this compose — it is `db_maglink` on the
existing `hauldr-db` container, reached over the shared `coolify` network.

## Configuration

Config is passed to the engine as base64 env vars (no file mounts), exactly like
the spike. Nothing secret lives in this repo.

- `SVC_B64`     — base64 of the service config (replication, storage, `client_auth` Supabase HS256)
- `SYNC_B64`    — base64 of the sync rules (global `ativacoes` stream, edition 3)
- `PS_JWT_SECRET` — maglink's `HAULDR_JWT_SECRET` (validates GoTrue access tokens)
- `REPL_PW`     — password of the `powersync_repl` role on `db_maglink`
- `PGPW`        — password for the local bucket Postgres

All set as Coolify env vars.

## Source DB setup (db_maglink)

A read-only replication role + publication is created on `db_maglink`:

```sql
CREATE ROLE powersync_repl WITH LOGIN REPLICATION PASSWORD '...';
GRANT USAGE ON SCHEMA public TO powersync_repl;
GRANT SELECT ON ativacoes TO powersync_repl;
ALTER TABLE ativacoes REPLICA IDENTITY FULL;   -- PowerSync needs full row on update/delete
CREATE PUBLICATION powersync FOR TABLE ativacoes;
```

To add more tables later: `GRANT SELECT`, set `REPLICA IDENTITY FULL`,
`ALTER PUBLICATION powersync ADD TABLE <t>`, then extend the sync rules.

## Auth

Clients authenticate with their **GoTrue access token** (HS256, `aud=authenticated`).
The engine validates it with `client_auth.supabase` + the shared secret;
`request.user_id()` resolves to the token `sub`. The Ativações stream is
workspace-shared (any signed-in staffer gets every row), so the sync query has
no per-user filter.

## UI

No admin UI. Inspect with the **PowerSync Diagnostics App**
(`journeyapps/powersync-diagnostics-app` or https://diagnostics-app.powersync.com):
engine endpoint + a dev JWT.
