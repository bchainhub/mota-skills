# Database connections and migration visibility

The template does not provision a database. Choose the app's existing driver/ORM and deployment runtime; keep DB access in src/lib/server or *.server.ts and use parameterized queries. UI config may hold a public feature flag or binding name, never a DB URL with credentials. Declare required secrets in src/env.ts and import from $app/env/private in app server code.

## Connection choices

| Runtime/database | Connection pattern |
| --- | --- |
| Node + PostgreSQL | pg Pool or the existing Postgres driver; bounded pool and TLS per provider |
| Node + MySQL | mysql2/promise pool or existing compatible driver |
| Node + SQLite | Local file path and installed SQLite driver; requires persistent writable storage |
| Edge/provider database | The platform's request-time binding or compatible HTTP driver; do not assume Node sockets/native SQLite binaries work |

Do not install every driver. Add only packages needed for the selected DB in the installed app. Keep credentials in ignored local env/deployment secrets and use the provider's certificate validation; do not disable TLS checks to get connected.

For example, on a Node deployment using pg:

```ts
// src/lib/server/db.ts
import { Pool } from 'pg';
import { DATABASE_URL } from '$app/env/private';

export const pool = new Pool({ connectionString: DATABASE_URL, max: 5 });
```

Use it only from server handlers, for example `await pool.query('select id from users where id = $1', [userId])`. Reuse a bounded pool per Node process rather than constructing one for each request. Configure shutdown and pool lifetime for the selected host. Keep schema/migration credentials separate from lower-privilege runtime credentials where available.

For request-scoped platform bindings, obtain the binding from the server event platform object, validate it exists, and inject it into the DB helper. Match the binding name, provider config and platform types. A name in vite.config.ts does not create the database.

## Drizzle example

Use an already chosen ORM/version. If introducing Drizzle, use mutually compatible drizzle-orm, drizzle-kit and the chosen driver; do not select a release candidate just because current website examples show one. [Drizzle PostgreSQL drivers](https://orm.drizzle.team/docs/get-started-postgresql).

```ts
// drizzle.config.ts — CLI configuration, not a browser module
import { defineConfig } from 'drizzle-kit';

const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) throw new Error('DATABASE_URL is required');

export default defineConfig({
    schema: './src/lib/server/db/schema.ts',
    out: './_migrations',
    dialect: 'postgresql',
    dbCredentials: { url: databaseUrl }
});
```

The migration CLI is outside Kit, so use process.env with an explicitly configured environment loader/CI secret. A SvelteKit .env file is not automatically loaded by every external CLI. Do not import $app/env/private into drizzle.config.ts. The schema path must exist. [Drizzle config](https://orm.drizzle.team/docs/drizzle-config-file).

Suggested package scripts for a project that has installed drizzle-kit:

```json
{
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:check": "drizzle-kit check"
}
```

Run `npm run db:generate` to prepare reviewable files; use `npm run db:check` for history consistency. Run `npm run db:migrate` only against the intended target after reviewing the generated SQL. `generate` creates SQL/history to review; `migrate` applies unapplied generated migrations to the configured database. `push` directly changes the DB from schema differences and is not equivalent to reviewing a migration history. Prefer generated, reviewed migrations for shared/production databases. Test on a disposable local DB and identify the actual target before application. Do not run live migrations during addon install or ordinary app startup. [Drizzle migrations](https://orm.drizzle.team/docs/migrations).

## Committed versus private history

The starter ignores /_migrations/ and /better-auth_migrations/ by default. This controls tracking, not DB permissions or encryption.

**Committed history:** remove only the matching migration ignore rules (and check global ignores); keep SQL plus required journal/snapshot metadata together. Review for credentials and data dumps. Verify with `git check-ignore -v _migrations/example.sql` and `git status --short --untracked-files=all`. Stage the reviewed files explicitly when committing is requested. Do not force-add the entire app. Teams and CI then use the same ordered migration history.

**Private history:** keep the ignore rule and provide an access-controlled migration repository/artifact store and deployment job. Package the full history and journal consistently; an ignored folder is absent from a clean CI checkout unless delivered separately. Do not place production data, connection secrets or signed identity payloads in migration files. If a file was already tracked, ignore rules do not untrack it; removing it from the index is a separate deliberate change and does not erase old git history.

Record the chosen folder in the ORM config and deployment process. An addon creating SQL under _migrations does not make it runnable in every dialect. Match PostgreSQL/MySQL/SQLite syntax and use the selected tool's migration journal.

## Passkey addon compatibility

The inspected passkey addon has DB choices d1/sqlite/postgres/mysql and a runMigration prompt that defaults true. Its generated DB module still contains legacy $env/dynamic/private imports, and its script can run a remote D1 migration or an auth CLI migration. Review and adapt these paths for Kit 3 before use; migrate its shell/template flow to the updated runner controls before use. The runner rejects truthy runMigration answers in generation and requires a separate declared migration-only invocation with a target. Do not copy legacy env snippets into new code or infer consent from a prompt default.

For auth schema, adapter selection and plugin migrations, see [Better Auth](auth/better-auth.md).

For Cloudflare D1/KV provisioning, runtime bindings and environment separation, see [Cloudflare](cloudflare.md).
