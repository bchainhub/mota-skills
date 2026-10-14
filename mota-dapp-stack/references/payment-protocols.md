# ICAN, blockies, FinTag, PayTo, PayPass and TxMS

These protocols describe payment routing and transport, not authorization to send money. Use actual user-provided recipients and validate network/asset semantics before creating actionable links. Examples below accept a validated address; never substitute a donation address copied from a library README.

## ICAN formatting and verification

Package @blockchainhub/ican ([source](https://github.com/bchainhub/ican.js), [concept](https://payto.onl/solutions/ican)). ICAN combines a network/country prefix, checksum and account portion; do not validate just by string length or uppercase/lowercase appearance.

```ts
import ICAN from '@blockchainhub/ican';

export function formatCryptoAccount(input: string) {
    const electronic = ICAN.electronicFormat(input);
    if (!ICAN.isValid(electronic, true)) throw new Error('Invalid crypto ICAN');
    return {
        electronic,
        display: ICAN.printFormat(electronic),
        short: ICAN.shortFormat(electronic, '…', 4, 4, 2)
    };
}
```

electronicFormat strips non-alphanumeric characters and uppercases, so validate allowed input syntax before normalizing when strict input rejection is required. Never treat normalization as checksum verification. `isValid(value, true)` limits to crypto definitions; false excludes crypto; main/mainnet, test/testnet and enter/enterprise filter network classes. For Core specifically, also validate the expected Core network/address type (e.g. the app's blockchain-wallet-validator helper), since generic crypto ICAN validation is broader than Core. Reject enterprise addresses where the payment flow forbids them.

printFormat provides grouped display; electronicFormat is for machine/copy use; shortFormat is display only, never a transfer destination. toBCAN/fromBCAN/isValidBCAN handle base account conversion with an explicit country/network code. Keep checksum and network verification after conversion.

## blo / blockies

Package @blockchainhub/blo ([source](https://github.com/bchainhub/blo)). Lowercase the normalized **Core ID first** so the same account has a consistent blockie across uppercase display and lowercase storage:

```ts
import { blo } from '@blockchainhub/blo';
import ICAN from '@blockchainhub/ican';

const electronic = ICAN.electronicFormat(coreId);
if (!ICAN.isValid(electronic, true)) throw new Error('Invalid Core ID');
const icon = blo(electronic.toLowerCase());
```

The caller must additionally enforce the Core network. Render the returned SVG data URI using an img with alt text. bloSvg returns SVG source and bloImage exposes pixels/palette. Use installed types for options rather than copying an older README's boolean option. Never lowercase arbitrary case-sensitive addresses from other chains. Blockies are visual cues, not ownership/authentication checks.

The existing MOTA getIcon helper currently forwards the string unchanged. Normalize at the caller or, when asked to fix the shared helper, update both template and installed app and verify case-consistent output.

## FinTag discovery

[FinTag](https://payto.onl/solutions/fintag) publishes payment identifiers through HTML metadata and /.well-known/fintag.json. It is financial discovery, separate from Open Graph tags and a payment confirmation. The well-known document is an array of objects in the current generator/parser.

For a validated Core mainnet recipient, add to themeConfig.metadata:

```ts
{ property: 'ican:xcb', content: validatedMainnetCoreId }
```

Equivalent rendered HTML is `<meta property="ican:xcb" content="YOUR_VALIDATED_MAINNET_CORE_ID" />`. Use the actual network key for other networks. For the static document create static/.well-known/fintag.json:

```json
[
    { "ican:xcb": "YOUR_VALIDATED_MAINNET_CORE_ID" }
]
```

Replace the placeholder before deployment. Additional supported identifiers include iban, bic, ach, pix, upi and void:plus; generate the correct structure with PayTo rather than inventing key combinations. Keep root metadata and well-known details consistent, serve the JSON publicly with the expected content type, and verify raw SSR head HTML. A recipient found on a site needs origin/context validation; discovery does not prove wallet control or payment settlement. Server-side lookup must bound size/time and reject private-network URLs/redirects to avoid SSRF.

## Payto Resource Locators

Package payto-rl ([source](https://github.com/bchainhub/payto-rl)) extends URL with typed routing fields and validation. It parses/builds payment instructions; it does not sign or broadcast them.

```ts
import Payto from 'payto-rl';

export function makeCorePayment(validatedCoreId: string): string {
    const payment = new Payto(`payto://xcb/${validatedCoreId.toLowerCase()}`);
    payment.amount = 'xcb:1.25';
    payment.receiverName = 'Example merchant';
    payment.message = 'Order reference';
    return payment.toString();
}
```

Use the expected validated address/network; other supported routes include iban, bic, intra, ach, upi and pix. Useful fields include network, address, amount, currency, fiat, donate, item, organization, colorBackground/colorForeground, language (the API property is lang), deadline and searchParams. Color fields use six hex digits without #. Catch invalid input errors and preserve decimal strings for authoritative amounts; a numeric convenience value is unsuitable for exact large balances. `.toJSONObject()` exposes parsed data; unsupported/unknown metadata must not become executable HTML or commands.

## PayTo:Money constructor and PayPass

At [payto.money](https://payto.money/) choose payment type/network, recipient/NS name, optional token and amount. Advanced options control supported routing metadata. Integrations provide a link, Markdown, HTML buttons, FinTag metadata and well-known JSON. Use the generated link rather than hand-guessing a private web-link encoding.

Expand **PayPass** to preview/customize the payment window. Set the item title, expand **Customization**, and adjust the offered foreground/background colors and organization/branding fields, then use the copy control for the payment link or **Open Weblink** to verify it. The screenshot's preview shows a QR, XCB, custom amount, issuer/organization and separate Apple/Google Wallet actions. Online preview and wallet passes differ; inspect each output. Optional Pro features such as notifications/fiat depend on the service/account and are not enabled by copying a URL. Never place private keys or merchant secrets in the link or pass.

For a website integration use a normal escaped href or generated button pointing to the copied public link. Do not execute copied scripts without review or treat a return-page visit as payment success.

## TxMS — offline signed transaction transport over SMS

TxMS means Transaction Message System in this stack. [Concept](https://payto.onl/solutions/txms), [txms.js](https://github.com/bchainhub/txms.js). It encodes an already signed raw transaction for SMS transport; it is not encryption, a signer or a substitute for nonce/fee/network validation.

Prepare the transaction for the correct network, obtain required nonce/fee inputs before disconnecting or via a deliberate offline workflow, show recipient/asset/amount/fees, and sign in the wallet. Encode only the signed raw transaction, never a private key/seed. Select a maintained endpoint for the same network, open the SMS composer, and let the user send. The recipient gateway needs network connectivity to broadcast; the sender needs SMS service. A message sent is not a confirmed on-chain payment.

```ts
import txms from 'txms.js';

const network = 'xab'; // Devin; xcb is mainnet
const encoded = txms.encode(signedRawTransactionHex);
const endpoint = txms.getNumber('SK', false, network);
if (!endpoint) throw new Error('No TxMS endpoint available');
const smsCount = txms.count(signedRawTransactionHex, 'sms');
const smsHref = txms.sms(endpoint, encoded, network, false, 'global');
```

count() takes raw hex and encodes internally in the inspected implementation. The false flag avoids double TxMS-encoding an already encoded body; the library still URI-encodes it. Use platform ios for the iOS &body= syntax when appropriate, global otherwise. Show smsCount and carrier cost implications before the user opens/sends; do not claim a fixed SMS price. Alternatively pass raw signed hex with the default encodeMessage=true, or download a .txms.txt file. `decode(encode(hex))` can check transport roundtrip after accounting for the returned 0x prefix and leading-padding normalization; do not treat this codec as a byte-preserving general binary archive. A library example hex is not a valid payment transaction.

Use the installed number pools (`txms.js/numbers` when supported) or getNumber rather than hardcoding forever. Previously documented global examples are mainnet +12019715152 and Devin +12014835939; verify against the installed pool and service status before presenting them as current. xcb/mainnet/1 and xab/devin/3 are aliases in the inspected source. Read-only receipt parsing with parseSMS can help UI status, but a sender-number match or SMS text alone is not sufficient server evidence for order fulfillment. Confirm the transaction on the intended chain when connectivity returns.
