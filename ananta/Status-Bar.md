# Status Bar

The bar is one screen row (`bar.position = "top" | "bottom" | "hidden"`).
It is composed of three independent groups drawn left, center and right:

```toml
[bar]
left   = ["workspaces", "columns", "mode"]
center = ["message"]
right  = ["cpu", "mem", "net", "battery", "time", "help"]
```

* On narrow screens the **right-hand group drops widgets from the outside
  in** (the clock and help hint, listed last, survive longest).
* The center group is centered only when it fits; otherwise it is placed in
  the gap between left and right.
* Workspace **tabs** in the bar are clickable (`workspaces` widget).
* `leader+b` hides/shows the whole bar.

## Built-in widgets

| Widget | Shows |
|---|---|
| `workspaces` | one clickable tab per non-empty workspace plus the current |
| `columns` | focus position (`█`/`░` per column and `2/3`) |
| `mode` | scroll-back depth (`↑N`), copy mode / prefix / overview badges, `centered` when scroll mode is `center` |
| `message` | transient notifications (copy counts, spawn errors, config warnings, ...) |
| `title` | title of the focused pane (its set title, else its command name) |
| `help` | the chord bound to the help action |
| `cpu` | busy percentage; red at >= 90% |
| `mem` | `3.1G/7.7G` (`mem_style = "used"`), `41%` (`percent`) or both (`both`) |
| `net` | `↓1.2K ↑300B` per second, all interfaces except `lo` |
| `battery` | percent, `+` while charging/full; red at <= 20% when discharging |
| `load` | 1-minute load average |
| `time` / `date` | formatted with `time_format` / `date_format` (Go layout) |
| `host` / `user` | hostname / `$USER` |

cpu/mem/net/battery/load read `/proc` and `/sys` on Linux and **hide
themselves** where those files don't exist (macOS, restricted Android) -
custom widgets keep working everywhere. Refresh rate is `bar.interval`
(seconds, minimum 0.2). These widgets (and the clock) repaint the bar via the
mux's render loop; each new sample nudges it.

## Custom widgets

Any shell command; the first line of its output is shown, trimmed to 60
rune-widths:

```toml
[bar.widgets.weather]
command  = "curl -s 'wttr.in/?format=%t'"
interval = 600            # seconds; default/min 5
prefix   = ""             # text drawn before the output
color    = "#ebcb8b"      # any colour accepted by the theme

[bar.widgets.tbat]
command  = "termux-battery-status | jq -r '\"bat \\(.percentage)%\"'"
interval = 30
```

Then add the name to `left`/`center`/`right`.

Notes:

* A command that times out or fails clears the widget; it still retries
  every `interval`.
* Unknown widget names in the lists trigger a startup warning
  (`unknown bar widget: NAME`).
* Custom widgets run `sh -c` under a context with the same timeout as their
  interval, so a hung command cannot hang the bar forever.
