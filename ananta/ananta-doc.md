# ananta — Complete Documentation

A scrollable tiling terminal multiplexer written in Go (~6.8k lines).
Inspired by niri and PaperWM on the layout side, and tmux on the session side.

Every pane is a real program running inside a background server. Closing the
terminal or dropping SSH never loses your work. The layout is an endless
horizontal strip of columns per workspace, with smooth animated scrolling and
zoom.

---

## 1. Project Layout

```
ananta/                      binary (built)
cmd/ananta/
  main.go                    CLI entry point: flags, command dispatch
  control.go                 control a running session (open, workspace, ...)
internal/
  app/
    app.go                   "local" mode: raw input loop around the mux
  config/
    config.go                TOML loading, keymap compilation, action table
    default.toml             embedded default config
    actions.go               action table + help screen generation
    theme.go                 color parsing
    wallpaper.go             wallpaper spec
    config_test.go
    wallpaper_test.go
  input/
    keys.go                  stdin -> Key/Mouse events, chord normalisation
    keys_test.go
  mux/
    mux.go                   types, construction, main event loop
    layout.go                columns/panes/workspaces, scrolling geometry, animation
    actions.go               what each bound action does
    keyboard.go              key dispatch (shortcuts, help, overview, prefix)
    mouse.go                 clicks, drag selection, wheel, forwarding to apps
    render.go                frame composition, overview thumbnails
    bar.go                   status bar + widgets
    help.go                  scrollable help overlay
    selection.go             selection model, text extraction, copy
    scrollback.go            logical rows (history + screen), viewport anchoring
    copymode.go              keyboard copy mode
    wallpaper.go             background program
    control.go               commands from outside (open, workspace, ...)
    control_test.go
  pane/
    pane.go                  one PTY + its emulator
    pane_test.go
  screen/
    screen.go                cell grid + SGR diffing renderer
    screen_test.go
  session/
    server.go                persistent session daemon
    client.go                attaching client + EnsureServer + List + Control
    proto.go                 framed unix-socket wire protocol
    session_test.go
  stats/
    stats.go                 /proc & /sys sampling + custom command widgets
  clipboard/
    clipboard.go             OSC 52 + native clipboard tools
  width/
    width.go                 display width of text (CJK / emoji take 2 cells)
    width_test.go
  vt/
    ...                      terminal emulator (vendored vt10x fork, MIT)
go.mod
go.sum
Makefile
README.md
```

Build: `go build -o ananta ./cmd/ananta` (Go 1.22+).
Modules: `creack/pty`, `BurntSushi/toml`, `golang.org/x/term`.

---

## 2. Process Shapes

### Attach mode (default)

```
ananta client   <== unix socket, framed messages ==>   ananta server
 relays keystrokes   MsgInput  ->         <- MsgOutput   owns panes, mux, emulators
 draws frames        MsgResize ->         <- MsgSetup    setsid, survives SIGHUP
                                                     <- MsgEnded
```

- **Client**: raw terminal, relays keystrokes in, draws frames out. One attached client per session at a time; a second attach displaces the first.
- **Server**: owns the mux and every pane/PTY/emulator. Runs with `setsid`, survives terminal close and SSH drops.
- Socket: `$XDG_RUNTIME_DIR/ananta-<uid>/NAME.sock` (mode 0600). Logs: `NAME.log`.

### Local mode (`ananta local`)

Single process: stdin → input assembler → mux loop → terminal. No server, no persistence. Everything dies with the terminal.

---

## 3. Command-Line Interface (`cmd/ananta/main.go`)

`main()` parses flags (`-config`, `-s`, `-init`, `-print-config`) and one subcommand:

| Command | Behavior |
|---------|----------|
| (none) | Attach to session `main`, creating the server if needed |
| `new NAME` | Create session and attach (error if exists) |
| `attach NAME` | Attach to an existing session |
| `ls` | List live sessions (panes, attached/detached, PID) |
| `kill NAME` | End the session and every program in it |
| `local` | Run in this terminal only; nothing survives it |
| `init` | Write default `config.toml` + `wallpaper.toml` to `~/.config/ananta/` |
| `print-config` | Print embedded default config to stdout |
| `print-wallpaper` | Print default wallpaper TOML to stdout |
| `help` | Print usage + control usage |
| `server NAME` | Internal: spawned by `EnsureServer` to run the daemon |
| `open \| workspace \| focus \| width \| close \| send \| list \| action` | Control a running session |

