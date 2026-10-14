# Site configuration, menus, branding, translations and SEO

Inspect vite.config.ts, src/app.d.ts, Header.svelte, Footer.svelte, Icon.svelte and `[[lang]]/+layout.svelte` together. Config types alone do not guarantee a renderer implements a feature. Edit the existing siteConfigClient object and languageConfig; avoid replacing app-specific modules/settings.

## Configuration layers

- `sveltekit({ preprocess, adapter, ... })`: framework options.
- Top-level Vite resolve/plugins/define/ssr: build tooling and aliases.
- `siteConfigClient`: client-visible title, canonical base URL, organization/project names, favicon, language, themeConfig, api and modules. JSON is injected as `__SITE_CONFIG__` and read with getSiteConfig().

Keep only serializable values in injected config. Use string icon names and auth action keys; JSON.stringify removes functions/component constructors. Keep secrets out of every nested field. `PUBLIC_SITE_URL` controls the current site's build URL; set the real HTTPS origin for deployment, not localhost.

## Navigation

Under themeConfig.navbar:

```ts
navbar: {
    logo: {
        src: '/img/logo.svg',
        srcDark: '/img/logo-dark.svg',
        alt: 'Project',
        attachTitle: true
    },
    style: 'blur',
    orientation: 'horizontal',
    hideOnScroll: false,
    itemsPosition: 'center',
    iconExternal: true,
    items: [
        { label: 'navbar.home', to: '/', icon: 'house', position: 'left' },
        {
            label: 'navbar.community',
            position: 'right',
            submenu: [
                { label: 'navbar.contact', to: '/contact' },
                { label: 'GitHub', href: 'https://github.com/your-org', icon: 'github', target: '_blank', rel: 'noopener noreferrer' }
            ]
        }
    ],
    authItems: [
        { label: 'common.logout', action: 'signout', show: 'loggedIn', icon: 'log-out' }
    ]
}
```

Use `to` for a real local route, `href` for an external URL or mailto/tel, and `action` for a handler key. Avoid defining competing actions/links on one item. `position: left/right` controls horizontal grouping; `itemsPosition: side/center` controls the main alignment. submenu contains nested NavbarItem entries. Provide labels for accessible social links.

Auth action keys currently include signin/login, signout/logout, register, manualConnect and disconnectWallet; Header resolves these at runtime. Visibility `show` supports loggedIn/loggedOut, connected/disconnected, authIn/authOut, always/hide, and `and`/`or` with parentheses. UI visibility is not server authorization. Confirm that auth modules and the referenced handlers exist.

Icons are lowercase kebab-case keys. Icon.svelte has a few inline social SVGs (including github and x); Lucide names resolve through src/lib/components/icon/iconRegistry.ts. If a requested icon shows ×, import the named component from @lucide/svelte and add it to that registry. Do not assume every Lucide name is already registered or that a URL is accepted as an icon. For custom image icons extend the renderer explicitly and validate the source.

## Fixed, hiding and vertical menu

| Desired behavior | Supported setting / change |
| --- | --- |
| Always-visible top menu | orientation: horizontal, hideOnScroll: false; current header uses fixed positioning |
| Hide on downward scroll, reappear on upward scroll | hideOnScroll: true |
| Desktop sidebar | orientation: vertical; layout reserves sidebar width, mobile stays responsive |
| Blurred / transparent / theme-adaptive surface | style: blur / transparent / auto |
| Sticky within document flow | No sticky flag exists; change Header positioning classes and layout offsets together |
| Draggable/movable by the visitor | No drag-position setting exists; implement state, pointer/keyboard behavior and bounds if requested |

Do not invent style: sticky, orientation: movable or a sticky boolean. For true sticky positioning remove conflicting fixed offsets, choose the sticky container/top, and test short/long content, mobile, scroll hiding and page overlap. Mirror a shared component change in the template and demo.

## Footer groups and liner

Under themeConfig.footer:

```ts
footer: {
    style: 'transparent',
    logo: { src: '/img/logo-footer.svg', alt: 'Project' },
    iconExternal: true,
    links: [
        {
            title: 'footer.community',
            items: [
                { label: 'footer.contact', to: '/contact' },
                { label: 'GitHub', href: 'https://github.com/your-org', icon: 'github', target: '_blank', rel: 'noopener noreferrer' }
            ]
        }
    ],
    liner: [
        { label: 'footer.privacyPolicy', to: '/terms/privacy' },
        { label: 'footer.termsOfService', to: '/terms/service' }
    ],
    copyright: 'footer.copyright'
}
```

