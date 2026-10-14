# Authentication provider: Better Auth

Better Auth is optional. Add it when the user requests application accounts or server sessions. A browser wallet connection alone does not create a Better Auth session. Keep credentials, database adapters and the auth instance server-only; browser code imports a separate client.

## Installer or existing application

The current ĐApp Starter blockchain toolset choices (`connect-protocol`, `corebc`, custom package) do not configure Better Auth. During scaffolding, select Better Auth if the underlying Svelte CLI offers it; otherwise finish the starter and add it inside the installed application:

```sh
npx sv add better-auth
```

The [Svelte CLI addon](https://svelte.dev/docs/cli/better-auth) supplies Drizzle integration, enables email/password and can generate demo login/signup pages. `npx sv add better-auth="demo:password"` selects those demos; `demo:github` selects GitHub OAuth demos, which still need provider credentials. Inspect the generated diff before merging it with existing auth, DB and hooks. Preserve `.npmrc` registry settings; see [starter repair instructions](../starter-addons.md). Do not run installation in the template-only mota-dapp repository.

For manual integration after scaffolding, install `better-auth` plus the chosen DB driver/adapter dependencies in the installed app. Select compatible releases using their peer requirements. The upstream `auth init` CLI currently targets Next.js with SQLite: use the Svelte addon or the manual SvelteKit path here instead. See [CLI documentation](https://better-auth.com/docs/concepts/cli).

The workspace's older `mota-addon-passkey` is not a drop-in Kit 3 installer: inspect its templates, private-env imports and EJS shell scripts before use. Static inspection with the MOTA runner is optional:

```sh
node bin/addon.mjs bchainhub/mota-addon-passkey passkey install --dry-run
```

Update incompatible templates and convert interpolated shell scripts to plain scripts with validated data first. Then use the runner's normal direct application and explicit execution controls as described in [starter and addons](../starter-addons.md); do not present legacy addon commands as a completed auth setup.

## Database and migrations

| Connection | Integration choice |
| --- | --- |
| PostgreSQL | Existing server-side `pg` pool, or Drizzle/Prisma adapter |
| MySQL | Server-side `mysql2` pool, or Drizzle/Prisma adapter |
| SQLite | Supported SQLite driver, or Drizzle/Prisma SQLite adapter; requires persistent storage |
| Cloudflare D1 | Supported D1/Drizzle SQLite adapter for the deployed runtime, constructed from request bindings |
| MongoDB | Better Auth MongoDB adapter with the selected release's driver requirements |

Reuse an appropriate existing DB connection; Better Auth owns its auth tables, not the entire application database. ORM adapters and dialect must match the actual database. Node filesystem SQLite does not become durable storage on an ephemeral/edge host. Node-only pools and native dependencies need a compatible runtime; inspect adapter support before deploying. See [Better Auth database concepts](https://better-auth.com/docs/concepts/database) and [project DB/migration conventions](../databases.md).

Generate the schema after configuring all chosen plugins: they may add tables and fields. The Better Auth CLI `generate` supports adapter-specific output; `migrate` applies changes only through its built-in Kysely path. For Drizzle or Prisma, use their schema generation and migration tools. Run the selected CLI's `--help` and point `--config` at the actual auth config; do not assume defaults find it.

For example, with a selected compatible `auth` CLI installed as a dev dependency, `npx auth generate --config ./auth-cli.ts` generates from a CLI-loadable configuration. Share database/plugin configuration through a factory, and supply `process.env` values in `auth-cli.ts`; keep SvelteKit virtual env modules, request-only helpers and cookie integration out of that CLI entrypoint. Match its plugins and dialect to the runtime config. Never commit real secrets or credential-bearing URLs.

Commit schema and migration SQL for visible migration history, or keep the migration directory in a private repository/artifact and configure the deployment job to retrieve it. Excluding migrations from Git does not apply them remotely. Apply to an explicitly selected DB using that adapter's migration tool; generate/review first, then migrate. See [public/private migrations](../databases.md).

## SvelteKit 3 manual implementation

Add private `BETTER_AUTH_SECRET` (a random secret of at least 32 characters) and `BETTER_AUTH_URL` (the exact application origin) to the project's typed `src/env.ts` declaration and deployment environment. Keep them out of `vite.config.ts` site config. This example assumes a Node PostgreSQL pool exported as `pool` from your server DB helper; adapt that export and database option to the selected driver:

```ts
// src/lib/server/auth.ts
import { betterAuth } from 'better-auth';
import { sveltekitCookies } from 'better-auth/svelte-kit';
import { getRequestEvent } from '$app/server';
import { BETTER_AUTH_SECRET, BETTER_AUTH_URL } from '$app/env/private';
import { pool } from '#lib/server/db.js';

export const auth = betterAuth({
    database: pool,
    secret: BETTER_AUTH_SECRET,
    baseURL: BETTER_AUTH_URL,
    basePath: '/api/auth',
    emailAndPassword: { enabled: true },
    plugins: [sveltekitCookies(getRequestEvent)]
});
```

Keep `sveltekitCookies` last when using server actions that set auth cookies. Wire one handler into the existing hook chain, before a generic API gate or catch-all can block `/api/auth/*`. This fragment maps the result into MOTA's existing session shape; retaining only `result.session` would lose the user expected by the header:

```ts
// Export from the chosen auth module, or compose once in hooks.server.ts.
import type { Handle } from '@sveltejs/kit/hooks';
import { building } from '$app/env';
import { svelteKitHandler } from 'better-auth/svelte-kit';
import { auth } from '#lib/server/auth.js';

export const authHandle: Handle = async ({ event, resolve }) => {
    if (building) return resolve(event);
    const result = await auth.api.getSession({ headers: event.request.headers });
    event.locals.session = result ? {
        user: result.user,
        expires: result.session.expiresAt.toISOString(),
        provider: 'better-auth'
    } : null;
    return svelteKitHandler({ event, resolve, auth, building });
};
```

Keep locale, geo, API and other existing handles when composing with `sequence`. If using MOTA's module loader, the selected module's `server.ts` exports `authHandle`; its browser `index.ts` implements the login/registration/logout flows used by menu actions. Do not mount the same auth handler through both routes. Keep `/api/auth` unlocalized; login/signup pages may live under `src/routes/[[lang]]/`. Update `App.Locals` and layout serialization deliberately if adding richer auth data; expose only needed user fields. Protect server loads/actions using the server session, not menu visibility or client wallet state. See [SvelteKit integration](https://better-auth.com/docs/integrations/svelte-kit).

```ts
// src/lib/auth-client.ts
import { createAuthClient } from 'better-auth/svelte';
export const authClient = createAuthClient();
```

Login and signup forms call `authClient.signIn.email({ email, password })` and `authClient.signUp.email({ name, email, password })`; display returned errors, clear pending state and navigate only on success. `authClient.useSession()` supplies the Svelte session store; `authClient.signOut()` logs out. Add labels and translated errors using existing i18n conventions. Configure actual delivery callbacks before enabling email verification or password reset; enabling email/password does not provision mail. See [email/password authentication](https://better-auth.com/docs/authentication/email-password).

## CorePass passkeys

Use [better-auth-corepass-passkey](https://github.com/CorePass/better-auth-corepass-passkey) for CorePass's passkey-first registration and signed profile enrichment. Install compatible versions of `better-auth-corepass-passkey` and `@better-auth/passkey`; inspect their peer requirements against Better Auth.

Add server plugins in this order: optional `anonymous()` from `better-auth/plugins` for provisional account bootstrap, `passkey()` from `@better-auth/passkey`, `corepassPasskey()` from `better-auth-corepass-passkey`, and finally SvelteKit cookies. Set WebAuthn `rpID` to the relying-party hostname and `origin` to the exact HTTPS application origin (localhost development is separate). Configure the browser with `passkeyClient()` from `@better-auth/passkey/client` and the anonymous client plugin when using that bootstrap flow.

This plugin enforces a passkey-only policy. Do not simply append it to the email/password configuration above or promise that every login method can coexist unchanged. Choose the intended policy and test the complete registration/login path. Default authenticator restrictions target CorePass; changing `allowedAaguids` is a deliberate policy choice.

The default `finalize: 'after'` leaves registration provisional until signed enrichment succeeds; `immediate` has different enrichment/recovery availability. Use the plugin's verification, timestamp and profile-expiry handling. A supplied Core ID or unsigned profile is not proof of ownership. Required profile/email/age/KYC gates must match what the app actually needs and the chosen network.

Expose the public `/webauthn/data` and recovery aliases expected by CorePass using `handlePasskeyDataRoute` from `better-auth-corepass-passkey/passkey-data-route`. In the server hook, pass `event.request`, `handler: request => auth.handler(request)` and the configured `basePath`; return its response when present, otherwise continue through the normal auth handler. Do this before catch-all routing. Enrichment is a signed POST; the HEAD availability check is not an enrichment request.

Generate migrations with the passkey and CorePass plugins configured: include passkey tables, `corepass_profile` and `restore_challenge`, plus anonymous fields if used. Recovery follows the plugin's signed challenge/restore flow and enrollment of a new passkey. Read profiles through the authenticated session and honor their expiry; do not extend expired enrichment locally.

## Connect wallet login with server sessions

Use [better-connect](https://github.com/bchainhub/better-connect) when Connect wallet proofs should establish Better Auth accounts and sessions. This is separate from the template's browser `connect-protocol` toolset. Install `better-connect` alongside compatible Better Auth; verify selected chain dependencies against the deployment runtime, especially native verifier dependencies.

Server integration uses `betterConnect` and the required chain definitions from `better-connect`. Configure `origin` to the exact application origin. Its endpoint contract requires Better Auth's URL to be that origin plus `/api/auth`: preserve this base path. `allChains` is available, but configure only the chains the application needs. Generate plugin schema/migrations and use its durable adapter-backed request store in production; an in-memory store is for tests or single-process development.

On the Svelte client, add `betterConnectClient()` from `better-connect/client` to `createAuthClient({ plugins: [...] })`. `createBetterConnectHandoff` from that same export coordinates callbacks to `authClient.walletConnect.create({})`, `.approve(proof)` and `.redeem(credentials)`. Unwrap successful `.data` and propagate `.error` in each callback. Display the returned handoff URI as a QR/deep link and feed accepted transport packets to the handoff, following the selected release's API.

The initiating browser creates a request, the wallet signs the origin/nonce/account/chain-bound challenge, the server verifies and approves it, and the initiating browser redeems it for a cookie session. Approval itself does not log in a different browser. Keep the redemption secret in the initiating browser, out of QR codes, URLs and logs; the public challenge/pairing handoff is separate. Handle expiry, denial and one-time redemption. Wallet keys stay in the wallet. Final session creation requires the server; offline transport of a wallet response is not an offline server login.

## Verification

Check/build the installed application. Test signup, login, logout, protected server routes, expired sessions, invalid credentials/proofs and production cookies/origin settings. For CorePass, test provisional registration, enrichment, rejection and recovery on the intended network. For Connect, test approval versus redemption, expiry and replay. Confirm migrations on a disposable matching DB before the intended deployment target. Documentation or a successful build alone does not prove end-to-end login works.

For deployment bindings and sending verification/reset email from Pages or Workers, see [Cloudflare](../cloudflare.md).
