# ĐApp Starter, npm configuration and addons

Source: dapp-starter README.md, start.mjs and scripts/addon.mjs. Code takes precedence when help text is stale. Installing or updating is conditional on the user's task, not required for ordinary edits.

## Installer CLI

Use a reviewed local checkout and selected ref of bchainhub/dapp-starter. From that checkout, `node start.mjs` starts the interactive installer. The documented remote distribution is the dapp-starter GitHub package; do not execute a moving remote installer just because a document mentions it.

| Invocation/option | Meaning |
| --- | --- |
| node start.mjs | Interactive sv scaffolding and setup |
| --template URL / -t URL | Alternate template; default bchainhub/mota-dapp |
| --template-version REF / --tv REF | Template ref when URL has no @version suffix |
| URL ending .git@VERSION | Starter template version syntax; distinct from addon #ref |
| --update / -u | Run from existing app root; overwrite template files except preserved config/env declarations |

Setup selects package manager, template, translations, skills, optional shared assets, blockchain toolset, license and optional initial commit. Toolset choices are None, Connect.js (`connect-protocol`), CoreBC (`corebc`), or a user-selected npm package. Neither library is universally required. Base runtime packages currently include @blockchainhub/blo, @blockchainhub/ican, @tailwindcss/vite, tailwindcss, blockchain-wallet-validator, device-sherlock, exchange-rounding, @lucide/svelte, payto-rl, txms.js, vite-plugin-pwa and zod. Addon tooling includes hygen, json5, ejs and prompts; current retrieval uses Git directly (older installations may still include tiged). typesafe-i18n is optional installer setup, but template translation imports need generated locale support in the installed app.

Update example from an existing app root (replace the path and ref with the selected checkout/ref):

```sh
node /path/to/reviewed/dapp-starter/start.mjs --update --tv SELECTED_REF
```

It preserves vite.config.ts and existing src/env.ts, rejects unmigrated Kit 2 apps, and can offer a checkpoint commit with `git add -A`. Review the working tree first: that choice may stage unrelated work or secrets. Do not select a commit or overwrite merely because the installer defaults to it. After update, review diffs, install with the app's manager, sync/check/build, and apply any needed Vite changes manually.

