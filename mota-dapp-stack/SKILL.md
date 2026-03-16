---
name: mota-dapp-stack
description: |
  How to work with the ĐApp Starter (dapp-starter), addon CLI, and mota-dapp (SvelteKit) project structure.
  Use this skill whenever the user is working on or asking about SvelteKit routes, addons, npx addon,
  dapp-starter, start.mjs, vite.config.ts modules, i18n in src/i18n, env variables ($env/static/private,
  $env/dynamic/private, import.meta.env), default packages, addon script execution order, or the mota-dapp
  folder layout. Also use it for route conventions ([param], [[optional]], (group)), _config/_lang/_scripts
  in addons, CONTRIBUTING/Code of Conduct, and template updates.
compatibility: []
---

# MOTA ĐApp Stack – SvelteKit, ĐApp Starter, Addons

This skill describes the **ĐApp Starter** installer, default packages, the **addon** CLI and its exact execution flow, the **mota-dapp** (sveltekit-mota) project structure, SvelteKit routing and env rules, Vite config, and project governance (CONTRIBUTING, Code of Conduct). Use it when editing or scaffolding SvelteKit apps created with this stack.

---

## ĐApp Starter (dapp-starter)

- **One-shot installer:** `npx github:bchainhub/dapp-starter` (or clone repo, `npm install`, `node start.mjs`).
- **Template:** Default template is **mota-dapp** (repo name; local project may be `sveltekit-mota`). Override with `--template URL` or `-t URL`. Pin version: append `@version` to URL (e.g. `...mota-dapp.git@1.2.3`) or use `--template-version REF` / `--tv REF`.
- **Update from template:** From project root run `node /path/to/dapp-starter/start.mjs --update` (or `-u`). Template is copied over **except `vite.config.ts`**, which is always preserved. Script may offer a git checkpoint commit before overwriting.
- **What it sets up:** Runs `sv create`, installs base + addon tooling deps, writes `bin/addon.mjs`, maps `addon` in `package.json`, composes project README, and optionally translations, skills, template merge, license, first commit. Requirements: Node.js 18+, git, one of npm/pnpm/yarn/bun.

---

## Default packages (installer)

**Base dependencies** (added by `start.mjs` after `sv create`):

- `@blockchainhub/blo`, `@blockchainhub/ican`, `@tailwindcss/vite`, `tailwindcss`
- `blockchain-wallet-validator`, `device-sherlock`, `exchange-rounding`
- `lucide-svelte`, `payto-rl`, `txms.js`, `vite-plugin-pwa`, `zod`

**Addon tooling** (dev):

- `hygen`, `tiged`, `json5`, `ejs`, `prompts`

**Optional (if user opts in):**

