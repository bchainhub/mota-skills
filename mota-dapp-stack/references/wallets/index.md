# Wallet integrations

Choose the wallet integration requested by the user or already used in the app. Wallet discovery, connection and account/network selection are distinct from server account sessions; signing requests require explicit wallet approval.

| Integration | Capabilities | Guide |
| --- | --- | --- |
| Injected Web3 connector (built into MOTA) | CIP-6963, EIP-6963 and configured 6963-style discovery; announced providers, account/network state and direct Connect action | [Web3 connector](web3.md) |
| Connect.js / Connect Protocol (`connect-protocol`) | Wallet discovery/adapters, signed challenges, cross-device handoffs and supported wallet operations | [Connect.js](connect.md) |

Add another integration as a separate guide in this folder and a row here when needed. Document its installation, supported adapters/networks, client lifecycle, account changes, signing capabilities and cleanup. Authentication-specific setup belongs in the [authentication catalog](../auth/index.md), with links between the relevant guides.