### Control commands (run against a live session)

These commands act on a running session over the unix socket. Every control
command accepts `-s NAME` to pick the session; without it, the session you are
inside (`ANANTA_SESSION`/`ANANTA_PANE` env vars) is used, else `main`. Inside
`ananta local` there is no session, so `-s NAME` is required.

Flags may appear before or after the program argument. The program always goes
in **double quotes, as one argument**.

---

#### `open` (alias `run`)

Starts a program in a new pane.

```
ananta open "htop" -n 3 -w 0.33 -s test
ananta open "tail -f /var/log/syslog" --below
ananta open "make watch" -k
ananta open "htop" --focus -n 3
ananta open                                  # no program: a new shell
ananta open "nvim" -d ~/projects -n 2        # start in a directory
```

| Flag | Meaning |
|------|---------|
| `-n N` / `--workspace N` | Workspace number (1–9) to open in |
| `-w W` / `--width W` | Column width: `0.33`, `33%`, or `33` |
| `-d DIR` / `--dir DIR` | Working directory (default: where you ran the command) |
| `--below` | Stack the pane under the focused one instead of a new column |
| `--focus` | Switch the view to that workspace after opening |
| `-k` / `--keep-focus` | Do not move keyboard focus to the new pane |
| `-p ID` | Target pane ID (not normally used with `open`) |
| `-s NAME` | Session name |

Returns the new pane's ID; pass it to `-p ID` on later `width`, `send`, `close`.

---

#### `workspace`

Switches the view to a workspace.

```
ananta workspace 3
ananta workspace 0   # current workspace (no-op)
```

Workspace must be `1`–`9`; `0` means the current one.

---

#### `focus`

Moves focus between columns and panes.

```
ananta focus left
ananta focus right
ananta focus up
ananta focus down
```

Direction must be `left`, `right`, `up`, or `down`. Equivalent to pressing the
bound arrow/hjkl focus keys.

---

#### `width`

