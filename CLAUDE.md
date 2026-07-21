# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page study tracker (学習トラッカー) for a Japanese university-exam student. It tracks study log entries, schedules spaced-repetition reviews, counts down to the exam date (共通テスト), and shows study-time statistics. The entire app is Japanese-language and mobile-first (fixed `max-width:480px`).

## The whole app is one file

**Everything lives in `index.html`** — markup, CSS (in a `<style>` block), and all application logic (in one IIFE inside a `<script>` block). There is:

- **No build system, no bundler, no transpile step.** It is hand-written ES5/ES6 vanilla JavaScript with zero runtime or dev dependencies.
- **No package.json, no test suite, no linter, no CI.**
- **No backend.** All state persists to `localStorage` under the key `kobe-study-tracker:v1`; there is no network/API layer.

To preview, open `index.html` directly in a browser, or serve the directory statically (e.g. `python3 -m http.server 8000` then visit `http://localhost:8000/`). "Deploying" means hosting this one static file. `README.md` is a two-line stub.

## Architecture (the parts that span the file)

**Three state objects** near the top of the IIFE, deliberately separated by lifetime:

- `data` — the persisted domain model (`school`, `examDate`, `themeColor`, `summerGoalHours`, `subjects[]`, `entries[]`, `colorVer`). This is the *only* thing written to `localStorage`.
- `ui` — ephemeral view state (active tab, which modal is open, stats range, edit toggles). Never persisted.
- `form` — the in-progress "record" form fields. Never persisted.

**Full re-render pattern.** `render()` rebuilds the entire `#app` innerHTML by concatenating strings from pure builder functions (`headerHTML`, `tabsHTML`, `reviewHTML`, `recordHTML`, `statsHTML`, `modalHTML`, `confirmHTML`). There is no virtual DOM and no partial DOM patching — every state change re-renders the whole screen.

**Event wiring is inline.** The generated HTML strings contain inline `onclick="App.foo(...)"` / `oninput` / `onchange` attributes. All handlers hang off the global `window.App` object defined near the bottom of the IIFE. To add interactivity you add a method to `App` and reference it from a builder-function string — there are no `addEventListener` calls.

**The one exception to full re-render is `refresh()`.** Re-rendering on every keystroke would blur the focused input, so while the user types in the record form, `setF`/`setDate` call `refresh()` instead of `render()`. `refresh()` mutates only the save button's disabled state and the review-date preview, leaving the inputs (and focus) untouched. Keep this distinction: form-field typing → `refresh()`; everything else → `render()`.

**Spaced-repetition core.** Each entry stores a `reviews:{day1,week1,month1}` boolean map. `STAGES` defines the 1-day / 1-week / 1-month offsets. `dueItems()` walks every entry × every unchecked stage, computing which reviews are due/overdue (`due`) vs. coming in the next 7 days (`up`); this drives the 復習 (review) tab and the badge count in the tab bar. `markDone(id, key)` flips a review flag.

**Manual SVG donut chart.** `statsHTML` renders a subject-breakdown donut with no charting library — `polar()`/`arc()`/`donut()` compute SVG arc paths by hand. Stats are windowed by `rangeWindow()` into week / month / summer / all-time; the "summer goal" (`summerGoalHTML`) back-calculates the required daily pace against `SUMMER_START`/`SUMMER_END`.

**Theme contrast is computed, not configured.** The header background is a user-chosen color; `luminance()` + `headerTheme()` derive readable text/sub/button colors from it, so any hex the user picks stays legible.

**Backup is text in / text out.** There is no cloud sync. Export serializes `data` to JSON shown in a textarea (with clipboard/Web Share fallbacks); import parses pasted text or a chosen file via `applyImport()`, which validates shape and asks for confirmation before replacing `data`.

## Conventions to follow when editing

- **After any mutation of `data`, call `persist()` then `render()`** (or `refresh()` for form typing). Forgetting `persist()` silently loses the change on reload.
- **All user-controlled strings interpolated into HTML must go through `esc()`.** Because rendering is `innerHTML` string concatenation, this is the only XSS guard — subject names, study content, school name, etc. are all escaped at the point of insertion.
- **Defensive load + schema migration.** On startup, after `load()`, missing/invalid fields are backfilled to defaults, and `colorVer` (`COLOR_VER`) gates a one-time migration of old subject colors. When you add a new persisted field, add a matching backfill in both the startup block and `applyImport()`, and bump `COLOR_VER` (with migration logic) only if existing stored values need rewriting.
- **Dates are handled as local-midnight and `YYYY-MM-DD` strings.** Use the existing helpers (`startOfDay`, `parseDate`, `addDays`, `isoDate`, `todayISO`, `DAY`) rather than raw `Date` math, so day-boundary and timezone behavior stays consistent.
- **Defaults and tunable constants live as `const`s at the top of the IIFE** (`DEFAULT_SUBJECTS`, `DEFAULT_EXAM`, `THEME_PRESETS`, `STAGES`, `SUMMER_START`/`END`, etc.). Change behavior there rather than hard-coding values inline.
- **UI copy is Japanese.** Match the existing tone and keep new user-facing strings in Japanese.
