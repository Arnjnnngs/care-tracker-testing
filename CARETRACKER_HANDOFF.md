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
> **Last updated:** July 17, 2026
> **Current version:** v34 (see README.md's versioning convention — this repo's version is always
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
  glassmorphism** (`#FFF0F3` background, `#AA5375`/`#9B5B8A` accents) with compact, high-density Quick
  Log medication cards — medication/generic names share the header line, status remains right-aligned,
  dose/availability metadata shares one line, and dose buttons are inline pills. Do not describe this
  as dark; that's production's old theme, not this one.
- **Backend:** Firebase Firestore, project `fuelforge-7c132` (shared with prod; isolated by
  collection name, see below). No auth.
- **Hosting:** GitHub Pages.

## 5. Firebase Collections

### `caretracker_test_entries` (this app's collection — `COL_NAME` when `TEST_MODE` is true)
Each document is one logged event. Fields actually written by this app's code:

- `medId` — `"dexamethasone" | "tylenol" | "zofran" | "compazine" | "morphine" | "lidocaine" | "imodium" | "protonix" | "buspirone" | "paroxetine" | "iron" | "senokot" | "temp" | "weight" | "chemo_date" | "cycle_start" | "cycle_end" | "inpatient_start" | "inpatient_end" | "inpatient"` (`inpatient` is the pre-v32 single-day marker, kept only for backward-compat reads of old data — new writes always use `inpatient_start`/`inpatient_end`)
- `ts` — ms-since-epoch of the event
- `dose` — human-readable label (e.g. `"1000 mg"`, `"½ tab · 7.5 mg"`) or `null`
- `mg` — numeric mg (0 for non-mg entries)
- `pills` — pill/application count, only present when relevant
- `temp` / `weight` — numeric value on vitals entries
- `override` — boolean, present when logged early past a lock
- `painLevel` — integer 1–10, present on Morphine and Tylenol doses (new in v29). **As of v30 this is required** — `confirmTimeAndLog()` rejects the log with a toast (`Select a pain level before logging <med>`) if `m.painLevel` is unset for a `painScale` med, so any entry for these two meds is guaranteed to carry a level.
- `loggedAt` — present on `chemo_date` records; used to find the most recently *set* chemo date (the
  record's own `ts` is the chemo date itself, which can be in the future)

`cycle_start` / `cycle_end` and `inpatient_start` / `inpatient_end` records carry no special fields beyond the common ones —
they're pure event markers, always logged at `Date.now()` (no time picker).

### `caretracker_entries`
Production's real collection. **This app must never write here.**

## 6. Medication & Feature Definitions

| ID | Display Name | Rules |
|---|---|---|
| `dexamethasone` | Dexamethasone | `chemoOnly: true` — only appears in Quick Log when `dexActiveOn(day)` (day −1 to +1 relative to chemo date). 2 tablets, 8 AM & 2 PM windows. Missed-dose tracked. |
| `tylenol` | Tylenol | Daily max 2500 mg, 4h min gap, 500/1000 mg doses. **`painScale: true` (v29), required as of v30** — same 1–10 "Pain level" dropdown as Morphine; Confirm is blocked until a level is chosen |
| `zofran` | Zofran | **As-needed (v29): `gapH: 0`, no lock, no reminder.** Still blocked on chemo days 0–1 via `zofranBlockedOn()` — that's a clinical rule, independent of the gap timer, and was not removed |
| `compazine` | Compazine | 6h min gap, shown in Evening meds card |
| `morphine` | Morphine | 4h min gap, ½ tab (7.5 mg) / full tab (15 mg). **`painScale: true` (v29), required as of v30** — time modal shows a 1–10 "Pain level" dropdown; `Not recorded` (blank) is no longer a valid choice for Confirm, `confirmTimeAndLog()` rejects the submission otherwise. Stored as `entry.painLevel`, shown in Today's Journal and History next to the dose. Tylenol has the same field/requirement (see above) |
| `lidocaine` | Lidocaine | 4h min gap, max 4 applications/day |
| `imodium` | Imodium | Daily limit 4 pills |
| `protonix` | Protonix | Windows 8 AM–noon & 8–10 PM, missed-dose tracked |
| `buspirone` / `paroxetine` / `iron` | — | Once daily, 10 PM window, missed-dose tracked |
| `senokot` | Senokot | As-needed, 8 AM & 10 PM windows |

### Vitals
- **Temperature** — °F. Input shows `tempDefault()` (`98.5`) as an HTML `placeholder` (grayed hint
  text only) — **the actual `value` is always `state.tempInput`, never auto-filled** (reverted in
  v30 after v29 briefly made it a real auto-submitted default, which risked silently logging a
  value the user never entered). `logTemp()` validates only the raw typed input; an empty field is
  rejected with a toast, not defaulted.
- **Weight** — lbs. Input shows `weightDefault()` (the last recorded weight, or `'156.0'` if none
  logged yet) as a `placeholder` only — same rule as Temperature: `value` is always
  `state.weightInput`, `logWeight()` rejects an empty field rather than silently using the last
  weight (fixed in v30; this was the same underlying bug as Temperature's, just not initially
  reported for Weight).

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
**As of v30, `missedDosesFor(dayTs, now)` returns `[]` immediately if `isInpatientDay(dayTs)` is
true** — a day marked In-Patient is fully excluded from missed-dose detection everywhere (Today
banner, Journal, History), since meds given by hospital staff that day aren't tracked by this app.
This is per-day: marking *today* in-patient doesn't suppress a genuine miss from *yesterday* still
showing in the Today banner's rollover section, and vice versa.

### Menstrual cycle (v29; Today banner added v30; own tab v32)
- `cycleEntries()` / `cycleActive()` — active whenever the most recent of `cycle_start`/`cycle_end` is a start.
- `daysSinceCycleStart()` — `dayStart(now) - dayStart(lastStart.ts)`, in days, +1 (start day = Day 1).
- `logCycleStart()` / `logCycleEnd()` — instant one-tap writes at `Date.now()`, **no time picker by design**.
- `cyclePeriods()` — pairs `cycle_start`/`cycle_end` chronologically into `{start, end}` periods
  (`end: null` if still open); `fmtCyclePeriod()` formats each as `M/D/YYYY – Active` or
  `M/D/YYYY – M/D/YYYY (N days)`.
- UI (**Cycle** tab, v32 — moved off the Weight tab, sits between Weight and In-Patient in the tab
  bar): a card at the top with an "Active" badge and a single context-sensitive button ("Log Period
  Start" / "Log Period End"), plus a Cycle History list below it built from `cyclePeriods()`. The
  underlying `cycle_start`/`cycle_end` event logic was **not touched** in the v32 move — only where
  it renders changed.
- UI (Today tab, v30): whenever `cycleActive()` is true, a **non-dismissible** "Period Active"
  banner renders near the top of the Today (home) screen — no close/dismiss control exists on it by
  design, it only goes away once `logCycleEnd()` is called (either from this banner's own "Log
  Period End" button, or from the Cycle tab's card — both write the same `cycle_end` event).

### In-Patient tracking (v29; missed-dose suppression v30; Start/End/Undo redesign v32)
As of v32 this is a **paired start/end event model**, mirroring the menstrual cycle, replacing the
original single daily toggle marker:
- `inpatientEntries()` / `inpatientPeriods()` — pairs `inpatient_start`/`inpatient_end` chronologically
  into `{start, end, startId, endId}` periods (`end: null` + `endId: null` if still open), most-recent-first.
- `currentInpatientPeriod()` / `isInpatientActiveNow()` — the open period (if any) / whether one exists.
- `daysSinceInpatientStart()` — `dayStart(now) - dayStart(open.start)`, in days, +1 (start day = Day 1).
- `logInpatientStart()` / `logInpatientEnd()` — instant one-tap writes at `simNow()`, each fires a
  confirmation toast ("In-Patient started — meds now show as Restricted" / "In-Patient ended").
- `undoInpatientStart()` — **two-tap confirm.** First tap sets `state.confirmUndoInpatient = true`,
  shows a "Tap Undo again to remove this In-Patient entry" toast, and starts a 5-second timer that
  resets the flag if not confirmed. Second tap (while the flag is set) deletes the open period's
  `inpatient_start` doc via `removeEntryDB(cur.startId)` and shows a removal-confirmed toast. This is
  specifically for undoing an accidental Start — closing a real stay should use End, not Undo.
- `isInpatientDay(ts)` — true if the day containing `ts` overlaps any period in `inpatientPeriods()`
  (open periods count as covering every day from their start through "now"), **or** if a legacy
  pre-v32 `inpatient` single-day marker exists on that day (kept for backward-compat with old test
  data, never written going forward). Feeds `missedDosesFor()`'s suppression check unchanged from v30.
- `fmtInpatientDateTime()` — `M/D/YYYY h:mm AM/PM`. `fmtInpatientPeriod()` — single period →
  `<start> – Active` or `<start> – <end> (N days)`, using real timestamps so a same-day stay of a
  few hours still shows correctly (half-day precision) instead of collapsing into a single date.
- UI (**In-Patient** tab): a Start/End/Undo card at the top (Log In-Patient Start when inactive; Log
  In-Patient End + Undo side-by-side when active, Undo relabeling to "Tap to confirm" mid-flow) plus
  an In-Patient History list below it built from `inpatientPeriods()`.
- UI (Today tab, v32): whenever `isInpatientActiveNow()` is true, a **non-dismissible, pinned**
  "In-Patient Active" banner renders at the top of Today with its own Log In-Patient End button — no
  close/dismiss control by design, same pattern as the Period Active banner.
- UI (Quick Log + Evening meds, v32): while `isInpatientActiveNow()` is true, every med card in both
  sections renders as `<Med Name> - In-Patient (Restricted)` with a "Given by hospital staff — not
  logged in this app" note, replacing all normal logging controls; the Evening meds "Take all" button
  is also hidden. This only affects display/interaction — no entries are written for restricted meds.

### Testing-only date override (v31)
This is the mechanism that makes the whole "wait for a real day to pass" problem go away during
manual testing:

- `simNow()` — the single source of truth for "what time is it right now" everywhere in the app.
  Returns `Date.now() + state.dateOffsetDays * 86400000` when `TEST_MODE` is true, and returns
  plain `Date.now()` unconditionally when `TEST_MODE` is false. **Every other function that used to
  call `Date.now()` for an actual event timestamp or "now" comparison was switched to `simNow()`**:
  `nowLocalISO()` (time-modal default), the future-timestamp check in `confirmTimeAndLog()`,
  `seedDemo()`, `logCycleStart()`/`logCycleEnd()`, `logInpatientStart()`/`logInpatientEnd()`
  (v32 — the old `toggleInpatientToday()` this replaced also used it), and the 1-second `setInterval`
  that drives `state.now`. `chemo_date`'s `loggedAt` audit field intentionally still uses real
  `Date.now()` (see below).
- `setSimDate(dateStr)` — takes a `YYYY-MM-DD` string from the header's date input, computes the day
  offset from the real calendar date, and stores it in `state.dateOffsetDays`.
- `shiftSimDate(days)` — adds/subtracts whole days from the current offset (powers the header's
  "− 1 Day" / "+ 1 Day" buttons).
- `resetSimDate()` — zeroes the offset, snapping back to real time.
- **Ordering gotcha:** all three setter functions mutate `state.dateOffsetDays` directly *before*
  calling `setState({ now: simNow() })`, rather than putting `dateOffsetDays` and `now: simNow()` in
  the same `setState()` patch object. If you refactor this, keep that ordering — `simNow()` reads
  `state.dateOffsetDays` at the moment it's called, and object-literal properties in a single
  `setState()` call are evaluated before `Object.assign` applies any of them, so putting both in one
  patch call would make `now` reflect the *previous* offset, one step behind. This was a real bug
  caught by the QA harness during development.
- **Why `loggedAt` on `chemo_date` records stays on real `Date.now()`:** that field is pure
  creation-order bookkeeping (`nextChemoTs()` picks the record with the highest `loggedAt`), not a
  simulated event time. If it used `simNow()` instead, jumping the simulated date backward after
  previously setting a chemo date at a later simulated date could make an older record look newer,
  breaking that ordering. The chemo date itself (`ts`, picked from the date input) is unaffected
  either way — it was never derived from "now".
- UI: a dashed-border amber panel at the top of the header, below the orange "🧪 Testing app" banner,
  visible on every tab (not just Today) since the picker affects Weight/History/In-Patient views
  too. Only rendered when `TEST_MODE` is true. Shows a "+N days from real today" label whenever the
  offset isn't zero.
- **Production safety:** confirmed via a dedicated QA pass that re-runs the whole app with
  `TEST_MODE` forced to `false` — the header control doesn't render, and `shiftSimDate()`/
  `setSimDate()` are no-ops (`simNow()` stays equal to real `Date.now()`) even if called directly.
## 7. Service Worker

**Cache name:** `caretracker-testing-v34` — bump this (using this repo's own version number, see
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

### v34 — July 17, 2026

**Compact medication-card redesign.** The Today tab's Quick Log cards were restyled for much higher
phone-screen density only: padding, radii, shadows, and inter-card gaps were reduced; medication and
generic names now share a compact header row with the status badge aligned right; last-dose and
availability information share one metadata row; and dose controls are compact inline pills. The
light-pink glassmorphism palette, Hanken Grotesk/IBM Plex Mono fonts, `#AA5375` accent, Firebase
routing, and all medication logic remain unchanged.

## Known Issues (Fixed)

### seedDemo() fake-data bug (fixed v33, Jul 17, 2026)

A dormant `seedDemo()` function fired whenever the app's first Firestore snapshot returned empty entries — intended only for a genuinely fresh install, but cold caches and brief network blips produce the same "empty" signal, so it could fire unpredictably during real usage. It silently wrote 11 hardcoded fake medication entries per trigger. The identical function existed in production `care-tracker` (introduced early in that repo's history, before this testing repo existed) and had already written fake entries into Brandi's real medical data before it was caught — see production's Known Issues section for the full incident. This repo's copy never reached `caretracker_entries` (confirmed via a `TEST_MODE`/`COL_NAME` routing audit) but did pollute `caretracker_test_entries` with the same pattern.

**Fix:** the trigger condition was changed to `if (false && wasEmpty && entries.length === 0)`, permanently disabling the call without deleting the dead function body (left as unreachable code, flagged for an optional future cleanup pass). All 11 fake entries were identified by matching their timestamps against `seedDemo()`'s known hour-offsets from a single trigger time, deleted via the Firebase Console's admin Data browser, and the collection was re-queried to confirm zero remain.

**Lesson for future fixture/demo-data functions:** never gate a write on "the collection looks empty" as a proxy for "this is the user's first real launch." A cold local cache or a dropped network request looks identical to genuine emptiness from the client's point of view. If a demo-seeding feature is ever wanted again, it should require an explicit user action (a button), not fire automatically off a snapshot listener.

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
