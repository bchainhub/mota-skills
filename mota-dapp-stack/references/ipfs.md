# IPFS and IPNS through ipf.sk

Use [ipf.sk](https://ipf.sk) as an HTTP gateway for reading IPFS files and resolving IPNS names. The [IPFS Worker repository](https://github.com/core-laboratories/ipfs-worker) implements the proxy on Cloudflare Workers. Gateway links need no client package. Publishing/pinning files and updating IPNS records are separate operations; this Worker supports GET/HEAD reads, not uploads.

## Compose a link

Replace the placeholder CID or name with the actual published identifier. A path is optional; omit it to access the root.

| Target | URL pattern |
| --- | --- |
| IPFS file/directory | `https://ipf.sk/ipfs/{CID}/path/to/file` |
| IPFS shortcut | `https://ipf.sk/{CID}/path/to/file` |
| IPFS subdomain | `https://{CIDv1-base32}.ipfs.ipf.sk/path/to/file` |
| IPNS key or DNSLink name | `https://ipf.sk/ipns/{NAME}/path/to/file` |
| IPNS subdomain | `https://{encoded-name}.ipns.ipf.sk/path/to/file` |

For a resource `ipfs://CID/images/logo.png`, remove `ipfs://` and prepend `https://ipf.sk/ipfs/`, producing `https://ipf.sk/ipfs/CID/images/logo.png`. Likewise, `ipns://docs.example.com/guide.pdf` becomes `https://ipf.sk/ipns/docs.example.com/guide.pdf` (illustrative name; it must have a valid published record).

IPFS identifies immutable content: changing the file gives a different CID. IPNS is a mutable name pointing to content, so the same IPNS link may resolve to an updated site or file. Resolution and caches can delay visibility; do not assume IPNS behaves like an immutable CID. A gateway does not ensure that somebody keeps the content pinned and available.

Use the path gateway for CIDv0 (`Qm…`) and preserve its case. For a subdomain, convert the CID to CIDv1 base32 with a proper CID library/tool first; lowercasing a CIDv0 string is not conversion. DNS hostnames are case-insensitive and have label-length constraints. Prefer explicit path URLs for names that cannot fit a valid subdomain.

### IPNS subdomain encoding

This Worker's IPNS subdomains encode each original `-` as `--` and each `.` as `-`. Replace hyphens first, then dots:

```ts
const encodedName = name.replace(/-/g, '--').replace(/\./g, '-');
```

For example, `my-site.com` becomes `my--site-com`, giving `https://my--site-com.ipns.ipf.sk/`. The path form uses the original name: `https://ipf.sk/ipns/my-site.com/`. Use a real DNSLink-enabled domain or published IPNS key when fetching content.

The inspected source also accepts direct `{CID}.ipf.sk` / `{encoded-name}.ipf.sk` hosts and interprets a non-CID apex shortcut as IPNS. Prefer the explicit `/ipfs/` and `/ipns/` forms in generated links to avoid that ambiguity. DNS/TLS deployment must support any chosen subdomain form.

## Application usage

A small helper can compose links from an already validated identifier and raw path segments:

```ts
export function ipfLink(
    kind: 'ipfs' | 'ipns',
    identifier: string,
    segments: string[] = []
): string {
    if (!identifier || /[\s/?#\\]/.test(identifier)) {
        throw new Error('Expected a CID or IPNS name without a path');
    }
    if (segments.some(segment => segment === '.' || segment === '..')) {
        throw new Error('Relative path segments are not supported');
    }
    const suffix = segments.map(segment => encodeURIComponent(segment)).join('/');
    return `https://ipf.sk/${kind}/${encodeURIComponent(identifier)}${suffix ? '/' + suffix : ''}`;
}
```

This helper encodes URL components; it does not validate CID structure or prove that an IPNS record exists. Pass unencoded segments, such as `['images', 'my logo.png']`, to avoid double encoding. Use a CID parser when accepting arbitrary CID input.

Use the URL in a normal link or image `src`. In MOTA, image-compatible `logo.src` settings can use a gateway link; `item.icon` still expects its existing registered icon/component contract. For JSON metadata, `fetch` the URL, check `response.ok` and content type, then parse/validate the expected schema. Browser fetch depends on the gateway/upstream CORS headers; this Worker forwards upstream headers rather than guaranteeing CORS. Use an authorized server-side helper if browser CORS is unavailable.

If CSP is configured, allow the chosen gateway origins in the relevant `img-src`, `media-src` or `connect-src` directives. Keep fetched HTML/SVG out of raw application markup. For offline use, explicitly cache required assets and test a production offline reload; remote content is not automatically precached. IPNS data needs a freshness policy, while immutable CID content can use longer-lived caching.

## Cloudflare Worker configuration

For a separately requested gateway deployment, inspect the current source and use [Cloudflare guidance](cloudflare.md). This source requires a `BANS` KV namespace and a `GATEWAY_PROVIDERS` secret containing upstream gateway base URLs as a JSON array or comma/newline-separated list. It races providers and returns the first successful response, aborting slower requests. Missing providers produce a configuration error; all providers failing produces 502. Do not configure the gateway itself as its own upstream.

`PRIVATE="true"` additionally requires a `REGISTRY` KV allow-list; bans still override allowed IDs. This restricts what the gateway serves, not who owns a decryption key or whether public IPFS data can be fetched elsewhere. Do not describe it as private encrypted storage. Use deployment-specific KV IDs and domain routes instead of copying another account's identifiers. The source currently hardcodes `ipf.sk` host matching, so a different hostname requires adapting its routing parser as well as DNS/Worker routes and TLS.

Active bans/allow-list rejection return 403. Image errors may be SVG placeholders with a failing HTTP status, not the requested image; check status rather than assuming an image body proves retrieval succeeded. Gateway use does not need permission to deploy a Worker, and documentation changes do not provision any service.

## Verification and sources

Verify root and nested paths, CIDv0 path access, CIDv1 subdomains, IPNS names, missing content and browser CORS for the intended use. Test image fallback and offline behavior where relevant. The URL formats above were checked against the workspace `ipfs-worker/README.md` and `src/index.ts`; its parser includes shortcuts beyond the README. Verify the deployed service before claiming successful content retrieval.
