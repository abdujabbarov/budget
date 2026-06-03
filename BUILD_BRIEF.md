# Build Brief — Personal Budget PWA

**For:** Claude Code (VSCode)
**Goal:** Turn the existing working prototype (`budget.html`) into a deployable, installable, offline-capable Progressive Web App for a single user, with on-device storage and JSON backup/restore. No backend, no server, no cloud database.

---

## 0. Read this first

You are given a **working single-file prototype**: `budget.html`. It already implements all the app logic and UI. **Do not redesign it and do not change its behavior or appearance.** Your job is purely to:

1. Swap the storage layer from `window.storage` (a Claude-only API) to real on-device storage (IndexedDB).
2. Add JSON **Import** alongside the existing Export.
3. Wrap it as an installable PWA (manifest + service worker + icons + correct meta tags).
4. Produce a clean deployable file tree and a short README with deploy steps.

Preserve every existing feature exactly (see §4 acceptance checklist). If you think something should change, leave it and note it at the end instead of changing it.

---

## 1. Constraints / decisions (already made — do not relitigate)

- **Single user, single device.** No login, no multi-user, no sync server.
- **Vanilla HTML/CSS/JS.** No React, no build step, no bundler, no framework. Keep it a small static site that runs by opening `index.html`.
- **Storage = IndexedDB** via the tiny `idb-keyval` library (ESM from a CDN, or vendored locally — see §3).
- **Backup = JSON export/import**, treated as the real safety net.
- **Offline-first PWA**, installable to iOS home screen via Safari "Add to Home Screen".
- Money records never leave the device.

---

## 2. Target file tree

```
/
├── index.html          ← the app (renamed from budget.html, with storage swapped)
├── manifest.webmanifest
├── sw.js               ← service worker
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   └── apple-touch-icon.png   (180×180)
└── README.md           ← deploy + install instructions
```

