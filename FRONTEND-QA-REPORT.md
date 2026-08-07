# Mandan Parivar — Frontend QA Report

**Site:** https://mandanparivar.com/
**Tested:** 7 August 2026
**Build:** `version.json` → `mp_frontend` v3.3.4 (build 23)
**Stack:** Flutter Web, CanvasKit renderer, Cloudflare, Firebase (auth/analytics/messaging)
**Method:** Automated Chromium (Playwright 1.56) driving the live site. Desktop 1440×900 and mobile 390×844. Elements located via the Flutter semantics tree and clicked by coordinate, then verified from screenshots. Signed-out only.

---

## How to read this

| Column | Meaning |
|---|---|
| **Severity** | P1 = blocks or badly damages the visit · P2 = high · P3 = medium · P4 = polish |
| **Confidence** | **Confirmed** = reproduced, or read straight from the API/headers · **Needs check** = observed once or twice, please verify on real hardware |

### Two caveats before you start

1. **Rendering was software (SwiftShader), not GPU.** The test browser logged `Automatic fallback to software WebGL`. CanvasKit paints far slower there than on a real machine, so **all timing figures are worst-case, not what your users see.** Findings about *what* rendered are unaffected; findings about *how long it took* are inflated.
2. **Payload sizes, HTTP headers, API content and text/typos are exact** — those came from direct requests, not from the browser, and are unaffected by the above.

---

## P1 — Critical

| # | Page URL | Where / how to reproduce | What's wrong | Suggested fix | Confidence |
|---|---|---|---|---|---|
| 1 | `https://mandanparivar.com/short-videos` | Paste the URL straight into a fresh tab (do **not** click through from the homepage) | Page renders **completely blank** — no header, no footer, nothing. 5 uncaught JS errors on one run. On a second run it stalled after Firebase init and never called its content API at all. Zero accessibility nodes, so it isn't just a paint delay — the screen was never built. | Reproduce with DevTools open and read the Dart stack. Likely an unguarded null/parse in the Short Videos screen's init that throws before first build. Wrap the screen's data load in a `try/catch` with an error state, and add a global `ErrorWidget.builder` so a screen-level crash shows a retry UI instead of a blank canvas. | **Confirmed** (2 runs) |
| 2 | `https://mandanparivar.com/contact-us`, `/search`, `/privacy-policy`, `/terms-conditions` | Open each URL **directly** in a new tab. Then compare: load the homepage and reach the same page by clicking the footer link. | On direct load these rendered nothing within ~20s. Reached by clicking inside the app, the **same** routes render fine (`/contact-us` → 22 nodes, `/privacy-policy` → 51 nodes). So deep links are the broken path — which is every Google result, every WhatsApp/social share, every bookmark. | Check the router's initial-route handling: state the screen depends on (app-settings, i18n, auth bootstrap) is probably resolved on the home screen only, so a cold start on a deep route waits forever. Make each screen fetch its own prerequisites, and add a load timeout with a visible error+retry. | **Needs check** — software rendering may exaggerate this. Verify on a real machine before estimating. |
| 3 | `https://mandanparivar.com/` | Homepage hero carousel. Also visible directly at `https://data.prodapi.mandanparivar.com/api/v1/hero-section` | **Production is serving hero banners from the dev/staging file host.** All three slides point at `https://files.devapi.mandanparivar.com/...` (`Vision_Jinshasan_Banner_3.jpg`, `Banner_3.jpg`, `Banner_1.jpg`). Every other image on the site correctly uses `files.prodapi`. If the dev environment is ever cleaned, resized or taken down, the homepage hero breaks. | Re-upload the three banners to `files.prodapi.mandanparivar.com` and update the `thumbnailUrl` of hero rows 11, 10 and 8. Then add a CI/CMS validation that rejects any `devapi` URL in production content. | **Confirmed** — read directly from the production API |
| 4 | `https://mandanparivar.com/` | Hard-reload with cache disabled and watch the screen | After the branded loader clears (~1.5s), the user gets a **plain colour gradient with no spinner, no skeleton and no text** until content appears. Nothing tells them the site is still working. | Keep the branded Lottie loader on screen until first meaningful paint instead of dismissing it early, or render a skeleton immediately. Fixing #10 (payload) shortens the window; this fixes the perception either way. | **Confirmed** that the gap exists (duration inflated by software rendering) |

