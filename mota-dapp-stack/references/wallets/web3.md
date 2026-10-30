# Injected Web3 connector (6963)

Use the template's existing `src/lib/modules/auth/web3.ts` for injected browser
wallets. This connector is separate from [Connect.js](connect.md) cross-device
handoffs and from [Better Auth](../auth/better-auth.md) server sessions. Inspect
`web3.ts`, `discovery.ts`, `nav-actions.ts`, `Header.svelte`, `ActionsDropdown.svelte`
and `src/app.d.ts` in the target checkout before changing the integration.

## Enable in the installed app

Merge into `siteConfigClient.modules` in `vite.config.ts`; preserve other modules.
The connector is included in the template and needs no extra wallet SDK. Keep the
template's auth configuration disabled unless the user requests enabling it there.
Do not put package installations or generated files in `mota-dapp`.

```ts
modules: {
	auth: {
		enabled: true,
		strategy: ['web3'],
		primaryStrategy: 'web3',
		web3: {
			provider: 'core',
			blockchain: 'Core',
			discovery: 'cip6963',
			methods: {
				requestAccounts: 'xcb_requestAccounts',
				accounts: 'xcb_accounts',
				chainId: 'xcb_chainId'
			}
		}
	}
}
```

Use the RPC methods supported by the target wallet/chain. For Ethereum configure
`provider: 'ethereum'`, `blockchain: 'Ethereum'`, `discovery: 'eip6963'` and the
corresponding `eth_requestAccounts`, `eth_accounts`, `eth_chainId` methods.
Other 6963-style integrations require their actual event namespace and RPC API;
`discovery` accepts a lowercase namespace ending in `6963`. Do not invent a
namespace and imply existing wallets support it.

`provider` is a preference matching announced RDNS or name, or an ecosystem label
when no specific wallet is preferred. An unmatched preference falls back to the
first announcement in that namespace. `blockchain` is the display label used in
errors. Required methods are `requestAccounts` and `chainId`; optional `accounts`
enables silent reconnection. If `discovery` is omitted, `xcb_` account methods infer
CIP-6963 and `eth_` account methods infer EIP-6963; other methods need an explicit
namespace. Discovery is not a network-support check or a network-switch request.

## Discovery and connection lifecycle

The header calls `initProviderDiscovery()` on client mount when Web3 is configured.
The discovery library registers `<namespace>:announceProvider` before dispatching
a standard `Event('<namespace>:requestProvider')`. It retains the listener for late
injection and deduplicates UUIDs. Wallets provide metadata and the provider object:

```ts
const detail = Object.freeze({
	info: {
		uuid: crypto.randomUUID(),
		name: 'Example Wallet',
		icon: 'data:image/png;base64,...',
		rdns: 'org.example.wallet'
	},
	provider: coreProvider
});
window.dispatchEvent(new CustomEvent('cip6963:announceProvider', { detail }));
```

This is wallet-side announcement code, not a provider the dApp should manufacture.
The wallet also reannounces the same provider/UUID on discovery requests. Metadata
is self-attested; RDNS and name do not authenticate the extension. Discovery alone
must not request accounts or signing permissions.

Use announcements from the configured namespace only. Keep CIP and EIP providers
separate, even if their request interfaces look alike. There is no hardcoded wallet
registry or fallback to `window.core`, `window.ethereum`, or wallet-specific globals.
Core Blockchain is distinct from Avalanche's Core Wallet; do not infer protocol
compatibility from a brand name.

`manualConnect` calls `connectWallet()` directly; there is no built-in wallet picker
in the shared layout. The connector prefers the configured RDNS/name, then the
first announcement. A custom caller can inspect `getAvailableProviders()` and pass
one of those detail objects to `connectWallet(true, messages, selectedDetail)`.
The displayed wallet name comes from `info.name`, not a wallet-name lookup table.

`autoLogin()` uses the previously connected RDNS in the same namespace and the
optional `methods.accounts` method. It never uses `requestAccounts` for automatic
reconnection. The header checks `shouldAutoConnect()` and `autoConnect !== false`
before attempting it. Keep explicit Connect as the permission-requesting action.

The connector maintains `walletConnected`, `walletType` (announced RDNS),
`walletDisplayName`, `walletAddress`, and `walletChainId` stores. It subscribes to
account/chain changes through the existing provider subscription helpers. Preserve
listener cleanup when changing providers or disconnecting.
`isCoreEcosystem()` uses CIP announcements rather than wallet brand heuristics.
Disconnect clears local connection state and reconnection preferences; it does not
promise that the wallet revoked its site permissions or that a server logged out.

Failed manual connections produce one error toast:

```text
Cannot connect wallet for Blockchain: Core.
```

Substitute the configured blockchain label. Missing provider, empty accounts and
rejected RPC requests must not each add another warning for the same attempt.
Automatic connection failures stay silent. Keep the explicit installation helpers
separate; connection failure does not automatically redirect to an extension store.

## Navigation and presentation

Add a navbar item where Connect/account should appear; preserve existing item order:

```ts
{ label: 'navbar.connect', action: 'manualConnect', position: 'right', show: 'disconnected' }
```

When logged in, the header replaces the first `manualConnect` slot with the account
menu, retaining its side and order relative to other links. Without a Connect slot,
the account menu falls back to the controls area. `authItems` configure account-menu
entries. The account label uses regular font weight. See [site config](../site-config.md).

Use serializable Tailwind `className` strings in config for requested desktop
Connect styles. The hamburger action uses its own plain styling; do not add a
shared layout picker or a global account simulation to achieve a visual preview.
For gradient outlines, avoid mask shorthands that reset `mask-composite`; preserve
the label and verify the actual rendered border.

## Icons, identity and verification

`discovery.ts` validates UUIDv4, name, RDNS and the request interface; CIP also
requires `on` and `removeListener`. Unknown metadata properties do not break discovery.
`walletIconUrl()` supports embedded WebP/PNG/JXL/SVG, HTTPS and validated IPFS CIDs/
paths with a configurable HTTPS gateway argument. Invalid icon URIs become a fallback;
remote HTTP is rejected. No icons are fetched automatically. If an application adds
icon UI, use an image element for SVG, handle decoding failures and avoid fetching
remote metadata before user interaction. Gateway resolution alone does not verify CID
integrity. Inspect this validator instead of using the general `getIpfsUrl()` helper
for untrusted wallet icons.

A browser wallet connection does not establish a server session, prove ownership
to the backend, or authorize protected routes. Use an explicitly chosen,
server-verified challenge/session integration for that requirement.

Validate changes in the installed app with its check/build scripts and relevant
behavior tests: listener-before-request, frozen announcement detail, late/duplicate
announcements, namespace isolation, selection from announced metadata, no permission
request during discovery or auto-reconnect, account/chain listener cleanup, exactly
one failure message, and account-menu ordering. Mirror shared fixes in the template
without enabling its auth config or copying temporary demo previews into it.
