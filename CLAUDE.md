# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Korean-language mobile-first facility guidance PWA for 동화약품(주) (Dongwha Pharmaceutical) HQ "빌딩1897" (서울특별시 중구 서소문로9길 20). Two standalone HTML files with no build system, no package manager, no dependencies installed locally — open `index.html` in a browser (or serve the directory) to run. Three.js is loaded from a CDN (`cdnjs ... three.js/r128`); fonts come from Google Fonts.

## Repository layout

- `index.html` — main app. Single-page, multi-screen flow for employees and visitors. Auth + dashboard + info pages.
- `vr-tour.html` — separate 360° panorama tour using Three.js. Linked from the employee menu in `index.html`. The file is ~3.6 MB because every panorama image is inlined as a base64 JPEG data URL inside `FLOOR_DATA`.

There is no shared code between the two files; each is self-contained (own CSS variables, own `goTo`/screen logic).

## index.html — screen-router architecture

The "app" is a single `#app` shell containing multiple `<div class="screen" id="screen-...">` children. Only one screen has `.active` at a time; transitions are pure CSS (`opacity` + `transform`). Navigation goes through two functions:

- `goTo(id)` pushes the current screen onto a `_history` array and activates the target.
- `goBack()` pops `_history`. **Do not call `goTo` to return to a previous screen — it will double-stack history.** When entering the dashboard, `enterDashboard()` deliberately resets `_history.length = 0` so the back button doesn't return to the login flow.

Screens of note (declaration order matches the `<!-- ① ② ③ -->` comments in markup): home → emp-menu → emp-gate → signup → verify → setpw → login → reverify → reverify-code → employee (dashboard) → visitor → parking / visit / hall.

## index.html — auth flow

The app expects a Firebase v9 modular SDK to be available on `window`:

- `window.$fb` — destructured for `createUserWithEmailAndPassword`, `signInWithEmailAndPassword`, `signOut`, `doc`, `getDoc`, `setDoc`, `updateDoc`, `serverTimestamp`.
- `window.$auth` — Auth instance.
- `window.$db` — Firestore instance, collection `employees` keyed by `cred.user.uid`, fields `{ email, createdAt, lastVerifiedAt, verifiedCount }`.

**The Firebase init script is not currently in this file** — the source comment `🔧 Backend: 추후 Supabase 연동 예정` and the `sendEmail()` stub indicate the backend is in transition. Login/signup will throw at runtime until `window.$fb`/`$auth`/`$db` are populated. When wiring this up, decide explicitly whether to add Firebase init or migrate the calls to Supabase rather than mixing both.

Constraints baked into the flow:

- `ALLOWED_DOMAIN = "@dong-wha.co.kr"` — only this domain is accepted; the UI shows the local part with a domain badge and concatenates on submit.
- `REVERIFY_MS = 6 months` — on login, if `lastVerifiedAt` is older than this, the user is routed to `screen-reverify` instead of the dashboard.
- 6-digit codes are generated client-side (`genCode`) and held in `_pendingCode` with a 10-minute `_codeExpiry`. The "send" step calls `sendEmail()`, which is currently a stub that just `console.warn`s — verification therefore "works" only because the code is also kept in memory client-side. Treat this as dev-only until `sendEmail` is replaced with a real backend call.

## index.html — PWA service worker

`window.load` registers an inline service worker built from a string, served via `URL.createObjectURL(new Blob(...))`. Cache name is `dw-v2` and the SW is cache-first for `'./'`. Bumping the SW source means changing the cache name (`dw-v2` → `dw-v3`) so old clients invalidate.

## vr-tour.html — Three.js panorama tour

Single config object drives everything:

```
FLOOR_DATA = {
  'bf'|'1f'|'3f'|'4f': {
    label, names[], imgs[],          // imgs = base64 data: URLs
    arrows: { [panoIdx]: [{target, lon, lat, label}, ...] },
    hotspots: { [panoIdx]: [{lon, lat, color, id}, ...] }
  }
}
```

- `names[i]`, `imgs[i]`, `arrows[i]`, `hotspots[i]` are all indexed by the pano index `cur`.
- Arrows navigate between panos on the same floor (`go(target)`); hotspots open a popup keyed by `id` (`openPopup`). Popup content for each `id` is built inside `openPopup` — adding a new hotspot id requires both a config entry and a popup case.
- `lon`/`lat` are degrees on the inside of a sphere; `toPos(lon, lat, r)` converts them to a Three.js Vector3. When tuning hotspot/arrow placement, edit those numbers directly in `FLOOR_DATA`.
- `startTour(floor)` decodes every base64 image in parallel via `b64toBlob` → `Image` → `THREE.Texture`, then swaps `sph.material.map` on `go()`. Floor switching disposes the previous textures (`clearObjects` + reassigning `tex=[]`).
- This file has its own landing page, visit-guide screen, and floor-select screen — independent from `index.html`'s screens. Don't try to share `goTo` between the two files.

Adding a new pano = append a base64 JPEG to `imgs`, append a name, and add the corresponding `arrows`/`hotspots` entries keyed by the new index. The file size will grow noticeably; consider whether the image really needs to be inlined or whether a regular URL would be acceptable.

## Style / tokens

Both files independently define CSS custom properties on `:root`. The brand palette is `--red: #C8102E` (primary), `--gold: #B8963E`, with `Noto Serif KR` for display headings and `Noto Sans KR` for body. Mobile shell width is capped at 480 px in `index.html`. When editing one file, do not assume the other shares the token.

## Conventions

- All user-facing copy is Korean. Keep new strings in Korean unless the surrounding context is English (e.g. taglines like `DONGWHA PHARMACEUTICAL · EST. 1897`).
- Phone number `02-2021-6000` and the HQ address appear in multiple places — search before changing.
- No linter, no formatter, no tests. Verify changes by opening the HTML in a browser (mobile viewport recommended) and walking the relevant screen flow.
