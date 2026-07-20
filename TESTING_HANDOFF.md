# CareTracker TESTING — AI Agent Handoff Document

> **THIS IS THE STAGING VARIANT.** `TEST_MODE = true` in `index.html` — writes to
> `caretracker_test_entries` (never `caretracker_entries`), push/local notifications fully disabled,
> orange "🧪 Testing app" banner in the header. sw.js cache is `caretracker-testing-vN`. Features
> under test here that are NOT (yet) in production: chemo cycle system, missed-dose alerts,
> menstrual cycle tracking, In-Patient day tracking, Morphine pain-level scale, Zofran as-needed,
> Bowel Movement/Symptoms tracking, persistent Firestore-backed missed-dose Clear, Tylenol Liquid
> (shared-ceiling + own volume cap), Appetite tracking.
> Promote to prod by porting the relevant changes into `care-tracker`'s `index.html` with
> `TEST_MODE = false` and a prod cache bump — **only when Aaron explicitly says to.**

> **Purpose:** Complete context for any AI assistant to understand, maintain, and extend this repo
> without prior knowledge. See `TESTING_CLAUDE.md` first for the non-negotiable rules.
>
> **Last updated:** July 20, 2026
> **Current version:** v55 (testing) (see TESTING_README.md's versioning convention — this repo's version is always
> "current live prod version + 1" while testing is ahead)
>
> **Known documentation gap:** versions v38–v49 (Jul 18–19, 2026) were built, QA'd, and pushed but
> never got Version History rows in TESTING_README.md at the time. See TESTING_README's table for a placeholder note
> and Section 9 below — flagged to Aaron on Jul 19, 2026 for a decision on backfilling from commit
> history as a separate pass. v50–v55 are fully documented.
>
> **Never edit `index.html` or `sw.js` directly through GitHub's web editor.** On the night of Jul
> 19–20, 2026, a direct web-editor commit (titled "v53: morphine half-dose window tracking, bump SW
> cache") replaced the entire contents of `index.html`, `sw.js`, and this file with the literal text
> `undefined` — the intended feature (a Morphine half-dose window) never actually landed; the commit
> only destroyed the three files. Almost certainly a paste gone wrong: GitHub's "Edit file" box
> replaces a file's whole contents with whatever's in the box, with no diff shown before commit, so a
> broken/empty clipboard paste silently becomes the entire file. The result: the live app served a
> 9-byte page, which read to Aaron as "an undefined page" and did **not** clear with a client-side
> cache reset, because the server itself — not the client cache — was serving the broken file. See
> the v53 Version History entry below for the full incident and fix.
> **Always make changes locally (or hand them to an AI agent working from the actual file contents)
> and push a real diff — never paste replacement content into GitHub's inline editor.**

---

## 1. What This Project Is

CareTracker is a **progressive web app (PWA)** that tracks medications, vitals, menstrual cycle,
hospital in-patient days, bowel movements, and other symptoms for a family caregiver (caring for
Brandi). It's a **single-file vanilla JavaScript app** — no build step, no framework. The entire app
lives in `index.html`. Firebase Firestore provides the database with real-time sync. This build has
push notifications disabled.

**Core user flow:** a caregiver opens the app, taps a quick-log button (e.g. "500 mg" Tylenol), and
the dose instantly syncs to Firestore across devices. The app enforces dosing limits/gaps where
relevant, tracks a chemo cycle and flags missed doses, and now also tracks menstrual cycle, in-patient
hospital days, and bowel movement/symptom episodes.

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
├── TESTING_CLAUDE.md           # Standing rules — read first
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
  Firestore, since jsdom does not reliably execute module scripts (this repo's QA harnesses now use a
  hand-rolled fake DOM instead of jsdom, which proved unreliable in the sandbox).
- **Rendering:** Custom `h()` hyperscript-style DOM builder + full `render()` on every `setState()`.
- **Styling:** Inline `<style>` + inline `style` objects on every element. **Theme is light pink
glassmorphism** (`#FFF0F3` background, `#AA5375`/`#9B5B8A` accents) with compact, high-density Quick
Log medication cards — medication/generic names share the header line, status remains right-aligned,
dose/availability metadata shares one line, and dose buttons are inline pills. A fixed native-style
bottom navigation (Home, Meds, Reports, In-Patient) is present on every primary route and reserves
safe-area space at the bottom of phone screens. Do not describe this as dark; that's production's old
theme, not this one.
- **Backend:** Firebase Firestore, project `fuelforge-7c132` (shared with prod; isolated by
  collection name, see below). No auth.
- **Hosting:** GitHub Pages.

## 5. Firebase Collections

### `caretracker_test_entries` (this app's collection — `COL_NAME` when `TEST_MODE` is true)
Each document is one logged event. Fields actually written by this app's code:

- `medId` — `"dexamethasone" | "tylenol" | "tylenol-liquid" | "zofran" | "compazine" | "morphine" | "lidocaine" | "imodium" | "protonix" | "buspirone" | "paroxetine" | "iron" | "senokot" | "appetite" | "temp" | "weight" | "chemo_date" | "cycle_start" | "cycle_end" | "inpatient_start" | "inpatient_end" | "inpatient"` (`inpatient` is the pre-v32 single-day marker, kept only for backward-compat reads of old data — new writes always use `inpatient_start`/`inpatient_end`)
- `ts` — ms-since-epoch of the event
- `dose` — human-readable label (e.g. `"1000 mg"`, `"½ tab · 7.5 mg"`, `"30 mL (1000 mg)"`, `"Little to none"`) or `null`
- `mg` — numeric mg (0 for non-mg entries)
- `pills` — pill/application count, only present when relevant
- `volumeMl` — numeric mL, present only on Tylenol Liquid doses (new in v54)
- `value` — raw Appetite selection (`"normal" | "little" | "none"`), present only on `appetite` entries (new in v54); `dose` carries the human-readable label for the same field
- `note` — optional free-text reason, present only on `appetite` entries when the user typed one (new in v54)
- `temp` / `weight` — numeric value on vitals entries
- `override` — boolean, present when logged early past a lock
- `painLevel` — integer 1–10, present on Morphine and Tylenol doses (new in v29), and Tylenol Liquid doses (v54). **As of v30 this is required for Morphine/Tylenol** — `confirmTimeAndLog()` rejects the log with a toast (`Select a pain level before logging <med>`) if `m.painLevel` is unset for a `painScale` med, so any entry for these meds is guaranteed to carry a level.
- `loggedAt` — present on `chemo_date` records; used to find the most recently *set* chemo date (the
  record's own `ts` is the chemo date itself, which can be in the future)

`cycle_start` / `cycle_end` and `inpatient_start` / `inpatient_end` records carry no special fields beyond the common ones —
they're pure event markers, always logged at `Date.now()` (no time picker). `appetite` records are one-per-day
(keyed by day-start), logged at noon on the target day (`dayStart + 12h`) so they sort predictably in History
regardless of what time of day the caregiver actually filled the dropdown in.

### `caretracker_entries`
Production's real collection. **This app must never write here.**

### `caretracker_test_prefs` (added v50)
Single document (`settings`) holding small, non-medical UI preferences — currently just
`missedClearedAt` (ms-since-epoch of the last time the missed-dose banner's Clear button was
pressed). Written via `clearMissedDoses()` / read via `subscribePrefs()`, both synced through
`onSnapshot` like everything else so the cleared state survives reloads and syncs across devices.
**Kept separate from production's `caretracker_prefs`** via `PREFS_COL_NAME = TEST_MODE ?
'caretracker_test_prefs' : 'caretracker_prefs'` — this app must never write to `caretracker_prefs`.

### `caretracker_prefs`
Production's preferences collection. **This app must never write here.**

## 6. Medication & Feature Definitions

| ID | Display Name | Rules |
|---|---|---|
| `dexamethasone` | Dexamethasone | `chemoOnly: true` — only appears in Quick Log when `dexActiveOn(day)` (day −1 to +1 relative to chemo date). 2 tablets, 8 AM & 2 PM windows. Missed-dose tracked. |
| `tylenol` | Tylenol | Daily max 2500 mg, 4h min gap, 500/1000 mg doses. **`painScale: true` (v29), required as of v30** — same 1–10 "Pain level" dropdown as Morphine; Confirm is blocked until a level is chosen. **`ceilingGroup: 'tylenol'` (v54)** — shares its 2500 mg daily ceiling with `tylenol-liquid` (see below) |
| `tylenol-liquid` | Tylenol Liquid | Added v54. Oral suspension form, tracked as a fully separate medication (not folded into the `tylenol` card) so History/Reports never have to disambiguate "which Tylenol." 30 mL (1000 mg) per dose, its own independent 6h gap (does not interact with pill Tylenol's 4h gap), `painScale: true` + required (same as pill Tylenol). `ceilingGroup: 'tylenol'` — its mg total is pooled with pill Tylenol against the same 2500 mg/24h ceiling. `volumeCeilingMl: 90` + `volumePerDoseMl: 30` — a second, independent ceiling on its own 90 mL/24h, tracked only from this medication's own entries (pill Tylenol doses never count toward the volume total, and Liquid doses never count toward anything but the shared mg ceiling and its own volume ceiling) |
| `zofran` | Zofran | **As-needed (v29): `gapH: 0`, no lock, no reminder.** Still blocked on chemo days 0–1 via `zofranBlockedOn()` — that's a clinical rule, independent of the gap timer, and was not removed |
| `compazine` | Compazine | 6h min gap, shown in Evening meds card |
| `morphine` | Morphine | 4h min gap, ½ tab (7.5 mg) / full tab (15 mg). **`painScale: true` (v29), required as of v30** — time modal shows a 1–10 "Pain level" dropdown; `Not recorded` (blank) is no longer a valid choice for Confirm, `confirmTimeAndLog()` rejects the submission otherwise. Stored as `entry.painLevel`, shown in Today's Journal and History next to the dose. Tylenol has the same field/requirement (see above) |
| `lidocaine` | Lidocaine | 4h min gap, max 4 applications/day |
| `imodium` | Imodium | Daily limit 4 pills |
| `protonix` | Protonix | Windows 8 AM–noon & 8–10 PM, missed-dose tracked |
| `buspirone` / `paroxetine` / `iron` | — | Once daily, 10 PM window, missed-dose tracked |
| `senokot` | Senokot | As-needed, 8 AM & 10 PM windows |

### Medication Management (v35)
- **Active medication configuration** lives in `state.meds`; the in-code `DEFAULT_MEDS` array remains
  the baseline used on a device with no saved configuration.
- `loadMedicationConfig()` and `persistMedicationConfig()` use browser-local storage key
  `caretracker_testing_med_config_v1`. This is intentionally a UI/configuration feature, not a new
  Firestore collection or an alteration to the existing entry schema.
- `archivedMeds` preserves a name/generic label locally when a configured medication is deleted, so
  historical entry rendering remains intelligible. Delete is always confirmation-protected.
- Configuration can differ by device/browser profile. Do not describe this as real-time shared
  medication configuration; only medication entries continue to sync through Firestore.

### Tylenol Liquid + shared-ceiling mechanism (v54)
- `ceilingGroupMedIds(med)` — returns every medication in `state.meds` sharing `med.ceilingGroup`
  (currently just `['tylenol', 'tylenol-liquid']`), or `[med.id]` if the med has no group.
- `dailyGroupMg(med)` — sums `dailyDoseMg(id)` across every id in that group, giving a combined
  24-hour mg total across both pill and liquid forms.
- `dailyCeiling(med)` — its mg-branch now uses `dailyGroupMg(med)` when `med.ceilingGroup` is set,
  `dailyDoseMg(med.id)` otherwise, so the existing ceiling-badge UI (shared by every medication, not
  Tylenol-specific) automatically reflects the combined total for both forms without any bespoke UI.
- `dailyVolumeMl(medId)` / `dailyVolumeCeiling(med)` — a wholly separate mechanism from the mg
  ceiling above, scoped to a single `medId` (no grouping): sums `entry.volumeMl` across that
  medication's own entries since midnight and compares against `med.volumeCeilingMl`. Pill Tylenol
  has no `volumeCeilingMl` and never accumulates a volume total; Tylenol Liquid's volume total is
  never affected by pill Tylenol entries.
- `status(med)` checks **both** ceilings independently for a `volumeCeilingMl` medication: the
  existing mg-ceiling check (now group-aware) runs first, then a second check locks the medication
  if `dailyVolumeMl(med.id) >= med.volumeCeilingMl`. Either one alone is sufficient to lock — it's
  possible to be locked on the 90 mL cap while still under 2500 mg combined, or vice versa (e.g. a
  large pill-Tylenol dose could hit the shared mg ceiling well before 90 mL of liquid is ever given).
- `afterLog()`'s post-log ceiling-warning toast was restructured to check both `tylenol` and
  `tylenol-liquid` medIds together: it computes the combined mg total and (for a Liquid dose) the
  volume total, and shows a combined/mg-only/volume-only warning depending on which limit(s) the
  dose that was just logged actually crossed.
- **Naming gotcha:** the medication id is `tylenol-liquid` (hyphen), not `tylenol_liquid`
  (underscore) — `safeMedicationId()` (used by `normalizeMedication()` on every medication, including
  `DEFAULT_MEDS` entries) silently converts any non-`[a-z0-9]` character, including underscores, to a
  hyphen. An id containing an underscore therefore never matches its own literal string in any
  hardcoded comparison — this was caught by the QA harness (`state.meds.find` returned `undefined`
  for the underscore spelling) before it shipped; every reference in `index.html` uses the hyphenated
  form consistently. If adding another custom-id medication in the future, either avoid underscores
  in the chosen id or always resolve the id via `state.meds.find(...).id` rather than a hardcoded string.

### Appetite tracking (added v54, moved to a Home alert v55)
- Retrospective one-entry-per-day pattern, same as Bowel Movement: `appetiteEntriesByDay()` builds a
  `Map` keyed by `dayStart(ts)` (last-write-wins per day), `appetiteFor(dayStartTs)` looks up a single
  day, `appetiteHistorySorted()` returns every day's entry most-recent-first.
- `logAppetite(value, dayStartTs, note)` writes one entry: `medId: 'appetite'`, `value` (raw
  `'normal' | 'little' | 'none'`), `dose` (human label via `APPETITE_LABELS`), `mg: 0`, `ts: dayStartTs
  + 12h` (midday, so the entry sorts predictably regardless of what time it was actually logged), and
  an optional `note` field only when the caregiver typed one (empty/whitespace-only notes are trimmed
  and omitted, not stored as an empty string).
- `submitAppetite(dayStartTs)` — the Log handler: rejects with a toast if no dropdown value is
  selected, otherwise deletes that day's existing entry (if any, relevant when re-answering after a
  Remove) and adds the new one (overwrite, not append — same delete-then-add pattern as Bowel
  Movement), wrapped in try/catch per the silent-failure lesson from the v51 Bowel Movement fix so a
  save failure always surfaces a toast rather than failing invisibly.
- **v55 UI change:** the input (select + optional notes textarea + button) moved off Reports entirely
  and now lives in `renderToday()` as a Home alert — see "Daily 'must be answered' Home alerts" below.
  `renderAppetite()` (**Reports → Appetite**) is now history-only: just the Appetite History list
  built from `appetiteHistorySorted()`, each row showing the date, the value label, the optional note,
  a color-coded status dot (green `#0F9D6B` normal / amber `#C77800` little / red `#C0453B` none), and
  a remove control. Removing an entry there makes the Home alert reappear for that day, since
  `appetiteFor(yesterdayStart(now))` becomes `null` again.
- `appetite` was added to `BYPASS_48H_IDS` (alongside `weight`, `cycle_start`, `cycle_end`,
  `bowel_movement`) so a day's answer stays editable/removable past the normal 48-hour permanent-
  history lock, matching the other retrospective daily-entry features.
- `reportDescriptor('appetite', now)` supplies the Reports-menu card's label/icon/meta (meta text
  pulls from `appetiteFor(yesterdayStart(now))` so the menu card previews yesterday's answer without
  opening the detail view).

### Daily "must be answered" Home alerts (v55) — Weight, Bowel Movement, Appetite
Weight already had a persistent, escalating Home alert (added earlier as "Batch G"); v55 gives Bowel
Movement and Appetite the identical treatment, per Aaron's explicit request that they "appear like the
Weight card where it is persistent until answered."
- `dailyAlertLevel(now)` — shared escalation-tier function (near `yesterdayStart`): returns `0`
  (quiet) before noon, `1` (firm) from noon–6 PM, `2` (urgent) from 6 PM on, based on `new
  Date(now).getHours()`. `dailyAlertStyle(level)` returns the matching `{ color, bg, border, shadow }`
  — lavender/amber/red, same values Daily Weight always used. All three alerts call these same two
  functions, so "the same rules as Weight" is enforced structurally, not just visually similar.
- Each alert only renders while its target day is unanswered, and disappears the instant it's logged
  — Weight checks `state.entries.some(e => e.medId === 'weight' && e.ts >= dayStart(now))` (asks about
  **today**, since Weight can be logged multiple times/day); Bowel Movement checks
  `!bowelMovementFor(yesterdayStart(now))` and Appetite checks `!appetiteFor(yesterdayStart(now))`
  (both ask about **yesterday**, being retrospective one-answer-per-day fields).
- Unlike Weight, Bowel Movement and Appetite have **no separate always-present quiet input card** —
  once answered, the only way to change that day's answer is via the matching Reports history list
  (Remove, then the alert reappears since the day is unanswered again). This is an intentional
  tradeoff: Weight supports multiple readings/day so its quiet card stays useful even after logging;
  Bowel Movement/Appetite are single-answer-per-day, so there's nothing left to log once answered.
- Render order on Home: Missed Doses → Chemo plan → Period Active → Bowel Issue Active → **Bowel
  Movement alert** → **Appetite alert** → Daily Weight alert → Demo/Warning banners → vitals/meds.
- **Bowel Issue Active banner logic change (v55):** this banner used to omit its own inline control
  and show a pointer message ("Update using the Bowel Movement card below") whenever its target day
  equaled yesterday, because the old retrospective card was always visible below it and duplicating a
  second identical control would've been confusing. Now that the retrospective control only shows
  while yesterday is unanswered, the "same day" case can only happen when yesterday **is** answered
  (that's what makes `bowelIssueActive()` true for that day in the first place) — meaning the card it
  used to point at is always hidden in that exact scenario. The pointer-message branch was removed
  entirely; the banner now **always** renders its own inline control, labeled with whichever day it
  targets (`bannerDayLabel`). See `test_bowel_dedup.js`, rewritten for this behavior.

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
- Home shows a phased red banner for offsets `[-2, 1]` and a "Chemo schedule" card to set/clear the date.

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

**Clear button (v50, persistent).** The Today/Home missed-dose banner has a Clear button
(`clearMissedDoses()`) that writes `{ missedClearedAt: state.now }` to the `caretracker_test_prefs`
Firestore document (merge:true), dismissing every currently-listed miss without deleting the
underlying entries. `renderToday()`'s `bannerItems` filters `bannerItemsAll` to only `m.ts >
(state.missedClearedAt || 0)`, so a new miss occurring after the clear timestamp reappears normally.
`missedClearedAt` is loaded on init via `subscribePrefs()` (an `onSnapshot` listener on the prefs
doc, mirroring `subscribeEntries()`), so the cleared state persists across reloads and syncs across
devices — this replaces a pre-v50 version that only set an in-memory `state.testMissedClearedAt`
field, which reset to 0 on every reload and therefore didn't actually solve the "stale entries
reappear on refresh" problem. The button itself is **not** `TEST_MODE`-gated (it's real
functionality, not testing scaffolding) — only the collection name it writes to differs from
production, via `PREFS_COL_NAME = TEST_MODE ? 'caretracker_test_prefs' : 'caretracker_prefs'`,
mirroring the existing `COL_NAME` pattern. This makes testing's and production's Clear-button code
paths structurally identical, so promoting this feature to prod is a no-op copy, not a rewrite.

### Menstrual cycle (v29; Today banner added v30; own tab v32)
- `cycleEntries()` / `cycleActive()` — active whenever the most recent of `cycle_start`/`cycle_end` is a start.
- `daysSinceCycleStart()` — `dayStart(now) - dayStart(lastStart.ts)`, in days, +1 (start day = Day 1).
- `logCycleStart()` / `logCycleEnd()` — instant one-tap writes at `Date.now()`, **no time picker by design**.
- `cyclePeriods()` — pairs `cycle_start`/`cycle_end` chronologically into `{start, end}` periods
  (`end: null` if still open); `fmtCyclePeriod()` formats each as `M/D/YYYY – Active` or
  `M/D/YYYY – M/D/YYYY (N days)`.
- UI (**Reports → Cycle**, v35 — previously a standalone top tab): a card at the top with an "Active"
  badge and a single context-sensitive button ("Log Period Start" / "Log Period End"), plus a Cycle
  History list below it built from `cyclePeriods()`. The underlying `cycle_start`/`cycle_end` event
  logic was **not touched** in the v35 routing change — only where it renders changed.
- UI (Home, v30): whenever `cycleActive()` is true, a **non-dismissible** "Period Active" banner
  renders near the top of Home — no close/dismiss control exists on it by design, it only goes away
  once `logCycleEnd()` is called (either from this banner's own "Log Period End" button, or from the
  Reports → Cycle card — both write the same `cycle_end` event).

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
- UI (persistent **In-Patient** bottom-nav route): a Start/End/Undo card at the top (Log In-Patient
  Start when inactive; Log In-Patient End + Undo side-by-side when active, Undo relabeling to "Tap to
  confirm" mid-flow) plus an In-Patient History list below it built from `inpatientPeriods()`.
- UI (Home, v32): whenever `isInpatientActiveNow()` is true, a **non-dismissible, pinned**
  "In-Patient Active" banner renders at the top of Home with its own Log In-Patient End button — no
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
  visible on every primary route (not just Home) since the picker affects Reports and In-Patient views
  too. Only rendered when `TEST_MODE` is true. Shows a "+N days from real today" label whenever the
  offset isn't zero.
- **Production safety:** confirmed via a dedicated QA pass that re-runs the whole app with
  `TEST_MODE` forced to `false` — the header control doesn't render, and `shiftSimDate()`/
  `setSimDate()` are no-ops (`simNow()` stays equal to real `Date.now()`) even if called directly.

### Bowel Movement & Symptoms tracking (v48–v49; retrospective card became a Home alert v55)
Added a **Bowel Movement** retrospective daily check-in and a **Symptoms** bottom-nav tab for logging
non-medication health events, plus a pinned **Bowel Issue Active** banner on Home (v49) mirroring the
pattern already established by the Period Active and In-Patient Active banners. This section is
intentionally brief pending the v38–v49 documentation backfill noted at the top of this file and in
Section 9 — see the actual `index.html` for the current implementation details until that pass is
done. **The retrospective check-in itself moved from an always-visible quiet card (under Weight) to a
persistent, escalating Home alert in v55** — see "Daily 'must be answered' Home alerts (v55)" above
for the current mechanism (`dailyAlertLevel`/`dailyAlertStyle`, hides once yesterday is answered).

**Two distinct Update controls (important for debugging).** When a Bowel Issue is active, both the
pinned "Bowel Issue Active" banner and the retrospective alert directly below it can render at the
same time — `submitBowelBannerUpdate()` (banner) always targets `latestBowelDay()` (whichever day is
currently driving the active streak — often today), while `submitBowelMovement()` (retrospective
alert) always targets yesterday specifically, regardless of what the banner is doing. **v52 through
v54:** the banner suppressed its own control (showing a pointer message, "Update using the Bowel
Movement card below") whenever its target day matched the card's, since the card was always visible
and a second identical control would've been redundant. **v55:** that pointer-message branch was
removed. Since the retrospective alert now only renders while yesterday is unanswered, the "banner
targets yesterday" case can only occur when yesterday already has an answer (that's what makes the
issue streak active) — meaning the retrospective alert is always hidden in exactly that scenario, so
there's nothing left to defer to. The banner now **always** shows its own inline control, explicitly
labeled with whichever day it targets (e.g. "Update status for today" / "…for yesterday"). If a user
reports one of these "not doing anything," check which control they actually used before assuming the
write failed.

**Silent-failure fix (v51).** Both functions used a delete-old-entry-then-add-new-entry pattern with
no error handling — if either Firestore call failed for any reason, the async function threw and
aborted before reaching `setToast()`, so the UI gave zero feedback that anything went wrong. This is
exactly what Aaron reported ("doesn't look like it does anything"). Fixed by wrapping both in
try/catch mirroring `clearMissedDoses()`'s pattern: on failure they now show "Could not save — check
connection and try again" and leave the pending selection in place rather than silently resetting it.
See the v51 entry in Version History below.

## 7. Service Worker

**Cache name:** `caretracker-testing-v55` — bump this (using this repo's own version number, see
README) on every deploy to force devices to refresh.

**Cached shell:** `'./'`, `'index.html'`, `'manifest.webmanifest'`, icons.

**Fetch strategy:** network-first for `firestore.googleapis.com` / `gstatic.com` / `googleapis.com`,
cache-first for everything else.

`firebase-messaging-sw.js` exists (copied from prod) but is inert here — `subscribePush()` returns
immediately when `TEST_MODE` is true, so no token is ever registered and this service worker never
receives a background message in practice.

## 8. Deployment

1. Edit `index.html` (and `sw.js`, docs) locally.
2. Bump `CACHE` in `sw.js` to the new version number (see TESTING_README's versioning convention).
3. Complete sandbox QA with a mocked Firestore harness, then push testing changes directly to `main`
   under the repository's standing testing-push authorization. GitHub Pages auto-deploys within about
   one minute. Production remains a separate repo and still requires Aaron's explicit approval.
4. Devices with the old service worker pick up the new version on next visit.
5. **New Firestore collections require a Security Rules update, not just an app-code change.**
   Firestore default-denies any collection without an explicit `allow` rule — adding a `collection()`/
   `doc()` call in `index.html` alone does nothing for security. See the `caretracker_test_prefs` rule
   added alongside v50 as the reference example. AI agents in this environment may be hard-blocked
   from editing Firestore Security Rules directly (a tool-level restriction, not a repo rule) — if so,
   hand Aaron the exact rule block/full rules file to paste and publish himself, and don't consider
   the feature done until he's confirmed it's published and you've verified live that the
   `permission-denied` error is gone.
6. **Never use GitHub's inline web editor to paste in replacement file content.** See the warning at
   the top of this document and the v53 Version History entry — a pasted/replacement edit made
   directly in GitHub's browser editor is what wiped `index.html`, `sw.js`, and this file down to the
   literal text `undefined` on the night of Jul 19–20, 2026. Always edit locally and push a real diff.

### Cache reset
Visit this app's `reset.html` — unregisters service workers, clears caches, redirects with a
cache-busting query string. **Note:** this only clears the client-side cache/service worker — it
cannot fix a broken *server-side* deploy (see the v53 incident below, where the live files themselves
were corrupted and a cache reset understandably did nothing).

## 9. Known Issues & Gotchas

1. **Documentation drift is a known past failure mode for this repo** — an earlier version of this
   handoff described a dark theme, push notifications, and a `send-reminders.js`/GitHub Actions
   system that never existed here; it had been copy-pasted from production almost verbatim. If
   anything in this document looks inconsistent with the actual `index.html`, trust the code and
   fix the doc — don't propagate the mismatch further.
2. **v38–v49 version-history gap (open, Jul 19 2026).** These versions were built, QA'd, and pushed
   to this repo — Dex/Zofran chemo-window fixes, testing UX fixes, a crash hotfix, the Batch A–I
   testing-only UI/feature work, Bowel Movement + Symptoms tracking, and a pinned Bowel Issue banner
   — but TESTING_README's Version History table was not updated at the time each shipped. Flagged to Aaron
   on Jul 19, 2026 (while documenting v50) for a decision on backfilling accurate rows from commit
   history as a separate pass, rather than guessing at the details here.
3. **No authentication** — anyone with the URL can read/write test data. Not a real risk since this
   collection isn't real medical data, but worth knowing.
4. **Shared Firebase project** (`fuelforge-7c132`) — isolated from prod only by collection name, not
   by a separate project. Don't touch project-level Firebase settings without checking prod impact.
5. **Zofran's chemo-day block is independent of its (now-removed) gap timer** — don't assume
   `gapH: 0` means Zofran is unrestricted; `status()` checks `zofranBlockedOn()` before the gap logic
   and still locks it (with override) on chemo days 0–1.
6. **`painLevel` is required for new Tylenol and Morphine entries** — `confirmTimeAndLog()` rejects a
   blank selection. Older test records from before v30 may not have a value, so history rendering must
   remain tolerant of it.
7. **Single-file architecture** — same tradeoff as prod: simple, but one large file to edit carefully.
8. **GitHub's web editor can silently destroy a file (fixed incident, Jul 19–20 2026)** — see the
   warning at the top of this document and the v53 entry below. There is no server-side undo besides
   pushing a corrected commit; GitHub's commit history is the only safety net, which is why this repo
   must never rely on uncommitted/local-only backups as the sole copy of a "last known good" state.
   **The Morphine half-dose window feature this incident's commit was named for was never actually
   built or added anywhere — production is unaffected and unchanged; if that feature is still wanted,
   it needs to be built fresh in a real editing session.**

## 10. Version History

See TESTING_README.md's **Testing Version History** table for the authoritative, dated list (this repo uses
`vN` numbers matching production's scheme, offset one ahead while testing leads — not an independent
counter). **v38–v49 have not yet been individually documented there — see Known Issues #2.**

### v55 — July 20, 2026

**Appetite + Bowel Movement moved to persistent Home alerts (matching Daily Weight).** Aaron asked
for both retrospective daily check-ins to "appear like the Weight card where it is persistent until
answered." Implemented by factoring Daily Weight's existing escalation logic into two shared
functions, `dailyAlertLevel(now)` and `dailyAlertStyle(level)` (quiet before noon, firm noon–6 PM,
urgent 6 PM+), and building new Bowel Movement and Appetite Home alerts on top of them that render
only while yesterday is unanswered and disappear the instant it's logged — Daily Weight itself was
refactored to use the same two functions rather than its own copy of the logic, so all three alerts
now share one implementation instead of three near-identical ones. Appetite's dropdown+notes input
moved off Reports entirely onto this new Home alert; **Reports → Appetite** is now history-only.
Bowel Movement's old always-visible quiet card (previously sitting under Weight, with an Update
button that worked even after answering) was replaced by the new hide-once-answered alert; a new
**Reports → Bowel Movement** tab was added, also history-only, so past entries are still reviewable
and removable. Removing an entry from either history list makes its Home alert reappear, since the
day becomes unanswered again — this is now the correction path for a wrong answer, replacing the old
"just re-select and hit Update" flow. Fixed a logic gap this exposed: the "Bowel Issue Active" banner
used to defer to the old always-visible retrospective card ("Update using the Bowel Movement card
below") whenever they targeted the same day; since that card no longer stays visible once answered,
the banner now always renders its own inline control instead — the old pointer-message branch was
dead code that would have pointed at nothing. QA: mocked-Firestore harness, a new 25/25 suite
(`test_daily_alerts_v54.js`) covering escalation tiers for both new alerts, hide-on-answer, and the
two history-only Reports pages; `test_bowel_dedup.js` rewritten for the new always-own-control banner
behavior (10/10); full regression pass across `test_tylenol_liquid_appetite.js` (29/29, one assertion
updated to match Reports > Appetite no longer having a dropdown), `test_missed_clear_testing.js`
(15/15), `test_bowel_update_fix.js` (7/7), `test_missed_clear.js` (10/10) — 96/96 total. `sw.js`
cache bumped to `caretracker-testing-v55`.

### v54 — July 20, 2026

**Tylenol Liquid + Appetite tracking.** Two independent additions, both requested together by Aaron.
(1) **Tylenol Liquid** — new medication `tylenol-liquid` (30 mL / 1000 mg per dose, its own 6h gap
independent of pill Tylenol's 4h gap), tracked as a fully separate medication rather than a combined
pill-vs-liquid toast on the existing Tylenol card, so History/Reports never need to disambiguate "which
form." A new `ceilingGroup` mechanism pools its mg total with pill Tylenol against their shared existing
2500 mg/24h daily ceiling (`dailyGroupMg()`/`ceilingGroupMedIds()`), while a wholly separate
`volumeCeilingMl`/`dailyVolumeMl()` mechanism enforces its own independent 90 mL/24h cap — either limit
alone is sufficient to lock the medication, and the lock/warning UI reports whichever was actually hit.
See the new "Tylenol Liquid + shared-ceiling mechanism" subsection in Section 6 above for full details,
including a naming gotcha caught by QA (`safeMedicationId()` silently converts underscores to hyphens,
so the id had to be spelled `tylenol-liquid`, not `tylenol_liquid`, everywhere it's referenced). (2)
**Appetite tracking** — new **Reports → Appetite** tab, following the same retrospective
one-entry-per-day pattern as Bowel Movement: a previous-day dropdown (Normal / Little to none / No
Appetite) plus an optional notes field, Update overwrites the day's existing entry rather than
appending, wrapped in try/catch per the v51 Bowel Movement silent-failure lesson, and included in
`BYPASS_48H_IDS` so a day's answer stays correctable. See the new "Appetite tracking" subsection in
Section 6 above. QA: mocked-Firestore harness (hand-rolled fake DOM, no jsdom), 29/29 new checks
covering dose shape, the required-pain-level gate, both independent gap timers, shared-ceiling mg math
across both Tylenol forms, the independent volume ceiling, Appetite submit/overwrite/empty-rejection/
history-sort/Reports-rendering — plus a 25/25 regression pass across the existing missed-dose,
bowel-movement-update, and bowel-dedup suites (`test_missed_clear_testing.js`, `test_bowel_update_fix.js`,
`test_bowel_dedup.js`, `test_missed_clear.js`). `sw.js` cache bumped to `caretracker-testing-v54`.

### v53 — July 20, 2026

**Incident: GitHub web-editor commit wiped index.html, sw.js, and this file.** A commit titled "v53:
morphine half-dose window tracking, bump SW cache" was pushed directly to `main` through GitHub's
inline web editor overnight (not via this AI agent, not via a local edit) and replaced the entire
contents of `index.html`, `sw.js`, and `TESTING_HANDOFF.md` with the literal 9-character text
`undefined` — almost certainly a paste gone wrong, since GitHub's "Edit file" box replaces a file's
whole contents with whatever's pasted into it, with no diff shown before commit. **No Morphine
half-dose feature was actually added anywhere — the commit only destroyed files, and production is
untouched.** Live impact: the testing app served a blank/broken page (the literal word "undefined")
to every visitor, and — critically — a client-side cache reset (which Aaron correctly tried first)
could not fix it, because the server itself was now serving the broken file, not a stale cached copy.
Diagnosed by fetching the live files directly (`fetch('/sw.js')` returned a 9-byte `"undefined"` body
with a 200 status) and confirmed via the GitHub commit API, which showed `+3 -2,944` lines across
exactly those three files. **Fix:** restored all three files to their exact pre-incident content
(commit `96c0bc5`, the last known-good state — fetched directly from GitHub's raw content for that
commit to guarantee a byte-exact restore rather than a manual reconstruction), then bumped `sw.js`'s
`CACHE` to `caretracker-testing-v53` to force every client (including ones that had cached or
received the broken version) to pick up the restored app on next load. No app functionality changed
versus v52 — this is a pure recovery (see the v52 entry below, documented for the first time here
alongside this incident). **New standing rule, added to Sections 8/9 above and TESTING_CLAUDE.md:** never
paste replacement content into GitHub's inline web editor — always edit locally and push a real diff.

### v52 — July 19, 2026

**Fix: redundant Bowel Issue banner Update control when it targets the same day as the retrospective
card.** Aaron asked for a design recommendation after the v51 fix surfaced that the pinned "Bowel
Issue Active" banner and the retrospective "Bowel Movement" card (under Weight) can render
simultaneously, each showing what looks like an identical "select a status, tap Update" control —
but they don't always target the same day (`submitBowelBannerUpdate()` targets whichever day is
driving the active streak; `submitBowelMovement()` always targets yesterday). Recommendation
implemented: when the banner's target day and the card's target day are the same, the banner now
shows a plain pointer message ("Update using the Bowel Movement card below.") instead of rendering a
second, functionally-duplicate control; when the two days genuinely differ, the banner keeps its own
control but now explicitly labels which day it will update (e.g. "Update status for today" / "…for
yesterday" / "…for Mon, Jul 14" for older dates), so it's never ambiguous which day either control
affects. QA: mocked-Firestore harness, 8/8 checks (same-day suppression, different-day labeling for
today/yesterday/an older date, and exact select-element counts per scenario), plus a 24/24 full
regression pass alongside the v50 and v51 suites. `sw.js` cache bumped to `caretracker-testing-v52`.
Verified live via a temporary test Firestore entry (created, screenshotted, then deleted) confirming
the labeled banner control rendered correctly for a same-day-as-today scenario.

### v51 — July 19, 2026

**Fix: Bowel Movement Update silently doing nothing on failure.** Aaron reported the Update button
on the Bowel Movement card "doesn't look like it does anything." Root cause: `submitBowelMovement()`
and `submitBowelBannerUpdate()` had no try/catch around their delete-old-entry + add-new-entry
Firestore calls — any failure threw and aborted the function before it reached `setToast()`, so the
UI gave no feedback at all. Fixed by wrapping both in try/catch matching `clearMissedDoses()`'s
existing pattern: failures now show "Could not save — check connection and try again" and leave the
pending selection in place. Also documented (see Section 6's Bowel Movement subsection) that the
pinned "Bowel Issue Active" banner and the retrospective card are two separate Update controls that
can be on screen simultaneously and target different days — a likely source of confusion distinct
from the error-handling bug itself. QA: mocked-Firestore harness, 7/7 checks (success path, delete
failure, add failure, banner handler). `sw.js` cache bumped to `caretracker-testing-v51`.

### v50 — July 19, 2026

**Persistent missed-dose Clear button (Firestore-backed).** Replaces the ephemeral in-memory
`state.testMissedClearedAt` Clear button (reset to 0 on every reload, so a "cleared" banner would
reappear as soon as the page refreshed) with the same durable version now live in production:
clearing writes `{ missedClearedAt: state.now }` to a new `caretracker_test_prefs/settings` document
via `clearMissedDoses()`, loaded back on every app start via `subscribePrefs()`'s `onSnapshot`
listener. Uses its own `caretracker_test_prefs` collection — never shared with production's
`caretracker_prefs` — via a `PREFS_COL_NAME` ternary mirroring the existing `COL_NAME` pattern, so a
Clear tap in one app can never affect the other; verified in the mocked-Firestore QA harness. The
button's old `TEST_MODE ? ... : null` gate was removed, since this is now real functionality rather
than testing scaffolding — testing's and production's Clear-button code are now structurally
identical, so a future promotion is a trivial copy instead of a regression risk. `sw.js` cache bumped
to `caretracker-testing-v50`. Required a new Firestore Security Rule for `caretracker_test_prefs`,
handed to Aaron to paste/publish per the Section 8 note above.

### v35 — July 18, 2026

**Mobile navigation and medication-management overhaul.** The legacy top-tab strip was replaced by a
fixed, safe-area-aware mobile bottom navigation with **Home**, **Meds**, **Reports**, and
**In-Patient**. Home retains the v34 compact Quick Log and all existing Firebase-backed logging,
chemo, missed-dose, cycle, vitals, and in-patient behavior. Meds is an editable configuration view
for active medications; its Add/Edit/confirmation-protected Delete controls persist configuration to
browser-local storage only and do not create a Firestore collection or alter the medication-entry data
model. Reports is now a menu leading to the preserved History, Weight, and Cycle detail views, each
with a themed right-aligned back control while bottom navigation remains available. The header now
keeps only CareTracker branding, Brandi's Meds, date/time, and Live sync.

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

### Add or edit a medication
1. Use the **Meds** bottom-nav page and select **Add medication** or **Edit**. The UI captures the
   display name, generic name, dose labels/mg values, gap, daily limit, and schedule type.
2. The active medication configuration persists only in browser-local storage under
   `caretracker_testing_med_config_v1`; it does **not** create or modify a Firebase collection.
3. Deleting a medication requires explicit in-app confirmation. Existing Firestore entries are not
   deleted, and their historical medication label remains resolvable through archived local metadata.
4. For a new built-in default or a medication that needs special clinical logic (missed-dose windows,
   chemo-only behavior, pain scale, etc.), update `DEFAULT_MEDS` and the relevant explicit render or
   safety rules in `index.html`, then complete the full mocked-Firestore regression pass.

### Bump the version
1. Check the **actual pushed** `care-tracker` (prod) repo for its current live version.
2. This repo's new version = that number + 1 (or +1 again if testing is already ahead and adding
   more changes before prod catches up).
3. Update `sw.js`'s `CACHE` constant and both `TESTING_README.md`/`TESTING_HANDOFF.md`.

### Debug the live testing app
1. Open the testing URL in Chrome, DevTools → Console for errors.
2. DevTools → Application → Service Workers to check SW status (`caretracker-testing-vN`).
3. Confirm the orange "🧪 Testing app" banner is visible — if it's missing, `TEST_MODE` may have been
   accidentally flipped, which is a serious bug (would point this app at prod's collection).
4. Look for the "Live sync" indicator in the header for Firestore connectivity.
5. **If the page looks entirely blank or shows literally nothing/"undefined"** even after a cache
   reset: don't assume it's a client-side caching problem — fetch the live file directly
   (`fetch('/care-tracker-testing/index.html').then(r=>r.text())` in a console) to check whether the
   *server-side* file itself is broken. See the v53 incident above for exactly this failure mode.

## 12. Keeping These Docs Updated

**When you make any change to this repo, update `TESTING_CLAUDE.md`, `TESTING_README.md`, and this file in the same
pass.** Specifically here: bump "Last updated" / "Current version" above, add a row to Section 10's
pointer target (TESTING_README's table), and revise any of Sections 4–9 that the change affects. Stale docs
in this exact repo have already caused real confusion once (see Known Issues #1) and again with the
v38–v49 gap (see Known Issues #2) — don't let it happen a third time.
