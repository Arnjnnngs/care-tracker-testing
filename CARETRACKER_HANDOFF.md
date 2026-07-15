# CareTracker TESTING — AI Agent Handoff Document

> **THIS IS THE STAGING VARIANT.** `TEST_MODE = true` in `index.html` — writes to
> `caretracker_test_entries` (never `caretracker_entries`), push/local notifications fully disabled,
> orange "🧪 Testing app" banner in the header. sw.js cache is `caretracker-testing-vN`. Features
> under test here that are NOT (yet) in production: chemo cycle system, missed-dose alerts,
> menstrual cycle tracking, In-Patient day tracking, Morphine pain-level scale, Zofran as-needed.
> Promote to prod by porting the relevant changes into `care-tracker`'s `index.html` with
> `TEST_MODE = false` and a prod cache bump — **only when Aaron explicitly says to.**

> **Purpose:** Complete context for any AI assistant to understand, maintain, and extend this repo
> without prior knowledge. See `CLAUDE.md` first for the non-negotiable rules.
>
> **Last updated:** July 14, 2026
> **Current version:** v29 (see README.md's versioning convention — this repo's version is always
> "current live prod version + 1" while testing is ahead)

---

## 1. What This Project Is

CareTracker is a **progressive web app (PWA)** that tracks medications, vitals, menstrual cycle, and
hospital in-patient days for a family caregiver (caring for Brandi). It's a **single-file vanilla
JavaScript app** — no build step, no framework. The entire app lives in `index.html`. Firebase
Firestore provides the database with real-time sync. This build has push notifications disabled.

**Core user flow:** a caregiver opens the app, taps a quick-log button (e.g. "500 mg" Tylenol), and
the dose instantly syncs to Firestore across devices. The app enforces dosing limits/gaps where
relevant, tracks a chemo cycle and flags missed doses, and now also tracks menstrual cycle and
in-patient hospital days.

## 2. Links

| Resource | URL |
|---|---|
| **Live App (testing)** | https://arnjnnngs.github.io/care-tracker-testing/ |
| **Live App (production)** | https://arnjnnngs.github.io/care-tracker/ |
| **GitHub Repository** | https://github.com/arnjnnngs/care-tracker-testing |
| **Firebase Console** | https://console.firebase.google.com/project/fuelforge-7c132 |

## 3. Repository Structure

```
care-tracker-testing/
├── CLAUDE.md                   # Standing rules — read first
├── firebase-messaging-sw.js    # FCM service worker, present but unused (TEST_MODE disables push)
├── icon-192.png                # PWA icon 192x192
├── icon-512.png                # PWA icon 512x512
├── index.html                  # THE ENTIRE APP — HTML + CSS + JavaScript
├── manifest.webmanifest        # PWA manifest (name: "CareTracker TESTING")
├── reset.html                  # Nukes service workers + caches, redirects to app
└── sw.js                       # App service worker — caching + notification click handler
```

**No build step.** Edits to `index.html` deploy by pushing to `main` — GitHub Pages serves the repo
root. **There is no `.github/workflows/` and no `send-reminders.js` in this repo** — unlike
production, the scheduled/gap-based reminder cron system doesn't exist here at all.

## 4. Tech Stack Details

- **Language:** Vanilla JavaScript, `<script type="module">` in the shipped app (imports Firebase
  from the gstatic CDN). A separate QA copy rewrites this to a classic `<script>` with a mocked
  Firestore, since jsdom does not reliably execute module scripts.
- **Rendering:** Custom `h()` hyperscript-style DOM builder + full `render()` on every `setState()`.
- **Styling:** Inline `<style>` + inline `style` objects on every element. **Theme is light pink
  glassmorphism** (`#FFF0F3` background, `#AA5375`/`#9B5B8A` accents) — do not describe this as dark;
  that's production's old theme, not this one.
- **Backend:** Firebase Firestore, project `fuelforge-7c132` (shared with prod; isolated by
  collection name, see below). No auth.
- **Hosting:** GitHub Pages.

## 5. Firebase Collections

### `caretracker_test_entries` (this app's collection — `COL_NAME` when `TEST_MODE` is true)
Each document is one logged event. Fields actually written by this app's code:

- `medId` — `"dexamethasone" | "tylenol" | "zofran" | "compazine" | "morphine" | "lidocaine" | "imodium" | "protonix" | "buspirone" | "paroxetine" | "iron" | "senokot" | "temp" | "weight" | "chemo_date" | "cycle_start" | "cycle_end" | "inpatient"`
- `ts` — ms-since-epoch of the event
- `dose` — human-readable label (e.g. `"1000 mg"`, `"½ tab · 7.5 mg"`) or `null`
- `mg` — numeric mg (0 for non-mg entries)
- `pills` — pill/application count, only present when relevant
- `temp` / `weight` — numeric value on vitals entries
- `override` — boolean, present when logged early past a lock
- `painLevel` — integer 1–10, present only on Morphine or Tylenol doses where a level was selected (new in v29; Tylenol added same day as Morphine)
- `loggedAt` — present on `chemo_date` records; used to find the most recently *set* chemo date (the
  record's own `ts` is the chemo date itself, which can be in the future)

`cycle_start` / `cycle_end` and `inpatient` records carry no special fields beyond the common ones —
they're pure event markers, always logged at `Date.now()` (no time picker).

### `caretracker_entries`
Production's real collection. **This app must never write here.**

## 6. Medication & Feature Definitions

| ID | Display Name | Rules |
|---|---|---|
| `dexamethasone` | Dexamethasone | `chemoOnly: true` — only appears in Quick Log when `dexActiveOn(day)` (day −1 to +1 relative to chemo date). 2 tablets, 8 AM & 2 PM windows. Missed-dose tracked. |
| `tylenol` | Tylenol | Daily max 2500 mg, 4h min gap, 500/1000 mg doses. **`painScale: true` (v29)** — same 1–10 "Pain level" dropdown as Morphine |
| `zofran` | Zofran | **As-needed (v29): `gapH: 0`, no lock, no reminder.** Still blocked on chemo days 0–1 via `zofranBlockedOn()` — that's a clinical rule, independent of the gap timer, and was not removed |
| `compazine` | Compazine | 6h min gap, shown in Evening meds card |
| `morphine` | Morphine | 4h min gap, ½ tab (7.5 mg) / full tab (15 mg). **`painScale: true` (v29)** — time modal shows a 1–10 "Pain level" dropdown (`Not recorded` is a valid choice); stored as `entry.painLevel`, shown in Today's Journal and History next to the dose. Tylenol has the same field (see above) |
| `lidocaine` | Lidocaine | 4h min gap, max 4 applications/day |
| `imodium` | Imodium | Daily limit 4 pills |
| `protonix` | Protonix | Windows 8 AM–noon & 8–10 PM, missed-dose tracked |
| `buspirone` / `paroxetine` / `iron` | — | Once daily, 10 PM window, missed-dose tracked |
| `senokot` | Senokot | As-needed, 8 AM & 10 PM windows |

### Vitals
- **Temperature** — °F. Input **defaults to `98.5`** (`tempDefault()`, v29) instead of a blank field; user can overwrite before logging.
- **Weight** — lbs. Input **defaults to the last recorded weight** (`weightDefault()`, v29); falls back to empty if none logged yet.

### Chemo cycle
- `nextChemoTs()` reads the most recently-set (`loggedAt`) `chemo_date` record; `ts: 0` means cleared.
- `chemoOffsetFor(dayTs)` = days between a given day and the chemo date (negative = before).
- `dexActiveOn(day)` = offset in `[-1, 1]`. `zofranBlockedOn(day)` = offset in `{0, 1}`.
- Today tab shows a phased red banner for offsets `[-2, 1]` and a "Chemo schedule" card to set/clear the date.

### Missed-dose alerts
`missedDosesFor(dayTs, now)` checks every med with `alerts: true` and `windows` (Dexamethasone,
Protonix, Buspirone, Paroxetine, Iron). A window counts as covered by any dose logged after the
previous window closed and before this one closes (so early logs count). Tracking starts
`MISSED_TRACK_SINCE` = Jul 12, 2026. Rendered as a non-dismissible red Today banner (today +
yesterday's misses), red rows in Today's Journal, and red rows + "N MISSED" summaries in History.

### Menstrual cycle (v29)
- `cycleEntries()` / `cycleActive()` — active whenever the most recent of `cycle_start`/`cycle_end` is a start.
- `daysSinceCycleStart()` — `dayStart(now) - dayStart(lastStart.ts)`, in days, +1 (start day = Day 1).
- `logCycleStart()` / `logCycleEnd()` — instant one-tap writes at `Date.now()`, **no time picker by design**.
- UI: a card at the top of the Weight tab with an "Active" badge and a single context-sensitive button ("Log Period Start" / "Log Period End").

### In-Patient day tracking (v29)
- `isInpatientToday()` — true if any `inpatient` entry's day matches today.
- `toggleInpatientToday()` — adds an `inpatient` entry at `Date.now()` if not marked, or deletes today's `inpatient` entry if already marked ("Undo"). Always uses real time, not a picker.
- `inpatientRanges()` — collects distinct in-patient calendar days, sorts, and merges any two days exactly 86,400,000 ms apart into a `{start, end}` range; returned most-recent-first.
- `fmtInpatientRange()` — single day → `M/D/YYYY`; multi-day → `M/D/YYYY – M/D/YYYY (N days)`.
- UI: a toggle banner at the very top of the Today tab (above the missed-dose banner), and a 4th
  "In-Patient" tab listing the ranges. `inpatient` entries are excluded entirely from Today's
  Journal and History's day rows/dose counts (they're not timestamped medical events).

## 7. Service Worker

**Cache name:** `caretracker-testing-v29` — bump this (using this repo's own version number, see
README) on every deploy to force devices to refresh.

**Cached shell:** `'./'`, `'index.html'`, `'manifest.webmanifest'`, icons.

**Fetch strategy:** network-first for `firestore.googleapis.com` / `gstatic.com` / `googleapis.com`,
cache-first for everything else.

`firebase-messaging-sw.js` exists (copied from prod) but is inert here — `subscribePush()` returns
immediately when `TEST_MODE` is true, so no token is ever registered and this service worker never
receives a background message in practice.

## 8. Deployment

1. Edit `index.html` (and `sw.js`, docs) locally.
2. Bump `CACHE` in `sw.js` to the new version number (see README's versioning convention).
3. **Ask Aaron before pushing** — this repo included. Once confirmed, push to `main`; GitHub Pages
   auto-deploys within ~1 minute.
4. Devices with the old service worker pick up the new version on next visit.

### Cache reset
Visit this app's `reset.html` — unregisters service workers, clears caches, redirects with a
cache-busting query string.

## 9. Known Issues & Gotchas

1. **Documentation drift is a known past failure mode for this repo** — an earlier version of this
   handoff described a dark theme, push notifications, and a `send-reminders.js`/GitHub Actions
   system that never existed here; it had been copy-pasted from production almost verbatim. If
   anything in this document looks inconsistent with the actual `index.html`, trust the code and
   fix the doc — don't propagate the mismatch further.
2. **No authentication** — anyone with the URL can read/write test data. Not a real risk since this
   collection isn't real medical data, but worth knowing.
3. **Shared Firebase project** (`fuelforge-7c132`) — isolated from prod only by collection name, not
   by a separate project. Don't touch project-level Firebase settings without checking prod impact.
4. **Zofran's chemo-day block is independent of its (now-removed) gap timer** — don't assume
   `gapH: 0` means Zofran is unrestricted; `status()` checks `zofranBlockedOn()` before the gap logic
   and still locks it (with override) on chemo days 0–1.
5. **`painLevel` is optional** — the time modal's dropdown defaults to "Not recorded"; don't assume
   every Morphine entry has one when reading history data.
6. **Single-file architecture** — same tradeoff as prod: simple, but one large file to edit carefully.

## 10. Version History

See README.md's **Testing Version History** table for the authoritative, dated list (this repo uses
`vN` numbers matching production's scheme, offset one ahead while testing leads — not an independent
counter).

## 11. Quick Reference for Common Tasks

### Add a new medication
1. Add an entry to the `MEDS` array in `index.html` with `id`, `name`, `sub`, dosing rules.
2. It appears in Quick Log automatically if `type === 'gap'` (and not `compazine`) or is explicitly
   included in the Quick Log filter, or in the Evening meds group if added to that id list.
3. If it needs missed-dose tracking, add `alerts: true` and a `windows` array.
4. No reminder system exists in this repo to wire up — that only applies to prod.

### Bump the version
1. Check the **actual pushed** `care-tracker` (prod) repo for its current live version.
2. This repo's new version = that number + 1 (or +1 again if testing is already ahead and adding
   more changes before prod catches up).
3. Update `sw.js`'s `CACHE` constant and both `README.md`/`CARETRACKER_HANDOFF.md`.

### Debug the live testing app
1. Open the testing URL in Chrome, DevTools → Console for errors.
2. DevTools → Application → Service Workers to check SW status (`caretracker-testing-vN`).
3. Confirm the orange "🧪 Testing app" banner is visible — if it's missing, `TEST_MODE` may have been
   accidentally flipped, which is a serious bug (would point this app at prod's collection).
4. Look for the "Live sync" indicator in the header for Firestore connectivity.

## 12. Keeping These Docs Updated

**When you make any change to this repo, update `CLAUDE.md`, `README.md`, and this file in the same
pass.** Specifically here: bump "Last updated" / "Current version" above, add a row to Section 10's
pointer target (README's table), and revise any of Sections 4–9 that the change affects. Stale docs
in this exact repo have already caused real confusion once (see Known Issues #1) — don't let it
happen again.
