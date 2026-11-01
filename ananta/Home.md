# Home

**ananta** is a scrollable tiling terminal multiplexer written in Go, heavily
inspired by [niri](https://github.com/YaLTeR/niri) and
[PaperWM](https://github.com/paperwm/PaperWM) on the terminal side, and by
tmux on the session side.

Panes live in columns on an endless horizontal strip. Focusing a column glides
the viewport to it. Square borders, animated scrolling and zoom, an overview
mode, a configurable status bar, mouse support, per-pane scrollback, vim-style
copy mode, and tmux-style persistent sessions (detach / reattach).

```
 ┌─────────────┬──────────────────────────┬───────────┐
 │             │                          │           │
 │  pane       │  focused pane            │  pane     │
 │             │                          │  ┌──────┐ │
 │             │                          │  │stack │ │
 │             │                          │  └──────┘ │
 └─────────────┴──────────────────────────┴───────────┘
  1 │ 2/2  · copy · ·—————————  connected  ———————————·  cpu mem net  21:47 ? help
```

## Quick start

```
git clone https://github.com/<you>/ananta
cd ananta
go mod tidy && make build
./ananta --init        # writes ~/.config/ananta/config.toml
./ananta               # attach to session "main" (created on first use)
```

That's it. The default leader key is **alt**; `alt+h/j/k/l` moves focus,
`alt+enter` opens a new pane, `alt+d` detaches, `alt+o` opens the overview.

Go deeper:

* [Getting Started](Getting-Started) - first run, detach, reattach
* [Key Bindings](Key-Bindings) - the full chord table (leader-based)
* [Sessions](Sessions) - persistent background servers, `ananta ls`, `ananta kill`, Control a running session
* [Workspaces and Layout](Workspaces-and-Layout) - columns, stacking, scrolling modes, overview
* [Configuration](Configuration) - every option in `config.toml`
* [Status Bar](Status-Bar) - built-in and custom shell widgets
* [Scrollback](Scrollback) - history, wheel, copy mode
* [Copy Mode](Copy-Mode) - vim-style visual selection across all history
* [Mouse and Selection](Mouse-and-Selection)
* [Copy and Paste](Copy-and-Paste) - OSC 52 + native clipboard tools
* [Architecture](Architecture) - package layout, the server/client wire protocol, hacking guide
* [FAQ](FAQ)
* [Troubleshooting](Troubleshooting)
* [Roadmap and Limitations](Roadmap-and-Limitations)

## Requirements

* Linux (macOS builds and runs; the system-stat bar widgets need `/proc`
  and hide themselves without it)
* Go 1.22+ to build
* A terminal. foot/kitty/konsole/wezterm/ghostty/alacritty work best; a
  terminal with the
  [kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)
  is required only if you bind `super`/`ctrl` as the leader.

## One-paragraph pitch

tmux gives you persistent sessions; window managers give you the freedom of a
scrollable strip of windows. ananta gives you both: every pane is a real
program in a background server, so closing the terminal or dropping SSH never
loses your work - and the layout is a niri-style horizontal strip with smooth
animation instead of a fixed grid.
