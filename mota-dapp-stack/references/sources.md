# Sources and verification notes

Prefer the installed package's exports and types over stale README examples. Local source may differ from published releases.

| Area | Reviewed source |
| --- | --- |
| Starter | [dapp-starter](https://github.com/bchainhub/dapp-starter): README.md, start.mjs, scripts/addon.mjs |
| Template | [mota-dapp](https://github.com/bchainhub/mota-dapp): vite.config.ts, src/app.d.ts, components, locale layout and helpers |
| Web3 connector | Local `mota-dapp` / `mota-demo`: `src/lib/modules/auth/web3.ts`, `discovery.ts`, `nav-actions.ts`, header/dropdown components, `src/app.d.ts`, demo `vite.config.ts` and behavior tests. User-supplied local `cip/cip/interface/cip-6963.md` (draft, 2026-10-30); [EIP-6963](https://eips.ethereum.org/EIPS/eip-6963). Local changes may precede publication. |
| SvelteKit | [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3), [private env](https://svelte.dev/docs/kit/$app-env-private), installed @sveltejs/kit types |
| Database | [Drizzle config](https://orm.drizzle.team/docs/drizzle-config-file), [migrations](https://orm.drizzle.team/docs/migrations), [PostgreSQL drivers](https://orm.drizzle.team/docs/get-started-postgresql); local passkey addon DB/script/prompt templates |
| CoreBC | [CorePass/corebc.js](https://github.com/CorePass/corebc.js): README, package exports and provider/unit APIs |
| Connect | [bchainhub/connect.js](https://github.com/bchainhub/connect.js): README, package.json, docs/BROWSER.md, docs/WALLETS.md and operation export availability |
| ICAN | [ican.js](https://github.com/bchainhub/ican.js): README, declarations and installed package; [ICAN concept](https://payto.onl/solutions/ican) |
| Blockies | [blo](https://github.com/bchainhub/blo): README and installed BloOptions |
| Payment URLs | [payto-rl](https://github.com/bchainhub/payto-rl): README, setters and installed package |
| FinTag | [concept](https://payto.onl/solutions/fintag), local paytoonl-web FinTag.tsx; payto generate.helper.ts and fintag.helper.ts array parser |
| PayPass | [PayTo:Money](https://payto.money/), user-provided UI reference, local payto constructor/generator source |
| SMS | [TxMS concept](https://payto.onl/solutions/txms), [txms.js](https://github.com/bchainhub/txms.js): README and src/index.ts; count takes raw hex despite ambiguous README wording |
| CorePort | Workspace coreport: (portal)/generate/+page.svelte, link/[...rest]/+page.svelte, ramp/[...rest]/+page.svelte, helpers/linkPrefill.ts, server/webhookSignature.ts and docs/webhook page |
| Core services | User-supplied mainnet/Devin hosts and [Postman collection](https://www.postman.com/core-labs/core-blockchain); endpoints were not used to broadcast transactions |
| Risk reports | [Snyk](https://skills.sh/bchainhub/mota-skills/mota-dapp-stack/security/snyk), [Gen](https://skills.sh/bchainhub/mota-skills/mota-dapp-stack/security/agent-trust-hub); fetched rendered reports showed Medium |
| Better Auth | [Repository](https://github.com/better-auth/better-auth), [SvelteKit integration](https://better-auth.com/docs/integrations/svelte-kit), [installation](https://better-auth.com/docs/installation), [CLI](https://better-auth.com/docs/concepts/cli), [Svelte addon](https://svelte.dev/docs/cli/better-auth) |
| Auth plugins | [CorePass passkey](https://github.com/CorePass/better-auth-corepass-passkey): README, exports and route helper; [better-connect](https://github.com/bchainhub/better-connect): README, server/client exports, request store and endpoint contract |
| CDN icons | [corecdn](https://github.com/bchainhub/corecdn): README and exported asset directories; hosted at [corecdn.info](https://corecdn.info) |
| Cloudflare | [Pages build image](https://developers.cloudflare.com/pages/configuration/build-image/), [bindings](https://developers.cloudflare.com/pages/functions/bindings/), [D1](https://developers.cloudflare.com/d1/), [KV](https://developers.cloudflare.com/kv/), [cron](https://developers.cloudflare.com/workers/configuration/cron-triggers/), [Email Service](https://developers.cloudflare.com/email-service/), [Durable Objects](https://developers.cloudflare.com/durable-objects/) |
| IPFS gateway | [ipfs-worker](https://github.com/core-laboratories/ipfs-worker): workspace README.md, src/index.ts and wrangler.jsonc; [ipf.sk](https://ipf.sk) gateway URL formats |
