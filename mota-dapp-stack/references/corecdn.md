# Optional CoreCDN icons

[CoreCDN source](https://github.com/bchainhub/corecdn) provides public icons hosted at [corecdn.info](https://corecdn.info). Image consumption needs no npm package or account. Choose an existing asset from the repository rather than guessing names.

Paths have the form `/{mark|badge}/{size}/{name}.{svg|png}`. `base` holds source SVGs; exported sizes include 16, 24, 32, 48, 64, 72, 96, 128, 144, 192, 256, 512 and 1024. Marks and badges are different visual treatments.

```svelte
<img
    src="https://corecdn.info/mark/256/xcb.png"
    alt="Core Blockchain"
    width="32"
    height="32"
/>
```

Set dimensions to reserve layout space and use meaningful alt text; use `alt=""` for a purely decorative icon next to equivalent text. Keep a controlled list of icon names for dynamic asset selection. The repository also supports jsDelivr delivery, for example <https://cdn.jsdelivr.net/gh/bchainhub/corecdn/mark/256/xcb.png>.

## MOTA integration

For navbar/footer logos, a CDN image URL can be used in the existing `logo.src` image field; configure light/dark variants as needed. See [site configuration](site-config.md).

The current `Icon.svelte` string API resolves registered Lucide/social icon names. It does not interpret a CDN URL as an image. To use CDN images in menu or footer icon slots, add an explicit image/component renderer and update its config type and callers, or use an image in a custom component. Do not paste a CDN URL into `item.icon` and claim it works automatically. Render remote SVGs through an image element instead of injecting fetched markup.

If the site has CSP, add the CDN host to its existing `img-src` policy while preserving the other required sources. Regular remote image display does not require fetching the SVG into JavaScript. Remote icons are not automatically available offline: use licensed local assets under `static/` for guaranteed app-shell availability, or configure and test an explicit runtime image cache. Test a production offline reload and the missing-image fallback; see [PWA checks](sveltekit-3.md).
