# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
make serve   # Start dev server at http://localhost:8815
make kill    # Kill the dev server
```

ES modules require an HTTP server — do not open `index.html` directly via `file://`.

## Architecture

Static, no-build modular ES module app. No npm, no bundler, no dependencies.

**Data flow:** `data.js` (static array) → `state.js` (filter state object) → `render.js` (DOM updates) ← `events.js` (user interactions)

### Snippet data model (`data.js`)

All snippets live in a single exported `snippets` array. Each entry has:

```js
{ id, title, description, command, platform, tags[], category }
```

`CATEGORIES` is the ordered list of valid category keys. `CATEGORY_LABELS` maps keys to display names. To add a snippet, append to `snippets` with one of the 10 existing category keys (`shell`, `macos`, `data`, `text`, `git`, `config`, `docker`, `k8s`, `devops`, `claude`).

### Filtering logic (`render.js`)

`matchesFilters(snippet)` applies all four active filters simultaneously (search query, category, platform, tags). Tag filter is OR-based: a snippet matches if it has **any** of the selected tags. Search query matches against the concatenated string of title + description + command + tags + platform.

Cards with more than 12 command lines get an expand/collapse toggle (`isLong = lines > 12`).

### State (`state.js`)

Plain exported object — no localStorage, no persistence. All state resets on page reload.

```js
{ searchQuery, activeCategory, activePlatform, activeTags[] }
```

### Dropdowns (`render.js` + `events.js`)

Custom dropdowns are re-rendered from scratch on every filter change (`renderDropdowns()`). The search inputs inside each dropdown (`dd-cat-q`, `dd-plat-q`, `dd-tag-q`) use `filterDropdownItems()` which shows/hides `<li>` items client-side without re-rendering. Tags dropdown is multi-select; category and platform are single-select.

### Suggest modal (`events.js`)

The "Suggest a snippet" modal has no backend. It composes a formatted string and copies it to the clipboard for the user to share manually.

### Utilities (`utils.js`)

- `$(id)` — cached `getElementById` wrapper
- `escHtml` — HTML-escapes all user-facing strings before innerHTML insertion
- `showToast(msg)` — creates and reuses a single `#app-toast` element; auto-hides after 2 s
- `debounce(fn, ms)` — used on search input (150 ms delay)
