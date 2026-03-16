# MOTA Skills

Skill pack for the **MOTA ĐApp stack**: SvelteKit, [ĐApp Starter](https://github.com/bchainhub/dapp-starter) (dapp-starter), addon CLI, and [mota-dapp](https://github.com/bchainhub/mota-dapp) project structure. These skills help AI assistants (Cursor, Claude Code, Copilot, Continue, etc.) follow the right conventions when working in your repo.

## Folder structure

This repo follows the [skills.sh skill anatomy](https://skills.sh/anthropics/skills/skill-creator): **one folder per skill**, with `SKILL.md` inside and optional `scripts/`, `references/`, and `assets/` for bundled resources.

```text
mota-skills/
├── README.md
└── mota-dapp-stack/
    └── SKILL.md
```

- **mota-dapp-stack** – ĐApp Starter installer, addon CLI (order of execution, options, versioning), addon file roles (prompt.js, _scripts, _config, _lang, _migrations), mota-dapp layout, SvelteKit routing (`[param]`, `[[param]]`, `(group)`), env vars, Vite config, CONTRIBUTING/Code of Conduct.

More skills can be added as new folders (e.g. `another-skill/SKILL.md`) later.

## How to install

### Using skills.sh (recommended)

If you use the [skills.sh](https://skills.sh) CLI (e.g. in Cursor, Claude Code, or from the terminal):

**Install the whole repo** (adds all skills in this repo):

```bash
npx skills add bchainhub/mota-skills
```

**Install only the MOTA ĐApp Stack skill** (same repo, single skill):

```bash
npx skills add bchainhub/mota-skills --skill mota-dapp-stack
```

From a GitHub URL:

```bash
npx skills add https://github.com/bchainhub/mota-skills --skill mota-dapp-stack
```

After installation, the skill is available to your agent when its description matches the task (e.g. editing SvelteKit routes, running addons, or working with dapp-starter/mota-dapp).

### Without skills.sh

- **Cursor / IDE:** Add this repo (or the `mota-dapp-stack` folder) to your project or agent context so the assistant can read `mota-dapp-stack/SKILL.md` when working on SvelteKit, addons, or dapp-starter.
- **Manual:** Clone or download the repo and point your workflow at `mota-dapp-stack/SKILL.md` as a reference doc.

## What the skill covers

- **ĐApp Starter:** `npx github:bchainhub/dapp-starter`, template (mota-dapp), `--update`, default packages, what the installer sets up.
- **Addon CLI:** `npx addon <repo> <generator> <action>`, versioning with `#ref`, flags (`-c`, `-d`, `-nt`, `-ns`, `-nc`), and the exact execution order (fetch → prompts → Hygen → _scripts → _config → _lang → typesafe-i18n).
- **Addon files:** prompt.js (array/function), *.ejs.t, _scripts, _config (client-only, no secrets), _lang ($path, $remove), _migrations.
- **mota-dapp:** `src/routes` (e.g. `[[lang]]`, `api/[version]`), `src/lib`, `src/i18n`, `app.d.ts`, `vite.config.ts` and `__SITE_CONFIG__`.
- **SvelteKit:** Route segments `[param]`, `[[param]]`, `(group)`; env: `$env/static/private`, `$env/dynamic/private`, `import.meta.env`, `VITE_*`.
- **Governance:** CONTRIBUTING (issue → fork → branch → PR), Code of Conduct summary.

## Links

- [skills.sh](https://skills.sh) – Discover and add skills for your agent.
- [Skill creator](https://skills.sh/anthropics/skills/skill-creator) – How to write and structure skills (folder per skill, YAML frontmatter, SKILL.md).
- [ĐApp Starter](https://github.com/bchainhub/dapp-starter) – Installer and docs.
- [mota-dapp](https://github.com/bchainhub/mota-dapp) – SvelteKit template repo.