---

## P2 — High

| # | Page URL | Where / how to reproduce | What's wrong | Suggested fix | Confidence |
|---|---|---|---|---|---|
| 5 | `https://mandanparivar.com/shala/gaanam`, `/shala/pathanam` (all `/shala/*`) | Look at the top navigation pill | The Devanagari nav labels render as **empty tofu boxes** (`▯▯▯▯`) instead of शाला / शोभा / सेवा / दानम् / संवाद. The same labels render correctly on the homepage. **Root cause:** `assets/FontManifest.json` bundles only `MaterialIcons` and `CupertinoIcons` — every Indic glyph is fetched at runtime from `fonts.gstatic.com` (7 TTFs, ~1.8 MB). When that fetch is slow or blocked, text paints with no font and Flutter doesn't re-layout. | Bundle the Devanagari/Gujarati fonts as local assets in `pubspec.yaml` instead of using runtime `google_fonts`. This removes the tofu **and** cuts ~1.8 MB of blocking network. If runtime fetch must stay, `await GoogleFonts.pendingFonts()` before first paint. | **Confirmed** — reproduced on multiple pages; root cause verified in FontManifest |
| 6 | `https://mandanparivar.com/shobha` | Open Mandan Shobha and wait | Grey **skeleton placeholder cards never resolve into content** — the events list stays empty, then a large blank band, then the footer. The event data *does* exist (it is in the accessibility tree), so the cards are failing to paint their content. | Check the Shobha events list for an image/layout exception swallowed per-card. Add an explicit empty/error state so a failed load says so rather than showing skeletons forever. | **Confirmed** |
| 7 | `https://mandanparivar.com/checkout.js` | Open that URL, or open DevTools → Console on any page | `index.html` contains `<script src="checkout.js">` but **the file does not exist**. The server answers `200` with the HTML of the homepage, so the browser parses HTML as JavaScript and throws a syntax error on **every page load**. | Either restore `checkout.js` or delete the `<script>` tag from `index.html`. Also see #8 — the soft-404 is what turns a missing file into a confusing parse error. | **Confirmed** |
| 8 | `https://mandanparivar.com/nonexistent-page-xyz` (any bad path) | `curl -I` any URL that doesn't exist | Every unknown path returns **HTTP 200** with the app shell. The app then draws a correct "Page not found" screen, but search engines see `200 OK` and will index unlimited junk URLs. It also masks missing assets (#7). | Serve a real `404` status for unmatched paths, or at minimum exclude them via `robots.txt` + a `noindex` meta on the not-found state. Keep the SPA fallback only for known route prefixes. | **Confirmed** |
| 9 | `https://mandanparivar.com/main.dart.js` | `curl -sI https://mandanparivar.com/main.dart.js \| grep -i cache` | The **5.7 MB** app bundle is served `cache-control: no-cache`. The browser must revalidate on *every* visit before it may reuse it. Same for `AssetManifest.bin.json`. | Give build outputs hashed filenames and serve them `cache-control: public, max-age=31536000, immutable`. Keep `no-cache` for `index.html` only. Biggest single win for repeat visits. | **Confirmed** |
| 10 | `https://mandanparivar.com/` | DevTools → Network, disable cache, reload | The homepage pulls **~19 MB** before it is usable: `main.dart.js` 5.7 MB, `canvaskit.wasm` 5.6 MB, **`pdfium.wasm` 3.9 MB**, **`rive_native.wasm` 2.7 MB**, plus ~1.8 MB of runtime fonts. PDFium and Rive are downloaded on the homepage even though nothing there shows a PDF. On Indian mobile data this is a very long wait. | Lazy-load `pdfrx`/PDFium only on screens that open a PDF, and Rive only where an animation plays (deferred imports). Bundle fonts locally (#5). Consider `--wasm` (skwasm) or trimming CanvasKit. Realistic target: under 3 MB to first paint. | **Confirmed** |
| 11 | `https://mandanparivar.com/` on mobile (390×844) | Load on a phone, look at the "360° Virtual Tour" hero slide | The pink/white headline over the light banner is **very close to unreadable** — well below WCAG AA contrast. The desktop crop hides this; the mobile crop puts the text over the brightest part of the photo. | Add a dark scrim/gradient behind hero text (e.g. `rgba(0,0,0,0.45)`), or bake a darkened band into the banner. Verify ≥4.5:1 against the *lightest* pixel the text can land on. | **Confirmed** |
| 12 | `https://mandanparivar.com/` | Scroll slowly past the Gyaanam and Vachanam sections and watch the floating header | The translucent header pill is **not opaque enough**: book covers and quote cards scroll visibly through it and collide with the nav labels, which become illegible over busy artwork. | Increase the header's background opacity (or add a stronger blur + solid tint) so content never reads through it. Simplest robust fix: solid background once `scrollOffset > 0`. | **Confirmed** |
| 13 | Every page | Compare the browser tab title on `/`, `/shala/gyanam`, `/danam`, `/contact-us` | **Every page has the identical title "Mandan Parivar".** Bookmarks, browser history, tab switching and Google results are indistinguishable. No per-page meta description either. | Set a route-specific title (and description) on navigation — e.g. `SystemChrome.setApplicationSwitcherDescription` / a `web` title updater per route: "Gyaanam — Books & Publications | Mandan Parivar". | **Confirmed** |
| 14 | `https://mandanparivar.com/` | `curl -s https://mandanparivar.com/ \| grep -c "Pravachanam"` → `0` | CanvasKit paints into a **single `<canvas>`**: the served HTML contains **zero** body text, links or images. Search engines and social/WhatsApp previews see only the static `<meta>` tags — none of your actual pravachans, books, articles or events are indexable. | This is inherent to CanvasKit. Options, cheapest first: (a) add per-route SSR/prerendered HTML snapshots for content pages; (b) publish an `articles`/`books` sitemap with server-rendered landing pages; (c) switch content pages to the HTML renderer. At minimum add a real `sitemap.xml` (#28). | **Confirmed** |

---

## P3 — Medium

| # | Page URL | Where / how to reproduce | What's wrong | Suggested fix | Confidence |
|---|---|---|---|---|---|
| 15 | `https://mandanparivar.com/` | Hero slide 1 title | Typo: **"Vison Jinshasan Action Plan 2026-27"** — should be **"Vision"**. Live on the homepage hero. (The same title is spelled correctly on `/shala/gyanam`.) | Fix `title` on hero-section row id 11 in the CMS. | **Confirmed** — from the API |
| 16 | `https://mandanparivar.com/articles` and homepage "Latest Articles" | Read the article card titles | Internal language tags are exposed to users: **"(GUJ) If Liberation is our Goal…"**, **"(HIN) Guidance on Chandanbala Jain Temple"**. 5 titles affected. | Move language into a proper `language` field and render it as a badge/filter, not as a title prefix. Strip the existing `(GUJ)`/`(HIN)`/`(ENG)` prefixes from stored titles. | **Confirmed** — from the API |
| 17 | `https://mandanparivar.com/shobha` | First event card, end of the venue line | Placeholder junk in live content: *"Parinati Parva Updhan 2025 — Shri Mahavirdham Jain Tirth, Shirsad **abc**"*. | Remove `abc` from that event's venue field. Add a CMS check for stub values before publish. | **Confirmed** — seen on desktop and mobile |
| 18 | `https://mandanparivar.com/` | Gyaanam book carousel | A book is titled **"VJAP_11 Jainism for the New Gen"** — an internal code leaking into a user-facing title. | Rename to the display title; keep `VJAP_11` in an internal reference field. | **Confirmed** |
| 19 | `https://mandanparivar.com/samvad`, `https://mandanparivar.com/shala/pathanam` | Click **MANDAN संवाद** in the top nav; and the **Pathanam** tab under Shala | Both are **"Coming soon"** empty pages — but both are promoted as primary navigation. 1 of 5 top-level nav items and 1 of 6 Shala tabs are dead ends. | Hide or visibly disable ("Coming soon" badge, not clickable) until content ships, so the nav doesn't promise what isn't there. | **Confirmed** |
| 20 | `https://mandanparivar.com/live-updates` | Open the URL | Route exists in the app's route table but renders **"Page not found"**. | Either implement it or remove the route so it can't be linked. | **Confirmed** |
| 21 | `https://mandanparivar.com/` | Click the hero's ▶/◀ arrows quickly, or watch autoplay advance | While a banner loads, the hero shows a **broken-image placeholder icon** and a grey bar where the title goes. It reads as "this site is broken" rather than "loading". Banners are 113–224 KB and are not preloaded. | Preload the next slide's image; use a blurred low-res placeholder or the brand gradient instead of the broken-image icon. | **Confirmed** |
| 22 | `https://mandanparivar.com/` | Turn on a screen reader, or inspect the Flutter semantics tree | Many controls have **no accessible name** — hero prev/next arrows, card `>` chevrons, share and bookmark buttons all announce as just "button". Fails WCAG 2.1 SC 4.1.2. | Wrap icon-only buttons in `Semantics(label: 'Next slide')`, `'Share article'`, `'Save to collection'`, etc. | **Confirmed** |
| 23 | `https://files.prodapi.mandanparivar.com/images/Anekant_Playlist.png` | DevTools → Network → sort by size on the homepage | Photographic images shipped as **PNG**: `Anekant_Playlist.png` is **1.26 MB**, `AGAM AMRUT PRAVACHAN SHRENI 1.png` 388 KB. These are thumbnails a few hundred px wide. | Convert photos to WebP/JPEG and serve responsive sizes (Cloudflare Images or `?width=` variants). Should drop ~1.5 MB from the homepage. | **Confirmed** |
| 24 | `https://mandanparivar.com/` | Hero slide 2 "Explore" → destination | The 360° tour links to a raw Netlify subdomain: `https://mandanparivar-udaygirikhandgiri-caves.netlify.app/`. Off-brand and looks untrustworthy to a cautious visitor. | Put it on a subdomain you own, e.g. `tour.mandanparivar.com`. | **Confirmed** |
| 25 | `https://mandanparivar.com/`, `/shala/pathanam` | Scroll to just above the footer | A **large empty band** (~350 px+) sits between the last content section and the footer, filled only by a faint "MANDAN PARIVAR" watermark. Reads as a failed-to-load section. | Tighten the bottom spacing, or make the watermark band an intentional sized element rather than leftover slack. | **Confirmed** |
| 26 | `https://mandanparivar.com/` | Footer | **"Contact Us" appears twice** — once in the link column, once in the bottom legal bar. | Remove one. | **Confirmed** |
| 27 | `https://mandanparivar.com/` | Hero slide 2 title | Title is `"Virtual Tour "` with a **trailing space**; 4 other titles in the dashboard feed contain **double spaces**. | Trim/collapse whitespace on save in the CMS. | **Confirmed** — from the API |
| 28 | `https://mandanparivar.com/sitemap.xml` | Open it | **No sitemap** — returns the homepage HTML. Combined with #14 (no crawlable text), search engines have almost nothing to index. | Generate a `sitemap.xml` covering all content routes and reference it from `robots.txt`. | **Confirmed** |
| 29 | Any page | `curl -sI https://mandanparivar.com/` | Missing security headers: **no `Content-Security-Policy`**, **no `Strict-Transport-Security`**, **no `X-Frame-Options`/`frame-ancestors`** (site can be framed → clickjacking). `X-Content-Type-Options`, `Referrer-Policy` and `Permissions-Policy` are correctly set. | Add via Cloudflare Transform Rules: `Strict-Transport-Security: max-age=31536000; includeSubDomains`, `X-Frame-Options: SAMEORIGIN`, and a CSP (start `report-only`). | **Confirmed** |
| 30 | `https://mandanparivar.com/` on mobile | Load on a phone and view the "Vision Jinshasan" slide | Landscape banners are letterboxed into a tall mobile hero, leaving **large empty margins** above and below the artwork, and the ◀ ▶ arrows sit on top of the image. | Supply a portrait crop per banner (`thumbnailUrlMobile`), or use `BoxFit.cover` with a defined mobile aspect ratio. | **Confirmed** |

---

## P4 — Polish

| # | Page URL | Where / how to reproduce | What's wrong | Suggested fix | Confidence |
|---|---|---|---|---|---|
| 31 | Any page | DevTools → Console | Recurring `Failed to load resource: net::ERR_INVALID_URL` — something requests a malformed/empty URL (likely an image with a null `src`). Harmless today but it hides real errors. | Find the empty URL and guard it; skip the network call when the field is blank. | **Confirmed** |
| 32 | `https://mandanparivar.com/shala/gaanam` | Look closely at the six section tabs | Each tab label has a **stray underscore-like mark before and after** it — `‾Pravachanam‾`, `‾Gyaanam‾`. Looks like a broken underline or literal underscore characters. | Inspect the tab label widget for a `TextDecoration.underline` on a padded/empty span, or literal `_` in the string. | **Confirmed** |
| 33 | `https://mandanparivar.com/samvad` | Look at "Coming soon" | Text is **not horizontally centred** (sits ~40 px right of centre on a 1440 viewport). | Centre the placeholder. | **Confirmed** |
| 34 | `https://mandanparivar.com/` | Hero slide 3 with the site language set to English | English headline "LISTEN PAUSE REFLECT" is paired with a **Gujarati** subtitle while `lang=en`. Confusing for an English-preferring visitor. | Provide per-language banner variants, or keep the whole slide in one language. | **Confirmed** |
| 35 | `https://mandanparivar.com/contact-us` | Contact section | Public contact address is a personal-style Gmail: `mandanparivar2023@gmail.com`. | Consider `contact@mandanparivar.com` for credibility. | **Confirmed** |

---

## What worked correctly

Worth recording so it isn't re-tested:

- **Header navigation** — MANDAN शाला → `/shala/pravachanam`, सेवा → `/seva`, दानम् → `/danam`, संवाद → `/samvad`, Signup/Login → `/profile`. All fire correctly.
- **Content cards** — clicking a Pravachanam card routes correctly (`/shala/pravachanam/4/1`); URL updates, so routing and browser history work.
- **Footer links** — Contact Us → `/contact-us`, Privacy Policy → `/privacy-policy` both work via in-app navigation.
- **Hero carousel** — autoplays and both arrows advance/reverse correctly.
- **404 screen** — unknown routes show a proper "Page not found" with a "Go to Home" action (though the HTTP status is wrong, see #8).
- **Legal pages** — `/return-refund-policy` and `/shipping-delivery-policy` render correctly on direct load.
- **Mobile layout** — below the hero, sections reflow sensibly at 390 px; no horizontal overflow observed.
- **APIs** — `hero-section`, `dashboard`, `app-settings` all returned `200` with valid payloads; no 4xx/5xx anywhere across 23 routes.

---

## Not covered

- **Signed-in areas** — profile, bookmarks/collection, cart, checkout, payment callbacks, notifications (no test account).
- **Hindi and Gujarati** (`?lang=hi`, `?lang=gu`) — English only this pass.
- **Safari / iOS** — Chromium only in this environment. Given the CanvasKit + wasm stack, iOS Safari deserves its own manual pass.
- **Tablet breakpoint** (~768 px).
- **Keyboard-only navigation and full contrast measurement** beyond the hero.

---

## Suggested order of work

1. **#1, #2** — pages that render blank. Nothing else matters if a shared link opens to nothing.
2. **#3** — get production off the dev image host before it bites.
3. **#9, #10** — caching and payload. Cheapest large win for real users on mobile data.
4. **#5, #6, #12, #11** — the visible breakages: tofu text, stuck skeletons, header collision, unreadable mobile hero.
5. **#15–18, #26, #27** — content/copy fixes; minutes of CMS work each.
6. **#13, #14, #28** — SEO, if search traffic matters to you.
