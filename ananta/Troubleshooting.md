# Troubleshooting

## Logs

The session server redirects its stdout/stderr to
`$XDG_RUNTIME_DIR/ananta-<uid>/NAME.log`. Crashes, panics and emulator debug
output land there. If something "disappeared", check that file first.

## Symptoms and fixes

### `ananta` says "session server did not start"
The server binary failed to launch or crashed instantly; `NAME.log` has the
reason. Common causes: a broken config file (reported at startup line
1), missing `/dev/pty` support in the container.

### Session exists but `attach` says "no session"
A stale `NAME.sock` from a crashed server is usually removed automatically
by `ananta ls` and by `EnsureServer`. If a hung (not dead) server holds the
socket, `ananta kill NAME` it, or `rm $XDG_RUNTIME_DIR/ananta-*/NAME.sock`
then start fresh.

### Config change did not take effect
The server reads the config at start: `ananta kill NAME && ananta`.

### Panes got tiny / layout squashed after resizing a terminal
The client's reported size drives everything. If your terminal sent a wrong
size once (SSH races, nested tmux), the panes kept that size. Attach anew in
a correct-size terminal, or resize the window to force a `SIGWINCH`.

### Colors wrong
Panes get `TERM=xterm-256color` + `COLORTERM=truecolor` independent of what
you set; if a specific program misbehaves, override its detection (e.g. it
may be seeing `ANANTA=1`). Theme-wise, `ParseColor` accepts `#rrggbb`,
0..255 indexes and `default` only - `#rgb` shorthand is **not** accepted yet.

### Keys don't do anything / "unknown action" warning
Chords are canonicalised (`Alt+Return` → `alt+enter`, `leader+H` →
`alt+shift+h`). If an action name in `[keys]` is misspelled the session
starts anyway with a warning badge; check the startup notification or
`NAME.log`. Also note: bindings only take effect on the *server* (restart).

### Copy still doesn't reach the system clipboard
OSC 52 is gated in some terminals (kitty: `allow_remote_control`; foot:
`osc52` settings). Native fallback runs `wl-copy`/`xclip`/... on the
**server machine**, not where your terminal is. See
[Copy and Paste](Copy-and-Paste).

### Help lists bindings I didn't set
The help overlay is generated from the *live* compiled keymap including
every default binding - to remove one ("none") or to reorder, use
`[keys]` in your config as usual.

### The server ate my CPU while detached
Detached sessions still redraw *nothing* but keep sampling stats and receive
pane output. If you want actual silence, `ananta kill`.

### `ananta local` behaves differently
`local` mode has no server: detach does nothing ("not in a session"), and
everything dies with the terminal; that is intended.

## Reporting

Please include `NAME.log`, your `config.toml` (trimmed is fine), terminal
name + `echo $TERM`, and the invocation (`ananta ls` output).