New and updated projects receive `#lib/` `#lib/*` → `src/lib` and `#data/` `#data/*` → `src/data` package imports. See [import shortcuts](sveltekit-3.md#import-shortcuts) for mappings and usage.

## Review package updates with npm-check-updates

Project dependencies and devDependencies can be checked and updated with tools such as [npm-check-updates](https://www.npmjs.com/package/npm-check-updates) (`ncu`). Recommend interactive review rather than applying every available upgrade:

```sh
ncu -i
```

Review the affected libraries and proposed versions, select which packages to update, then confirm. Interactive mode writes the selected version changes to package.json. If `ncu` is not installed, use `npx npm-check-updates -i`; do not use `npx ncu`, which is not the documented package name. Check the tool's own Node requirements before choosing its release. See the [upstream usage guide](https://github.com/raineorshine/npm-check-updates#interactive-mode).

After selecting updates, run the project's package-manager install command (for npm, `npm install`) to update installed packages and the lockfile, then run check/build and relevant behavior tests. Preserve `legacy-peer-deps=true` and install required missing peers explicitly as described below. Review major-version migrations and compatibility across SvelteKit, Vite, adapters and plugins; keep TypeScript on a release that supplies the JavaScript API required by the installed Kit version. A newer version is not automatically compatible or a proven security fix.

Run updates in the installed application. Mirror intentional shared dependency choices in dapp-starter defaults; keep installations and lockfiles out of the template-only mota-dapp repository. Review package.json/lockfile diffs and preserve unrelated work.

## Existing .npmrc and install errors

The npm installer adds/restores `legacy-peer-deps=true` after merging the template. This accepts narrower/older peer declarations (for example VitePWA versus Vite); it neither upgrades vulnerable packages nor guarantees runtime compatibility.

1. Read the actual error, package manager and lockfile. Check Node and peer ranges. Fix a wrong Node version, invalid registry, credentials or genuinely incompatible API directly.
2. Inspect only relevant `.npmrc` settings; avoid printing auth tokens. Preserve scoped registries, registry URLs, env placeholders and other project settings. Do not replace the file with a one-line printf, delete it, or modify global npm configuration as a routine fix.
3. If this app intentionally uses the starter's compatibility setting, check `npm config get legacy-peer-deps --location=project`. If missing, merge it into the project config with `npm config set legacy-peer-deps=true --location=project`. Record the specific peer mismatch and verify check/build.
4. Use the same setting for npm ci as for lockfile creation. Keep the lockfile, and do not mix managers to bypass a failure. A config containing tokens must not be committed; use environment substitution for credentials. A non-secret compatibility setting can be committed for CI consistency.
5. `npm warn gitignore-fallback` is a packaging notice, not a failed install; add .npmignore only when publishing needs a deliberate exclusion policy.

### Missing peer packages with legacy-peer-deps

Keep the project's intentional `legacy-peer-deps=true`. npm ignores peer dependencies when constructing the installation tree with this setting, so a package can install successfully yet fail later with `ERR_MODULE_NOT_FOUND`. Resolve the missing package explicitly instead of removing the setting or disabling `engine-strict`. See [npm legacy-peer-deps](https://docs.npmjs.com/cli/v11/using-npm/config/#legacy-peer-deps).

Read the failing import and identify its owner. Inspect that owner's installed package.json peerDependencies, or use `npm view PACKAGE@SELECTED_VERSION peerDependencies --json`. Install only the required compatible peer as a direct dependency: build tools/adapters belong in devDependencies; packages required by the deployed application at runtime belong in dependencies. Do not add every peer or guess a version from the newest release. Ensure the build environment installs devDependencies.

For example, Cloudflare adapter-auto may install `@sveltejs/adapter-cloudflare` during adaptation and then fail because it imports missing `wrangler`. Install the latest Wrangler release explicitly:

```sh
npm install -D wrangler@latest
npm run check
npm run build
```

Check that the latest Wrangler satisfies the selected adapter's peer range and Node engine requirements; resolve any incompatibility explicitly. Keep `.npmrc` unchanged, review package.json/lockfile changes, and commit them when preparing the deployment commit. A tool present globally or accidentally left in local node_modules does not satisfy a clean CI install. If the project tracks a lockfile, validate with `npm ci` using the same project configuration before rebuilding. Missing-package repair is separate from deprecation/security warnings and does not establish that all dependencies are vulnerability-free.

For a project committed to Cloudflare, explicitly installing the compatible `@sveltejs/adapter-cloudflare` alongside Wrangler and selecting it in Vite can avoid build-time adapter installation; preserve Kit 3's existing `sveltekit({ adapter: adapter() })` structure. This provider-specific change is optional; adding the missing Wrangler peer does not require replacing adapter-auto. See [Cloudflare adapter](https://svelte.dev/docs/kit/adapter-cloudflare) and [Cloudflare deployment guidance](cloudflare.md).

The current new-project installer may remove a scaffold .npmrc for non-npm managers; do not copy that behavior into repairs of an existing app with custom settings. Review manager-specific configuration instead.

## Addon CLI

Use the project's installed runner. Applying directly to the current project is the default; `--dry-run` optionally performs a **static inspection** instead:

```sh
node bin/addon.mjs owner/repo generator install --dry-run
```

The report shows the normalized repository, resolved full commit SHA, files, static destination hints, declared packages/migrations and enabled stages. Dynamic output paths and arbitrary code effects remain unknown until execution. Inspection does not import prompt.js, render EJS, execute hooks/scripts, install dependencies, regenerate translations or write project files. Fetching source uses a temporary directory outside the project.

Apply directly using the selected repository/ref:

```sh
node bin/addon.mjs owner/repo generator install --answers '{"route":"account"}'
```

`--apply` is an optional explicit alias for the default behavior and cannot be combined with `--dry-run`. `--commit FULL_SHA` optionally pins the full SHA printed by inspection; without it, the requested repository ref (or default HEAD) is resolved at invocation time. There is no project copy or execution sandbox: approved EJS, prompts and shell commands retain host permissions. Review their effects before enabling them.

Repository input accepts owner/repo, github:owner/repo, HTTPS URL, or an absolute local Git repository path. Addon refs use `#branch`, `#tag` or `#commit` (quote the argument), not the installer's `@version` template syntax. Generator and action must each be one letters/digits/hyphen/underscore segment. HTTPS URLs cannot contain credentials or query parameters. Git hooks/global config are disabled for retrieval; use a local reviewed checkout for sources needing custom authentication configuration.

`npx addon ...` remains a convenience alias only when it resolves to this installed runner. Prefer `node bin/addon.mjs` to avoid an unintended npm-registry fallback.

### Stage controls

| Option | Effect |
| --- | --- |
| No execution flag, or `--apply` | Render/generate files and merge enabled config/translations directly |
| `--commit SHA` | Optionally pin an exact full commit instead of resolving the repository ref |
| `--dry-run`, `-d` | Static inspection only; no addon execution or project writes |
| `--answers JSON` | Non-secret prompt data, supplied without executing prompt.js |
| `--allow-prompts` | Execute reviewed prompt.js; explicit --answers values override collected values |
| `--allow-scripts` | Run the selected reviewed plain hidden shell script after file/package stages |
| `--allow-expressions` | Permit raw `$expr(...)` config/translation values |
| `--allow-packages` | Install exact registry versions declared in addon.json; npm lifecycle scripts disabled |
| `--allow-package-scripts` | Also permit npm dependency lifecycle scripts; requires --allow-packages |
| `--run-migrations --migrations-only --migration-target NAME` | Separate migration-only invocation with an explicit named target |
| `--overwrite` | Allow generated files to replace existing files; template force does not bypass this |
| `--cache`, `-c` | Cache source Git objects by normalized repository and full commit outside the project during apply |
| `--no-scripts`, `-ns` | Skip the hidden script stage |
| `--no-config`, `-nc` | Skip hidden config merge |
| `--no-translations`, `-nt` | Skip hidden translation merge |

Plain _config/_lang merges run during application (the default); EJS used in them is executable code. Source symlinks/submodules are rejected. Destination validation rejects absolute paths, .. components, Windows/backslash paths, symlink/hard-link destinations, .git, node_modules, .addon-cache and overwriting the runner itself. All runner-managed output is validated before the first write. This constrains the file writer, not arbitrary explicitly enabled JavaScript or shell code.

Raw shell `sh`/`setup` template hooks, addon index.js, project .hygen.js and unsafe YAML tags are not executed by the controlled template renderer. The installed Hygen front-matter/context/injection utilities are used without invoking its CLI engine. Reviewable output is kept in memory until validation, then written directly into the project. File application is not a multi-file transaction; inspect the working tree if a later I/O or explicit command fails.

### Exact source and cache identity

Each invocation resolves the selected branch/tag to a commit and reports it. Without --commit, a later invocation can resolve to a newer commit. With optional --commit SHA, apply fetches **that SHA directly**, even if the branch moves afterward. Cache keys include repository and full commit; cached Git objects are integrity-checked, and blob IDs are verified before materializing addon source. No working checkout, Git filters or addon package lifecycle scripts run during fetching. A cache is not evidence that code is trustworthy. Current apply still verifies/fetches the selected commit, so the cache is not an offline-install guarantee.

## Addon action structure

Both generator/action and _templates/generator/action layouts are supported:

```text
generator/install/
    addon.json
    prompt.js
    page.svelte.ejs.t
    other.ts.t
    _scripts/_scripts.sh
    _config/_config.json5
    _lang/en.json5
```

Normal `.t` templates (including `.ejs.t`) are rendered. Control files are not copied as generated output. Config and script search checks action root before the optional subfolder and uses the first supported candidate. Locale processing collects matching _lang files. The controlled renderer supports to, inject, before/after, prepend/append, at_line, eof_last, skip_if, unless_exists, force, from and message; unsupported attributes fail explicitly.

### Prompts and inputs

prompt.js exports either an array of `prompts` questions or an async function receiving `{ prompts, cwd, generator, action, repo, commit }`. It is imported only with --allow-prompts. Otherwise --answers supplies data directly. Values must be JSON-compatible and keys cannot override runner/template internals. Do not collect credentials: values can be available to subprocess environments and generated files.

```js
export default [
    {
        type: 'text',
        name: 'route',
        message: 'Route segment',
        initial: 'account',
        validate: (value) => /^[a-z][a-z0-9-]*$/.test(value) || 'Use a lowercase route segment'
    },
    {
        type: 'select',
        name: 'provider',
        message: 'Provider',
        choices: [{ title: 'Local', value: 'local' }, { title: 'External', value: 'external' }]
    }
];
```

Validate data for its destination, including values supplied through --answers. The runner checks generated paths independently. Serialize strings in TypeScript output rather than inserting them into quoted literals:

```text
---
to: src/routes/<%= route %>/+page.svelte
---
<h1>Account</h1>
```

```text
---
to: src/lib/account-config.ts
---
export const provider = <%- JSON.stringify(provider) %>;
```

A truthy legacy runMigration answer is refused during generation. Declare migration commands separately instead of embedding auto-migration into a prompt/default.

### Scripts and packages

Hidden scripts are opt-in and must be plain shell source. Shell templates containing `<%` interpolation are rejected, even with --allow-scripts. Convert `_scripts.ejs.sh` into `_scripts.sh` that reads validated/quoted environment data:

```bash
#!/usr/bin/env bash
set -euo pipefail
case "${ADDON_VAR_ROUTE:-}" in
    account|profile) printf 'Configured route: %s\n' "$ADDON_VAR_ROUTE" ;;
    *) printf 'Unsupported route\n' >&2; exit 1 ;;
esac
```

Context variables include `ADDON_CONTEXT_JSON`, `ADDON_REPO`, `ADDON_COMMIT`, `ADDON_GENERATOR`, `ADDON_ACTION` and `ADDON_VAR_<UPPERCASE_NAME>`. The runner passes data through environment variables, never EJS interpolation into shell source. Do not use eval or unquoted expansions in addon scripts. Explicitly permitted scripts still have full host permissions and can perform undeclared package installs or DB operations; stage flags cannot constrain arbitrary code without a sandbox.

Declare package installs in the action's static addon.json instead:

```json
{
    "version": 1,
    "description": "Account feature",
    "packages": {
        "dependencies": ["zod@4.6.7"],
        "devDependencies": []
    },
    "migrations": [
        {
            "name": "Account schema",
            "command": "node",
            "args": ["bin/migrate-account.mjs", "--target", "{{target}}"]
        }
    ]
}
```

Use actual selected versions; only exact registry package specs are accepted (no Git/file URLs, shell strings or floating ranges). --allow-packages currently supports npm projects; other managers must use their own reviewed package commands separately. npm is invoked with an argument array, --save-exact and --ignore-scripts unless lifecycle execution was explicitly allowed. No new permission is inferred from an addon README.

### Databases and migrations

A generated _migrations folder is SQL/history, not a live database connection. The starter ignores `/_migrations/` and `/better-auth_migrations/` by default. To commit migrations, remove only those ignore rules, review SQL plus journal/snapshot files, and stage the chosen files. To keep history private, retain the ignores and supply a private migration artifact/repository to CI; an ignored directory will not appear in a clean checkout. Never store credentials or production data in migration SQL.

Configure the selected ORM's output directory explicitly, for example Drizzle `out: './_migrations'` with an existing server-side schema path and dialect. Keep DB URLs in server environment/deployment secrets; migration CLI config uses its own environment loader, not browser site config. Generate/review SQL and test it against a disposable DB before applying to the intended target. Drizzle push is a direct schema change, not equivalent to reviewed generated migration history.

The manifest's migration commands run only through a separate invocation:

```sh
node bin/addon.mjs owner/repo generator install --run-migrations --migrations-only --migration-target staging
```

Add --dry-run to inspect the migration-only invocation without executing it; --commit SHA is optional here too. Every migration must have a name, executable command and string args array, including `{{target}}`. The runner substitutes the named target as a literal argument and sets ADDON_MIGRATION_TARGET; it does not interpolate a shell command. The migration program must resolve that name to its configured database and validate the target. It runs from the current project, so generate its migration files/entrypoint first. No templates, prompts, package installs, merges or hidden scripts run during migration-only execution. The command still has host permissions; inspect it and its DB mapping before execution. Keep DB credentials out of target names and command arguments.

### Config composition

_config.json5 or `_config.ejs.json5` merges into the client-visible modules block of vite.config.ts. Never include credentials or private environment values. $remove applies before merging. `$navbarItems` and `$authItems` accept `$append/$remove` arrays for idempotent navbar/user-menu composition; matches use the supplied fields.

```json5
{
    support: { enabled: true },
    $navbarItems: { $append: [{ label: "navbar.support", to: "/support", position: "left" }] }
}
```

Uninstall uses `$remove: [{ to: "/support" }]` under $navbarItems and module-specific $remove entries. Avoid removing other addons' items. `$expr(...)` injects raw TypeScript only with --allow-expressions; use reviewed fixed expressions, never user strings or secrets. Reserved prototype/internal wrapper keys are rejected in incoming data.

### Translations

Use _lang.en.json5 in the action root or `_lang/en.json5`. `$path` defaults to `modules.<cleanedGenerator>;` explicit paths target existing locale dictionaries. $remove accepts a key/string, key array or nested deletion object.

```json5
{
    $path: "modules.support",
    $remove: ["oldTitle"],
    title: "Support"
}
```

Translation writes obey destination checks. The runner prints a reminder after dictionary changes; it does **not** automatically execute typesafe-i18n or npx. After reviewing the generated diff, run the installed project's i18n:extract script yourself. Do not hand-edit generated translation types.

## Compatibility with older addons

Older instructions that say “install runs everything”, “dry-run runs Hygen”, “put migrations in _scripts”, or “interpolate prompt answers into `_scripts.ejs.sh`” are obsolete for this runner. Use --dry-run when inspection is wanted, optionally pin with --commit SHA, migrate package/DB effects to addon.json, and replace interpolated scripts with quoted environment data. Unsupported hooks fail instead of executing implicitly.

For optional login/signup during or after scaffolding, see [Better Auth installation](auth/better-auth.md). The blockchain toolset selection alone does not configure server authentication.
