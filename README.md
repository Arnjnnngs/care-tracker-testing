# ⚠️ CareTracker TESTING

**This is the staging/testing app.** Changes are built and tested here before being promoted to the
real app at https://arnjnnngs.github.io/care-tracker/. It writes to a separate test database
collection (`caretracker_test_entries`) — nothing here touches Brandi's real med history. Push
notifications are disabled (`TEST_MODE = true` short-circuits `subscribePush()` and
`checkNotifications()`).

**Governance:** see `CLAUDE.md` for the standing rules any contributor (human or AI) must follow in
this repo — most importantly: never point this app at the real `caretracker_entries` collection, and
never push here (or to prod) without asking first.

**Testing-only date override:** at the top of the header (visible on every tab) there's a date
picker plus ± 1 Day buttons and a "Reset to Today" button. Use it to simulate any date without
waiting for real time to pass — every date-dependent feature (chemo offsets, missed-dose windows,
the cycle day counter, in-patient date ranges, and the timestamp on anything you log while the
override is active) respects the simulated date. This control only exists and only functions when
`TEST_MODE` is true — it's a no-op with no UI in production.

---

## Versioning (matches production, offset ahead)

This repo's version number is **`(current live production version) + 1`** while testing carries
features prod doesn't have yet — not an independent "t1, t2..." counter. Example: production is
live at v27, so this repo is v28; the next round of testing changes becomes v29; and so on until
those features are promoted to prod and prod's version catches up, at which point testing goes back
to being exactly one ahead.

Always check the **actual pushed** `care-tracker` repo (not a local/unpushed copy) for its real
current version before assigning the next testing version number.

## Testing Version History (this repo)

