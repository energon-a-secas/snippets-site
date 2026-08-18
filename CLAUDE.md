# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
make serve   # Start dev server at http://localhost:8815
make kill    # Kill the dev server
```

ES modules require an HTTP server: do not open `index.html` directly via `file://`.

## Architecture

Static, no-build modular ES module app. No npm, no bundler, no dependencies.

**Data flow:** `snippets/*.js` → `data.js` (aggregate) → `state.js` (filters + pins) → `render.js` (DOM) ← `events.js` (interactions)

### Snippet data (`js/snippets/<category>.js`)

Each category owns one module exporting an array named after its category key.
`data.js` imports all of them, concatenates in `CATEGORIES` order, and exports
`snippets`, `CATEGORIES` and `CATEGORY_LABELS`. Adding a category means: new module,
plus an import, a `CATEGORIES` entry, a `CATEGORY_LABELS` entry and a `BY_CATEGORY`
entry in `data.js`, plus a `--cat-<key>` colour and a `[data-cat="<key>"]` rule in
`css/style.css`.

```js
{ id, title, description, command, platform, tags[], category }
```

`id` must be unique across every module (a duplicate silently breaks pins and keyboard focus).
Current keys: `shell`, `macos`, `windows`, `git`, `data`, `text`, `media`, `config`,
`devops`, `docker`, `k8s`, `claude`, `ref`. Platforms in use: `bash`, `macOS`,
`PowerShell`, `Windows`, `Claude Code`, `any`.

**Escaping: the one real trap.** Commands are template literals, so:

- Backslashes (`C:\Users`, `.\app.log`, `\d`, `\;`, shell line continuations) need
  `` String.raw`…` ``. In a plain template `.\app.log` silently becomes `.app.log`
  and `\b` becomes a backspace character.
- Bash parameter expansion (`${VAR%.txt}`) needs a **plain** template with `\${`,
  `String.raw` does not disable `${}` interpolation.
- Backticks always terminate the literal, so avoid them (PowerShell line
  continuation, shell command substitution), use `$(...)` or one long line instead.

### State (`state.js`)

Filters live in the URL hash (`#q=…&cat=…&plat=…&tags=a,b&pinned=1`) via
`history.replaceState`, so any view is shareable and back/forward works through the
`hashchange` listener in `events.js`. Category, platform, tags and the pinned toggle
*also* persist to `localStorage` (`neo_snippets_filters`) and restore on a fresh
visit: the free-text query deliberately does not, because a stale search on load is
disorienting. Pins live separately in `neo_snippets_pins` and are filtered against
known ids on load, so removing a snippet cannot leave a dangling pin.

### Rendering (`render.js`)

`matchesExceptCategory()` is the basis for the rail's per-category counts, counts
reflect every *other* active filter, so a chip showing `0` really would be empty.
Search is multi-term AND across title + description + command + tags + platform.

`highlight()` in `utils.js` splits on the term regex and escapes each segment
*before* wrapping matches in `<mark>`: never escape first and then insert marks, or
a search for `amp` will corrupt `&amp;`.

Pinned snippets sort first (stable sort preserves catalogue order within each group).
Cards over 12 command lines get an expand toggle.

### Keyboard focus (`events.js`)

`state.focusId` tracks a snippet **id**, not an index, so it survives re-renders;
`renderSnippets()` clears it if the card is filtered out. `updateFocus()` toggles the
ring without a full re-render. `--toolbar-h` is published by a `ResizeObserver` in
`app.js` and feeds `scroll-margin-top`, so keyboard scrolling never parks a card
under the sticky toolbar.

Shortcuts: `/` search · `j`/`k` or arrows move · `Enter`/`c` copy · `p` pin ·
`?` shortcuts · `Esc` close. Shift+`/` is accepted alongside `?` because layouts
disagree on which one they report.

### Shared kits

Header and footer come from `packages/neorgon-ui/` and are vendored here as
`css/neorgon-header.css`, `css/neorgon-themes.css`, `js/neorgon-header.js`,
`css/neorgon-footer.css`, `js/neorgon-footer.js`. **Never edit those files**: edit
the canonical source and re-run `sync-header.sh` / `sync-footer.sh`.
