# Existing MOTA helpers

Reusable utilities live in `src/lib/helpers/`. Import from `#lib/helpers/<name>.js` (e.g. `import { getSiteConfig } from '#lib/helpers/siteConfig.js'`). Installed apps declare `#lib` imports in package.json; preserve these mappings.

| Helper | Purpose |
| -------- | -------- |
| **siteConfig** | Client-safe site config from `__SITE_CONFIG__`. `getSiteConfig()` returns cached `Config`; `isApiVersionEnabled(version, api?)` checks `api.activeVersions` (string or string[]). |
| **modules** | Module enablement from site config. `getPrimaryAuthModuleName()` → e.g. `auth.passkey`; `isModuleEnabled(module)` (e.g. `auth.passkey`, `banking`); `getModuleRoute(module)` → a module route; `loadModule(module)` → dynamic import when enabled (client-safe). |
| **modules.server** | Server-only: `loadAuthModuleServer(module)` loads `#modules/auth/**/server.ts` (e.g. for `authHandle` in hooks); use in `hooks.server.ts` so private env never hits client. |
| **keys** | Clipboard and download: `copyToClipboard(text, successMsg, errorMsg)` (toast on success/fail); `downloadTextFile(content, filename, mimeType)`; `downloadGPGKey(keyContent, keyName)` → `.asc` file. |
| **i18n** | Locale and translations: `detectLocale(navigatorLanguage, availableLocales)`; `getAvailableLocales()` / `getAvailableLocalesWithNames()` from config; `getLocale()`; `applyLocale(locale)` (load + set + persist); `loadLocaleAsync(locale)` (preload); `createI18nLayoutLoad()` for layout; `t(key, LL, vars?)` resolve dotted key on `$LL` with optional vars; `deepMergeDict(base, overlay)` for locale overlays. |
| **nav** | Navbar/footer visibility: `ItemWithShow` has optional `show` rule string. `shouldShowItem(item, context)`; `filterNavItemsByShow(items, context)`; `filterNavItemsByAuth(items, context, authEnabled)`. Context: `{ loggedIn, connected, authIn }`. |
| **showRule** | Evaluate show rules: `evaluateShowRule(show, context)`. Terms: `always`, `hide`, `loggedIn`, `loggedOut`, `signedOut`, `connected`, `disconnected`, `authIn`, `authOut`. Combine with `and` / `or` and `()`. |
| **shortFormat** | Shorten identity strings for UI: `shortFormat(value, kind)` with `kind`: `'core'` (Core ID, 4+⋯+4 upper), `'wallet'` (0x…), `'user'` (user id). |
| **storageKeys** | localStorage keys: `STORAGE_KEY_LOCALE`, `STORAGE_KEY_THEME`; `getStoredLocale` / `setStoredLocale`, `getStoredTheme` / `setStoredTheme` / `removeStoredTheme` (with legacy migration). |
| **redirect** | Post-auth redirect: `getRedirectTo(configFallback?, queryName?, options?)` — reads `?redirect=` (or custom param), validates same-origin path, strips `//`; optional `keepOtherQueryParams`. |
| **dev** | `isDev` — true when SvelteKit `dev` or the explicit `__DEV_MODE__` build flag is true (e.g. Cloudflare Preview with `DEV_MODE=1`). |
| **geo** | **Server-only.** `getGeoData(event)` — sets `event.locals.country` / `event.locals.city` from Cloudflare `platform.cf`, Vercel/Netlify headers; respects `CAPTURE_COUNTRY` / `CAPTURE_CITY` in private env. |
| **icon** | `asDynamicIcon(icon)` — cast for dynamic icon component (e.g. when icon comes from config). |
| **ipfs** | `getIpfsUrl(url, ns?)` — convert `ipfs://<cid>` to gateway URL (e.g. `https://ipf.sk/<cid>` or `https://ipf.sk/<ns>/<cid>`). |
| **coreIdValidation** | `validateCoreId(id, updateField, messages?)` — validate Core ID via blockchain-wallet-validator (network xcb); returns `{ isValid, isEnterprise, description, error, field }`; enterprise addresses rejected for payments. |
| **urlValidation** | `validateSecureUrl(url)` — HTTPS + valid domain; skips in development. |
| **globalPolyfill** | Sets `window.global = window` when `global` is undefined (browser). Side-effect import. |
| **navigation** | `goUpOneLevel(currentPath)` — navigate to parent path (e.g. `/exchange-rates/usd` → `/exchange-rates`), preserves query params; uses SvelteKit `goto`. |
| **euclideanDistance** | `hexToRgb(hex)`; `calculateColorDistance(color1, color2)` — RGB Euclidean distance for HEX colors. |
| **blo** | `getIcon(address)` — returns identicon URL (normalize Core IDs to lowercase before calling) from `@blockchainhub/blo` for a wallet/address. |

**How to use (typical):**

- **Config / modules:** Call `getSiteConfig()` in components or load functions; use `isModuleEnabled('auth.passkey')` before loading auth UI; in `hooks.server.ts` use `loadAuthModuleServer(getPrimaryAuthModuleName())` for the auth handle.
- **i18n:** In `[[lang]]/+layout.ts` use `loadLocaleAsync(data.locale)`; in layout load return `{ locale }`; use `$LL` and `t('key', $LL, { var: value })` when you need manual key resolution.
- **Nav:** Get items from config, then `filterNavItemsByAuth(items, showRuleContext, authEnabled)` and render; each item can have `show: "loggedIn and connected"` etc.
- **Redirect:** After login/connect call `getRedirectTo(configRedirect)` and `goto(url)` or redirect response.
- **Storage:** Use `getStoredLocale()` / `setStoredLocale()` for language; `getStoredTheme()` / `setStoredTheme()` for theme.
- **Geo:** In `hooks.server.ts` handle, call `getGeoData(event)` so `event.locals.country` / `city` are set for server load/layout.

---

Check the current helper implementation before changing a caller. In particular, the current SSR i18n helper writes a shared store; new request-specific translations should use request-local dictionaries, avoiding locale leakage between concurrent requests.