Resizes the focused column (or a targeted pane's column).

```
ananta width 50%
ananta width 0.33
ananta width 33
ananta width -p 5 33%      # resize column containing pane ID 5
```

Width is a fraction of the screen: `0.33`, `33%`, or bare `33` (treated as a
percentage when `> 1`). Range is 1%–100%. `-p ID` targets a specific pane;
otherwise the focused pane's column is resized.

---

#### `close`

Closes a pane.

```
ananta close
ananta close -p 5
```

`-p ID` targets a specific pane (get IDs from `ananta open` or `ananta list`);
otherwise the focused pane is closed. If the column empties, it is removed.

---

#### `send`

Types text into a pane's PTY.

```
ananta send "make test"
ananta send -e "make test"        # types text then presses Enter
ananta send -p 5 "hello world"    # target pane ID 5
ananta send -p 5 -e "ls -la\n"    # target pane + Enter
```

| Flag | Meaning |
|------|---------|
| `-e` / `--enter` | Press Enter after sending the text (appends `\r`) |
| `-p ID` | Target pane ID; otherwise the focused pane |

If `Enter` is set, the text runs immediately (useful for commands like `make
test`). Without `-e`, the text is typed but not executed.

---

#### `list`

Lists every pane in the session.

```
ananta list
```

Output table: `ID  WS  COL  WIDTH  PROGRAM` with `*` marking the focused pane.
Use the IDs with `-p ID` on `close`, `send`, and `width`.

---

#### `action`

Runs any bound action by name, exactly as if its key chord was pressed.

```
ananta action overview
ananta action goto_workspace 2
ananta action detach
ananta action toggle_bar
ananta action move_to_workspace 3
ananta action new_pane htop
```

The action name must exist in `internal/config/actions.go` (`ActionInfo`);
unknown names produce an error. Everything after the action name is passed as
its argument (e.g. `goto_workspace 2`, `new_pane htop`).

---

### Session targeting rules

1. If `-s NAME` is given → that session.
2. Else, if running inside a pane (`ANANTA_SESSION` set) → that session.
3. Else, if `ANANTA=1` (`ananta local` mode) → error: no session to control;
   use `-s NAME`.
4. Else → session `main`.

`ananta help` prints this control usage.

---

## 4. Session Layer (`internal/session/`)

### `proto.go` — Wire protocol

Every frame: `1 byte type | uint32 BE length | payload` (max 16 MiB).

| Type | Dir | Meaning |
|------|-----|---------|
| `H` | client→server | Hello: JSON `{Cols, Rows, Cwd}` |
| `I` | client→server | Raw terminal input bytes |
| `R` | client→server | Resize: JSON size |
| `K` | client→server | Kill session |
| `P` / `p` | both | Info request / response (`ananta ls`) |
| `S` | server→client | Setup: `setup + NUL + teardown` escape sequences |
| `O` | server→client | Bytes to paint to the terminal |
| `X` | server→client | "you were detached" (payload = reason) |
| `Q` | server→client | Session ended |

Attach sequence: `H → S → (I/O loop) → X or Q`. The server allows a single attached client; a second `H` displaces the first.

### `server.go` — The daemon

`Serve(cfg, name)`:
1. Validates the session name (`^[A-Za-z0-9._-]{1,48}$`).
2. Removes any stale socket, listens on unix socket (0600).
3. Creates a `Server` wrapping a `mux.Mux`, sets `OnDetach = detachCurrent`.
4. Ignores `SIGHUP`/`SIGPIPE`; handles `SIGTERM`/`SIGINT` → `requestStop()` → `m.Quit()`.
5. Waits for the first client `MsgHello` (60s timeout), then calls `m.Start()` and `m.Loop()`.
6. `handle(conn)` dispatches frame types:
   - `MsgInfo` → replies with `Info{Name, Panes, Attached, PID}` (`ananta ls`).
   - `MsgKill` → requests stop, closes connection.
   - `MsgCommand` → executes a control request on the mux goroutine, returns JSON `Response{Output, Error}`.
   - `MsgHello` → attaches the client.
7. `attach(conn, sz)`: swaps the current client (old client gets `MsgDetached` and is closed), sends setup sequences, records start directory, sets attached + size, then reads input frames in a loop until disconnect → `drop()` → `SetAttached(false)`.

### `client.go` — The attaching client

- `Exists(name)`: dial test to check if a session is live.
- `EnsureServer(name, cfgPath)`: if no server exists, forks `ananta server name` with `setsid`, redirects stdout/stderr to `NAME.log`, sets working dir to `/`, polls until the socket answers.
- `Attach(name)`: enters raw mode, sends `MsgHello`, reads `MsgSetup`, spawns goroutines for stdin→`MsgInput` and `SIGWINCH`→`MsgResize`, then loops reading `MsgOutput` (writes to stdout) until `MsgDetached` or `MsgEnded`.
- `Control(name, req)`: dials the session, sends JSON `mux.Request` as `MsgCommand`, waits for `MsgCommandResp`, returns output or error.
- `List()`: scans the socket directory, pings every socket, collects `Info`, removes stale sockets.

---

## 5. Multiplexer (`internal/mux/`)

All mutable layout/selection state is owned by a single **mux goroutine** (`Mux.Loop`). Other goroutines hand work in through `post` closures — they never write mux fields directly.

### `mux.go` — Types and main loop

Core types:

```go
type Column struct {
    Panes     []*pane.Pane
    Focus     int
    WFrac     float64 // target width, fraction of screen
    SavedFrac float64 // width to restore after un-maximize
    CurW      float64 // animated, displayed width in cells
}

type Workspace struct {
    Cols    []*Column
    Focus   int
    ViewX   float64 // animated viewport offset (unscaled cells)
    TargetX float64
}

type Mux struct {
    cfg       *config.Config
    ws        []*Workspace   // always 9
    cur       int            // current workspace index
    W, H      int            // terminal size
    scale     float64        // 1 = normal, OverviewScale = overview
    ovY       float64        // animated vertical offset across workspace rows
    overview  bool
    scrollMode string        // "fit" | "center"
    ...
}
```

`Loop(keys, post)` selects on:
- `keys` — decoded key/mouse batches → `handleKeys`
- `post` — closures from other goroutines (resize, attach, stats-triggered SetSize)
- `dirty` — nudge from pane I/O or stat change
- `exits` — dead panes → `removePane`
- ticker at `cfg.FPS` → `step()` (animation)

A render frame is emitted only when data changed or an animation is running; otherwise only bookkeeping happens.

### `layout.go` — Geometry, structure, scrolling

- `tileH()` / `barY()` / `tileTop()` — the status bar reserves one screen row.
- `targetW(c)` — column width in cells, clamped to `minColW = 12`.
- `paneHeight(total, n, i)` — divides tile height evenly among panes in a column.
- `detachAt(ci, pi)` — removes a pane from its column, drops the column if empty, fixes focus indices.
- `newColumnFor(ws, p, frac)` — inserts a new column to the right of the focused one.
- `spawn(cmdline)` / `spawnWith(cmdline, dir, frac)` — starts a pane PTY + emulator; inherits cwd from the focused pane or `startDir`.
- `newPane` / `newPaneBelow` / `stackBelow` — create a new column or stack under the focused pane.
- `findPane(p)` / `removePane(p)` — locate a pane and remove it; clears copy mode / selection if affected.
- `moveWorkspaceRel(d)` — cycles through visible workspaces.
- Scrolling geometry:
  - `geom()` — total width, focused column x and width.
  - `centerNow()` — target X to center the focused column.
  - `updateTarget()` — fit mode (scroll just enough to show column) vs center mode (always centered).
  - `effTarget()` — overview zoomed target.
  - `gap()` / `rowGeom()` — overview row height and pitch.
- `syncSizes()` — resizes PTYs to the **target** layout so apps reflow exactly once per resize; on-screen rectangles animate towards the target.
- `step(dt)` — exponential easing for scale, column widths, viewport X, and overview Y offset.

### `actions.go` — Bound actions

`run(a config.Action)` dispatches every action:

| Action | What it does |
|--------|--------------|
| `focus_left` / `focus_right` | step between columns |
| `focus_up` / `focus_down` | step between panes in a stack |
| `first_column` / `last_column` | jump to column edge |
| `move_left` / `move_right` | swap columns |
| `move_up` / `move_down` | swap panes within a column |
| `new_pane` | spawn a new column to the right |
| `new_pane_below` | stack a new pane under the focused one |
| `close_pane` | remove the focused pane |
| `stack_left` | stack the focused pane into the left column |
| `unstack` | pop the focused pane out into its own column |
| `grow` / `shrink` | widen / narrow by `resize_step` |
| `cycle_width` | cycle through `column_widths` presets |
| `maximize` | maximize to full screen, restore |
| `center_mode` | toggle scrolling: fit ↔ center |
| `overview` | toggle overview |
| `workspace_next` / `workspace_prev` | cycle visible workspaces |
| `goto_workspace N` | jump to workspace N |
| `move_to_workspace N` | move focused pane to workspace N |
| `copy_mode` | enter vim-style copy mode |
| `scroll_up` / `scroll_down` | scroll focused pane through history |
| `detach` | detach client; session keeps running |
| `toggle_bar` | show / hide status bar |
| `help` | toggle help overlay |
| `quit` | end the session |

### `keyboard.go` — Key dispatch

`handleKeys(ks)`:
1. Mouse events → `handleMouse` (if `cfg.Mouse`).
2. Clears the selection highlight on any key.
3. Copy mode active → `copyKey`.
4. Help overlay active → `helpKey`.
5. Prefix key pending → complete prefix or re-arm.
6. Keymap lookup → `m.run(action)`.
7. Overview active → `overviewKey` (h/j/k/l navigate columns/workspaces; enter/space zooms in).
8. Otherwise accumulate bytes → forward to the focused pane (`p.Anchored = false` on typing).

`translate(k)` converts arrow keys to SS3 form (`\x1bO[A-D]`) for apps in application-cursor mode.

### `mouse.go` — Mouse handling

- `hitAt(x, y)` / `hitFor(p)` — hit-test against drawn pane rectangles.
- `leftPress(ev)`:
  - Bar click → workspace tab switch (from `m.tabs`).
  - Overview click → focus the pane, zoom in, center it.
  - Pane click → focus it (or forward to the app if it has mouse mode).
  - Border click → focus only.
  - Otherwise → start a pane-local drag selection.
- `dragSelection(ev)` — update selection cursor; auto-scroll history at pane edges.
- `finishSelection()` — if movement occurred, `copyText(extract(sel))`.
- `handleWheel(ev)`:
  - Overview: vertical → workspace rows, horizontal → column focus.
  - Pane with mouse mode → forward raw scroll events.
  - Normal screen → scroll history (3 lines/tick).
  - Alternate screen without mouse → arrow keys (vim/less-style paging).
- `forwardMouse(h, ev)` — re-encodes SGR or legacy mouse reports relative to the pane and writes to its PTY.

### `control.go` — External control requests

`Request{Op, Command, Dir, Workspace, Width, Below, Focus, KeepFocus, Pane, Text, Enter, Direction, Action, Arg}` (JSON serializable).

`Control(r)`:
- `open` → `ctlOpen`: spawns a pane, places it as a new column or stacks below; honors `keep-focus`, `focus` (switch workspace); returns the new pane ID.
- `workspace` → sets `m.cur`.
- `focus` → runs `focus_<dir>` action.
- `width` → sets column `WFrac` by pane ID or focused pane.
- `close` → finds pane by ID (or focused), removes it.
- `send` → finds pane, writes text (+ optional `\r`) to its PTY.
- `list` → `ctlList`: table `ID WS COL WIDTH PROGRAM` with `*` on the focused pane.
- `action` → validates against `config.ActionInfo`, runs `m.run`.

`ParseWidth(s)` — parses `0.33`, `33%`, `33` into a fraction of screen width.

### `bar.go` — Status bar and widgets

`widget(name, snap, now)` renders one named widget into `[]seg`. Built-ins:
- `session` — session name.
- `workspaces` — clickable tabs (`tab: i`) for non-empty workspaces plus current.
- `columns` — focus position (`█`/`░` per column + `2/3`).
- `mode` — scroll depth (`↑N`), copy mode / prefix / overview badges, `centered`.
- `message` — transient notifications.
- `title` — focused pane title.
- `help` — chord bound to the help action.
- `cpu`, `mem`, `net`, `battery`, `load` — system stats (Linux `/proc`, `/sys`).
- `time`, `date`, `host`, `user` — clock and environment.

Custom widgets from `[bar.widgets.NAME]` run shell commands (via `stats.Custom`); first line of output is shown.

`drawBar()`: builds left/center/right groups, drops right widgets from outside-in on narrow screens, draws segments. `drawSegs()` records `tabHit{x0, x1, ws}` for clickable segments.

### `help.go` — Scrollable help overlay

Generated from `Config.HelpLines()` — the live keymap merged with `ActionInfo` descriptions. Sorted by action order, keys truncated to 34 cells. Scrollable with vim motions.

### `selection.go` — Mouse selection model

`Selection{P, AX, AY, CX, CY, Kind}` over logical rows (history first, then live screen). `extract()` pulls text from the emulator using row addressing. Selection is clipped to one pane — never borders, never neighbours.

### `scrollback.go` — Logical row addressing

Unifies the emulator's scrollback ring and live screen into a single logical-row address space used by selections and copy mode.

### `copymode.go` — Vim-style copy mode

Motions: `hjkl`, arrows, `0`, `^`, `$`, `end`, `w`, `b`, `e`, `W`, `B`, `E`, `f`, `F`, `t`, `T`, `;`, `,`, `g`, `G`, `{`, `}`, `H`, `M`, `L`, `ctrl+u`, `ctrl+d`, `ctrl+b`, `ctrl+f`, pgup/pgdn. Counts work (`3j`, `12w`, `5f,`).

Selections: `v` (char-wise), `V` (line-wise), `ctrl+v` (block), `o`/`O` (swap ends). Copy with `y` or `Enter`; `Y` forces line-wise.

### `wallpaper.go` — Background program

Runs a command behind panes (display-only, receives no input). Drawn wherever no pane covers: empty workspace, space beside a narrow column, every row of the overview. Configurable command, directory, dim, and auto-restart.

---

## 6. Config Layer (`internal/config/`)

### `config.go` — Loading and compilation

Embeds `default.toml` via `//go:embed`. `Load(path)` decodes defaults, overlays the user file (map values merge key-by-key), calls `compile()`, then `loadWallpaper(dir)`.

`Config` fields: `Leader`, `ExtendedKeys`, `Shell`, `Mouse`, `Animate`, `AnimationSpeed`, `FPS`, `ScrollMode`, `OverviewScale`, `BorderStyle`, `ColumnWidths`, `DefaultColumnWidth`, `ResizeStep`, `ScrollbackLines`, `AppClipboard`, `Prefix`, `Wallpaper`, `Bar`, `Theme`, `Keys`, `PrefixKeys`, plus compiled fields (`LeaderMask`, `Keymap`, `PrefixMap`, `Warnings`, `WallpaperSpec`).

`compile()`:
- Parses the leader modifier; bad leader warns and falls back to `alt`.
- Normalizes every chord via `input.NormalizeChord`.
- Validates action names against `ActionInfo` (unknown → warning).
- Clamps `FPS` (10–240), `OverviewScale` (0.15–0.6), `DefaultColumnWidth`, `ScrollbackLines` (≥0).
- Validates `bar.position` (top/bottom/hidden), `bar.mem_style` (used/percent/both), `bar.interval` (≥0.2).
- Warns on unknown bar widgets.

`UseExtendedKeys()` — requests the extended (CSI-u) keyboard protocol if the leader is `super`/`ctrl`, or any binding contains `super+`.

`TermModes()` — returns alt-screen/hide-cursor setup + teardown sequences, plus mouse reporting (`?1002h`/`?1006h`) and extended keys (`>5u`) when needed.

### `actions.go` — Action table

`ActionInfo[]` is the single source of truth (name + description), in help-screen order. `ActionIndex(name)`, `KeyFor(action)` (shortest bound chord), `HelpLines()` (builds help from the live keymap).

### `default.toml` — Embedded defaults

```toml
leader = "alt"
extended_keys = "auto"
shell = ""
mouse = true
animate = true
animation_speed = 18.0
fps = 60
scroll_mode = "fit"
overview_scale = 0.3
border_style = "single"
column_widths = [0.33, 0.5, 0.66, 1.0]
default_column_width = 0.5
resize_step = 0.05
app_clipboard = true
scrollback_lines = 10000
wallpaper = "default"
prefix = ""

[bar]
position = "bottom"
interval = 1.0
separator = " │ "
left   = ["session", "workspaces", "columns", "mode"]
center = ["message"]
right  = ["cpu", "mem", "net", "battery", "time", "help"]
time_format = "15:04"
date_format = "Jan 2"
mem_style = "used"

[theme]
border       = "#4c566a"
focus_border = "#88c0d0"
...
```

Full `[keys]` table: leader+h/j/k/l (focus), leader+H/J/K/L (move), leader+enter/s (new pane/stack below), leader+w (close), leader+,. (stack/unstack), leader/r/f/-/= (widths), leader+c (center_mode), leader+o (overview), leader+b (toggle_bar), leader/home/end (first/last column), leader+pagedown/pageup (workspace_next/prev), leader+1..9 (goto_workspace), leader+!..( (move_to_workspace), leader+/ (help), leader+Q (quit), alt+a (copy_mode), shift+pageup/down (scroll), leader+d (detach).

### `theme.go` / `wallpaper.go`

`ParseColor` accepts `#rrggbb`, 0–255 palette index, or `default`. Wallpaper spec: `command`, `dir`, `dim`, `restart`.

---

## 7. Input Layer (`internal/input/keys.go`)

Mod bits: `ModShift=1`, `ModAlt=2`, `ModCtrl=4`, `ModSuper=8`.

`Key{Name, Raw, Mouse}` — `Name` is a canonical chord like `alt+h`; `Raw` is the legacy byte sequence forwarded to a pane when the key isn't a shortcut.

- `Canon(mask, key)` — builds canonical chord string; modifier order: ctrl, alt, super, shift.
- `NormalizeChord(s, leader)` — expands `leader`; uppercase implies shift; aliases (`return`→`enter`, `pgup`→`pageup`, `bs`→`backspace`, etc.).
- `plainKey(b)` — decodes a single non-escape byte (ctrl chars, backspace, enter, tab, UTF-8).
- `csiName(params, final)` — decodes CSI sequences (arrows, home, end, F-keys).
- `csiUKey(params)` — decodes the extended keyboard protocol (`CSI code ; mods u`), builds both the canonical name and the legacy bytes a pane expects.
- `parseKeys(buf, partial)` — full parser; holds incomplete sequences as `rest`.
- `parseMouse(params, final)` — decodes SGR mouse reports (`<Btn;X;Y M/m`).
- `Assembler` — stream reassembler for chunked input (common over SSH). `loneEscDelay=40ms`, `partialDelay=400ms`.

---

## 8. App Layer (`internal/app/app.go`)

`Run(cfg)`:
1. Checks stdin is a terminal; gets terminal size.
2. `term.MakeRaw(fd)`; writes `cfg.TermModes()` setup sequences.
3. `mux.New(cfg, out)`; `m.SetSize(w, h)`; `m.Start()` (stats sampler + wallpaper).
4. Goroutines:
   - `readInput`: stdin → input assembler → `keys` channel; on EOF → `m.Quit`.
   - `SIGWINCH` → `m.SetSize(new w, h)`.
   - `SIGTERM`/`SIGHUP` → `m.Quit`.
5. `m.Loop(keys, post)` until quit, then cleanup (teardown + restore terminal).

---

## 9. Pane Layer (`internal/pane/pane.go`)

`Pane{ID, Name, Cmd, PTY, Term vt.Terminal, Cols, Rows, Dead, Anchored, Top}`.

- `New(opts)`: starts `shell` or `sh -c cmdline` on a PTY (`pty.StartWithSize`). Sets `TERM=xterm-256color`, `COLORTERM=truecolor`, `ANANTA=1`, plus `ANANTA_PANE`/`ANANTA_SESSION`. Reader goroutine pumps PTY output into the emulator and pokes `dirty`. On EOF → `exits` channel.
- `Resize(cols, rows)` — PTY size + emulator resize.
- `Close()` — SIGHUP to child, close PTY.
- `Cwd()` — `/proc/<pid>/cwd` readlink (Linux).
- `Write(b)` — bounded write (250ms deadline) to PTY so a blocked child can't freeze the mux.

---

## 10. Terminal Emulator (`internal/vt/`)

Vendored fork of vt10x (MIT, © 2013 James Gray; Go port of suckless st's state table). Glyph grid, attribute bit flags, cursor management, DEC private modes, OSC/D scanners, full line-graphics handling. Plus scrollback ring (`SetScrollbackLimit`, `ScrollbackLen`, `Row`, `ClearScrollback`) and logical-row addressing used by the mux for selections and copy mode.

---

## 11. Stats Layer (`internal/stats/stats.go`)

Samples `/proc` and `/sys` for cpu, mem, net, battery, load at `bar.interval`. Custom widgets run shell commands with per-interval timeouts. `Snapshot()` returns current values; nudges `m.dirty` on change so the bar repaints.

---

## 12. Clipboard Layer (`internal/clipboard/clipboard.go`)

OSC 52 encoding (`ESC ] 52;c;<base64> BEL`) plus native tool drivers (`wl-copy`, `xclip`, `xsel`, `termux-clipboard-set`, `pbcopy`, and read-back variants). Copy text goes: OSC 52 to the client terminal + native tool on the server host + session `paste` buffer.

---

## 13. Width Layer (`internal/width/width.go`)

Display width of text (CJK / emoji = 2 cells via `uniseg`). Used by the renderer and help truncation.

---

## 14. Screen Layer (`internal/screen/screen.go`)

Double-buffered cell grid. Each frame is composed into `cur`, diffed against `prev`, and changed cells are written with absolute cursor positioning + attribute runs. Frames are bounded by synchronized-update markers (`CSI ? 2026 h/l`) with blinking-cursor suppression, so no tears appear mid-frame. `Set`/`Text`/`Fill` primitives; no layout engine beyond a handful of numbers.

---

## 15. Complete Data Flow (Attach Mode)

```
User runs: ananta
   │
   ├─ main.go: default case → session.EnsureServer("main", cfgPath)
   │      forks "ananta server main" (setsid) if not running
   │
   ▼ server Serve():
   │  listen on unix socket, wait for MsgHello (60s timeout)
   │  m.Start() → m.Loop()
   │
   ▼ client Attach():
   │  dial socket, send MsgHello{Cols,Rows,Cwd}
   │  receive MsgSetup{setup\0 teardown}
   │  enter raw mode, spawn input + winch goroutines
   │  loop: read MsgOutput → write to stdout
   │
   ▼ User types:
   │  client stdin → MsgInput frames → server assembler → mux.keys
   │  Loop → handleKeys → action OR forward bytes to pane PTY
   │
   ▼ Pane output:
   │  PTY reader → emulator → dirty → Loop → render()
   │  clientWriter.Write → MsgOutput frames → client stdout
   │
   ▼ Resize:
   │  client SIGWINCH → MsgResize → server → post → m.SetSize
   │
   ▼ leader+d (detach):
   │  detach action → Server.detachCurrent() → SetAttached(false)
   │  client gets MsgDetached, exits. Panes keep running.
   │
   ▼ ananta again:
   │  new client displaces old → re-lays out for new size
```

---

## 16. Screen Layout (Per Frame)

- **Tile area** = full height minus one row for the status bar (if visible).
- **Workspace** = one strip of columns. There are always 9.
- **Column** = vertical stack of 1..N panes; every pane in a column has the same width.
- **Column width** = fraction of screen (`default_column_width`, default 0.5); proportions survive resizes.
- **Viewport** `ViewX` glides to show the focused column (`fit` mode) or keeps it centered (`center` mode).
- **Overview**: scale to `overview_scale` (default 0.3), stack workspaces vertically with a gap, show live thumbnails of real pane content (PTYs are not resized for it).
- **Status bar**: left / center / right widget groups; on narrow screens, right-hand widgets drop from the outside in.

---

## 17. How to Extend

### New action
1. Add `{Name, Desc}` to `ActionInfo` in `internal/config/actions.go`.
2. Add a `case` in `run` in `internal/mux/actions.go`.
3. Bind it in `internal/config/default.toml`.
The help overlay picks it up automatically.

### New bar widget
- Built-in: add a `case` in `internal/mux/bar.go:widget`.
- Custom: define `[bar.widgets.NAME]` in `config.toml` with `command`, `interval`, `prefix`, `color`; add the name to `left`/`center`/`right`.

### New key events
`internal/input/keys.go` is where raw terminal bytes become named chords.

### Clickable widget (not yet implemented)
`seg.tab >= 0` marks a clickable segment; only the `workspaces` widget uses it today. `leftPress` in `internal/mux/mouse.go` handles tab hits. Custom widgets have no click target yet.

---

## 18. Known Limitations

- No double-width glyph handling (CJK/emoji corrupt display).
- History is not re-wrapped on pane resize.
- One attached client per session (a second attach displaces the first).
- No bracketed paste emulation into panes.
- No insertion (IRM) mode in the emulator.
- No session state persistence (everything in memory; server restart starts fresh).
- No screenshot / dump mode.

---

## 19. Environment Variables

| Variable | Purpose |
|----------|---------|
| `XDG_CONFIG_HOME` | config directory (fallback `~/.config`) |
| `XDG_RUNTIME_DIR` | session sockets / logs (fallback temp dir) |
| `SHELL` | default pane shell |
| `TERM` / `ANANTA` | exported into panes (`TERM=xterm-256color`, `COLORTERM=truecolor`, `ANANTA=1`) |

---

## 20. Testing

```
make test     # go test ./...
make vet      # go vet ./...
make build    # go build -o ananta ./cmd/ananta
make install  # go install ./cmd/ananta
```

Test coverage focuses on key parsing (`input`), config normalisation, session protocol round-trip, detach keeps-panes-alive, and the vt scrollback ring.

---

*End of documentation.*
