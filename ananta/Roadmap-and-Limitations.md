# Roadmap and Limitations

Known gaps, honest list. Pull requests welcome.

## Hard limitations today

| Limitation | Why / impact |
|---|---|
| **No double-width glyphs** (CJK, emoji, ambiguous-width) | `internal/screen` treats every rune as one cell; the renderer's column bookkeeping desyncs from the real terminal's cursor. Text display with CJK/emoji corrupts; width math in the bar/help also counts runes. |
| **History is not re-wrapped on resize** | Old lines keep their width at capture time; widening shows columns of air on the right rather than re-flowing like tmux's `screen`/zellij fully reflow. |
| **One client per session** | A second attach displaces the first; no shared read-only observer. |
| **No bracketed paste emulation into panes** | Pasting multi-line text works, but shells cannot distinguish it from typing. |
| **No insertion (IRM) mode in the emulator** | A rare requirement; apps toggling mode 4 render imperfectly. |
| **No session state persistence** | Everything is in memory; server restart restarts from your shell. Sessions are not saved to disk ever. |
| **No screenshot / dump mode** | Panes render only to the attached client; there is no `capture-pane`-style text dump of a pane's visible state. |

## Planned

* wcwidth / wide-rune handling in the renderer/core.
* ` termination-aware` line rewrap on pane resize.
* Multiple observers (read-only "spectator" attach).
* Sixel / image protocols: not on the list, but not forbidden either.
* Better copy-mode `word-wrap`-aware selection across wrapped lines.

## Notes towards your own edits

* The action table is the cornerstone (config/actions.go): the help screen is
  generated from it and the keymaps validate against it.
* The emulator (`internal/vt`) is a vendored check-in; changes there should
  stay conservative (upstream upstream is unmaintained) and re-run the
  scrollback tests.
* Rendering lives on a one goroutine invariant; if you add "hot" info
  sources for the bar, keep they notify-only (`m.dirty <- struct{}{}` +
  `default:`), which is beautifully readable.

---

*Want to help?* The contributing surface area is deliberately tiny: mux
actions, bar widgets, key parsing, emulator fixes. See
[Architecture](Architecture) for the map.
