# Cloudflare deployment and services

Use this reference for MOTA apps hosted on Pages or Workers. Inspect the app's adapter, Wrangler config and deployment settings before changing them. Creating resources, deploying Workers, applying remote migrations and sending emails are separate operations from documenting or preparing configuration.

## Pages versus Workers

Pages hosts static output and can run SvelteKit server routes through Pages Functions. A fully prerendered site has no server handler for DB access or email. Use a Cloudflare-compatible SvelteKit adapter and preserve the existing Kit 3 configuration; do not add a competing `functions/` router beside framework-generated routing without checking the adapter.

Workers can host HTTP APIs, scheduled handlers and Durable Object classes. A Pages application can use companion Workers for these features through appropriate service/DO bindings. Keep deployment units and their environments explicit. See [Pages Functions](https://developers.cloudflare.com/pages/functions/) and [Workers handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/).

If Cloudflare adaptation fails with `Cannot find package 'wrangler'`, keep `legacy-peer-deps=true` and install a compatible Wrangler version explicitly as a devDependency. adapter-auto's installation of adapter-cloudflare does not guarantee its peer is present with this npm setting. See [missing peer package repair](starter-addons.md#missing-peer-packages-with-legacy-peer-deps) for the diagnosis, latest-version install command and compatibility checks. Verify that CI installs devDependencies before the build.

## Node build version: prefer the latest LTS

Keep the minimum requirement in `package.json`, currently `"engines": { "node": ">=22.17.0" }`. This validates compatibility; it does not install Node. Cloudflare Pages does not select Node from engines, so its default can fail before the build command starts.

In the Cloudflare dashboard, select the Pages project under Workers & Pages, open Settings and its environment variables section, and set `NODE_VERSION` to a numeric Node version. `NODE_VERSION` is configured per environment: select every relevant environment (Production and Preview when both are used) when adding the variable, or add it separately to each. Setting it only for Production does not configure Preview builds, and vice versa. Verify the value in both environment settings, then redeploy the affected environments. Alternatively, commit `.node-version` or `.nvmrc` at the configured project root with the numeric version. Remove conflicting overrides. Confirm the selected version in the new build log. See [Pages build image and overrides](https://developers.cloudflare.com/pages/configuration/build-image/).

Prefer the newest supported LTS release that satisfies engines and the project's dependencies. Check the [Node release table](https://nodejs.org/en/about/previous-releases) when configuring it: Node 24 is LTS as of October 2026, whereas Node 26 is still Current. Use the latest available patch of the selected LTS, not a permanently hardcoded patch copied from this guide.

For reproducible builds, record the exact numeric LTS patch and update it periodically or through automated update PRs. Do not rely on named LTS aliases in Pages: its documented v3 limitations include codename selectors. Truly automatic latest-LTS selection needs a custom bootstrap/version manager before dependency installation, with `SKIP_DEPENDENCY_INSTALL=1`; it must enforce engines, then install dependencies and build using that Node. An npm hook or ordinary build command cannot fix an earlier automatic install failure.

Workers Builds has its own [build image and NODE_VERSION setting](https://developers.cloudflare.com/workers/ci-cd/builds/build-image/). Neither setting changes the deployed Workers runtime: runtime APIs depend on compatibility date/flags, including `nodejs_compat` where required. Do not weaken engine checks to hide an unsupported build version.

## Bindings and runtime access

A binding gives server code a resource handle such as `DB`, `CACHE`, `ROOMS` or a Worker service. A string in `vite.config.ts` neither provisions nor attaches it. Configure bindings per environment using the supported Pages dashboard settings or Wrangler config; redeploy after changes. When adopting Wrangler for Pages, reconcile existing dashboard values because the file becomes the configuration source of truth for supported settings. See [Pages bindings](https://developers.cloudflare.com/pages/functions/bindings/) and [Wrangler configuration for Pages](https://developers.cloudflare.com/pages/functions/wrangler-configuration/).

In SvelteKit server handlers, use `event.platform?.env`; a standalone Worker receives `env` in its handler. Generate matching platform types with the installed Wrangler version, and extend `App.Platform` where needed. Inject request bindings into server helpers rather than initializing a DB from a browser global or build-time config. Keep tokens in runtime secrets, not public env/site config. Local emulation and Preview should use separate data from Production.

## D1 SQL database

D1 is managed SQLite-compatible SQL, useful for structured application/auth records. A representative binding fragment is:

```json
{
    "d1_databases": [
        {
            "binding": "DB",
            "database_name": "mota-app",
            "database_id": "REPLACE_WITH_CREATED_DATABASE_ID",
            "migrations_dir": "_migrations"
        }
    ]
}
```

Create or select the database, replace its ID and attach the binding; declaring this fragment alone does not create it. Use server-side prepared statements with bound values:

```ts
// Inside a SvelteKit server handler; userId is already validated.
const db = event.platform?.env.DB;
if (!db) throw new Error('DB binding is missing');
const user = await db.prepare('SELECT id, name FROM users WHERE id = ?')
    .bind(userId)
    .first();
```

Use a D1-compatible ORM adapter when desired. Do not pass a D1 binding to a Node PostgreSQL pool. [D1 binding API](https://developers.cloudflare.com/d1/worker-api/d1-database/).

With the selected Wrangler CLI, `wrangler d1 migrations create DB initial` creates a migration file, `wrangler d1 migrations apply DB --local` tests local migrations, and `wrangler d1 migrations apply DB --remote` changes the bound remote DB. Select the intended environment explicitly and inspect the resolved database before remote application. See [D1 commands](https://developers.cloudflare.com/d1/wrangler-commands/) and [committed/private migration delivery](databases.md). Binding IDs are configuration; credentials and production records do not belong in SQL history.

## KV key-value storage

Use Workers KV for read-heavy settings, cached responses and other data that tolerates eventual consistency. Bind a namespace, for example `"kv_namespaces": [{ "binding": "CACHE", "id": "REPLACE_WITH_NAMESPACE_ID" }]`. From server code, `await env.CACHE.get(key, 'json')`, `await env.CACHE.put(key, JSON.stringify(value), { expirationTtl: 300 })` and `await env.CACHE.delete(key)` access it. In Kit, `env` here is the request's `platform.env`.

Writes can take time to appear in other locations. KV is unsuitable for atomic counters, wallet challenge redemption, immediate session revocation or coordinating concurrent payments. Choose transactional DB logic or a Durable Object when correctness requires atomic coordination. See [KV consistency](https://developers.cloudflare.com/kv/concepts/how-kv-works/). Keep local, Preview and Production namespaces separate; local emulator state is not the deployed namespace.

## Workers and cron jobs

Put recurring maintenance in a companion Worker with a `scheduled` handler. Pages route files do not register cron events. Example Worker config fragment:

```json
{
    "triggers": { "crons": ["*/15 * * * *"] }
}
```

Cron expressions run in UTC; this example runs every 15 minutes. Export `scheduled(controller, env, ctx)` and await the job or register its promise with `ctx.waitUntil()`. Bind the same intended DB/KV resources to this Worker explicitly; Pages bindings are not automatically inherited. Make jobs idempotent and handle overlapping runs and partial failure using a durable job key/transaction where needed. Use Queues or Workflows for work requiring durable retries and longer orchestration rather than assuming cron guarantees exactly-once execution. Test locally with the installed Wrangler's scheduled-event support and inspect deployed trigger/log status. See [Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/).

## Email: API from Pages, binding from Workers

Configure an authorized sending domain and its required DNS records first. Cloudflare Email Sending is currently beta and transactional sending requires the appropriate account access/plan; incoming Email Routing alone is not unrestricted outbound sending. Recheck availability and [domain setup](https://developers.cloudflare.com/email-service/configuration/domains/) for the account. See [Email Service overview](https://developers.cloudflare.com/email-service/).

For Pages, call the [Email Sending REST API](https://developers.cloudflare.com/email-service/api/send-emails/rest-api/) from a server route/action or server-only mail helper. Keep the account ID and scoped sending API token in server runtime configuration/secrets. Send an authenticated POST to `https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/send`, using `Authorization: Bearer TOKEN` and a JSON body with `from`, `to`, `subject`, and `text`/`html`. Check both HTTP status and API errors before reporting success. Never expose the token in browser code or make a public unrestricted email relay. Validate recipients and escape untrusted template values. Better Auth's verification/reset callbacks can await this helper; see [Better Auth](auth/better-auth.md).

For Workers, add a send binding:

```json
{
    "send_email": [{ "name": "EMAIL" }]
}
```

Then, inside the authorized server operation:

```ts
await env.EMAIL.send({
    from: 'welcome@your-verified-domain.example',
    to: recipient,
    subject: 'Welcome',
    text: 'Your account is ready.'
});
```

Replace the sender with an onboarded domain. The binding needs no REST token in application code; sender/recipient restrictions still apply. This structured API is distinct from older raw MIME-only examples. Incoming mail uses a separate Worker `email()` handler and routing rules. Test failure handling without logging tokens or private email content. See [Workers sending API](https://developers.cloudflare.com/email-service/api/send-emails/workers-api/) and [binding restrictions](https://developers.cloudflare.com/email-service/configuration/send-bindings/).

## Durable Objects for WebSockets

Use a Durable Object per room, tenant or coordination entity for realtime collaboration. A Worker accepts the HTTP upgrade route, checks session/origin and room membership, then forwards to the selected object, for example `env.ROOMS.getByName(roomId).fetch(request)`. Do not use one global object for unrelated rooms.

The object creates a `WebSocketPair`, registers the server side with `this.ctx.acceptWebSocket(server)` and returns the client side in a 101 response. Implement `webSocketMessage`, `webSocketClose` and error handling. Prefer this hibernation API: clients stay connected while the object sleeps. Store durable state in storage; restore connection metadata through serialized attachments because in-memory fields reset after hibernation. Validate message size/schema and authorize operations; add reconnect/backoff and state resynchronization on the browser. See [Durable Object WebSockets](https://developers.cloudflare.com/durable-objects/best-practices/websockets/).

Deploy the class in a Worker, with a namespace binding and initial SQLite class migration:

```json
{
    "durable_objects": {
        "bindings": [{ "name": "ROOMS", "class_name": "Room" }]
    },
    "migrations": [{ "tag": "v1", "new_sqlite_classes": ["Room"] }]
}
```

The Worker must export the `Room` class. Preserve existing migration tags and append later changes rather than rewriting deployed history. Durable Object class migrations are different from D1 SQL migrations. Pages cannot deploy a DO class itself: bind to the separately deployed Worker namespace (external binding uses `script_name`) and forward upgrades through supported routing. See [Pages DO bindings](https://developers.cloudflare.com/pages/functions/bindings/#durable-objects) and [DO configuration](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/).

## Verify the intended deployment

Check/build the installed app with the selected LTS. Test platform bindings under the adapter's local runtime, migrations on a disposable database, API authorization, cron idempotency, email errors and WebSocket reconnect/hibernation. Verify Preview and Production bindings separately. A local Vite run does not prove deployed bindings or runtime compatibility. Never report resources provisioned, messages delivered or cron executed from configuration checks alone.

For the ipf.sk gateway Worker, KV ban/allow-list bindings and IPFS/IPNS URLs, see [IPFS gateway](ipfs.md).