- **Translations:** `typesafe-i18n`; scripts `i18n:extract`, `i18n:watch`, `typesafe-i18n`
- **Skills:** [skills.sh](https://skills.sh/) (e.g. MOTA Skills, custom repo)
- **Other:** `.editorconfig`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, issue templates, license

---

## Addon CLI

After installation, projects run:

```bash
npx addon <repo> <generator> <action> [options]
```

Examples:

```bash
npx addon bchainhub@mota-support auth install
npx addon owner/repo auth uninstall
npx addon owner/repo auth install -c
npx addon owner/repo auth install -d
npx addon owner/repo#v1.2.3 auth install
```

- **Versioning:** Pin with `#<ref>` (tiged): e.g. `owner/repo#v1.2.3`, `owner/repo#main`, `owner/repo#<commit-hash>`. Do **not** use `@1.2.3` (that's npm; addon uses `#`).
- **Options:** `-c`/`--cache`, `-d`/`--dry-run`, `-nt`/`--no-translations`, `-ns`/`--no-scripts`, `-nc`/`--no-config`.

---

## How the addon script works (exact order)

The addon script (`bin/addon.mjs`) runs in this order:

1. **Parse argv** – repo, generator, action, and flags (`-c`, `-d`, `-nt`, `-ns`, `-nc`). Cache dir: `.addon-cache/<repo_sanitized>`; else temp dir.
2. **fetchAddon()** – Clone the addon repo with **tiged** (mode `git`) into cache or temp. If `-c` and cache exists, skip clone.
3. **resolveActionDir()** – Path is `tmpDir/generator/action`. If that folder doesn't exist, exit with "Addon action not found".
4. **loadPrompts(actionDir)** – If `prompt.js` exists, load it (array or function). If function: call with `{ prompts, cwd, generator, action, repo }`. Result is `locals` for the rest of the run.
5. **runHygen(locals)** – Run Hygen with `locals`; only **\*.ejs.t** templates generate project files. Hidden files are not Hygen targets.
6. **runHiddenScript(actionDir, locals)** – Skip if `-ns`. For each dir in `[actionDir, actionDir/_scripts]` (if `_scripts` exists), look for `_scripts.ejs.sh` or `_scripts.sh`. First found: render EJS if needed, write to temp script, run with env `ADDON_REPO`, `ADDON_GENERATOR`, `ADDON_ACTION`, `ADDON_CONTEXT_JSON`, `ADDON_VAR_<name>`.
7. **applyHiddenConfig(actionDir, locals)** – Skip if `-nc`. For each dir in `[actionDir, actionDir/_config]`, look for `_config.ejs.json5` or `_config.json5`. First found: render EJS if needed, parse JSON5, apply `$remove` then deep-merge into the **modules** block in `vite.config.ts` (finds block via regex), write back. Config is **client-side** — no secrets.
8. **applyHiddenLang(actionDir, locals)** – Skip if `-nt`. Collect all _lang entries from `[actionDir, actionDir/_lang]`: in action root files named `_lang.<code>.json5` (or `.ejs.json5`); in `_lang/` folder files named `<code>.json5` (or with path suffix). For each: read, EJS if needed, parse JSON5, resolve `$path` (default `modules.<generator>`), apply `$remove`, deep-merge into `src/i18n/<lang>/index.ts` (parse TS object, merge, serialize back). Only normal `.ejs.t` and these merges write to the project; prompt.js, _scripts, _config are never copied.
9. **runTypesafeI18n()** – If any _lang file was applied (and not `-d`), run `npx typesafe-i18n --no-watch` in project cwd so i18n types update.
10. **Cleanup** – If not using cache, delete the temp clone.

Dry-run (`-d`): no script execution, no config write, no lang write; only Hygen and prompts run, and addon logs what it would do.

---

## Addon structure and file roles

Addon repo layout: `<repo>/<generator>/<action>/`. Hidden files can live in the **action root** or in **optional subfolders** `_scripts/`, `_config/`, `_lang/`.

```text
<generator>/<action>/
  prompt.js
  *.ejs.t
  _scripts.ejs.sh    or  _scripts/_scripts.ejs.sh
  _scripts.sh            _scripts/_scripts.sh
  _config.ejs.json5  or  _config/_config.ejs.json5
  _config.json5          _config/_config.json5
  _lang.sk.json5     or  _lang/sk.json5, _lang/en.ejs.json5, ...
  _lang.en.json5         _lang/en.json5
```

- **prompt.js** – Optional. Collects answers once; same values reused by Hygen, _scripts, _config. Export **array** of prompt definitions or **function** returning answers.

  **Example (array):**
  ```js
  export default [
    { type: 'text', name: 'provider', message: 'Auth provider' },
    { type: 'text', name: 'route', message: 'Route name', initial: 'auth' }
  ];
  ```
  **Example (function):**
  ```js
  export default async ({ prompts, cwd, generator, action, repo }) => {
    const answers = await prompts([{ type: 'text', name: 'provider', message: 'Auth provider' }]);
    return answers;
  };
  ```

- **\*.ejs.t** – Normal Hygen templates; **only these** generate project files. Example:
  ```text
  ---
  to: src/routes/auth/+page.svelte
  ---
  <h1>Hello</h1>
  ```

- **_scripts** – Executed after Hygen; never copied. EJS example:
  ```bash
  #!/usr/bin/env bash
  set -euo pipefail
  npm install <%= packageName %>
  echo "Route: <%= routeName %>"
  ```
  Env vars: `ADDON_CONTEXT_JSON`, `ADDON_REPO`, `ADDON_GENERATOR`, `ADDON_ACTION`, `ADDON_VAR_<NAME>`.

- **_config** – Merged into `modules` in `vite.config.ts`. **Client-only; no secrets.** Example with `$remove` and `$expr`:
  ```json5
  {
    "auth": { "enabled": true, "provider": "github" },
    "$remove": { "legacyAuth": true }
  }
  ```
  Raw TS expression: `"strategy": "$expr(resolveAuthStrategy())"` → written as TypeScript, not a string.

- **_lang** – Merged into `src/i18n/<lang>/index.ts`. `$path` (default `modules.<generator>`), `$remove` (array, string, or object). Example:
  ```json5
  { "$path": "modules.myAddon", "title": "Názov", "actions": { "submit": "Odoslať" } }
  ```
  With EJS: `"welcome": "Welcome to <%= featureName %>"`. Removing keys: `"$remove": ["oldTitle"]` or `"$remove": { "oldTitle": true, "nested": { "key": true } }`.

- **_migrations** – If present, copied into project; in `.gitignore` by default. Configure ORM (e.g. Drizzle `out: "./_migrations"`).

**These are never copied into the project:** `prompt.js`, `_scripts*`, `_config*`, `_lang*` files (only their **effects** — script run, config merge, lang merge — are applied).

---

## mota-dapp (sveltekit-mota) project structure

Template repo: **mota-dapp**; local app often **sveltekit-mota**. Typical layout:

```text
src/
  app.html
  app.d.ts           # Config, NavbarItem, LayoutData, $env/dynamic/private
  routes/
    +layout.svelte
    +error.svelte
    [[lang]]/        # Optional lang segment
      +layout.svelte
      +layout.server.ts
      +layout.ts
      +page.svelte
    api/[version]/   # e.g. /api/v1/ping
  lib/helpers/, lib/components/, lib/modules/, lib/server/
  i18n/en/, i18n/sk/, i18n-util.ts, i18n-svelte.ts, formatters.ts
static/
vite.config.ts
svelte.config.js
```

Site config is built in `vite.config.ts` and injected as `__SITE_CONFIG__`; types in `app.d.ts`; access via e.g. `getSiteConfig()` from `$lib/helpers/siteConfig`. Never put secrets in that config.

---

## SvelteKit routing

- **`[param]`** – Required dynamic segment (e.g. `[version]` → `/api/v1`).
- **`[[param]]`** – Optional dynamic segment (e.g. `[[lang]]` → `/` or `/en`).
- **`(group)`** – Layout group; does **not** add to URL (e.g. `(auth)/login` → path `/login`).

Files: `+page.svelte`, `+page.server.ts`, `+layout.svelte`, `+layout.server.ts`, `+layout.ts`, `+server.ts`, `+error.svelte`. Params in load/server as `event.params`.

---

## Environment variables

- **Server-only (secrets):** `import { X } from '$env/static/private'` or `$env/dynamic/private`. Declare in `app.d.ts`; never in client or in `define`.
- **Client/build:** `import.meta.env`, `VITE_*` only. Addon _config and `__SITE_CONFIG__` are client-visible — no secrets.

---

## Vite config (mota-dapp)

Plugins: `sveltekit()`, Tailwind, optional VitePWA. Client config object → `define: { __SITE_CONFIG__: JSON.stringify(siteConfigClient) }`. `modules` is the object addons merge into (via _config). Types in `app.d.ts`.

---

## Contributing and Code of Conduct

**Contributing (CONTRIBUTING.md):** Open an issue → fork and clone → create a branch → make changes → push and open a PR. Follow existing style; ensure tests/docs as needed. Contributions under same license as project (see `LICENSE`).

**Code of Conduct (CODE_OF_CONDUCT.md):** Community pledges a harassment-free, inclusive environment. Welcoming language, respect, constructive criticism, empathy. No trolling, harassment, or publishing others' private info. Reports to maintainers; appropriate response.

---

## Quick reference

| Topic | Use |
|-------|-----|
| New project | `npx github:bchainhub/dapp-starter` or clone + `node start.mjs` |
| Update from template | `node start.mjs --update` (keeps `vite.config.ts`) |
| Install addon | `npx addon <repo> <generator> <action>`; pin with `#tag` or `#branch` |
| Addon order | fetch → prompts → Hygen → _scripts → _config → _lang → typesafe-i18n |
| Required route param | `[param]` |
| Optional route param | `[[param]]` |
| Route group (no URL) | `(groupName)` |
| Server secrets | `$env/static/private`, `$env/dynamic/private` |
| Client env | `import.meta.env`, `VITE_*`; no secrets in _config or __SITE_CONFIG__ |
| i18n | `src/i18n/<lang>/index.ts`; addon _lang merges; then `typesafe-i18n --no-watch` |
| Migrations | `_migrations` (gitignored); set ORM `out` to `./_migrations` |

Use this skill whenever you edit or generate code in a SvelteKit project that uses the ĐApp Starter, addons, or the mota-dapp layout and conventions.