Generate placeholder icons (a simple solid-background square with a "₿"-style or "$"/"B" glyph in the app's gold `#e0a83c` on the dark `#0e1413` background is fine). Note in the README that the user can replace them.

---

## 3. Storage swap (the one critical change)

In `budget.html` the storage layer currently looks like this (search for `window.storage`):

```js
const KEY = 'budget-data-v2';
async function loadState(){ try{ const r=await window.storage.get(KEY); return r?JSON.parse(r.value):null; }catch(e){ return null; } }
async function saveState(){ try{ await window.storage.set(KEY, JSON.stringify(state)); }catch(e){ console.error(e); } }
```

Replace **only these two functions** with IndexedDB via `idb-keyval`. Add near the top of the `<script>`:

```js
import { get as idbGet, set as idbSet } from 'https://cdn.jsdelivr.net/npm/idb-keyval@6/+esm';
```

(If you prefer no CDN dependency for offline robustness, vendor `idb-keyval` into the repo and import locally — your call, but document it.)

Then:

```js
const KEY = 'budget-data-v2';
async function loadState(){ try{ return (await idbGet(KEY)) || null; }catch(e){ return null; } }
async function saveState(){ try{ await idbSet(KEY, state); }catch(e){ console.error(e); } }
```

Note: `idb-keyval` stores structured objects directly, so store the `state` object itself (no `JSON.stringify`). The existing `await saveState()` call sites all continue to work unchanged.

**Important:** because the `<script>` now uses `import`, change the script tag to `<script type="module">`. Confirm the IIFE still runs (module scripts defer automatically; that's fine).

### Durable storage request
On boot, after `loadState()`, request persistent storage so iOS is less likely to evict data:

```js
if (navigator.storage && navigator.storage.persist) { navigator.storage.persist(); }
```

---

## 4. Feature acceptance checklist (must all still pass)

The prototype already does these. After your changes, verify each still works:

- [ ] **Income / Spend** entries via the **+** FAB. FAB shows only on Home and Ledger tabs and offers only Spend and Income.
- [ ] **Available cash** on Home updates live with every entry.
- [ ] **Three currencies** (UZS / KGS / USD) editable in Settings, each with a rate relative to a selectable base; all totals convert to base.
- [ ] **Debts = per-person accounts.** Adding a loan is done only from the Debts tab ("+ New loan"), never the FAB. Each person shows one net balance (green = owes you, red = you owe).
- [ ] **Partial & full repayment:** tapping a person opens their account with a payment box; "Pay in full" autofills the outstanding; direction (money in vs out) is automatic; net shrinks toward zero; per-currency breakdown and full history are shown.
- [ ] **Recurring payments** are clickable; the detail view has a "Pay this month" button that logs the expense and charges cash, with an "Undo" that reverses it; paid months are tracked to avoid double-paying.
- [ ] **Statistics** on Home: monthly income, monthly spent, owed-to-me, I-owe, monthly recurring commitment, balances by currency.
- [ ] **Export** JSON backup works.
- [ ] Data **persists across full app restarts** (close tab / kill app, reopen → data still there). This is the proof the storage swap worked.

### New: Import
Add an **Import backup (JSON)** button in Settings next to Export. It should:
- open a file picker (`<input type="file" accept="application/json">`),
- parse the file, validate it has the expected top-level keys (`base`, `currencies`, `tx`, `debts`, `recur`),
- confirm with the user before overwriting (`confirm()` is fine),
- replace `state`, `saveState()`, and `render()`.

---

## 5. PWA shell

### `manifest.webmanifest`
```json
{
  "name": "Budget",
  "short_name": "Budget",
  "start_url": "./index.html",
  "scope": "./",
  "display": "standalone",
  "background_color": "#0e1413",
  "theme_color": "#0e1413",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### `<head>` additions to `index.html`
```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="theme-color" content="#0e1413">
<link rel="manifest" href="manifest.webmanifest">
<link rel="apple-touch-icon" href="icons/apple-touch-icon.png">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<title>Budget</title>
```
(The prototype is currently a fragment with no `<head>`/`<html>`. Wrap it into a complete HTML document.)

### `sw.js` — minimal offline cache
Cache-first for the app shell so it launches offline. Keep it simple and bump `CACHE` on changes:

```js
const CACHE = 'budget-v1';
const ASSETS = ['./', './index.html', './manifest.webmanifest',
  './icons/icon-192.png', './icons/icon-512.png', './icons/apple-touch-icon.png'];
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(ASSETS)).then(() => self.skipWaiting()));
});
self.addEventListener('activate', e => {
  e.waitUntil(caches.keys().then(ks => Promise.all(ks.filter(k => k !== CACHE).map(k => caches.delete(k)))).then(() => self.clients.claim()));
});
self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
```

Register it at the end of the app script:
```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => navigator.serviceWorker.register('./sw.js'));
}
```

**Caveat to handle:** if `idb-keyval` is loaded from the CDN, it won't be cached by the service worker and the app won't fully work offline on first cold launch. Prefer **vendoring `idb-keyval` locally** (download the ESM file into the repo, import it relatively, and add it to the `ASSETS` array). Do this and note it in the README.

---

## 6. Deployment (put both options in the README)

The app is a static site and needs an HTTPS URL for installability.

- **Option A — Cloudflare Pages or Netlify from a PRIVATE repo (recommended if keeping code closed):** free tier serves a private repo. Connect the repo, no build command, output dir = repo root. Gives an HTTPS URL.
- **Option B — GitHub Pages:** free Pages requires the repo to be **public**. The code contains nothing sensitive (no keys, no data), so this is acceptable. Settings → Pages → deploy from branch (root).

After deploy: open the HTTPS URL in **Safari on iPhone → Share → Add to Home Screen.** Launches full-screen, runs offline, data stored on-device.

---

## 7. Explicitly OUT of scope for v1 (do not build these)

- Cloud sync / accounts / auth
- Charts/graphs beyond the existing simple spend bar
- Multi-device sync
- Auto-posting recurring payments without a tap (keep it manual/confirmed)
- Editing existing transactions in place (delete + re-add is fine for now)

If you have spare time, the single highest-value optional add is **partial-payment history clarity** and **an Import validation guard** — both already requested above. Otherwise stop at a clean, installable, offline PWA that passes §4.

---

## 8. Definition of done

- `index.html` runs by double-click (works as plain file) AND when served.
- All §4 boxes pass; Import added and guarded.
- Installs to iOS home screen; launches offline after first load.
- Data survives app kill/restart.
- README documents: how to run locally, the storage choice, how to replace icons, and both deploy options.
- A short note at the bottom of the README listing anything you intentionally left unchanged or any assumptions you made.