| Version | Date | Based on / ahead of prod | Changes under test | Status |
|---|---|---|---|---|| v33 | Jul 17, 2026 | v32 | **Data-integrity fix.** Removed a dormant `seedDemo()` function that silently wrote 11 hardcoded fake medication entries into `caretracker_test_entries` whenever the app's first Firestore snapshot came back empty — which happens on a cold cache or a brief network blip on load, not only on a genuinely fresh install. The identical bug existed in production `care-tracker` (see its v28/v29 history) and had already written fake entries into real patient data there; this repo's fake entries never reached `caretracker_entries`, confirmed via a `TEST_MODE`/`COL_NAME` routing audit. Trigger permanently disabled (`if (false && wasEmpty && entries.length === 0)` in the Firestore subscription callback); all 11 fake entries identified by timestamp fingerprint and deleted from Firestore, re-verified via a fresh collection query (0 matches). The dead `demo` state flag and its banner UI were left in place — unreachable, harmless, flagged for an optional cleanup pass. | Testing |
| v32 | Jul 16, 2026 | v31 | Menstrual Cycle moved off the Weight tab into its own **Cycle** tab (tab order: Today / History / Weight / Cycle / In-Patient), with a Cycle History list of past periods (unchanged underlying Start/End logic). Chemo banner rewrite: Zofran-restricted days now say "restricted" in the banner body and get a standalone red **Zofran — Restricted** badge; days Dexamethasone is due get a standalone light-green **Dexamethasone Due** badge — both replace the old plain-text wording so they can't be missed. In-Patient tracking redesigned from a single daily toggle to a **Start / End / Undo** model (mirrors the Cycle tab): Start and End are instant one-tap logs with a confirmation toast; Undo requires a second confirming tap (toast prompt) and deletes the open Start entry entirely — for correcting an accidental Start, not for closing a real stay. A non-dismissible **In-Patient Active** banner is pinned to the top of Today whenever a stay is open, with a Log In-Patient End button on the banner itself. While active, every med (Quick Log cards and Evening meds) displays as `<Med Name> - In-Patient (Restricted)` instead of normal logging controls. In-Patient history now stores real start/end timestamps (supports half-day precision) instead of day-buckets, e.g. `7/13/2026 3:15 PM – 7/14/2026 9:00 AM (2 days)`. New `inpatient_start`/`inpatient_end` entry types replace the old single `inpatient` marker going forward; legacy `inpatient` marker entries from before this version are still recognized for missed-dose suppression. | Testing |
| v31 | Jul 15, 2026 | v30 | Added a **testing-only date override** at the top of the header (every tab, TEST_MODE-gated): pick any date, or step ± 1 day, to simulate a different "today" without waiting for real time to pass — a "Reset to Today" button appears whenever the offset isn't zero. All date-dependent logic (chemo offsets, missed-dose windows, the cycle day counter, in-patient ranges, and the timestamp on every newly-logged entry) now flows through a single `simNow()` helper instead of `Date.now()`, so everything stays internally consistent while a simulated date is active. `simNow()` returns real `Date.now()` whenever `TEST_MODE` is false, so this has zero effect if ever copied into production. | Testing |
| v30 | Jul 15, 2026 | v29 | Tylenol/Morphine now require a pain level (1–10) before the dose can be confirmed — Confirm is rejected with a toast if none is selected. Temperature and Weight inputs went back to true placeholders (grayed hint text — 98.5 / last recorded weight) instead of pre-filled values, so an untouched field can no longer be accidentally logged; both now require the user to type a value. Added a non-dismissible "Period Active" banner to the Today (home) screen that stays up for the whole cycle, from Period Start until Period End is logged, with a one-tap Log Period End button on the banner itself. Marking a day In-Patient now hides that day's missed-dose alerts everywhere (Today banner, Journal, History) — meds given by hospital staff aren't tracked by this app, so they shouldn't be flagged as missed. | Testing |
| v29 | Jul 14, 2026 | v28 | Menstrual cycle tracking (Period Start/End one-tap logging, no time picker, "Day N since last period" counter, Active badge — on the Weight tab). In-Patient day tracking (one-tap toggle banner at the top of Today, new In-Patient tab grouping consecutive hospital days into date ranges, e.g. `7/13/2026` alone or `7/13/2026 – 7/14/2026 (2 days)`). Weight input now defaults to the last recorded weight; Temperature input now defaults to 98.5°F. Added a 1–10 pain-level dropdown to Morphine and Tylenol logging, shown in Today's Journal and History. Zofran changed from an 8-hour gap timer to a plain as-needed med — no lock, no scheduled or gap-based reminder (the chemo-day clinical block is unchanged). New `cycle_start`, `cycle_end`, and `inpatient` entry types; `inpatient` entries excluded from Today's Journal and History's per-day rows/counts entirely (they have their own tab); `cycle_start`/`cycle_end` still show as History rows but are excluded from the day's dose count. | Testing |
| v28 | Jul 13, 2026 | v27 | *(retroactively renumbered from "t1" to match the new versioning convention below — no functional change)* Chemo cycle: chemo date scheduling, Dexamethasone auto-appearing med (2 tablets, 8 AM & 2 PM, day before chemo through day after), Zofran locked on chemo days 1–2 with override (superseded in v29 — see above), phased red chemo banners (day −2 through +1), missed-dose alert system | Testing |

**Rollback:** every test version gets a full-history bundle saved before changes, same as
production. To roll back, redeploy the previous test version's `index.html`/`sw.js` and revert this
table's top row.

**Testing notes:** log test doses freely — this app writes to `caretracker_test_entries`, not the
real database. Use the date picker in "Chemo schedule" to simulate different days relative to chemo.

---

# Brandi's CareTracker (Testing)

Real-time family medication & vitals tracker — a progressive web app (PWA) for logging medications,
temperature, weight, menstrual cycle, and hospital (in-patient) stays with live Firebase sync. This
build also carries an in-development chemo-cycle scheduling system and missed-dose alerts, ahead of
what's live in production.

**Live App (testing):** https://arnjnnngs.github.io/care-tracker-testing/
**Live App (production):** https://arnjnnngs.github.io/care-tracker/
**Repository:** https://github.com/arnjnnngs/care-tracker-testing

## Overview

CareTracker is a single-page PWA built with vanilla JavaScript and Firebase Firestore. It tracks
daily medication doses, temperature, weight, menstrual cycle onset/end, and hospital in-patient days
for a caregiver workflow, plus a chemo-cycle scheduling system and missed-dose alerting that are
still under test here. The app uses real-time Firestore listeners for instant multi-device sync.
Push notifications are disabled in this build (`TEST_MODE`).

