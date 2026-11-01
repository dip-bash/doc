# Scrollback

Every pane keeps `scrollback_lines` (default 10000) history lines in the
emulator (`internal/vt`, a vendored vt10x fork with an added scrollback
ring). History includes the primary screen output only - programs that
switch to the **alternate screen** (vim, less, htop, ...) write there and
never touch scrollback, exactly as in a regular terminal.

## Ways to scroll

| Input | Effect |
|---|---|
| Mouse wheel over a pane | 3 lines towards/away from history |
| `shift+pageup` / `shift+pagedown` | half the pane height at a time |
| [Copy mode](Copy-Mode) | full vim motions across all history |
| Dragging a selection past an edge | auto-scroll while dragging |

Scrolling *anchors* the pane to an absolute history row; the pane shows the
same content even as new output lands below (`Anchored`/`Top` in
`internal/pane`). A small `↑N` badge in the bar's `mode` widget tells you how
far back you are.

**Typing returns to live.** Any keypress forwarded to a pane first clears the
anchor.

Full-screen apps without mouse mode get **arrow keys** for the wheel (with
application-cursor mode respected); apps that asked for mouse reports
receive raw wheel events instead.

## What's kept and what's trimmed

* The emulator stores history trimmed of trailing blank cells, so a
  panicking program spitting 10k empty-ish lines doesn't bloat memory.
* The limit is enforced lazily: over the limit the oldest lines are dropped
  from the front of the ring.
* `CSI J 3`-style "clear scrollback" sequences from the app are honoured
  (`clearScrollback`).
* Scrolling off the *alternate* screen never records history.

## History vs. resize

History is **not** re-wrapped when a pane is resized: the stored lines keep
whatever width they had at the time. That is the same trade-off tmux' default
made before `on`: widening shows extra space on the right of old lines rather
than re-flowing words. (See [Roadmap and Limitations](Roadmap-and-Limitations).)
