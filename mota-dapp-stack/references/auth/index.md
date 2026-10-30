# Authentication providers

Authentication is optional. Choose the provider requested by the user or already used by the application, then read its implementation guide. A browser wallet connection alone does not establish a server session. Keep provider credentials and database access server-only, and integrate with the existing MOTA auth module, hooks and session shape.

For `manualConnect`, browser wallet state and CIP/EIP-6963 discovery, read the
[built-in Web3 connector](../wallets/web3.md). It participates in MOTA navigation
but does not create a server-authenticated session.

## Available providers

| Provider | Methods and integrations | Guide |
| --- | --- | --- |
| Connect.js | Browser-only wallet challenge authentication; optional server sessions through the separate Better Connect integration | [Connect.js authentication](../wallets/connect.md#authentication) |
| Better Auth | Email/password signup and login; optional CorePass passkey and Better Connect wallet-session plugins | [Better Auth implementation](better-auth.md) |

Connect.js is also listed under [wallet integrations](../wallets/index.md). CorePass passkeys and Better Connect are integrations under Better Auth in this catalog. Their installation, database schema, session behavior and compatibility constraints belong in that provider's guide.

## Adding another provider

Add a separate provider reference and a row in the table when another implementation is requested. Document installation during scaffolding and afterward, supported databases/adapters, migrations, server/client setup, login/signup/logout, session mapping and verification. Keep provider-specific plugins under their parent provider. Do not install multiple providers merely because they are listed here.

Shared deployment and storage guidance remains in [Cloudflare](../cloudflare.md) and [databases](../databases.md); provider references explain how their own configuration uses those services.
