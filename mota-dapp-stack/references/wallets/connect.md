# Wallet integration: Connect.js

Package `connect-protocol`, [repository](https://github.com/bchainhub/connect.js). It handles browser wallet discovery, challenge signing and local proof verification. It is not a Core RPC provider. Import browser APIs inside client-only lifecycle/actions so wallet code does not execute during SSR.

```ts
import { BrowserConnect, signBrowserChallenge } from 'connect-protocol';

const request = new BrowserConnect({ origin: location.origin }).create();
// Host UI displays the challenge, discovers wallets and obtains an account
// selected by the user. Signing belongs in an explicit user action.
const proof = await signBrowserChallenge(request.challenge, selectedWalletAccount);
const identity = await request.verify(proof);
```

selectedWalletAccount comes from the current wallet adapter API, not a fabricated address object. See the repository's examples/dapp and docs/WALLETS.md for supported adapters/account selection. Keep the original request instance for verification. Challenges are held in memory, expire (two minutes by default in the inspected version) and are single-use; reload invalidates them.

The main export is the browser API; connect-protocol/wallet contains discovery/adapters and /verification contains proof profiles. Do not invent /server, /dapp, /browser or /evm entry points. Browser identity is local UI state, not a server session. A backend must verify its own authentication evidence and issue its own session; the separate better-connect project owns Better Auth integration.

For cross-device use, export the full challenge or use createBrowserHandoff(request), display its uri as QR/deep link, and accept the returned packet against that original handoff. A short request-ID URI alone cannot support offline proof verification. QR/Bluetooth/manual response transport and permissions belong to the host UI. Disconnect/clean up listeners on sign-out.

Newer local source also has wallet operation dispatch/transport APIs; check installed package exports and docs/OPERATIONS.md before using them. Do not assume that a verified login proof grants transaction-signing capability, or that every deployed package version contains unpublished local features. Wallet operations preserve native chain formats and require explicit host approval.

## Installation and MOTA integration

Connect.js is optional. Select Connect.js (`connect-protocol`) in the starter's blockchain interaction toolset, or install `connect-protocol` afterward in the installed app using its package manager. Match the installed exports and wallet adapter API before wiring UI. Do not install packages in the template-only repository.

Use MOTA's wallet module and navbar connect/disconnect actions to expose connection state, account and network. Keep wallet connection distinct from application login. Wallet capabilities determine which reads/signing operations can be requested; connecting alone does not authorize a payment. On disconnect, clear local identity and release listeners/transports; a separate server session needs its own logout flow.

For signed-challenge login using the same library, see [authentication](#authentication) below. For Better Auth cookie sessions, see [Better Connect](../auth/better-auth.md#connect-wallet-login-with-server-sessions). Core RPC and CoreBC setup remain in [blockchain communication](../blockchain-connect.md).

## Authentication

### Browser wallet authentication

Create a `BrowserConnect` request bound to the application's origin, obtain a user-approved signature through the selected wallet account, and verify the returned proof with that same request instance. Only after verification succeeds should the UI accept the returned local identity. Keep the original challenge/request for QR or other cross-device response transport; handle cancellation, expiry and replay. Reload discards in-memory requests and local login state unless the app deliberately implements another persistence mechanism.

Use the library's challenge/signature verification rather than accepting a wallet-supplied address as proof. Validate the intended origin/network/account and show the signing request to the user. A verified identity does not automatically grant transaction-signing permission or identity enrichment/KYC attributes.

### Server-backed application authentication

Browser verification establishes local state. A client-provided `authenticated` flag or verified identity object is not evidence a backend can trust. To protect server routes, the application needs a backend-owned challenge/proof verification flow and session issuance, with expiry and one-time consumption enforced on the server. Connect.js has no built-in HTTP handlers or server session store; do not invent a `connect-protocol/server` import.

For an existing supported implementation, use the separate [Better Connect plugin under Better Auth](../auth/better-auth.md#connect-wallet-login-with-server-sessions). That integration verifies wallet proofs and issues Better Auth cookie sessions. Choose browser-only identity or server-backed sessions according to the application's needs; do not install Better Auth merely to display a connected wallet.

### Authentication verification

Test rejected/expired/replayed proofs, origin/account selection, cancellation and disconnect. For server-backed login, also verify protected server routes and session logout independently from wallet disconnect. Check the selected release against [Connect.js source and browser examples](https://github.com/bchainhub/connect.js).
