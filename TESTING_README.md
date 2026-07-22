# ⚠️ CareTracker TESTING

**This is the staging/testing app.** Changes are built and tested here before being promoted to the
real app at https://arnjnnngs.github.io/care-tracker/. It writes to a separate test database
collection (`caretracker_test_entries`) — nothing here touches Brandi's real med history. Push
notifications are disabled (`TEST_MODE = true` short-circuits `subscribePush()` and
`checkNotifications()`).

**Governance:** see `TESTING_CLAUDE.md` for the standing rules any contributor (human or AI) must follow in
this repo — most importantly: never point this app at the real `caretracker_entries` collection. After
sandbox QA, testing changes may be pushed directly to this repo; any production push still requires
Aaron's explicit, in-the-moment approval.

**Testing-only date override:** at the top of the header (visible on every primary view) there's a
date picker plus ± 1 Day buttons and a "Reset to Today" button. Use it to simulate any date without
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
|---|---|---|---|---|
| v65 | Jul 22, 2026 | v64 | **Bug fix: an already-open tab could keep running old, already-fixed JS for a long time.** Found while re-verifying UC11 during the 30-use-case QA pass — after v64 was pushed and confirmed live via a fresh navigation, a later check in the *same browser* showed the v64 fix wasn't applied (`caches.keys()` showed the cache was still stuck on `caretracker-testing-v63`). Root cause: the service worker's `install`/`activate` handlers already call `skipWaiting()`/`clients.claim()`, but nothing on the page side ever proactively asked the browser to check for a newer `sw.js`, and the browser's own background update check is slow/throttled — so a tab could sit on a stale cached app shell across multiple reloads even though the fix was already live on GitHub Pages. Added `registration.update()` right after registering, plus a `controllerchange` listener that reloads the tab once a new service worker actually takes control, so any future fix pushed here reaches an open tab within seconds instead of an indeterminate wait. No app logic changed. Verified by forcing the stale v63 tab to pick up v65 and confirming `caches.keys()` then showed only the new cache name and the dose-label text was correct for the simulated date. | Testing |
| v64 | Jul 22, 2026 | v63 | **Bug fix: dose "Last dose" labels used the real clock instead of the simulated date.** Found during the live 30-use-case QA pass — after advancing the testing-only date override by a day, doses logged the sim-previous day still showed "Today - Wednesday 7/22" instead of "Wednesday, 7/22", because `fmtDateLabel()` computed its "today" baseline from raw `Date.now()` rather than `state.now` (which the v31 date-override feature routes everything else through — see this file's "Testing-only date override" note above: "every date-dependent feature ... respects the simulated date"). `fmtDateLabel` was the one holdout. Fixed to use `state.now`. New 5/5 regression suite (`test_fmt_date_label.js`) plus full regression pass. | Testing |
| v63 | Jul 22, 2026 | v62 | **Fix: stale Symptoms-tab copy still referenced removed Diarrhea/Constipation options.** Found during the live 30-use-case QA pass. v57 removed Diarrhea/Constipation from the Symptoms dropdown (replaced with a required-note "Other") but left the on-screen subtitle ("Nausea, vomiting, diarrhea, or constipation noticed during or after treatment") unchanged, misleading the caregiver into expecting options that no longer exist. Updated the subtitle to describe current behavior and point to the Bowel Movement tracker for bowel changes. Also corrected the matching stale line in this file's "App Views" section (still listed Diarrhea/Constipation as Symptoms options). | Testing |
| v62 | Jul 22, 2026 | v61 | **Follow-up to v61: the Zofran "next dose" display time was still off by a day.** v61 fixed `zofranBlockedOn()`'s boundary but missed that `status()` computed the displayed unlock timestamp with its own separately hardcoded `+ 3 * 86400000` (chemo day + 3 days) — a leftover from the old 3-day-block bug. Corrected to `+ 2 * 86400000` to match the fixed 2-day block. Caught by extending `test_zofran_chemo_window.js` to also assert the displayed unlock time, not just the boolean block/unblock outcome (7/7). | Testing |
| v61 | Jul 22, 2026 | v60 | **Bug fix: Zofran chemo-day block was 3 days instead of the documented 2.** Found during a live 30-use-case end-to-end QA pass. `zofranBlockedOn()` read `o >= 0 && o <= 2`, blocking chemo day + the two following days — but this repo's own docs (this file's "Chemo Cycle" section and `TESTING_HANDOFF.md`'s precise `zofranBlockedOn(day) = offset in {0, 1}` note) both specify a 2-day block (chemo day + 1 following day only). Fixed to `o >= 0 && o <= 1`. New 6/6 regression suite (`test_zofran_chemo_window.js`) locks in the offset boundary (-2, -1, 0, 1, 2, 3) against the documented spec. **Production `care-tracker` has the identical bug — not fixed here, flagged to Aaron separately since production requires explicit go-ahead per repo.** | Testing |
| v60 | Jul 22, 2026 | v59 | **Meds tab: reorder Home Quick Log cards + always-alphabetical full list.** (1) New "Home screen order" section at the top of the Meds tab lists just the standalone (non-grouped) medications that render as their own Home Quick Log card, with ▲/▼ buttons to move each one up or down — the order set here is exactly the order those cards render in on Home (`reorderableHomeMeds()` swaps positions directly within `state.meds` via `moveReorderableMed()`, persisted the same way as any other medication-config change). Grouped Morning/Evening meds aren't included since they don't have individual Home cards to reorder. (2) The Meds tab's full medication list (below the reorder section) now always renders alphabetically by name, regardless of the underlying array order — so it stays easy to scan no matter how many medications have been added or reordered. QA'd with a new 19/19 mocked-DOM suite (`test_med_reorder.js`), including a regression guard for a boundary-button bug caught before shipping (the `h()` helper's `setAttribute` fallback would have rendered every non-boundary arrow button as `disabled="false"` — still disabled per the DOM spec — had `disabled: false` been passed instead of omitting the attribute entirely), plus a full regression pass. | Testing |
| v59 | Jul 22, 2026 | v58 | **Bug fix: new default medications now reach devices with an existing saved config.** Root-caused why Tylenol Liquid wasn't showing for Aaron: `loadMedicationConfig()` loads a device's saved medication list wholesale and never checked for medications added to `DEFAULT_MEDS` after that device's config was last saved — so any device that had saved a config before Tylenol Liquid existed would never see it, even though a fresh/no-saved-config session renders it correctly. Added `mergeMissingDefaultMeds()`: on load, any `DEFAULT_MEDS` entry missing from both the saved list and `archivedMeds` (i.e. not deliberately deleted) is appended, leaving all existing customizations for already-present meds untouched. QA'd with a new 12/12 mocked-localStorage suite (`test_med_config_merge.js`) covering the stale-snapshot case, the fresh-device case, the already-merged case, and the deliberately-archived-med case, plus a full regression pass. | Testing |
| v58 | Jul 20, 2026 | v57 | **Correction: Buspirone/Paroxetine default window fixed to 10 AM (was 8 AM–noon).** Per Aaron's clarifying feedback, the "no Protonix log yet" default is a single fixed clock time — 10 AM, i.e. Protonix's own typical 8 AM dose time + 2h — not a window matching Protonix's own 8 AM–noon window. This exactly mirrors how Iron's evening default (10 PM) already sits at Protonix's evening time (8 PM) + 2h. `DEFAULT_MEDS` windows changed from `{start:8,end:12}` to `{start:10,end:24,name:'Morning'}`; `morningWindowsFor()` comment corrected to match. `checkNotifications()`'s Buspirone/Paroxetine check moved from the fixed 8:30 AM block to a new 9:55–10:05 AM block (`sched-10am` key), matching the corrected default. QA'd: `test_v57_batch.js` updated (28/28) plus full regression (100/100) — 128/128 total. | Testing |
| v57 | Jul 20, 2026 | v56 | **Batch: Morning-linked Buspirone/Paroxetine, card-text/sizing fixes, medication editor clarity, Cycle/Journal separation, Symptoms dropdown cleanup.** (1) Buspirone/Paroxetine moved off the 10 PM evening window onto a new Morning window with Protonix (`morningLinkedToProtonix`, new `protonixMorningLogTs()`/`morningWindowsFor()` mirroring the existing evening pattern): default 8 AM–noon, shifting to 2h after Protonix's actual logged morning dose if later, open through end of day. New shared `renderGroupedMedsCard()` helper now powers both a new "Morning meds" Home card (Buspirone, Paroxetine) and the existing "Evening meds" card (now just Iron, Compazine). Also fixed the local notification scheduler (`checkNotifications()`), which still had Buspirone/Paroxetine wired to the old 8 PM evening reminder — moved to the 8:30 AM morning reminder. (2) Fixed redundant Evening/Morning meds card status text (e.g. "10 PM or 2h after Protonix" *and* "Opens 10 PM" both showing at once). (3) Chemo Schedule card resized to match the compact padding/border-radius of other Home cards. (4) `cycle_start`/`cycle_end` entries no longer appear in Today's Journal or Reports → History (only Reports → Cycle History, via `cyclePeriods()`, still shows them). (5) Medication editor: "Home quick log" toggle relabeled "Show as its own Home card", "Chemo plan" badge became a real "Chemo-day only" toggle (previously read-only), and a new "Group with morning meds" toggle added alongside the existing evening one — all with explicit helper text so a configured value is unambiguous. (6) Symptoms dropdown: Diarrhea/Constipation options removed (the dedicated frequency picker for them was removed too, now unreachable), a new "Other" option added requiring a note to submit. QA'd with a new 28/28 mocked-Firestore suite (`test_v57_batch.js`) plus a full regression pass across all existing suites (100/100) — 128/128 total. *(Superseded by v58 — the 8 AM–noon default described here was corrected to a fixed 10 AM default.)* | Testing |
| v56 | Jul 20, 2026 | v55 | **Morphine: rolling 4-hour / 15 mg dosing ceiling (replaces the old flat "last dose + 4h" lockout).** Fixes the exact bug Aaron reported: logging a ½ tab (7.5 mg) dose was locking Morphine out for a full 4 hours, same as a full 15 mg dose — but her prescription allows up to 15 mg *total* per rolling 4-hour window, so a lone half dose should leave 7.5 mg of room, not trigger a full lockout. Morphine is now modeled the same way Tylenol's daily ceiling already works (`ceiling:true`), except the mg total is summed over a trailing 4-hour rolling window instead of resetting at midnight (new `rollingCeilingH` config flag + `rollingDoseMg()` helper) — so it only locks once cumulative mg in the last 4 hours actually reaches 15 (one full dose, or two half doses). The per-dose buttons already had ceiling-aware blocking (reused from Tylenol, no new UI code needed): after a lone half dose, the ½ tab button stays enabled and the Full 15 mg button is disabled as "over limit." When the ceiling is hit, "Next dose at" now shows the real roll-off time — the moment enough of the counted doses age out of the 4-hour window to free up room for another half dose — computed by `rollingCeilingAvailableAt()`, instead of the old flat "last dose + 4h" math. QA'd with a new 14/14 mocked-Firestore suite (`test_morphine_rolling.js`) covering: full dose locks 4h from that dose; a lone half dose does not lock; two half doses reaching 15mg lock until the older one rolls off; doses spaced apart in the window roll off in the correct order; doses older than 4h no longer count; and live-rendered Quick Log output (badge, blocked button, and lockout time). Full regression pass across all current suites (14 + 10 + 7 + 15 + 25 + 29 = 100 checks) — 100/100. | Testing |
| v55 | Jul 20, 2026 | v54 | **Appetite + Bowel Movement moved to persistent Home alerts (matching Daily Weight).** Per Aaron's request, both retrospective daily check-ins now behave like the existing Daily Weight card: a persistent, escalating "must be answered" alert on Home (quiet before noon, firmer through the afternoon, urgent by evening — sharing the exact same `dailyAlertLevel()`/`dailyAlertStyle()` rules as Weight) that disappears the moment yesterday's entry is logged. Appetite's dropdown+notes input moved off Reports entirely; Bowel Movement's old always-visible quiet card (previously sitting under Weight) was replaced by the new alert. **Reports → Appetite** and the new **Reports → Bowel Movement** tab are now history-only — no input controls, just a reviewable/removable list; removing an entry there makes its Home alert reappear since the day becomes unanswered again. Also fixed a logic gap this change exposed: the "Bowel Issue Active" banner used to defer to the old always-visible retrospective card when they targeted the same day ("Update using the Bowel Movement card below"); since that card no longer stays visible once answered, the banner now always renders its own inline control instead. QA'd with a mocked Firestore harness: a new 25/25 suite (`test_daily_alerts_v54.js`) covering escalation tiers, hide-on-answer for both alerts, and the two new history-only Reports pages, plus the existing `test_bowel_dedup.js` suite rewritten for the new banner behavior (10/10) and a full regression pass (29/29 Tylenol Liquid/Appetite, 15/15 + 10/10 missed-dose, 7/7 bowel-update) — 96/96 total. | Testing |
| v54 | Jul 20, 2026 | v53 | **Tylenol Liquid + Appetite tracking.** New medication, Tylenol Liquid (Acetaminophen, oral suspension): 30 mL (1000 mg) per dose, its own independent 6-hour gap (separate from pill Tylenol's 4-hour gap), and its own 90 mL/24h volume ceiling — logged as a distinct medication (`tylenol-liquid`) rather than folded into the existing Tylenol card, so a toast/dose picker never has to ask "pills or liquid?" and the two forms can't be confused in History. The two forms *share* the existing 2,500 mg/24h Tylenol daily ceiling via a new `ceilingGroup` mechanism — logging either form counts against the same combined total, and hitting the shared cap locks both. Volume and mg ceilings are tracked independently: it's possible to be locked on the 90 mL liquid cap while still under the 2,500 mg shared cap, or vice versa, and the lock reason (ceiling badge) reflects whichever was actually hit. Also added Appetite tracking: a new **Reports → Appetite** tab asks about the previous day's appetite (Normal / Little to none / No Appetite, via dropdown) with an optional notes field for the reason, following the same retrospective one-entry-per-day pattern as Bowel Movement (Update overwrites the existing day's entry rather than appending) and the same 48-hour edit-lock bypass, so a day's answer stays correctable. Appetite History lists past entries with a color-coded status dot. QA'd with a mocked Firestore harness, 29/29 new checks (dose shape, independent gap timers, shared-ceiling math across both forms, independent volume ceiling, Appetite submit/overwrite/empty-rejection/history/Reports rendering) plus a 25/25 regression pass across the existing missed-dose, bowel-movement, and bowel-dedup suites. | Testing |
| v53 | Jul 20, 2026 | v52 | **Incident recovery: GitHub web-editor commit wiped `index.html`, `sw.js`, and `TESTING_HANDOFF.md`.** Overnight, a commit titled "v53: morphine half-dose window tracking, bump SW cache" was pushed directly to `main` through GitHub's inline web editor and replaced the entire contents of those three files with the literal text `undefined` (a paste-gone-wrong — GitHub's file editor has no diff preview before commit, so a bad clipboard paste silently becomes the whole file). No Morphine half-dose feature was actually built; the commit only destroyed files, and production was never touched. Live impact: the app served a blank/broken page, and a client-side cache reset (correctly tried first) could not fix it since the *server* was serving the broken file, not a stale client copy. Diagnosed via direct `fetch()` of the live files (200 OK, 9-byte `"undefined"` body) and the GitHub commit API (`+3 -2,944` lines across exactly those 3 files). Fixed by restoring all three files byte-for-byte from the last-known-good commit and bumping the SW cache to force every client to refresh. New standing rule: never paste replacement content into GitHub's inline web editor — always edit locally and push a real diff. | Testing |
| v52 | Jul 19, 2026 | v51 | **Fix: redundant Bowel Issue banner Update control.** When a Bowel Issue is active, the pinned "Bowel Issue Active" banner and the retrospective "Bowel Movement" card can render at once, each showing what looks like an identical select+Update control, but they don't always target the same day. Fixed per Aaron's request for a design recommendation: when the banner's target day and the card's target day are the same, the banner now shows a pointer message instead of a duplicate control ("Update using the Bowel Movement card below."); when the days genuinely differ, the banner keeps its own control but now explicitly labels which day it updates (e.g. "Update status for today"). QA'd with a mocked Firestore harness, 8/8 new checks plus a 24/24 full regression pass alongside the v50/v51 suites. Verified live with a temporary test Firestore entry (created, screenshotted, deleted). | Testing |
| v51 | Jul 19, 2026 | v50 | **Fix: Bowel Movement Update silently doing nothing on failure.** Aaron reported the Update button on the Bowel Movement card "doesn't look like it does anything." Root cause: `submitBowelMovement()` and `submitBowelBannerUpdate()` had no error handling around their delete-old-entry + add-new-entry Firestore calls — if either call failed for any reason, the async function threw and aborted silently, with no toast and no visible change. Fixed by wrapping both in try/catch (mirroring the existing `clearMissedDoses()` pattern): on failure they now show "Could not save — check connection and try again" and leave the user's pending selection in place instead of silently resetting it. Also documented for future debugging: when a Bowel Issue is active, two separate Update controls can be on screen at once — the pinned "Bowel Issue Active" banner (updates whichever day is driving the active streak) and the retrospective "Bowel Movement" card under Weight (always updates yesterday specifically) — easy to conflate since both look like a dropdown + Update button. QA'd with a mocked Firestore harness, 7/7 checks covering the success path and both functions' failure paths. | Testing |
| v50 | Jul 19, 2026 | v49 | **Persistent missed-dose Clear button (Firestore-backed).** Replaces the old ephemeral `state.testMissedClearedAt` Clear button (which reset on every reload) with the same durable version now live in production: clearing writes `{ missedClearedAt: ts }` to a new `caretracker_test_prefs/settings` Firestore document via `subscribePrefs()`/`clearMissedDoses()`, so the cleared state survives reloads and syncs across devices. Kept in its own `caretracker_test_prefs` collection — separate from production's `caretracker_prefs` — via a `PREFS_COL_NAME = TEST_MODE ? 'caretracker_test_prefs' : 'caretracker_prefs'` ternary mirroring the existing `COL_NAME` pattern, so clearing the banner in one app can never affect the other. The Clear button's old `TEST_MODE ? ... : null` gate was removed since this is now real, non-scaffolding functionality. This also makes testing's and production's Clear-button code structurally identical, so a future promotion of this feature is a trivial no-op instead of a regression risk. QA'd with a mocked Firestore harness confirming persistence across a simulated reload *and* that a testing Clear never touches the prod collection. | Testing |
| v38–v49 | Jul 18–19, 2026 | — | **Documentation gap — not yet backfilled.** These versions were built, QA'd, and pushed to this repo (Dex/Zofran chemo-window fixes, testing UX fixes, a crash hotfix, the Batch A–I testing-only UI/feature work, Bowel Movement + Symptoms tracking, and a pinned Bowel Issue banner) but this table was not updated at the time. Flagged to Aaron on Jul 19, 2026 for a decision on backfilling from commit history as a separate pass. | Testing |
| v37 | Jul 18, 2026 | v36 | **Senokot as-needed (mirrors prod v33).** Schedule windows removed, quick-log offers 1 pill or 2 pills. Includes a one-shot localStorage migration (`migrateSenokotV37`) so devices with a saved med configuration pick up the new shape — fires only while the saved Senokot is still window-typed, so later manual edits via the Meds page persist. | Testing |
| v36 | Jul 18, 2026 | v35 | **Backport of production v32 fixes.** Missed-dose coverage is now two-pass dose-to-window assignment: in-window/early doses claim their window first, then late doses (after a window closed, before the next opened) credit the window they follow — fixes the false MISSED alert when a dose was logged the same day (e.g., Dexamethasone at 11 AM + 6 PM flagged Afternoon as missed). A genuinely skipped window still alerts. Early tag now only applies to doses logged before the day's first window. Testing scaffolding untouched and verified intact: TEST_MODE, `caretracker_test_entries` routing, date-override control (simNow), header Testing chip. | Testing |
| v35 | Jul 18, 2026 | v34 | **Mobile navigation and medication-management overhaul.** Replaced the five top tabs with a fixed four-item bottom navigation: Home, Meds, Reports, and In-Patient. Home preserves the compact v34 Quick Log and all existing medication, chemo, missed-dose, cycle, vitals, and in-patient behavior. Meds adds an editable medication configuration page; configuration is stored only in browser-local storage and does not create or change a Firebase collection or the existing medication-entry data model. Reports is now a menu for the preserved History, Weight, and Cycle detail views, each with a themed back control while the bottom nav remains available. The header is simplified to CareTracker branding, Brandi's Meds, date/time, and the Live sync indicator. | Testing |
| v34 | Jul 17, 2026 | v33 | **Compact medication-card redesign.** Restyled the Today tab's Quick Log cards for substantially higher phone-screen density without changing medication logic or Firebase behavior: reduced card padding, corner radii, shadows, and inter-card gaps; placed medication and generic names on one compact baseline with the status badge on the right; combined last-dose and availability information into one line; and converted dose controls into smaller inline pill buttons. The light-pink glassmorphism palette, Hanken Grotesk/IBM Plex Mono typography, and `#AA5375` accent remain intact, with darker text for improved contrast. | Testing |
| v33 | Jul 17, 2026 | v32 | **Data-integrity fix.** Removed a dormant `seedDemo()` function that silently wrote 11 hardcoded fake medication entries into `caretracker_test_entries` whenever the app's first Firestore snapshot came back empty — which happens on a cold cache or a brief network blip on load, not only on a genuinely fresh install. The identical bug existed in production `care-tracker` (see its v28/v29 history) and had already written fake entries into real patient data there; this repo's fake entries never reached `caretracker_entries`, confirmed via a `TEST_MODE`/`COL_NAME` routing audit. Trigger permanently disabled (`if (false && wasEmpty && entries.length === 0)` in the Firestore subscription callback); all 11 fake entries identified by timestamp fingerprint and deleted from Firestore, re-verified via a fresh collection query (0 matches). The dead `demo` state flag and its banner UI were left in place — unreachable, harmless, flagged for an optional cleanup pass. | Testing |
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
- **Visual theme:** Light pink glassmorphism (`#FFF0F3` background, `#AA5375` accent) with compact, high-density Quick Log medication cards and a fixed native-style mobile bottom navigation — **not** production's dark theme
- **Backend/Database:** Firebase Firestore (project `fuelforge-7c132`, shared with prod, isolated by collection name)
- **Hosting:** GitHub Pages
- **Fonts:** Hanken Grotesk, IBM Plex Mono (Google Fonts)
- **Firebase SDK:** v10.12.0 (ESM imports from gstatic CDN)

## Project Structure

```
care-tracker-testing/
├── TESTING_CLAUDE.md          # Standing rules for AI/human contributors — read first
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
| `caretracker_test_prefs` | Small app-preferences doc (currently just the missed-dose banner's `missedClearedAt` timestamp, v50). **Isolated from prod's `caretracker_prefs`** so clearing the banner in one app never affects the other. |
| `caretracker_prefs` | Production's preferences collection — this app must never write here |

## Service Worker Strategy

- Cache name: `caretracker-testing-v65` (bump this — matching the app version above — to force updates on all devices)
- Static assets (cache-first): `./`, `index.html`, `manifest.webmanifest`, icons
- Firebase/API calls (network-first): `firestore.googleapis.com`, `gstatic.com`, `googleapis.com` — falls back to cache if offline
- As of v65, the page also calls `registration.update()` on load and reloads once on `controllerchange`, so a newly pushed cache version reaches an already-open tab within seconds instead of waiting on the browser's own background update check.

## Tracked Medications

| Medication | Generic | Tracking Type |
|---|---|---|
| Dexamethasone | Steroid, chemo premed | 2 tablets, 8 AM & 2 PM, only appears day before chemo through day after (chemoOnly) |
| Tylenol | Acetaminophen | Daily limit (2500 mg, resets midnight), 4h min gap, 500/1000 mg doses. **1–10 pain-level required at log time (v29; enforced as required in v30)**. Shares its 2500 mg daily ceiling with Tylenol Liquid (v54) |
| Tylenol Liquid | Acetaminophen, oral suspension | 30 mL (1000 mg) per dose, 6h min gap (independent of pill Tylenol's 4h gap), max 90 mL/24h (own volume ceiling, independent of the shared 2500 mg mg-ceiling). 1–10 pain-level required at log time. Added v54 |
| Zofran | Ondansetron | **As-needed — no gap timer, no reminders** (changed in v29). Blocked on chemo days 1–2 with override, per care team |
| Compazine | Prochlorperazine | 6h min gap; 10 PM routine + earlier as needed (in Evening meds card) |
| Morphine | Immediate release | Rolling 4h / 15 mg ceiling (v56, replaces the old flat 4h gap) — up to 15 mg total per any trailing 4 hours, so a ½ tab (7.5 mg) dose leaves room for one more ½ tab before locking. ½ tab (7.5 mg) / full tab (15 mg) doses. **1–10 pain-level required at log time (v29; enforced as required in v30)** |
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
Journal, or in History. **The banner has a Clear button (v50, Firestore-backed via `caretracker_test_prefs`)** that dismisses the current list of misses without deleting the underlying data — new misses occurring after the clear timestamp still show, and the cleared state persists across reloads and devices.

## Bowel Movement Tracking

**As of v55, the daily check-in lives on Home as a persistent "must be answered" alert** (same rules
as Daily Weight, see below) rather than an always-visible quiet card — it only appears while
yesterday is unanswered, escalating from a quiet reminder before noon to firm in the afternoon to
urgent by evening, and disappears the instant yesterday's entry is logged. Logging Diarrhea or
Constipation on the Symptoms tab also starts (never ends) the same tracker for that entry's day, if
that day doesn't already have its own answer. Consecutive days of None/Diarrhea count as an active
"Bowel Issue," shown via a separate pinned, non-dismissible **Bowel Issue Active** banner on Home
(v49) with its own inline update control, positioned directly below the retrospective alert — **as of
v55 this banner always shows its own control** (it previously deferred to the retrospective card when
they targeted the same day, back when that card stayed visible permanently; now that the card only
shows when unanswered, there's nothing left to defer to). **Reports → Bowel Movement (new in v55)** is
a history-only list of past entries — no input there; removing an entry makes the Home alert reappear
for that day. **Both save paths report failures instead of failing silently (v51)**.

## Appetite Tracking

**As of v55, the daily check-in lives on Home as a persistent "must be answered" alert**, matching
Bowel Movement and Daily Weight exactly (same escalation rules, same hide-once-answered behavior) —
it asks about the previous day's appetite via dropdown (Normal / Little to none / No Appetite) with
an optional notes field, right on Home instead of a Reports-page input card. **Reports → Appetite is
now history-only** (the dropdown/notes input that lived there through v54 was removed) — it just
lists past entries sorted most-recent-first with a color-coded status dot (green/amber/red) and the
optional note; removing an entry there makes the Home alert reappear since the day becomes unanswered
again. Submitting again for an already-answered day (via the Home alert, before it's answered) still
overwrites rather than duplicates. Entries bypass the normal 48-hour edit lock, same as Weight, Cycle,
and Bowel Movement.

## Daily "Must Be Answered" Home Alerts (Weight, Bowel Movement, Appetite)

Weight, Bowel Movement, and Appetite each get a persistent Home alert that only appears while that
day's entry is missing, sharing one set of escalation rules (`dailyAlertLevel()`/`dailyAlertStyle()`
in `index.html`): quiet (lavender) before noon, firm (amber) from noon–6 PM, urgent (red) from 6 PM
on. Weight asks about **today**; Bowel Movement and Appetite ask about **yesterday** (retrospective).
Each alert disappears the moment its entry is logged — Weight always keeps its separate always-present
quiet input card too (for logging additional readings), but Bowel Movement and Appetite do not, since
they're one-answer-per-day fields: once answered, correcting a mistake is done via the matching
Reports history list (Remove, then the Home alert reappears since the day is unanswered again).

## Chemo Cycle

Set a chemo date on the Home view's "Chemo schedule" card. The app derives offsets from that date:
Dexamethasone appears (and is required) days −1 through +1, phased red chemo banners run days −2
through +1, and Zofran is blocked on days 0–1 (override available if the care team says otherwise).

## Vitals, Cycle & In-Patient Tracking

- **Temperature** — logged in °F with timestamp; input shows **98.5°F as a grayed placeholder** (v29/v30) — the user must still type a value; an untouched field is rejected, not silently logged
- **Weight** — logged in lbs with timestamp; input shows **the last recorded weight as a grayed placeholder** (v29/v30) — same rule, must be typed to submit
- **Menstrual Cycle** — available at **Reports → Cycle** (v35; previously a standalone top tab). Logged as two one-tap events, Period Start and Period End — tapping logs immediately at the current time (no date/time picker). Shows a running "Day N" counter (days since the most recently logged period start), an "Active" badge while a period is in progress, and a Cycle History list of past periods. **A non-dismissible "Period Active" banner also appears on the Home screen for the whole cycle (v30)** — it has no close control and stays up until Period End is logged, with a one-tap Log Period End button right on the banner.
- **In-Patient (hospital) tracking** — **Start / End / Undo** model (v32; previously a single daily toggle), reached from the persistent **In-Patient** bottom-nav item. It has Log In-Patient Start / Log In-Patient End buttons (one-tap, timestamped to the second) plus an Undo button that only appears while a stay is open; Undo requires a second confirming tap (toast prompt) and deletes the open Start entry — for correcting an accidental Start, not for closing a real stay. History shows real start/end timestamps supporting half-day precision, e.g. `7/13/2026 3:15 PM – 7/14/2026 9:00 AM (2 days)`, or `7/13/2026 3:15 PM – Active` while still open. While a stay is active: a non-dismissible **In-Patient Active** banner is pinned to the top of Home (with its own Log In-Patient End button), and every med in Quick Log and Evening meds displays as `<Med Name> - In-Patient (Restricted)` in place of normal logging controls, since those doses are given by hospital staff and aren't tracked here.

## App Views

The fixed bottom navigation is present on every primary route: **Home**, **Meds**, **Reports**, and
**In-Patient**. It replaces the former top-tab bar and reserves bottom safe-area space on phone-sized
screens.

- **Home** — the former Today experience: In-Patient Active banner (pinned, only when a stay is open), missed-dose banner (with persistent Clear button, v50), Bowel Issue Active banner (pinned, only when an issue streak is active, v49), the Bowel Movement and Appetite "must be answered" alerts (v55 — only while yesterday is unanswered), the Daily Weight "must be answered" alert (only while today is unanswered), chemo banner with Dexamethasone/Zofran badges (when applicable), Period Active banner (when a cycle is open), dose counters, vitals inputs, compact Quick Log cards (shown as Restricted while In-Patient is active), and a grouped "Evening meds" card with a one-tap "Take all" (hidden while In-Patient is active).
- **Meds** — medication-management page for the active configuration. It lists each medication's display/generic names, dose options, gap/frequency rules, daily limit, and schedule type; supports Add, Edit, and confirmation-protected Delete. The configuration persists in browser-local storage only; dose and vitals history remains in the existing Firestore test collection.
- **Reports** — a menu of five cards that open preserved detail views while the bottom nav remains visible: **History** (grouped per day into Overnight/Morning/Afternoon/Evening; per-day summaries and rows exclude Weight, Temperature, and In-Patient start/end entries), **Weight** (trend chart and reading history), **Cycle** (Menstrual Cycle Start/End card plus Cycle History), **Bowel Movement** (v55, new — history-only, no input), and **Appetite** (history-only as of v55 — the input moved to a Home alert; just Appetite History here). Each detail view has a themed right-aligned back control to return to Reports.
- **In-Patient** — Start/End/Undo card plus a history list of hospital stays with real start/end timestamps.
- **Symptoms** — ad hoc, timestamped incident logging (Nausea, Vomiting, Other — Diarrhea/Constipation were removed here in v57 in favor of the dedicated Bowel Movement tracker), fully editable/deletable, no 48h lock.

## Troubleshooting: "All Blank" / Stale Cache

1. Visit the reset page (`reset.html` on this app's URL) — unregisters service workers, clears caches, redirects back
2. Or manually: Chrome DevTools → Application → Service Workers → Unregister, then hard refresh
3. On mobile: Settings → Site settings → this URL → Clear & reset

When deploying new versions, bump the `CACHE` constant in `sw.js` to match the new app version.

## Maintaining This Documentation

When making changes here, update these docs in the same pass:

- **TESTING_CLAUDE.md** — the standing rules file; update if a rule or gotcha changes
- **TESTING_README.md** (this file) — new row in the Version History table (numbered per the versioning
  rule above), and revise any sections affected by the change
- **TESTING_HANDOFF.md** — update "Last updated" and "Current version" at the top, add the new
  version to its Version History table, and revise any affected sections

All three files live in the repo root and are the source of truth for onboarding new contributors or
AI agents — treat doc updates as part of the feature, not a follow-up task.
