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
3. **Push to *this* repo (`care-tracker-testing`) directly once a change is built and QA'd —
   no confirmation round-trip needed (per Aaron, Jul 16, 2026: "when I ask for changes, you can
   push to testing only from now on... we'll cut down on the back and forth"). This is the one
   exception to rule 2 — it applies ONLY to `care-tracker-testing`, never to production. Still
   build and QA in the sandbox first; just don't wait for a go-ahead before pushing testing.
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

## GitHub web-editor cautions (learned Jul 17, 2026)

This repo's docs get edited via GitHub's web editor (Find & Replace panel), since the sandbox has no git push credentials. Failure modes that have bitten us here — know them before editing:

**Ctrl+A can wipe the whole file.** If focus isn't actually in the Find/Replace field when you press Ctrl+A, it selects the entire document body instead. Always confirm via screenshot which element has focus before using Ctrl+A or Delete. Triple-click to select existing field text instead of a blind Ctrl+A.

**Multi-line Replace-field content silently truncates.** Typing or pasting a replacement with several embedded newlines (e.g. a multi-paragraph section) into the Replace field only applies the first line/fragment — the rest is silently dropped, with no error shown. Single-line edits are unaffected.

**Multi-line Find patterns can silently match zero results.** Combining two edits into one Find string with an embedded newline can fail to match anything, with Replace All doing nothing and no error shown. Split into separate single-line Find/Replace operations instead.

**The editor auto-continues markdown numbered/bulleted lists.** Typing a line that starts with a digit-dot-space (e.g. "1. ") and then pressing Enter makes the editor auto-insert the next number on the following line — so directly typing your own pre-numbered list corrupts it with duplicated markers (e.g. "2. 2. text"), and can even inject a stray number into an unrelated heading right after the list. Avoid numbered-list syntax when typing multi-line content directly into the editor; use bold lead-in phrases as paragraphs instead, or add list markers in a separate pass after verifying the plain text landed correctly.

**Verified-safe pattern for large multi-line insertions:** use Find-only (leave Replace blank) to locate and highlight an anchor line, press Escape to close the panel (this keeps the highlight), click at the start of the matched line, press Home, then type the full multi-line content directly into the main editor body — bypassing the Replace field entirely.

**Always verify against the live file, not the editor UI.** After every commit, fetch the file from `raw.githubusercontent.com/<owner>/<repo>/main/<file>` and check the actual committed content before declaring a doc update done — don't trust what the editor displayed. Use small boolean/`.includes()` checks rather than dumping large raw content, since large text dumps from raw.githubusercontent.com pages can trigger a false-positive content-block.

**QA loop rule (Aaron, Jul 17, 2026):** after any change, verify against the live/actual state. If anything is found incomplete or wrong, fix it and re-run the entire verification pass from the start — not just the fixed item — before calling the task done.

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
6. Push to this repo's `main` once built and QA'd — no confirmation needed for testing (see Hard
   Rule 3). Production promotion always still needs Aaron's explicit go-ahead.