## Tech Stack

- **Frontend:** Vanilla JavaScript (ES module in production build; classic script in the QA harness), inline CSS, single-file `index.html`
- **Visual theme:** Light pink glassmorphism (`#FFF0F3` background, `#AA5375` accent) — **not** production's dark theme
- **Backend/Database:** Firebase Firestore (project `fuelforge-7c132`, shared with prod, isolated by collection name)
- **Hosting:** GitHub Pages
- **Fonts:** Hanken Grotesk, IBM Plex Mono (Google Fonts)
- **Firebase SDK:** v10.12.0 (ESM imports from gstatic CDN)

## Project Structure

```
care-tracker-testing/
├── CLAUDE.md                  # Standing rules for AI/human contributors — read first
├── firebase-messaging-sw.js   # FCM service worker, present but unused while TEST_MODE is on
├── icon-192.png                # PWA icon (192x192)
├── icon-512.png                # PWA icon (512x512)
├── index.html                  # Main app (all HTML/CSS/JS in one file)
├── manifest.webmanifest        # PWA manifest (name: "CareTracker TESTING")
├── reset.html                   # Cache reset utility for stuck service workers
└── sw.js                        # Service worker (caching + notification clicks)
```

There is **no** `.github/workflows/` and **no** `send-reminders.js` in this repo — the scheduled/
gap-based push reminder system that exists in production does not run here at all, because
`TEST_MODE` disables push registration entirely.

## Firebase Collections

| Collection | Purpose |
|---|---|
| `caretracker_test_entries` | All logged test data — meds, temperature, weight, cycle start/end, in-patient days. **Isolated from real data.** |
| `caretracker_entries` | Production's real collection — this app must never write here |

## Service Worker Strategy

- Cache name: `caretracker-testing-v31` (bump this — matching the app version above — to force updates on all devices)
- Static assets (cache-first): `./`, `index.html`, `manifest.webmanifest`, icons
- Firebase/API calls (network-first): `firestore.googleapis.com`, `gstatic.com`, `googleapis.com` — falls back to cache if offline

## Tracked Medications

| Medication | Generic | Tracking Type |
|---|---|---|
| Dexamethasone | Steroid, chemo premed | 2 tablets, 8 AM & 2 PM, only appears day before chemo through day after (chemoOnly) |
| Tylenol | Acetaminophen | Daily limit (2500 mg, resets midnight), 4h min gap, 500/1000 mg doses. **1–10 pain-level required at log time (v29; enforced as required in v30)** |
| Zofran | Ondansetron | **As-needed — no gap timer, no reminders** (changed in v29). Blocked on chemo days 1–2 with override, per care team |
| Compazine | Prochlorperazine | 6h min gap; 10 PM routine + earlier as needed (in Evening meds card) |
| Morphine | Immediate release | 4h min gap, ½ tab (7.5 mg) / full tab (15 mg) doses. **1–10 pain-level required at log time (v29; enforced as required in v30)** |
| Lidocaine | Topical cream | 4h min gap, max 4 applications per day |
| Imodium | Loperamide | Daily pill count limit (4 pills) |
| Protonix | Pantoprazole | Twice daily windows (8 AM–noon, 8–10 PM), missed-dose alerts |
| Buspirone | BuSpar | Once daily, 10 PM, missed-dose alerts |
| Paroxetine | Paxil | Once daily, 10 PM, missed-dose alerts |
| Iron | Ferrous sulfate | Once daily, 10 PM, missed-dose alerts |
| Senokot | Senna | 2 pills, 8 AM & 10 PM windows, as needed |

## Missed Dose Alerts

Dexamethasone (chemo days only), Protonix, Buspirone, Paroxetine, and Iron are tracked for missed
doses. When a schedule window closes with no dose logged, the app shows a red alert banner at the
top of Today (covering today's and yesterday's misses), a red MISSED row in Today's Journal, and red
MISSED rows plus a "N MISSED" day summary in History. Tracking starts July 12, 2026. Zofran, Tylenol,
Morphine, Lidocaine, Imodium, Compazine, and Senokot are as-needed and never flagged. **Any day marked
In-Patient is fully excluded from missed-dose detection (v30)** — meds given by hospital staff on
that day aren't tracked here, so nothing for that day is ever flagged as missed, on Today, in the
Journal, or in History.

