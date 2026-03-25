# occult.el - Functional Specification

From Latin *occultus* ("hidden, secret"). Collapse any buffer region into a
single-line summary while keeping the underlying text fully intact.

## Problem

When working in Emacs buffers - LLM chat sessions, org documents, eshell, etc. -
verbose sections become visual clutter. Folding mechanisms like outline-mode or
org-cycle are structure-aware and don't work on arbitrary regions. We need a way
to visually collapse any selected region, with the guarantee that:

- The hidden text remains in the buffer (accessible to `buffer-string`,
  `buffer-substring`, org-export, copy/kill, LLM context extraction)
- Search (isearch AND evil-ex-search) can find text inside folds
- It works in any buffer: read-only, special-mode, TUI Emacs

## Core Mechanism

Emacs overlays with the `invisible` property + `before-string` for the summary.

- `invisible` hides the region text via `buffer-invisibility-spec`
- `before-string` displays the summary line (indicator + first line preview + ellipsis)
- `buffer-string` / `buffer-substring-no-properties` return the full original
  text regardless of overlay state - this is what LLM packages, org-export, and
  copy/kill use

## Public API

Two commands. That's it.

### `occult-toggle`

Interactive, DWIM behavior:

- Region active: collapse the region into a summary overlay
- Point on an occult overlay (no region): expand it (remove the overlay)
- No region, no overlay at point: no-op

### `occult-reveal-all`

Remove all occult overlays in the current buffer.

### `occult-hide-region` (beg end)

Non-interactive. Programmatic entry point for creating a fold.

## No User-Facing Minor Mode

There is no `occult-mode` the user toggles. The user calls `occult-toggle` and
it works. An internal minor mode (`occult--mode`) activates/deactivates
automatically to manage buffer-local hooks when folds exist. The user never
interacts with it directly.

## Overlay Properties

Each occult overlay carries:

| Property                         | Value                                         |
|----------------------------------|-----------------------------------------------|
| `occult`                         | `t` (marker for finding our overlays)         |
| `invisible`                      | `occult`                                      |
| `before-string`                  | Summary line with `occult-summary` face        |
| `isearch-open-invisible`         | `occult--isearch-reveal`                      |
| `isearch-open-invisible-temporary` | `occult--isearch-reveal-temporary`           |
| `modification-hooks`             | Remove overlay if underlying text is edited   |
| `evaporate`                      | `t`                                           |
| `keymap`                         | TAB and mouse-1 toggle the fold               |
| `help-echo`                      | "Press TAB to expand"                         |

## Summary Line Format

```
⨁ First line of the region, truncated to max-length...
```

- Indicator: customizable via `occult-indicator`, default `"⨁ "`
- Ellipsis: customizable via `occult-ellipsis`, default `"..."`
- Max length: customizable via `occult-summary-max-length`, default 50
- Extracts the first line of the folded region (up to the first newline)
- Truncates to `occult-summary-max-length` characters

## Faces

Inherit from standard faces to work in light and dark themes without custom colors.

- `occult-summary` - the summary text. Inherits from `shadow`.
- `occult-indicator` - the prefix glyph. Inherits from `font-lock-constant-face`.

## Search Integration

### isearch (C-s / C-r)

Native integration via `invisible` property:

- `isearch-open-invisible-temporary`: temporarily reveals the fold while
  searching, re-hides when search moves on
- `isearch-open-invisible`: permanently reveals (deletes the overlay) when
  isearch exits with point inside a fold

### evil-ex-search (/ and ?)

Optional integration, only when evil is loaded. After `evil-ex-search-forward`,
`evil-ex-search-backward`, `evil-ex-search-next`, `evil-ex-search-previous` - if
point lands inside an occult overlay, temporarily reveal it. Re-hide when point
moves out.

Implemented via advice on evil search commands, guarded by `(featurep 'evil)`.

## Auto-Reveal

Controlled by `occult-auto-reveal`:

- `nil` (default): folds stay collapsed until explicitly toggled
- `echo`: show full text (truncated to a few lines) in echo area when point
  is on a fold
- `expand`: temporarily expand when point enters, re-collapse when point leaves

isearch integration is always active regardless of this setting.

## Revert-Buffer Persistence

Folds survive `revert-buffer` (important for LLM chat buffers, eshell, etc.):

- `before-revert-hook`: save `(beg end content-hash)` tuples for all
  occult overlays into a buffer-local variable
- `after-revert-hook`: for each saved tuple, verify text at `(beg . end)`
  matches the stored hash. If yes, re-create the overlay. If hash doesn't
  match, the fold is lost (graceful degradation).

This works reliably for append-only buffers (LLM, eshell) where old content
doesn't shift. For buffers that rebuild entirely (Dired `g`), folds are
lost - which is the expected behavior.

## Edge Cases

- Overlapping regions: refuse to create a fold if the region overlaps an
  existing occult overlay
- Nested folds: refuse (same check as overlapping)
- Empty / whitespace-only region: no-op
- Single-line region: works (collapses to truncated summary)
- Read-only buffers: works (overlays don't modify buffer text)

## Customizable Variables

| Variable                    | Default   | Description                              |
|-----------------------------|-----------|------------------------------------------|
| `occult-indicator`          | `"⨁ "`   | Prefix string for summary line           |
| `occult-ellipsis`           | `"..."`   | Suffix string for summary line           |
| `occult-summary-max-length` | `50`      | Max chars from first line to show        |
| `occult-auto-reveal`        | `nil`     | Auto-reveal mode: nil, echo, or expand   |
| `occult-lighter`            | `" Occ"`  | Mode-line lighter (internal mode)        |

## Package Metadata

- Requires: Emacs 29.1
- No external dependencies (evil integration is optional/lazy)
- License: GPL-3.0-or-later
- Single file: `occult.el`
