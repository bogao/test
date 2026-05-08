# ccs

`ccs` - short for **Common Components** — shared browser modules, and more.

Small, framework-free, jsDelivr-friendly browser modules (button state
machines, modals, drawers, popovers, toasts, i18n, theming) plus a
handful of server-side helpers for Cloudflare Workers. Each browser
module is one self-contained IIFE / CSS file. No bundler, no module
loader; just `<script>` / `<link>` tags.

## Install

Via npm (resolves through jsDelivr — no `npm install` needed):

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/overlay/style.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/toast/style.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/spinner/style.min.css">

<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/i18n-engine/client.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/action/client.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/overlay/client.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/popover/client.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/toast/client.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/footer-brand/client.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@pckgs/ccs@1.0.1/theme/client.min.js"></script>
```

Pin a specific version (recommended) or drop `@<version>` for "latest".

`<script>` tags are parser-blocking and run in document order — load
`i18n-engine` and `action` *before* `overlay` (overlay reads
`window.tr` and `window.Action`).

## Modules

### action — async-button state machine

Wraps a `<button>` into `idle → (validate) → pending → success / error → idle`
states. Use it for any button that triggers async work (form submit,
fetch, multi-step flow).

```js
const btn = Action.create({
  text: 'Save',
  pendingText: 'Saving…',
  successText: 'Saved',
  retryText: 'Retry',
  validate: () => emailIsValid() || 'Email looks wrong',
  asyncFn:  async () => { await fetch('/api/save', { method: 'POST' }); },
  onSuccess: () => location.reload(),
});
container.appendChild(btn);

// Or wrap an existing button:
Action.wrap(document.querySelector('#submit'), {
  asyncFn: async () => { /* ... */ },
});
```

`window.Action.create(opts)` returns the button element (controller on
`btn._ctl`); `Action.wrap(btn, opts)` returns the controller directly.
The controller exposes `getState()`, `setEnabled()`, `setText()`,
`reset()`, `trigger()`, `detach()`.

### overlay — Modal, Drawer, inline overlay

A unified covering-layer primitive. Three variants cover the common cases:

- `box` — page-fixed centered box (Modal-style)
- `edge-right` — page-fixed slide-in panel (Drawer-style)
- `flat` — container-absolute overlay (in-section loading / busy state)

Two scopes: `'page'` (full-screen, with focus-return / scroll lock /
Esc / click-outside) or any `HTMLElement` (scoped, no page-level side
effects).

```js
// Modal sugar — back-compat aliases on window.Modal:
Modal.confirm('Delete this item?', { onOk: () => doDelete() });
Modal.alert('Saved successfully.');
Modal.input('Project name?', { onOk: (val) => createProject(val) });

// Drawer (slide from right):
const drawer = Overlay.show({
  variant: 'edge-right',
  title: 'Settings',
  body: settingsPanelEl,
});
// later: drawer.close()

// Loading overlay scoped to a section:
await Overlay.run({
  variant: 'flat',
  scope: tableContainerEl,
  asyncFn: async () => { await reloadTable(); },
});
```

Requires `window.tr` (a `(key) => string` lookup, caller-provided)
and `window.Action` (load `action/client.min.js` first). If `tr`
isn't set, falls back to English defaults for built-in labels.

### popover — non-modal click-outside lifecycle

For caller-built popups (context menus, dropdowns) where you want
click-outside / Esc to close, but don't want the modal-style focus
trap or scroll lock.

```js
triggerBtn.addEventListener('click', (e) => {
  e.stopPropagation();   // prevent the click-outside detector from firing immediately
  myDropdownEl.style.display = 'block';
  Popover.show({
    el: myDropdownEl,
    closable: { escape: true, clickOutside: true },
    onClose: () => { myDropdownEl.style.display = 'none'; },
  });
});
```

You manage markup, positioning, and show/hide; `Popover` only manages
*when to close*.

### toast — bottom-right notifications

Top-down stacked, auto-dismissed.

```js
Toast.ok('Saved');                        // green, ~2s
Toast.err('Failed: ' + err.message);      // red, ~4.5s
Toast.show('Hello', 'ok');                // explicit kind
```

Container `<div id="toasts"></div>` is auto-created if missing. Drop
`<div id="toasts"></div>` in your markup if you want to control its
position. RTL pages get bottom-left placement automatically.

### spinner — CSS-only loading circle

```html
<div class="spinner"></div>
```

20×20 px ring, `@keyframes spin` rotation, no JS. Good as a button label,
inline-with-text indicator, or inside other components.

### footer-brand — branded `<footer>` IIFE

Drops a small, lang-aware footer at the bottom of the page. After your
i18n setup runs, call:

```js
FooterBrand.applyLang(currentLang);
```

Override `--footer-color` / `--footer-border` CSS vars if you want it
tinted differently.

### i18n-engine — language detection + DOM translation + LangSelect

20-language detection (including RTL: `ar`, `he`) with sensible Chinese
fallback (`zh-{hant,tw,hk,mo}` → `zh-tw`, other `zh*` → `zh-cn`).

```js
const SUPPORTED = ['en','fr','de','zh-cn','zh-tw','ja','ar', /* ... */];
const lang = detectLang(SUPPORTED);     // walks navigator.languages
applyLocaleAttrs(lang);                  // sets <html lang> + <html dir>

