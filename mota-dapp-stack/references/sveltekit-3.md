# SvelteKit 3 and template maintenance

Read package.json and the installed Kit declarations before adapting older addons. Current starter targets Kit ^3.0.0, TypeScript ~6.0.3, Svelte 5 and Vite 8; Node must satisfy both Kit and installed Vite engines (Kit minimum 22.17).

## Migration map

| Older convention | Current convention |
| --- | --- |
| svelte.config.js | Options passed to sveltekit({ ... }) in vite.config.ts; former kit options move to this object |
| $lib imports | #lib package subpath imports, typically #lib/helpers/name.js |
| $app/environment | $app/env |
| $app/stores | $app/state |
| $env/static/private or $env/dynamic/private | Typed declarations in src/env.ts; named imports from $app/env/private |
| $env public modules | Explicit public declarations; $app/env/public |
| .svelte-kit/tsconfig.json base | extends "$app/tsconfig"; generated base lives in node_modules after sync |
| $service-worker | $app/service-worker for native Kit workers |
| Hook types from @sveltejs/kit | Hook types from @sveltejs/kit/hooks |

The official migration tool is `sv migrate sveltekit-3`; run the reviewed/selected CLI version as part of an authorized migration, inspect its diff, then check/build. Starter update mode is not a substitute for migrating Kit 2 first. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3).

Migrate legacy project aliases to package imports: $lib → #lib, $components → #components, $modules → #modules and $data → #data. Remove their duplicate Vite aliases and TypeScript paths after migrating callers; preserve SvelteKit built-ins such as $app/*. Package imports must point to actual project paths, e.g.:

```json
{
    "imports": {
        "#lib": "./src/lib/index.js",
        "#lib/*": "./src/lib/*",
        "#data": "./src/data/index.js",
        "#data/*": "./src/data/*",
        "#components": "./src/lib/components/index.js",
        "#components/*": "./src/lib/components/*",
        "#modules/*": "./src/lib/modules/*"
    }
}
```

## Import shortcuts

| Shortcut | Source target | Example |
| --- | --- | --- |
| `#lib` | `src/lib/index.ts` barrel | `import { helper } from '#lib';` |
| `#lib/*` | Files under `src/lib` | `import { getSiteConfig } from '#lib/helpers/siteConfig.js';` |
| `#components` | `src/lib/components/index.ts` barrel | `import { Header } from '#components';` |
| `#modules/*` | Files under `src/lib/modules` | `import { walletAddress } from '#modules/auth/web3.js';` |
| `#data` | `src/data/index.ts` barrel | `import { settings } from '#data';` |
| `#data/*` | Files under `src/data` | `import { settings } from '#data/settings.js';` |

Examples using helper/settings require those exports to exist. For bare #lib/#data imports, export the desired values from the corresponding index.ts; an empty barrel exports nothing. Subpath imports access their files directly. Use .json for JSON imports, for example `import settings from '#data/settings.json';` in application code handled by the Vite toolchain.

ĐApp Starter adds these package.json mappings through applyKit3PackageDefaults() for both new projects and template updates, preserving existing custom mappings. Keep the corresponding index.ts files in the template and installed app. These standard package imports do not need duplicate Vite aliases or TypeScript paths. Migrate addon imports before using them rather than reintroducing the removed project aliases.

The .js import suffix is intentional for TypeScript source resolved by the toolchain. Leave unrelated package.json fields intact.

## Typed environment

Merge declarations into existing src/env.ts rather than overwriting addon variables:

```ts
import { defineEnvVars } from '@sveltejs/kit/env';

export const variables = defineEnvVars({
    DATABASE_URL: {
        schema: (value) => {
            if (!value) throw new Error('DATABASE_URL is required');
            return value;
        }
    },
    PUBLIC_SITE_URL: { public: true, schema: (value) => value },
    CAPTURE_COUNTRY: { schema: (value) => value === 'true' }
});
```

Server: `import { DATABASE_URL } from '$app/env/private'`. Client-safe: `import { PUBLIC_SITE_URL } from '$app/env/public'`. A name beginning PUBLIC_ does not replace the explicit public declaration. `static: true` opts into build-time inlining; use it only when needed. src/env.ts describes values, not credentials. Keep real credentials in ignored local env files or deployment secret configuration. [Typed private env](https://svelte.dev/docs/kit/$app-env-private).

In Vite config use Node/build env loading; app virtual env modules belong to application code. Only Vite VITE_* values are exposed through import.meta.env by default. Everything in `__SITE_CONFIG__` and addon `_config` is client-visible. Preserve Vite's import.meta.env.DEV; use the separate `__DEV_MODE__` flag for DEV_MODE=1 preview behavior.

## Svelte and routes

Use $props, $state, $derived and $effect in runes components; dynamic components render directly (`<item.icon />`). Do not blindly convert working legacy components during an unrelated fix. Read page state from $app/state; clone page.url with `new URL(page.url)` before changes. Use snippets/children for new layouts. Hook types and sequence come from @sveltejs/kit/hooks.

Routes stay file-based: `[param]` required, `[[lang]]` optional, (group) omitted from the URL, `[...rest]` catch-all. Use generated ./$types for load handlers. Preserve server-only import boundaries and request-local auth/locale data.

## PWA verification

Current template uses VitePWA, manual manifest markup in app.html, and registration on mount outside development. Its compatibility mapping removes unused transformIndexHtml hooks and limits the build plugin to the client environment. Check this behavior against the installed VitePWA version before changing it; do not swap in an integration whose peer range excludes Kit 3.

The October 14, 2026 browser check found a valid manifest, an active worker and cached JS, but offline homepage reload failed: no HTML navigation handler/fallback existed, and logo SVGs were not cached. This is a known gap, not a claim of full offline support. When fixing it, decide which public pages or offline shell to cache; avoid caching authenticated HTML, API responses, payment status or credentials. Authenticated actions and live settlement require connectivity. Verify first-load install, control after reload, manifest/icons, navigation offline, unvisited routes, update behavior and reconnection in production preview. Keep template and demo changes paired.

For Cloudflare build-time Node selection and latest-LTS guidance, see [Cloudflare](cloudflare.md#node-build-version-prefer-the-latest-lts).
