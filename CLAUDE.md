# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A Korean-language, mobile-first single-page web app for tracking fridge/pantry
inventory, a todo list, and drafting SMS restock reminders ("냉장고 재고 ·
할일 · 문자" — "Fridge inventory · To-dos · SMS"). There is no backend and no
build system: the app is a single static `index.html` plus a `data.json` seed
file, meant to be opened directly or served as static files.

## Repository layout

- `index.html` — the entire app: inline `<style>` (design tokens, layout,
  components) and, when complete, inline `<script>` (state, rendering, event
  wiring). There is no separate CSS/JS file and no bundler.
- `data.json` — the seed data loaded on first run (see "Data model" below).
- `README.md` — one-line project description only.

No `package.json`, linter, formatter, or test suite exists in this repo.
There is nothing to install, build, or lint — treat this as a static site.

## Current state (important)

`index.html` on this branch is a **work-in-progress markup/CSS shell**: it
defines the header, stat tiles, and the tab bar (냉장고/할일/문자 =
fridge/todo/sms), and the styles for rows, steppers, todo items, SMS
bubbles, bottom sheets, and toasts — but the file is truncated mid-markup
(it currently ends inside an unclosed `<svg>` in the fridge search box) and
has **no `<script>` tag and no closing `</body></html>`**. The todo and SMS
sections, the add/edit bottom sheets, and all JS logic (state management,
rendering, persistence) are not present yet. Do not assume app behavior
exists just because a CSS class for it does — check whether the
corresponding markup/JS has actually been (re)written.

An earlier commit (`40f780e`, "Initial commit of fridge management HTML
layout") contained a complete but visually/structurally different prototype
with working JS. It used:
- `localStorage` key `fridge_todo_json_v2` to persist state across reloads.
- A fallback `fetch("data.json", {cache:"no-store"})` to seed state the
  first time there's nothing in `localStorage`.
- A `db = { foods, todos }` in-memory model, a `save()` that writes `db` to
  `localStorage`, and per-feature `render*()` functions re-rendering from
  `db` on every mutation (no diffing/virtual DOM).
- Simple event delegation on list containers (`onclick` handlers reading
  `dataset` attributes to route to `changeFood`/`editFood`/`deleteFood`/etc.)
  rather than one listener per row.

That prototype's element IDs/classes (`fridgeTab`, `foodList`, `.item`, …)
do **not** match the current markup's classes (`.row`, `.card`, `.stepper`,
`.tabs[data-tab]`, …), so its JS cannot be dropped in as-is — but the
overall pattern (single global `db` object, `save()` to `localStorage`,
full re-render per mutation, delegated click handlers keyed off
`dataset`) is the intended architecture and is the reasonable model to
follow when implementing the current markup's logic.

## Data model (`data.json`)

```json
{
  "foods":    [{ "id": "food001", "name": "계란", "qty": 10, "place": "냉장" }],
  "todos":    [{ "id": "todo001", "text": "계란 주문하기", "done": false, "bg": "" }],
  "settings": {
    "phone": "",
    "low": 2,
    "places": [{ "name": "냉장", "color": "#2f86b4" }]
  },
  "sent": []
}
```

- `foods[].place` is a free-text zone name that must match one of
  `settings.places[].name` (used to look up the zone's color, exposed as the
  CSS var `--zone` on food rows/chips).
- `settings.low` is the quantity threshold below which a food is flagged
  "부족" (low stock) / `qty === 0` is "품절" (out of stock) — see `.tag--low`
  / `.tag--out` in the CSS.
- `todos[].bg` is an inline `style="..."` string (e.g. `"background:#ddeefb"`)
  applied directly to a todo row for user-chosen highlighting — see
  `.todo.has-bg` styles.
- `sent` is intended to log SMS messages composed/sent from the "문자" tab
  (see `.sent`/`.sent__body` styles); it's an empty array in the seed data
  since no message-composition JS exists yet.
- IDs (`foodNNN`, `todoNNN`) are plain strings; the prior prototype
  generated new ones with `Date.now().toString(36) + "_" + Math.random()...`
  via a `uid()` helper — reuse that scheme for consistency if adding items.

## Conventions used in the existing markup/CSS

- All UI copy is Korean; keep new UI text Korean and consistent in register
  with the existing strings (e.g. 상품 종류/전체 수량/부족·품절 stat labels).
- Theming is done entirely via CSS custom properties on `:root`, redefined
  under `@media (prefers-color-scheme: dark)` — there is no JS theme
  toggle. Add new colors as tokens in both blocks rather than hardcoding.
- Layout is a single centered column, `max-width: 560px`, iOS-app-like
  (bottom fixed action bar with `env(safe-area-inset-bottom)`, bottom
  sheets for forms, a toast for transient feedback). Follow this shell
  rather than introducing a different page layout.
- Class naming is a light BEM variant (`.stat__k`, `.stat__v`,
  `.hint--plus`, `.todo.has-bg`) — match this style for new components.
- `--zone` is set inline per-element (e.g. `style="--zone:#2f86b4"`) to
  color a chip/row/dot per storage place; it's consumed by CSS
  (`background:var(--zone,currentColor)`), not hardcoded per-place classes.
- Accessibility touches already in place to preserve: `.sr` screen-reader-only
  class, `:focus-visible` outline, and a
  `@media (prefers-reduced-motion: reduce)` block that zeroes animation/
  transition durations — keep new interactive elements covered by these.

## Development workflow

There is no build/lint/test tooling. To work on this app:
- Edit `index.html` / `data.json` directly.
- Preview by opening `index.html` in a browser, or serving the repo root
  with any static file server (needed for the `fetch("data.json")` seed
  load to work — opening via `file://` will fail that fetch due to CORS,
  so prefer something like `python3 -m http.server` from the repo root).
- There are no automated tests; verify changes manually in the browser
  (check both the fridge/todo/sms tabs and light/dark color schemes).
