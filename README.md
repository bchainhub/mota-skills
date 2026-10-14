# MOTA Skills

Project guidance for MOTA ĐApp templates, installed SvelteKit 3 applications, ĐApp Starter addons and Core payment integrations.

## Included skill

[mota-dapp-stack](mota-dapp-stack/SKILL.md) has a small entrypoint and focused references. Load only the material needed for the task.

| Guide | Contents |
| --- | --- |
| [SvelteKit 3](mota-dapp-stack/references/sveltekit-3.md) | Migration map, typed env, #lib imports, template/demo parity and PWA verification |
| [Starter and addons](mota-dapp-stack/references/starter-addons.md) | Installer CLI, existing .npmrc repairs, addon execution, prompts and merges |
| [Databases](mota-dapp-stack/references/databases.md) | Server connections, Drizzle setup and committed/private migration history |
| [Site config](mota-dapp-stack/references/site-config.md) | Navigation, footer/liner, icons, logos, themes, translations and OG/SEO |
| [Wallets](mota-dapp-stack/references/wallets/index.md) | Wallet integration catalog, Connect.js connection, signing and operations |
| [Authentication](mota-dapp-stack/references/auth/index.md) | Provider catalog; Connect.js authentication, Better Auth setup and plugins |
| [CoreCDN icons](mota-dapp-stack/references/corecdn.md) | Hosted icons, image rendering, menus/logos and offline use |
| [Cloudflare](mota-dapp-stack/references/cloudflare.md) | Pages/Workers, Node LTS selection, D1/KV bindings, cron, email and Durable Object WebSockets |
| [IPFS gateway](mota-dapp-stack/references/ipfs.md) | ipf.sk IPFS/IPNS links, subdomains, application usage and gateway Worker configuration |
| [Helpers](mota-dapp-stack/references/helpers.md) | Existing template helper APIs |
| [Blockchain and Connect](mota-dapp-stack/references/blockchain-connect.md) | CoreBC, browser Connect, mainnet/Devin RPC/WebSocket and identity boundaries |
| [Payment protocols](mota-dapp-stack/references/payment-protocols.md) | ICAN, lowercase Core blockies, FinTag, Payto-RL, PayPass and SMS/TxMS |
| [CorePort](mota-dapp-stack/references/coreport.md) | Payment, donation, booster and ramp link generation; merchant integration and webhook verification |
| [Sources](mota-dapp-stack/references/sources.md) | Source files and upstream documentation |

## Install

Using a reviewed version of the skills CLI:

```sh
npx skills add bchainhub/mota-skills --skill mota-dapp-stack
```

For local review, point the agent at this checkout's mota-dapp-stack/SKILL.md. Copy the whole skill folder, including references, when installing manually. The GitHub installation command fetches the published repository; unpushed local changes are not available through it. Reinstallation can replace local installed edits.

The installed demo copy is under `.agents/skills/mota-dapp-stack`. Package installations, lockfiles and generated output belong in the installed application, not the template-only mota-dapp repository. Mirror shared application fixes without overwriting demo-specific configuration.
