# Core Blockchain, CoreBC and Connect

Core Blockchain's native currency is XCB. Core IDs use network-prefixed ICAN addresses and Ed448, not Ethereum's secp256k1 key/address rules. Use bigint and decimal-string unit helpers for money. Do not transplant Ethereum transaction fields or chain IDs into Core code.

## Network communication

User-supplied service hosts, with explicit transport schemes:

| Network | JSON-RPC | WebSocket |
| --- | --- | --- |
| Mainnet | <https://xcbapi.coreblockchain.net> | wss://xcbws.coreblockchain.net |
| Devin testnet | <https://xcbapi-devin.coreblockchain.net> | wss://xcbws-devin.coreblockchain.net |

Use `xcb_*` RPC methods through the library or the [Core Blockchain Postman collection](https://www.postman.com/core-labs/core-blockchain). Verify endpoint network identity and connectivity at integration time; listing a public endpoint is not an availability guarantee. Keep mainnet and testnet providers, assets, accounts and TxMS pools paired. WebSocket handles subscriptions; disconnect it when finished. Do not broadcast while merely checking a connection.

## CoreBC

Package `corebc`, [source/API](https://github.com/CorePass/corebc.js). Starter can optionally install it as the blockchain toolset. Use JsonRpcProvider/WebSocketProvider for chain reads, Wallet for owned signing, and Contract for ABI reads/writes. Prefer an external wallet for end-user keys; secrets never belong in siteConfigClient.

```ts
import { JsonRpcProvider, formatXCB } from 'corebc';

export async function readCoreBalance(coreId: string): Promise<string> {
    const provider = new JsonRpcProvider('https://xcbapi-devin.coreblockchain.net');
    try {
        return formatXCB(await provider.getBalance(coreId));
    } finally {
        provider.destroy();
    }
}
```

The input must be a validated Devin address for this example. `formatXCB` formats base-unit bigint values; `parseXCB` converts decimal strings to base units. Contract accepts a provider for reads and a signer for writes. Core transaction fields include energyLimit, energyPrice and networkId. The inspected library supports ESM/CJS and subpaths corebc/providers, corebc/abi, corebc/crypto, corebc/cip and corebc/ipfs.

Wallet.createRandom(networkIdToPrefix(3)) is a Devin wallet pattern, but do not generate or print keys just to demonstrate a connection. Before signing or broadcasting an actual payment, show and validate the selected network, recipient, amount, asset and fees in the app's wallet approval flow. A transaction hash alone does not prove final settlement.

CoreBC compatibility details from its README: private/public keys are 57-byte Ed448; shared-secret output is 56-byte X448 in the current version; legacy sha256/sha512 exports compute SHA3-256/SHA3-512, while keccak256 is distinct. Use installed version documentation before touching signing/hash compatibility. CIP-150/151/152 helpers expose metadata, lifecycle times and lab structures; metadata presence or structural validation does not prove issuer authenticity. Keep metadata fetches bounded and enforce server outbound URL policy.

## Wallet connections

For Connect.js (`connect-protocol`) discovery, account selection, signing and wallet operations, see [wallet integrations](wallets/index.md) and [Connect.js](wallets/connect.md). For signed-challenge login, see [Connect.js authentication](wallets/connect.md#authentication).

## Identity enrichment

CorePass identity enrichment and Connect proofs are different flows. For the passkey/CorePass addon, accept signed enrichment only after verifying Core ID signature, challenge/time bounds and consent. Preserve dataExp expiry semantics: no extension without new signed data. Do not infer KYC/age attributes from a wallet address or identicon.

For verified wallet proofs backed by application accounts and cookie sessions, see [Better Connect with Better Auth](auth/better-auth.md#connect-wallet-login-with-server-sessions).
