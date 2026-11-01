# Configuration

ananta reads `~/.config/ananta/config.toml` (override the location with
`-config PATH` or by setting `XDG_CONFIG_HOME`). Start or end with
`./ananta --print-config` to see every option with comments.

The embedded defaults are always merged **under** your file: a minimal
config with three lines is valid, and anything you leave out keeps its
default. Map values (`[keys]`, `[bar.widgets]`) merge key by key, so you
cannot delete a default entry - rebind unwanted shortcuts to `"none"`.

Changes take effect when the *session server* restarts: after editing,
`ananta kill NAME && ananta` (or start a new session).

## The whole default file

This is essentially `--print-config` output; keep this page in sync when the
default changes:

```toml
leader = "alt"             # alt | super | ctrl | ctrl+alt | ...
extended_keys = "auto"     # auto (on when leader needs it) | on | off

shell = ""
mouse = true
animate = true
animation_speed = 18.0
fps = 60
scroll_mode = "fit"        # fit | center
overview_scale = 0.3       # 0.15 - 0.6
border_style = "single"    # single | double | heavy | ascii
column_widths = [0.33, 0.5, 0.66, 1.0]
default_column_width = 0.5
resize_step = 0.05
scrollback_lines = 10000

prefix = ""                # e.g. "ctrl+b" to enable prefix mode

[bar]
position = "bottom"        # top | bottom | hidden
interval = 1.0             # widget refresh period, seconds
separator = " │ "
left   = ["session", "workspaces", "columns", "mode"]
center = ["message"]
right  = ["cpu", "mem", "net", "battery", "time", "help"]
time_format = "15:04"
date_format = "Jan 2"
mem_style = "used"         # used | percent | both

[theme]
border       = "#4c566a"
focus_border = "#88c0d0"
title        = "#81a1c1"
focus_title  = "#eceff4"
status_bg    = "#2e3440"
status_fg    = "#d8dee9"
accent       = "#88c0d0"
selection_bg = "#5e81ac"
selection_fg = "#eceff4"

[keys]
"leader+h" = "focus_left"
# ... (see [Key Bindings](Key-Bindings) for the full default table)

[prefix_keys]
```

## Option by option

### General

| Option | Default | Meaning |
|---|---|---|
| `leader` | `alt` | The modifier used in every `leader+...` binding. `super`/`ctrl` require a terminal that speaks the [kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/). Plain terminals cannot send super, and ctrl+h/i/j/m overlap backspace/tab/enter, so `alt` is the safe default. |
| `extended_keys` | `auto` | `on` always requests the extended keyboard protocol, `off` never, `auto` enables it when the leader or any binding needs it. |
| `shell` | `$SHELL` | The program each empty pane runs. |
| `mouse` | `true` | Master switch for all mouse handling; turns off the terminal mouse-report injection. |
| `animate` | `true` | Smooth scroll / zoom / column-resize animation. |
| `animation_speed` | `18.0` | Higher = snappier (exponential smoothing constant). |
| `fps` | `60` | Render tick rate; clamped to 10..240. |
| `scroll_mode` | `fit` | `fit` (PaperWM) or `center` (always centered). |
| `overview_scale` | `0.3` | Overview zoom; clamped to 0.15..0.6. |
| `border_style` | `single` | `single`, `double`, `heavy`, or `ascii` (square corners only). |
| `column_widths` | `[0.33, 0.5, 0.66, 1.0]` | Presets for `cycle_width`. |
| `default_column_width` | `0.5` | Width fraction of a new column. |
| `resize_step` | `0.05` | Step for `grow`/`shrink`. |
| `scrollback_lines` | `10000` | History lines per pane (0 disables). |
| `wallpaper` | `"default"` | Wallpaper program behind panes: `"default"` (wallpaper.toml), another file from the same folder, or `"none"`. |
| `prefix` | empty | tmux-style prefix chord; enables `[prefix_keys]`. |

Colours accept `"#rrggbb"`, a 0..255 palette index, or `"default"`
(terminal default / transparent).

### Validation and warnings

Out-of-range numeric values are clamped to the documented ranges; unknown
action names, unknown bar widgets and an invalid `bar.position` produce
warnings that flash in the status bar for a few seconds when the session
starts. `leader = "super"` additionally warns if your terminal lacks the
kitty protocol.

### Keys / prefix keys

```toml
[keys]
"alt+h" = "focus_left"           # exact chord, ignores leader
"leader+w" = "none"              # unbind
"leader+/" = "help"
"leader+3" = "goto_workspace 3"

[prefix_keys]                    # only consulted after the `prefix` chord
"n" = "new_pane"
```

Chords are normalised: `Alt+Return` = `alt+enter`, `leader+H` = `alt+shift+h`,
single uppercase letters imply shift, aliases (`return`, `escape`, `pgup`,
`bs`, `spc`, ...) map to canonical names. The modifier order in canonical
chords is ctrl, alt, super, shift.

## Environment

| Variable | Used for |
|---|---|
| `XDG_CONFIG_HOME` | config dir (fallback `~/.config`) |
| `XDG_RUNTIME_DIR` | session sockets / logs (fallback temp dir) |
| `SHELL` | default pane shell |
| `TERM/ANANTA` | exported into panes (`TERM=xterm-256color`, `COLORTERM=truecolor`, `ANANTA=1`) |