## Chemo Cycle

Set a chemo date on the Today tab's "Chemo schedule" card. The app derives offsets from that date:
Dexamethasone appears (and is required) days −1 through +1, phased red chemo banners run days −2
through +1, and Zofran is blocked on days 0–1 (override available if the care team says otherwise).

## Vitals, Cycle & In-Patient Tracking

- **Temperature** — logged in °F with timestamp; input shows **98.5°F as a grayed placeholder** (v29/v30) — the user must still type a value; an untouched field is rejected, not silently logged
- **Weight** — logged in lbs with timestamp; input shows **the last recorded weight as a grayed placeholder** (v29/v30) — same rule, must be typed to submit
- **Menstrual Cycle** — its own **Cycle** tab (v32; previously a card on the Weight tab). Logged as two one-tap events, Period Start and Period End — tapping logs immediately at the current time (no date/time picker). Shows a running "Day N" counter (days since the most recently logged period start), an "Active" badge while a period is in progress, and a Cycle History list of past periods. **A non-dismissible "Period Active" banner also appears on the Today (home) screen for the whole cycle (v30)** — it has no close control and stays up until Period End is logged, with a one-tap Log Period End button right on the banner.
- **In-Patient (hospital) tracking** — **Start / End / Undo** model (v32; previously a single daily toggle). The **In-Patient** tab has Log In-Patient Start / Log In-Patient End buttons (one-tap, timestamped to the second) plus an Undo button that only appears while a stay is open; Undo requires a second confirming tap (toast prompt) and deletes the open Start entry — for correcting an accidental Start, not for closing a real stay. History shows real start/end timestamps supporting half-day precision, e.g. `7/13/2026 3:15 PM – 7/14/2026 9:00 AM (2 days)`, or `7/13/2026 3:15 PM – Active` while still open. While a stay is active: a non-dismissible **In-Patient Active** banner is pinned to the top of Today (with its own Log In-Patient End button), and every med in Quick Log and Evening meds displays as `<Med Name> - In-Patient (Restricted)` in place of normal logging controls, since those doses are given by hospital staff and aren't tracked here.

## App Views

- **Today** — In-Patient Active banner (pinned, only when a stay is open), missed-dose banner, chemo banner with Dexamethasone/Zofran badges (when applicable), Period Active banner (when a cycle is open), dose counters, vitals inputs, quick-log cards (shown as Restricted while In-Patient is active), and a grouped "Evening meds" card with a one-tap "Take all" (hidden while In-Patient is active)
- **History** — grouped per day into Overnight/Morning/Afternoon/Evening; per-day summaries and rows exclude Weight, Temperature, and In-Patient start/end entries (which have their own views); Period Start/End rows appear but aren't counted as doses
- **Weight** — weight trend chart only (v32 — Menstrual Cycle moved to its own tab)
- **Cycle** — Menstrual Cycle Start/End card plus Cycle History list (v32, moved off Weight)
- **In-Patient** — Start/End/Undo card plus a history list of hospital stays with real start/end timestamps

## Troubleshooting: "All Blank" / Stale Cache

1. Visit the reset page (`reset.html` on this app's URL) — unregisters service workers, clears caches, redirects back
2. Or manually: Chrome DevTools → Application → Service Workers → Unregister, then hard refresh
3. On mobile: Settings → Site settings → this URL → Clear & reset

When deploying new versions, bump the `CACHE` constant in `sw.js` to match the new app version.

## Maintaining This Documentation

When making changes here, update these docs in the same pass:

- **CLAUDE.md** — the standing rules file; update if a rule or gotcha changes
- **README.md** (this file) — new row in the Version History table (numbered per the versioning
  rule above), and revise any sections affected by the change
- **CARETRACKER_HANDOFF.md** — update "Last updated" and "Current version" at the top, add the new
  version to its Version History table, and revise any affected sections

All three files live in the repo root and are the source of truth for onboarding new contributors or
AI agents — treat doc updates as part of the feature, not a follow-up task.