const TRANSLATIONS = {
  en: { hello: 'Hello', save: 'Save' },
  fr: { hello: 'Bonjour', save: 'Sauvegarder' },
  // ...
};
applyI18nAttrs(TRANSLATIONS[lang]);      // walks [data-i18n] / [data-i18n-ph] / [data-i18n-title]

// Optional <select> for switching:
//   <select id="lang-select">...</select>
LangSelect.init(lang, (newLang) => {
  localStorage.setItem('lang', newLang);
  location.reload();
});
```

The engine functions (`detectLang`, `isRTL`, `applyLocaleAttrs`,
`applyI18nAttrs`) plus constants (`SUPPORTED_LANGS`, `RTL_LANGS`) are
exposed at script global scope. `LangSelect` is on `window.LangSelect`.

### theme — dark/light toggle button

Drop in a button, load the IIFE, the rest is automatic — reads
`localStorage('theme')`, applies `data-theme` to `<html>`, wires
the click handler:

```html
<button id="themeToggle">🌓</button>
<script src=".../theme/client.min.js"></script>
```

Define `--surface` / `--text` / etc. with two values gated on
`html[data-theme="dark"]` in your CSS.

> **Caveat:** theme reads `localStorage`. In strict tracking-prevention
> browsers (Edge), cross-origin scripts may be blocked from accessing
> `localStorage`, breaking persistence. If you need theme persistence
> in Edge, vendor this module into your own first-party bundle instead
> of loading it from jsDelivr.

## Required CSS variables

Modules read CSS custom properties on `:root`. Provide:

- All modules: `--surface`, `--border`, `--text`, `--text-muted`, `--accent`, `--err`
- `toast`: also `--ok`
- `spinner`: uses `--border` and `--accent`
- `footer-brand`: optionally `--footer-color`, `--footer-border`

## License

MIT — see [LICENSE](LICENSE).

---

## Internal (Gitea-only)

If you're consuming `ccs` from npm or jsDelivr, the section above is all
you need. The rest is for `ccs` source-repo developers.

### Layout

```
ccs/
├── action/          async-button state machine (browser IIFE)
├── footer-brand/    branded <footer> markup + IIFE
├── i18n-engine/     detectLang / applyI18nAttrs / LangSelect (browser IIFE)
├── lock/            lock-page assembly (server runtime + browser IIFE + i18n)
├── markdown-editor/ toolbar + textarea + preview-modal (browser IIFE + i18n)
├── overlay/         unified covering layer (Modal / Drawer / inline)
├── popover/         click-outside / Esc lifecycle for app popups
├── response/        server-side json/html/text Response factory
├── spinner/         loading-circle CSS
├── theme/           dark/light toggle button + IIFE
├── toast/           bottom-right notifications (browser IIFE + CSS)
└── upload2kv/       blob-chunked KV upload (server runtime + browser IIFE)
```

The 4 modules NOT in the npm package (`lock`, `markdown-editor`,
`upload2kv`, `response`) are build-time composers (lock,
markdown-editor) or server-side helpers (response, upload2kv/runtime).
They need access to the source tree, not just minified IIFEs, so they
ship only via sibling-clone consumption — not jsDelivr.

### Sibling clone layout

```
~/Documents/                 ← parent dir (any name)
├── ccs/                     ← this repo (Gitea = source of truth)
└── tinycfw/                 ← consumer; build.mjs imports ../../../ccs/<mod>/build.mjs
    └── apps/<name>/build.mjs
```

Initialize a new machine:

```bash
git clone https://git.11270115.xyz/gaobo/ccs.git
cd ccs && npm install && cd ..
git clone https://git.11270115.xyz/gaobo/tinycfw.git
```

### Publishing

| Channel | URL form | What's published | Trigger |
|---|---|---|---|
| `bogao/test/ccs/` (GitHub mirror) | jsDelivr `gh/bogao/test@<SHA>/ccs/<mod>/<file>.min.*` | minified browser modules + README + LICENSE | `npm run publish:test` |
| `@pckgs/ccs` (npm) | jsDelivr `npm/@pckgs/ccs@<ver>/<mod>/<file>.min.*` + npmjs.com | the `bogao/test/ccs/` tree wrapped with workflow-generated `package.json` | `npm run publish:release` (dispatches `bogao/test`'s `publish-ccs.yml` via GitHub Actions; OIDC trusted publishing — no npm token needed locally) |

`scripts/publish.mjs` minifies in-memory, lazy-clones `/tmp/ccs-publish-<target>/`,
writes minified files plus README.md / LICENSE, then commits + pushes.

### Cross-repo invariant

Any `ccs` change → all sibling consumers regression-test (rebuild +
redeploy). Current consumers:

- `tinycfw` (apps: `mailgunfire`, `mixssl`, `shurl`)
- (add new consumers here)

**Exception (`[no-runtime-impact]`):** if the change has zero impact on
the published browser-module bytes (e.g. README / publish.mjs / CI yml
/ pure comments / build.mjs internal refactor), tag the commit message
subject with `[no-runtime-impact]` to signal "skip sibling regression".
Default = full regression.