links creates titled groups; liner is the separate single-row legal/utility menu. Footer style supports light, dark or transparent. FooterLinkItem supports label, to/href, target, rel, icon, className, action and show; verify renderer support when adding unusual combinations. Define referenced local routes instead of shipping dead links. Copyright translations can contain `{year}` as supported by the footer helper.

## Dark, light and automatic theme

colorMode is under themeConfig:

| Initial behavior with no saved choice | Configuration |
| --- | --- |
| Dark | defaultMode: dark, respectPrefersColorScheme: false |
| Light | defaultMode: light, respectPrefersColorScheme: false |
| System/automatic | defaultMode: dark or light fallback, respectPrefersColorScheme: true |

Use quoted strings in TS config. defaultMode only accepts light/dark, not auto/system. With system enabled the switch rotates system → light → dark; otherwise it toggles light/dark. disableSwitch hides the control; existing stored choices can still override the default, so test a clean context and a context with mota.theme set. Use the storageKeys helper to reset preference rather than clearing unrelated localStorage. Navbar style auto is a visual surface mode, independent from the colorMode system preference.

## Logos, favicon and social icons

Place assets in static/img; reference them as /img/name.svg. Navbar uses src, srcDark, alt, optional attachTitle and className; footer has its own logo. favicon is the siteConfigClient field. Keep PWA manifest icons and apple-touch icon consistent with branding. Social icons are ordinary nav/footer link items backed by the icon renderer, not SEO metadata. Logo rendering selects an asset according to theme and navbar surface; test both light/dark with the chosen style.

## Translations

Set the separate languageConfig used by `siteConfigClient.language`:

```ts
const languageConfig = {
    enabled: true,
    icon: 'languages',
    showName: true,
    availableLocales: ['en', 'sk'],
    defaultLocale: 'en',
    autoDetect: true
};
```

Every code must have a dictionary under src/i18n/{code}/index.ts and belong to the generated typesafe-i18n locale union. Set language.code, language.descriptiveName and optional language.icon/rtl consistently with existing dictionaries. Names can be loaded asynchronously from those fields. Add nav/footer/meta keys to base and target dictionaries. Keep variable placeholders aligned.

Run `npm run i18n:extract` (currently typesafe-i18n --no-watch) to regenerate types/utilities; this command generates from dictionaries and does not automatically translate arbitrary UI prose. Use i18n:watch during translation edits when appropriate. Do not hand-edit generated i18n-types. With translations disabled the current t() helper returns the key itself: use readable literals or enable dictionaries when replacing visible labels.

`[[lang]]` is optional: the default locale has an unprefixed URL and /en redirects when en is default. Non-default locales use /sk/..., etc. Explicit URL locale wins over stored/browser preference; autoDetect controls browser fallback. Do not add a locale only in Vite without generating its files. Request-specific SSR dictionaries must not leak through shared stores.

## SEO, Open Graph and social images

Site title is a literal title; Header splits it at `|` for brand/powered-by display. themeConfig.metadata entries are rendered by the locale layout: name/property + content create meta tags; rel + href create link tags. Meta content goes through t(), so it can be a dictionary key; canonical href is not automatically translated.

```ts
metadata: [
    { name: 'description', content: 'seo.description' },
    { property: 'og:type', content: 'website' },
    { property: 'og:title', content: 'seo.title' },
    { property: 'og:description', content: 'seo.description' },
    { property: 'og:url', content: 'https://example.com/' },
    { property: 'og:image', content: 'https://example.com/img/social/home.png' },
    { property: 'og:image:alt', content: 'seo.imageAlt' },
    { name: 'twitter:card', content: 'summary_large_image' },
    { name: 'twitter:title', content: 'seo.title' },
    { name: 'twitter:description', content: 'seo.description' },
    { name: 'twitter:image', content: 'https://example.com/img/social/home.png' },
    { rel: 'canonical', href: 'https://example.com/' }
]
```

Create the referenced image in static/img/social, using a crawler-friendly PNG/JPEG (commonly 1200×630) and absolute HTTPS image URLs. Set the actual domain and real descriptions, not example.com or untranslated keys. For per-page title/canonical/images, render route-aware head values and avoid duplicate global/page tags; one global canonical to / is wrong on other routes. Supply locale-aware canonical/hreflang where appropriate. Use server-rendered head tags for social crawlers, verify raw production HTML and image HTTP responses, and check robots/noindex. Changing an image path avoids stale social preview caches more reliably than editing bytes behind the same URL.

For financial discovery add FinTag metadata separately; see [payment protocols](payment-protocols.md).

For optional hosted image icons and the difference from registered menu icon names, see [CoreCDN](corecdn.md). For login/signup menu flows, see [Better Auth](auth/better-auth.md).
