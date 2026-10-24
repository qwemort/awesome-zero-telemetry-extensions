# Awesome Zero-Telemetry Extensions (2026)

> A practical guide to browser extensions that work **fully offline, collect no data, and survive Manifest V3** — with reproducible tests you can run yourself.

**Disclosure:** this list is maintained by the developer of [PureJSON](#purejson), [PureSession](#puresession) and [PureHeaders](#pureheaders), which appear in it. We're biased, so every claim about another product links to a public source, every claim about ours can be checked with the [test fixtures](fixtures/) in this repo, and corrections are welcome via issues and pull requests. Last reviewed: **October 2026**.

**Our extensions:**
- **PureJSON** (JSON viewer) — [Microsoft Edge](https://microsoftedge.microsoft.com/addons/detail/jgaejimgiijadilkfklimmmgacjabijd) · [Firefox](https://addons.mozilla.org/firefox/addon/json-viewer-formatter-purejson/) · Chrome: coming soon
- **PureSession** (tab & session manager) — [Microsoft Edge](https://microsoftedge.microsoft.com/addons/detail/ojfadjpeagojacihpdnbepiljnadeiio) · [Firefox](https://addons.mozilla.org/firefox/addon/puresession-tab-manager/) · Chrome: coming soon
- **PureHeaders** (HTTP headers & CORS) — [Microsoft Edge](https://microsoftedge.microsoft.com/addons/detail/ilodajgedphnnmafkapjcldjbjbkcmfj) · Firefox: in store review · Chrome: coming soon

---

## Contents

- [Why this list exists: the Manifest V3 reality in 2026](#why-this-list-exists-the-manifest-v3-reality-in-2026)
- [How we evaluate extensions](#how-we-evaluate-extensions)
- [Category 1 — JSON viewers & formatters](#category-1--json-viewers--formatters)
- [Category 2 — Tab & session managers](#category-2--tab--session-managers)
- [Category 3 — HTTP header & CORS tools](#category-3--http-header--cors-tools)
- [FAQ](#faq)
- [Coming next](#coming-next)
- [Contributing](#contributing)

---

## Why this list exists: the Manifest V3 reality in 2026

Manifest V3 is now the only extension platform in Chrome and Edge. It changed the trade-offs for extensions that handle your data:

**1. Background code can be stopped at any moment.**
MV3 extensions run their background logic in a service worker. Chrome stops it after **30 seconds without events**, when a single event takes **more than 5 minutes**, or when a `fetch()` takes **more than 30 seconds** to respond ([Chrome docs](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)). Anything kept only in memory is lost when that happens. Extensions that store your work must persist it continuously and in small steps, not in one big write at the end.

**2. IndexedDB is atomic — but your data model has to be, too.**
An IndexedDB transaction either commits fully or not at all, so a stopped worker does not half-write a record. What breaks is application logic: deleting the previous backup *before* the new one is committed, overwriting a good state with an empty one after a crash, or migrating storage formats without a fallback. Storage migrations are the riskiest moment — Session Buddy's move from Web SQL to IndexedDB in v4 is the best-known example of users reporting lost saved sessions ([user reports summarised here](https://supasidebar.com/blog/session-buddy-alternatives-2026) — note: written by a competing vendor).

**3. Timers are coarse.**
`chrome.alarms` fires at most every **30 seconds** (Chrome 120+) ([Chrome docs](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)). Backup and sync features must be event-driven, with alarms only as a safety net.

**4. Extensions with access to every page are a supply-chain risk.**
An extension with a content script on `<all_urls>` can read and change every page you open. When such an extension changes owners or business model, an update can quietly change what it does. In 2026, developers reported that the popular **JSON Formatter** extension went closed-source and began injecting geolocation tracking and donation prompts into websites ([Wes Bos](https://wesbos.com/json-alexander)). Prefer extensions that request the narrowest permissions, explain each one, and don't send data anywhere.

---

## How we evaluate extensions

| Criterion | What we check |
|---|---|
| **Zero telemetry** | No analytics, no remote config, no requests except ones the user explicitly triggers. Checked in DevTools → Network and in the store's privacy disclosure. |
| **Offline** | Core features work with the network disabled. |
| **Permissions** | Each permission is necessary and explained. `<all_urls>` content scripts are flagged. |
| **Safe rendering** | Untrusted text (page titles, JSON values) is rendered as text, not HTML. Test: [`fixtures/xss.json`](fixtures/xss.json). |
| **Correctness** | JSON: integers above 2^53 stay exact ([`fixtures/bigint.json`](fixtures/bigint.json)), 20 MB+ files open without freezing ([how to generate](fixtures/README.md)). |
| **Data ownership** | Full export in an open format, free; import from competitors. |
| **Pricing** | One-time vs subscription; account required or not. |

Cells marked **—** mean *not yet tested by us*. Send a PR with your result and the browser version.

---

## Category 1 — JSON viewers & formatters

Chrome and Edge show raw JSON as plain text; Firefox has a built-in viewer. Extensions add a tree view, search and export — and, with `<all_urls>` access, a trust question.

| | Open source | Telemetry / injected content | Exact integers > 2^53 | 20 MB+ files | Export | Price |
|---|---|---|---|---|---|---|
| **Firefox built-in viewer** | ✅ (part of Firefox) | ✅ none | — | — | Save raw JSON only | Free |
| **JSON Formatter** (Chrome Web Store) | ❌ closed in 2026 ([source](https://wesbos.com/json-alexander)) | ⚠️ tracking & injected prompts reported ([source](https://wesbos.com/json-alexander)) | — | — | — | Free |
| **JSON Formatter** (arnav-kr fork) | ✅ | ✅ none reported | — | reported slow above ~5 MB ([source](https://dev.to/wayknow123/i-tested-every-json-viewer-after-the-formatter-scandal-heres-what-i-found-4dcl)) | — | Free |
| **JSON Alexander** (Wes Bos) | ✅ ([source](https://wesbos.com/json-alexander)) | ✅ none | — | — | — | Free |
| <a id="purejson"></a>**PureJSON** *(ours)* | ❌ (unminified, reviewable in the package) | ✅ none; only request is an opt-in license check | ✅ custom parser keeps the exact source digits | ✅ lazy rendering; tested with 21 MB / 120k records | CSV, Excel-friendly CSV, TSV, TypeScript (Pro) | Free core; Pro one-time |

### Why big numbers break in most JSON viewers

`JSON.parse` turns every number into an IEEE-754 double. Integers above `2^53 − 1` (`9007199254740991`) lose their last digits: `1847293001200007919` becomes `1847293001200008000`. Twitter/X and Discord IDs (Snowflake IDs), many database keys and some banking identifiers are that large. A viewer that uses plain `JSON.parse` silently shows the wrong ID. Test any viewer with [`fixtures/bigint.json`](fixtures/bigint.json).

---

## Category 2 — Tab & session managers

| | Where data lives | Account needed | Free tier | Paid | Side panel / sidebar | Import from others | Encrypted sessions |
|---|---|---|---|---|---|---|---|
| **Session Buddy** | Local | No | ✅ | — | — | ✅ (own JSON format) ([formats](https://sessionbuddy.com/formats)) | — |
| **OneTab** | Local | No | ✅ | — | — | Text import | — |
| **Toby** | Cloud | — | Up to 60 saved tabs ([source](https://toolradar.com/tools/toby/pricing)) | $6 per member / month (≈ $72 / year) ([source](https://toolradar.com/tools/toby/pricing)) | — | — | — |
| **Workona** | Cloud | Yes | No permanent free plan ([source](https://toolradar.com/tools/workona/pricing)) | Custom pricing ([source](https://toolradar.com/tools/workona/pricing)) | — | — | — |
| <a id="puresession"></a>**PureSession** *(ours)* | Local (IndexedDB) | No | ✅ live tabs, sleep tabs, dedupe, sessions, crash recovery, automatic backups (restore the newest), 1 password-locked session, full JSON backup | One-time Pro: full backup history, unlimited encrypted sessions, deep search, Markdown/CSV | ✅ Chrome/Edge Side Panel, Firefox sidebar | ✅ Session Buddy, Toby, OneTab, bookmark HTML, URL lists | ✅ AES-256-GCM, PBKDF2-SHA-256 600,000 iterations |

**OneTab note:** OneTab's own troubleshooting page warns that reinstalling the extension wipes saved data ([one-tab.com/troubleshooting](https://www.one-tab.com/troubleshooting)). Whatever manager you use, export a backup regularly.

### What "crash-proof" should mean

A session manager is only as good as its worst moment. Questions worth asking any of them:

1. Does it write state continuously (debounced, small atomic writes), or only on demand?
2. After a crash the browser often opens with one empty window. Can that empty state overwrite your last good backup?
3. When it rotates old backups, does it delete only **after** the new backup is committed?
4. Can you export **everything**, free, in an open format?

PureSession's answers: debounced atomic writes 3 s after tab changes; empty states are never stored; new backups are committed before the oldest is deleted; full JSON export is free.

---

## Category 3 — HTTP header & CORS tools

Header extensions need the most trust of all: they can see every request to the sites they work on, and you usually store API tokens in them. In **July 2026** Microsoft and Google removed **ModHeader** (about 1.6 million users) after researchers found a dormant data-collection pipeline in a signed release: it could fingerprint the browser, record visited domains and upload them, switched on by a server-side change. Analyses found no evidence it had actually been switched on ([analysis](https://pasqualepillitteri.it/en/news/8063/modheader-pulled-chrome-dormant-spyware), [TechRadar](https://techradar.com/pro/security/experts-get-google-microsoft-to-pull-trusted-modheader-with-1-6-million-installs-after-finding-it-could-harvest-all-kinds-of-data)). Store removal does not uninstall it from browsers that already have it.

| | Open source | Site access at install | CORS with cookies | Firefox | Imports ModHeader | Price |
|---|---|---|---|---|---|---|
| **ModHeader** | ❌ | — | — | — | — | Removed from Chrome Web Store and Edge Add-ons in July 2026 ([source](https://pasqualepillitteri.it/en/news/8063/modheader-pulled-chrome-dormant-spyware)) |
| **Modify Header** (SelectorsHub) | ✅ ([source](https://selectorshub.com/modify-header/)) | — | — | — (Chrome and Edge listed) | — (generic bulk import) | — |
| **Headerbench** | Source-available, PolyForm Shield ([source](https://github.com/Darkhouse13/headerbench)) | ✅ none — asked per profile ([source](https://github.com/Darkhouse13/headerbench)) | — | ❌ Chrome and Edge ([source](https://github.com/Darkhouse13/headerbench)) | ✅ ([source](https://github.com/Darkhouse13/headerbench)) | Free + Pro |
| <a id="pureheaders"></a>**PureHeaders** *(ours)* | ❌ (unminified, reviewable in the package) | ✅ none — asked per site you add, released when you remove it | ✅ exact page origin + `Allow-Credentials`, only for the pages you list | ✅ | ✅ old and new JSON formats, with a report of what didn't carry over | Free core; Pro one-time |

### What to do if you used ModHeader

1. **Uninstall it** — removing it from the store doesn't remove it from your browser — and check synced profiles so it isn't reinstalled.
2. **Rotate** the API keys, bearer tokens and session cookies you stored in its headers.
3. Before uninstalling, **export your profiles** (Export → Export JSON) if you want to move them to another tool. The file holds your tokens: keep it private and delete it after importing.

### Why "Allow CORS: *" breaks logins

Browsers refuse a credentialed request (`fetch(url, { credentials: 'include' })`, cookies, most logins) when the response says `Access-Control-Allow-Origin: *` ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/Errors/CORSNotSupportingCredentials)). A CORS fix that works with logins has to echo the **exact** page origin and add `Access-Control-Allow-Credentials: true`.

The bigger risk is scope. A switch that adds those headers **for every site** lets any page you visit read responses from sites where you're logged in. Questions worth asking any CORS or header tool:

1. Does it work only on the sites and pages you list, or globally?
2. Does it ask for site access when you add a site, or for all sites at install?
3. Is it obvious when rules are on (badge, timer), so nothing stays on by accident?
4. Does it store your tokens only locally, without cloud sync or remote config?

One limit no extension can lift: if the server answers the browser's `OPTIONS` preflight with an error (404, 405), the request fails whatever headers are added — the server has to allow `OPTIONS`.

---

## FAQ

**What is a local, offline alternative to Session Buddy in 2026?**
Any manager that stores data in the browser (IndexedDB) and offers a full export. OneTab and Session Buddy are local; PureSession is local and adds side-panel UI, crash recovery and encrypted sessions. Before switching, export your Session Buddy collections (Session Buddy → Export → JSON); Session Buddy's format is documented at [sessionbuddy.com/formats](https://sessionbuddy.com/formats).

**Which tab manager works in the Chrome Side Panel without a cloud account?**
Extensions built on the `chrome.sidePanel` API (Chrome 114+) can run entirely locally. PureSession is one; check the store's privacy disclosure of any other for "No data collected".

**How do I stop `JSON.parse` from rounding 64-bit IDs (Snowflake IDs)?**
Use the reviver's `context.source` argument, which gives the original text of each primitive value, and convert large integers to `BigInt` or keep them as strings ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse)):

```js
const data = JSON.parse(text, (key, value, context) =>
  typeof value === 'number' && !Number.isSafeInteger(value) && /^-?\d+$/.test(context?.source ?? '')
    ? BigInt(context.source)
    : value
);
```

Check browser support for the `context` argument on MDN; a `core-js` polyfill exists. Better still, have the API send IDs as strings.

**Why does my JSON viewer freeze on large files?**
Rendering every node into the DOM at once. A 20 MB file can have millions of nodes. Viewers that render lazily (only expanded nodes, paged long arrays) stay responsive.

**Is a JSON viewer extension safe to install?**
It needs to read pages to detect JSON, so it usually has access to every site. Prefer one that discloses "No data collected", renders values as text (test with [`fixtures/xss.json`](fixtures/xss.json)), and makes no network requests you didn't trigger. The 2026 JSON Formatter incident ([source](https://wesbos.com/json-alexander)) shows why the developer's track record matters.

**How do I move from OneTab to another tab manager?**
OneTab → *Export URLs* gives `url | title` lines. Most managers, including PureSession, import that text directly.

**Why does a tab manager say it can "read your browsing history"?**
Browsers show that warning for the `tabs` permission, which is needed to read tab titles and URLs. It does not by itself give access to page content or the network. Check whether the extension also has content scripts or host permissions, and what its privacy disclosure says.

**Does Manifest V3 delete my saved sessions?**
No — MV3 itself doesn't touch extension storage. Data loss happens when an extension migrates its storage format or overwrites good data with an empty state. Keep an exported backup.

**What is a safe ModHeader alternative after the 2026 removal?**
Pick one that asks for site access per site instead of all sites at install, stores profiles only locally, has no remote configuration, and imports ModHeader's JSON export. See the [Category 3 table](#category-3--http-header--cors-tools) and rotate any tokens you kept in ModHeader.

**How do I fix "Credential is not supported if the CORS header 'Access-Control-Allow-Origin' is '*'"?**
The response must name the exact origin of your page (for example `http://localhost:3000`) and include `Access-Control-Allow-Credentials: true`; `*` is not allowed with cookies ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/Errors/CORSNotSupportingCredentials)). Fix it on the server, or for local development use a tool that sets those headers only for your dev page.

---

## Coming next

### Category 4 — User-Agent & Client Hints switchers
*Being tested.* What we'll check: consistent `User-Agent` + `Sec-CH-UA-*` headers + `navigator.userAgentData`, per-tab scope, no telemetry.

---

## Contributing

- Found a wrong cell or an outdated price? Open an issue with a link to the source.
- Tested an extension with the fixtures? Send a PR with the result, browser and version.
- Want an extension added? It must work offline for its core feature and disclose its data collection.

Maintainer: [@qwemort](https://github.com/qwemort), developer of PureJSON, PureSession and PureHeaders · Contact: papana44444@gmail.com
