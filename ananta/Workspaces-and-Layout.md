# Workspaces and Layout

ananta is a scrolling strip manager, not a classic grid: workspaces contain
columns, columns contain vertically stacked panes, and the strip is usually
wider than the screen.

## The model

```
strip (endless width)              screen (your terminal)
[ col A ][ col B ][ col C ][ ... ]   <------ viewport W cells ------>
```

* There are always **9 workspaces** (`alt+1`..`alt+9`).
* A **column** is a vertical stack of 1..N panes; every pane in a column has
  the same width.
* Column width is stored as a *fraction of the screen*
  (`default_column_width`, default 0.5), so proportions survive resizes.
* The focused column gets bright borders / a bright title; within a column
  the focused pane is what receives your keystrokes.

## Focus navigation

* `alt+h/j/k/l` (and arrows): step between columns and between panes inside a
  stack.
* In the default `fit` scroll mode the strip glides just enough to reveal the
  focused column; in `center` mode it glides to put the focused column in the
  exact middle.
* Clicks focus too, so does the mouse wheel left/right (`dir == wheel` on a
  pane scrolls it, but horizontal wheel focuses columns).

## Stacks

A column with more than one pane is a *stack*: its panes split the height
evenly (`paneHeight` divides the tile height).

* `alt+enter` opens a new **column** to the right of the focused one.
* `alt+s` opens a new pane **below** the focused one (new stack entry).
* `alt+,` stacks the focused pane into the column to its left.
* `alt+.` pops it back out into its own column.
* `alt+H/J/K/L` (or shift + arrows) move columns/panes around.

## Widths

* `alt+r` cycles the focused column through `column_widths` (default
  0.33 / 0.5 / 0.66 / 1.0).
* `alt+f` maximizes to a full screen and restores.
* `alt+=` / `alt+-` widen / narrow by `resize_step` (default 0.05).
* `alt+c` switches scrolling mode; `alt+C` centers the focused column once.

## Scrolling modes (`scroll_mode` in config, toggled live with `alt+c`)

* **`fit`** (default, PaperWM style): the viewport only scrolls as far as
  needed for the focused column to be fully visible.
* **`center`** (niri-like "always"): the focused column is kept centered.

Animation can be disabled entirely with `animate = false` (positions are then
set instantly).

## Workspaces

* `alt+1..9` - go to workspace N directly.
* `alt+pagedown` / `alt+pageup` - cycle through **all** workspaces,
  including empty ones.
* The mouse wheel in overview mode and the bar's workspace tabs step only
  across *visible* (non-empty) workspaces plus the current one.
* `alt+1` never creates panes; entering an empty workspace shows
  `"empty workspace - press alt+enter to open a terminal"`.

## Overview (`alt+o`)

The overview zooms the whole layout out to `overview_scale` (default 0.3) and
stacks all workspaces vertically with a small gap. It shows live-scaled
**thumbnails** of the real pane content - the PTYs are never resized for it,
only redrawn smaller.

* Mouse wheel / `k` / `j` move between workspaces.
* Horizontal wheel / `h` / `l` move between columns.
* Click a pane (or press `enter`) to zoom in and focus it; a clicked window
  glides to the center.

## Geometry notes

* While a resize animation is playing, apps are already reflowed to the
  *target* layout (`syncSizes`), so a resize triggers exactly one reflow.
* The status bar occupies one row (`bar.position`, default `bottom`);
  `barVisible` and `tileH` exclude it from the tile area.
* Narrow terminals clamp columns to a minimum of 12 columns
  (`minColW`), so layouts degrade gracefully instead of mangling panes.
