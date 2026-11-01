# Key Bindings

All bindings are in `[keys]` of `~/.config/ananta/config.toml` with the syntax
`"chord" = "action [argument]"`. A chord is a modifier chain plus a key, e.g.
`"alt+h"`, `"ctrl+shift+pagedown"`, `"leader+D"`. `leader` expands to the
configured leader modifier.

**This page documents the defaults.** Press `leader+/` inside ananta to see a
scrollable help overlay generated from your *live* config, including any
bindings you added or removed.

## Lexicon

* **Column** - a vertical stack of panes that always occupies the full height of the tile area.
* **Workspace** - one strip of columns. There are always 9; `alt+1..9`.

## Focus and movement (leader = alt)

| Chord | Action |
|---|---|
| `alt+h` / `alt+left` | focus column left |
| `alt+l` / `alt+right` | focus column right |
| `alt+j` / `alt+down` | focus pane below |
| `alt+k` / `alt+up` | focus pane above |
| `alt+home` | focus first column |
| `alt+end` | focus last column |

## Rearranging

| Chord | Action |
|---|---|
| `alt+H` (shift+h) / `alt+shift+left` | move column left |
| `alt+L` / `alt+shift+right` | move column right |
| `alt+J` / `alt+shift+down` | move pane down |
| `alt+K` / `alt+shift+up` | move pane up |
| `alt+,` | stack the focused pane into the column to its left |
| `alt+.` | pop the focused pane out into its own column |

## Creating and closing panes

| Chord | Action |
|---|---|
| `alt+enter` | open a new column to the right (`new_pane`) |
| `alt+s` | stack a new pane below the focused one (`new_pane_below`) |
| `alt+w` | close pane |

`new_pane` and `new_pane_below` take an optional command; a bound value of
`"leader+g" = "new_pane git status"` replaces the shell with that command.

## Widths

| Chord | Action |
|---|---|
| `alt+r` | cycle through `column_widths` presets |
| `alt+f` | maximize / restore column |
| `alt+=` | widen (`grow`) |
| `alt+-` | narrow (`shrink`) |
| `alt+c` | toggle always-centered vs PaperWM-style scrolling |

## Workspaces

| Chord | Action |
|---|---|
| `alt+pagedown` | next workspace (however empty) |
| `alt+pageup` | previous workspace |
| `alt+1` .. `alt+9` | go to workspace N |
| `alt+!` `alt+@` `alt+#` | move the focused pane to workspace 1 / 2 / 3 |

## Views and modes

| Chord | Action |
|---|---|
| `alt+o` | toggle overview (all workspaces, zoomed out) |
| `alt+a` | enter copy mode (vim visual selection) |
| `alt+b` | show / hide the status bar |
| `alt+/` | help overlay |
| `shift+pageup` / `shift+pagedown` | scroll history / back to live |
| `alt+d` | detach (session keeps running) |
| `alt+Q` | quit the session |

## Unbound keys

Anything not bound is forwarded to the focused pane verbatim, mouse events
included (see [Mouse and Selection](Mouse-and-Selection)).

The prefix key itself is never lost when a prefix is configured: while ananta
waits for the key that completes a prefix, pressing the prefix chord again
sends the prefix key *as input* to the focused app (`keyboard.go` appends
`k.Raw` instead of re-arming the prefix).
## Rebinding

```toml
[keys]
"leader+h" = "focus_left"        # default shown; change freely
"alt+1"    = "goto_workspace 1"  # explicit chord, ignores the leader
"leader+w" = "none"              # remove a default binding
```

The first word of an action value is the action name; everything after the
first space is its argument. The full action list lives in
`internal/config/actions.go`; unknown names produce a warning in the status
bar on startup.

## Prefix mode (tmux-style)

Off by default. Set

```toml
prefix = "ctrl+b"

[prefix_keys]
"n" = "new_pane"
```

and `ctrl+b` becomes a one-shot prefix exactly like tmux: the next key press
is looked up in `[prefix_keys]`, or typed through to the app if unmatched
(after the prefix key is sent as-is when pressed twice).
