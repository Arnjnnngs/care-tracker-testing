# CLAUDE.md — care-tracker-testing

Instructions for any AI agent (Claude or otherwise) working in this repository.

## What this repo is

This is the **staging/testing** counterpart to `Arnjnnngs/care-tracker` (production). It exists so new
features can be built and verified here first, using fabricated test data, before they're ever
promoted to the app Brandi's caregiver actually relies on.

- **Live app:** https://arnjnnngs.github.io/care-tracker-testing/
- **Production app (do not confuse with this one):** https://arnjnnngs.github.io/care-tracker/
- **Firestore project:** `fuelforge-7c132` (shared with prod, but writes to a separate collection — see below)

## Hard rules — read before touching anything

1. **Never write test/QA data to `caretracker_entries`.** That collection is Brandi's real medical
   history. This app writes to `caretracker_test_entries` (`COL_NAME`, gated by `TEST_MODE = true`
   near the top of `index.html`). If `TEST_MODE` is ever flipped to `false` here, stop — that would
   point this staging app at production data.
2. **Never push to the production `care-tracker` repo's `main` branch without Aaron's explicit,
   in-the-moment go-ahead.** This applies even if a change looks trivial or was already approved for
   testing. Production and testing are promoted as a deliberate, separate step.
3. **Ask before pushing to *this* repo too.** Build and QA changes locally/in-sandbox first; confirm
   with Aaron that a change is ready before committing and pushing to `care-tracker-testing`.
4. **Never run real QA against live Firestore.** Use a mocked Firestore harness (in-memory store +
   pub/sub, matching the shape of `subscribeEntries`/`addEntryDB`/`removeEntryDB`) driven by jsdom or
   a real headless browser. Only manual, deliberate testing by a human should touch the actual
   `caretracker_test_entries` collection.
5. **Keep documentation current.** Every change to `index.html` that affects behavior, medications,
   Firebase fields, or the service worker gets a matching update to `README.md` and
   `CARETRACKER_HANDOFF.md` in the same pass — see "Maintaining documentation" in both files.

## Repo-specific things that trip people up

- This repo has **no** `.github/workflows/`, no `send-reminders.js`, and no live push notifications —
  `subscribePush()` and `checkNotifications()` both short-circuit when `TEST_MODE` is true. Don't
  assume the production reminder/cron system applies here; it doesn't exist in this repo at all.
- `firebase-messaging-sw.js` is present (copied from prod) but effectively unused while `TEST_MODE`
  is on, since the app never registers for a push token here.
- The visual theme is **light pink glassmorphism**, not dark — don't copy prod's dark-theme
  descriptions into these docs.
- This repo currently has features prod does not: the chemo-cycle system (chemo date, Dexamethasone,
  Zofran chemo-day block, chemo banners), missed-dose alerts, menstrual cycle tracking, In-Patient day
  tracking, a pain-level (1–10) scale on Morphine logs, and Zofran treated as a plain as-needed med
  (no gap timer, no reminders). Confirm feature parity/divergence against prod before promoting
  anything — don't assume the two `index.html` files are close to each other structurally.
- **Versioning matches prod, offset ahead.** This repo's version number is always
  `(current live/pushed prod version) + 1` while testing is ahead of production — e.g. if prod is
  live at v27, testing is v28, and the next testing change becomes v29, and so on, until those
  features are promoted to prod and prod catches up. Do NOT use a separate "tN" counter — check
  the *actual pushed* `care-tracker` repo's `sw.js`/README (not any local unpushed copy) before
  assigning the next testing version number, since prod may have moved since this repo was last
  touched. Service worker cache name here is `caretracker-testing-vN` using that same number
  (e.g. `caretracker-testing-v29`), not a separate testing-only counter.

## Workflow for a new feature request

1. Understand the actual current `index.html` in *this* repo (don't assume it matches prod — it
   frequently doesn't). Read the real file, not stale docs.
2. Implement against this repo's actual theme, state shape, and existing features so nothing
   regresses (chemo banners, missed-dose alerts, evening-meds flow, etc.).
3. Build a mocked-Firestore QA harness and verify both the new behavior and a regression pass over
   existing features.
4. Update `README.md`'s Version History table (new row, numbered per the versioning rule above)
   and `CARETRACKER_HANDOFF.md`.
5. Bump the `sw.js` cache version.
6. Ask Aaron for confirmation before committing/pushing to this repo's `main`.
