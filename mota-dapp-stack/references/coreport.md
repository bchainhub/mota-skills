# CorePort payment gateway integration

[CorePort](https://coreport.net) offers hosted payment, donation, booster and on/off-ramp flows. These notes are derived from the workspace coreport source (generator and URL parsers), not an assertion that every local feature is deployed. Do not copy its legacy Kit 2 env imports into a Kit 3 app.

## Generate and copy a link

Open <https://coreport.net/generate> and choose Payment, Donation, Booster or Ramp. For the first three, set the recipient Core ID, settlement asset, display currency/currencies and optional amount; the generator validates required inputs. Set branding/name, description, colors, language and blockies/photo as applicable. Payment also supports item details and tip visibility. Preview the result and copy the generated link, QR or normal/Tailwind button code. Use the hosted generator for advanced combinations and provider availability.

For Ramp choose enter (buy/on-ramp) or exit (sell/off-ramp), supported asset/network, wallet, amount and the available payment/payout method. Provider minimums, supported countries/assets and identity requirements are enforced by the service; a link does not bypass them. Amount/fees displayed by the flow are authoritative at confirmation, not a stale estimate embedded in a button.

## Link schema from the source

Payments, donations and boosters use /link:

| Parameter | Meaning |
| --- | --- |
| id | Recipient/merchant Core ID |
| settle | Settlement asset code (generator emits lowercase) |
| currency | Display/quote currency code, or comma-separated allowed codes |
| amount | Fixed amount when supplied; omit for editable/custom-amount flow |
| type | Omitted for payment; donation or booster for those modes |
| name / desc | Merchant/creator name and description |
| item / item-desc | Payment item title/details |
| showtip=0 | Disable payment tip UI |
| method | Preselect an available payment rail |
| asset | PayTo/CorePay asset prefill where supported |
| lang | Supported locale |
| blo=1 | Blockies instead of a custom photo |
| photo / item-photo | Valid public image URLs; use generator encoding |
| color-f / color-b | Both six-digit hex colors without #; generator checks contrast |
| strict=1 | Strict settlement preference |
| client-coreid / force-coreid=1 | Customer Core ID prefill / required input |
| rd | Valid return URL, not payment verification |
| wh / wh-data | HTTPS merchant webhook and public JSON correlation data |
| wh-init=1 | Request early payment.initiated notification, subject to release policy |
| split-id / split-percent | Valid split recipients and percentages; use generator validation |

Generator limits currently include name/item to 25 characters and desc/item-desc to 40. Payment method aliases in linkPrefill.ts include payto, corepay, wallmoney/intrabank, stripe/card, phajay, mypos and sepa; feature flags can disable rails. `/link?method=card` retains Stripe compatibility, while ramp's card mapping differs. Do not copy one page's method map into another. A method prefill can skip the summary only with a payable fixed amount, valid merchant and valid required customer identity.

A minimal builder for validated merchant data (no money is sent):

```ts
export function corePortLink(
    validatedMerchantCoreId: string,
    type: 'payment' | 'donation' | 'booster' = 'payment'
): string {
    const url = new URL('/link', 'https://coreport.net');
    url.searchParams.set('id', validatedMerchantCoreId);
    url.searchParams.set('settle', 'usdx');
    url.searchParams.set('currency', 'usd');
    if (type !== 'payment') url.searchParams.set('type', type);
    if (type === 'payment') url.searchParams.set('amount', '25.00');
    return url.href;
}
```

Use only an asset the intended deployed service supports. For Booster, a fixed amount is the per-zap unit price in the current source, multiplied by the selected zap count; do not reuse invoice-amount assumptions. Amounts are decimal strings in URLs. Establish the invoice amount/recipient/currency on your server, since query parameters are editable by the payer.

Ramp uses /ramp:

| Parameter | On-ramp (enter) | Off-ramp (exit) |
| --- | --- | --- |
| action | enter | exit |
| asset | Asset to receive | Asset to send |
| wallet | User receiver wallet | User sender wallet |
| method | Selected funding rail | Usually not the destination selector |
| asset-to | Not required for normal enter flow | Target payout asset |
| wallet-to | Not required for normal enter flow | Destination wallet when needed |
| network | Supported network, when disambiguation is needed | Same |
| amount / currency | Amount and quote/display code(s) | Same, interpreted by selected flow |

Examples as URL patterns, not valid transactions:

```text
https://coreport.net/ramp?action=enter&asset=usdx&wallet=VALIDATED_RECEIVER&amount=25&currency=usd&method=intrabank
https://coreport.net/ramp?action=exit&asset=usdx&wallet=VALIDATED_SENDER&asset-to=eur&amount=25&currency=usd
```

The second pattern still needs the selected provider's payout details in its flow. Bank offramps have special identity/payout-account handling; do not guess wallet-to or promise a universal bank deep link. Generate the deployed provider's supported URL through the UI.

## Integrate into a MOTA app

Store public merchant display settings in client config if useful; keep invoice records and verification server-side. Build the URL using URL/URLSearchParams and render an ordinary Svelte `<a href={paymentUrl} target="_blank" rel="noopener">Pay</a>`. The generator also produces styled HTML and Tailwind buttons. Escaping attributes is required when inserting generated code into non-Svelte HTML. Do not assume iframe embedding is supported by provider/CSP; use a normal link unless embedding is documented.

For the advanced rd/wh/photo fields, the current generator applies encodeURIComponent before URLSearchParams, and parsers perform matching decoding. Prefer copying the generator output for these fields; do not apply extra encoding to an already generated URL or blindly decode arbitrary strings repeatedly. wh-data is JSON serialized into the URL: only use a non-secret order/reference ID, not credentials or personal data. Links and QR contents can be logged/shared publicly.

## Merchant completion and webhooks

1. Create an order with expected recipient, amount, asset and unique correlation ID on the merchant server.
2. Supply a public HTTPS webhook/return route through the generator, keeping wh-data non-secret. The generator defaults to requesting early notification where allowed; uncheck it if only final settlement is desired.
3. Treat redirects, browser messages, QR scans and payment.initiated as progress only. The current code distinguishes initiated/executable from payment.completed/full settlement.
4. Verify CorePort's X-JWS-Signature header as detached JWS ES256 against the **raw body** using a trusted public verification key obtained through the service's key publication/onboarding (see <https://coreport.net/keys> and /docs/webhook). Honor key rotation/alg restrictions; never trust a key or key URL supplied by the webhook itself.
5. Match the verified event to the server order: recipient, amount, asset/currency, status, transaction/order identity and replay/idempotency. Fulfill once on final confirmed evidence. Persist delivery IDs/status transitions and acknowledge duplicates safely.

The local implementation can emit unsigned notifications if signing keys are missing. A merchant must reject missing/invalid signatures for automatic fulfillment, or independently verify the transaction through a trusted chain/provider path; do not silently downgrade verification. Public callback data and customerCoreId are not authentication credentials. Keep the body unmodified during signature verification, then parse/validate it.

CorePort source can lag deployment and the public docs contain older placeholder payloads. Confirm the current callback schema/key distribution and settlement contract before implementing production fulfillment. Link construction and source review here do not establish that a real payment or provider flow has been tested end to end.
