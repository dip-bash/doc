# Mouse and Selection

Mouse handling is on by default (`mouse = true` in the config).

## Clicking

* **In a pane** - focus it (its column, then the pane). The click is swallowed
  unless the app asked for mouse reporting.
* **Apps with mouse mode** (vim, htop, ranger...) get every event, including
  full motion reporting, remapped to their own coordinates at ananta's
  layout. Hold **shift** with a click to bypass an app that owns the mouse
  and use ananta's own selection / focusing instead (the same
  convention terminals use).
* **On the status bar** - workspace tabs are clickable.
* **On a border** - focuses only; no drag begins.
* **In the [overview](Workspaces-and-Layout)** - focuses the clicked pane,
  zooms back in and centers it; the *release* right after is swallowed so the
  app doesn't receive a phantom click.

## Drag selection

Dragging inside one pane selects text there, with the selection clipped to
that pane - never borders, never neighbours, even though the strip may be
scrolled.

1. Press and drag: a char-wise selection (`selection.go`'s `Selection`,
   Kind `'c'`) highlights as you move.
2. Dragging past the top/bottom edge of a pane auto-scrolls its history.
3. Release copies the text immediately - see
   [Copy and Paste](Copy-and-Paste). A press+release without movement
   cancels (`finishSelection` drops a zero-size selection).

Selection coordinates are **logical**: history rows first, then the live
screen, so a selection holds meaning across app output.

* Any keypress clears the highlight (`handleKeys` nils `m.sel`).
* A pane dying clears its selection; so does entering copy mode.

## Wheel

| Mode | Wheel effect |
|---|---|
| Pane on the primary screen | scroll its history (3 lines / tick) |
| App in mouse mode | forwarded as raw scroll events |
| Full-screen app without mouse mode | arrow keys (`ctrl+A/B` cursor form chosen per app-cursor mode) - less / vim-style pagers behave like scrolling |
| [Overview](Workspaces-and-Layout) | vertical wheel changes workspace rows, horizontal wheel changes column focus |
| Help overlay | scrolls the help text |

## What's forwarded when

Deciding happens per event in `mouse.go`:

1. A drag in progress owns motion + its release.
2. Wheel: dispatch by pane mode, as tabled above.
3. Motion with a button (`Btn&32`) - forwarded only to apps in a motion
   mouse mode (`1003` etc.), else ignored.
4. Releases/other buttons: forwarded to the focused app if it requested
   mouse reporting (SGR "1006" normal encoding preferred, legacy
   `ESC [ M` coordinates clamped to 222 like xterm).

Mouse events reach the mux only when the *client terminal* is in mouse mode
(`?1002h / ?1006h`, requested by `mouse = true`); a keystroke stream
including `<`-prefixed SGR mouse sequences is parsed by `internal/input` and
routed here.
