# Budget — Personal Budget PWA

A single-user, offline-first Progressive Web App for tracking cash, multi-currency
balances, per-person debts, and recurring payments. No backend, no accounts, no cloud.
**All data stays on your device** (IndexedDB), with JSON export/import as the backup.

---

## File tree

```
/
├── index.html            ← the app
├── manifest.webmanifest  ← PWA manifest
├── sw.js                 ← service worker (offline app shell)
├── idb-keyval.js         ← vendored IndexedDB helper (idb-keyval v6, ESM)
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   └── apple-touch-icon.png   (180×180)
└── README.md
```

---

## Run locally

The app needs to be **served over HTTP(S)** for the service worker and the
local ES-module import (`./idb-keyval.js`) to work. Opening `index.html` from a
`file://` path will load the UI but the module import and service worker are
blocked by browsers, so use a tiny static server:

```bash
# Python 3 (already on most systems)
python -m http.server 8000
# then open http://localhost:8000/
```

```bash
# or Node, if you prefer
npx serve .
```

The app still runs as a plain static page — there is **no build step, no bundler,
no framework.** It's vanilla HTML/CSS/JS.

> Note: the original prototype opened by double-clicking the file. Because the
> storage layer now uses an ES-module import, a local server is required for
> module loading and offline caching. This is the standard way to serve a PWA.

---

## Storage choice

- **On-device storage = IndexedDB**, accessed through the tiny
  [`idb-keyval`](https://github.com/jakearchibald/idb-keyval) library.
- `idb-keyval` is **vendored locally** as `idb-keyval.js` (no CDN dependency), so
  the app works fully offline on a cold launch — the service worker caches it
  alongside the app shell.
- `idb-keyval` stores structured objects directly, so the whole `state` object is
  persisted as-is (no `JSON.stringify`).
- On boot the app calls `navigator.storage.persist()` to request durable storage,
  making iOS less likely to evict your data.
- **Backups are the real safety net:** use **Settings → Export backup (JSON)**
  regularly. Restore with **Settings → Import backup (JSON)** (it validates the
  file and asks for confirmation before overwriting).

---

## Replacing the icons

The icons in `icons/` are simple placeholders — a gold **B** glyph (`#e0a83c`) on
the dark app background (`#0e1413`). Replace them with your own art at the same
sizes and filenames:

- `icons/icon-192.png` — 192×192
- `icons/icon-512.png` — 512×512 (also used as the maskable icon)
- `icons/apple-touch-icon.png` — 180×180 (iOS home-screen icon)

After replacing icons (or any cached asset), bump the `CACHE` name in `sw.js`
(e.g. `budget-v1` → `budget-v2`) so clients pick up the new files.

---

## Deploy

The app is a static site and needs an **HTTPS URL** to be installable.

### Option A — Cloudflare Pages or Netlify (recommended; keeps the repo private)
Free tier serves a **private** repo.
1. Push this folder to a private Git repo.
2. Connect it in Cloudflare Pages / Netlify.
3. **Build command:** none. **Output directory:** repo root (`/`).
4. You get an HTTPS URL.

### Option B — GitHub Pages (repo must be public)
Free GitHub Pages requires a **public** repo. This code contains no secrets and
no data, so that's acceptable.
1. Push to a public GitHub repo.
2. **Settings → Pages → Deploy from a branch → `main` / root.**
3. You get an HTTPS URL.

### Install on iPhone
Open the HTTPS URL in **Safari → Share → Add to Home Screen.**
It launches full-screen, runs offline after the first load, and stores all data
on-device.

---

## Notes / assumptions (things intentionally left unchanged)

- **The prototype's behavior and appearance are untouched.** Only the storage
  layer was swapped (`window.storage` → IndexedDB via `idb-keyval`), the document
  was wrapped into a complete HTML page with PWA `<head>` tags, the **Import**
  button was added in Settings, and the service worker was registered.
- The `<script>` was changed to `<script type="module">` so it can `import`
  `idb-keyval`. Module scripts defer automatically; the app's IIFE still runs on
  load.
- **v1 → v2 migration:** the prototype migrated an old `budget-data-v1` store from
  the Claude-only `window.storage` API. Since that API doesn't exist in a
  standalone PWA, the migration now reads `budget-data-v1` from IndexedDB instead.
  For a fresh install there's nothing to migrate and it's a no-op.
- **Out of scope for v1 (not built):** cloud sync, accounts/auth, extra
  charts/graphs, multi-device sync, auto-posting recurring payments without a tap,
  and in-place editing of transactions (delete + re-add is the supported flow).
- The Google Fonts `@import` still loads from the network. It is **not** required
  for the app to function offline — without it the UI falls back to the system
  sans/mono stack defined in the CSS. If you want fonts cached offline too, vendor
  them locally and add them to `ASSETS` in `sw.js`.
