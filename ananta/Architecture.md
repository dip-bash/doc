# Architecture

A tour of the source. ananta is ~6.8k lines of Go, no external dependencies
beyond `creack/pty`, `BurntSushi/toml` and `golang.org/x/term`.

## Process shapes

```
attach (default):
  ┌────────────────────┐    unix socket, framed messages
  │  ananta (client)   │<--------------------------------> ┌─────────────────────┐
  │  relays keystrokes │  MsgInput ->         <- MsgOutput │ ananta server/Setsid│
  │  draws frames      │  MsgResize ->         <- MsgSetup │ ends every PTY/mux  │
  └────────────────────┘                       <- MsgEnded └─────────────────────┘

local (ananta local):
  ┌────────────────────────────────────────────┐
  │ one process: input -> mux loop -> terminal │
  └────────────────────────────────────────────┘
```

## Package map

| Package | Files | Role |
|---|---|---|
| `cmd/ananta` | `main.go` | flags, command dispatch (new/attach/ls/kill/local/server/init) |
| `internal/session` | `proto.go`, `server.go`, `client.go` | persistent sessions: the framed unix-socket protocol, the daemon, the attaching client |
| `internal/app` | `app.go` | "local" mode: raw input loop around the mux |
| `internal/config` | `config.go`, `default.toml`, `actions.go`, `theme.go` | TOML loading, keymap compilation, action table, themes |
| `internal/input` | `keys.go` | stdin -> `Key`/`Mouse` events, chord normalisation |
| `internal/screen` | `screen.go` | cell grid + SGR diffing renderer with synchronized-update markers |
| `internal/vt` | `state.go`, `parse.go`, `csi.go`, `str.go`, ... | terminal emulator (vendored vt10x fork, MIT) + scrollback |
| `internal/pane` | `pane.go` | one PTY, one emulator, one reader goroutine |
| `internal/mux` | `mux.go` + friends | the multiplexer (all user-visible behaviour) |
| `internal/stats` | `stats.go` | `/proc` & `/sys` sampling + custom command widgets |
| `internal/clipboard` | `clipboard.go` | OSC 52 pack + native tool drivers |
| `internal/width` | `width.go` | display width of text (CJK / emoji take 2 cells) |

The `internal/mux` sub-map is documented at length in the package comment
(`mux.go`, lines 1-16) and in the README - the same list is also the
recommended reading order for a new contributor:

```
mux.go layout.go actions.go keyboard.go mouse.go render.go bar.go
help.go selection.go copymode.go scrollback.go wallpaper.go control.go
```

## The main loop (mux)

`Mux.Loop` is the single owner of all mutable layout/selection state. It
selects on:

* `keys` - decoded key/mouse batches,
* `post` - closures other goroutines run on this goroutine (attach/resize/
  stats triggered SetSize, count, ...),
* `dirty` - nudges (I/O in pane, stat change),
* `exits` - dead panes to remove,
* a ticker at `fps`.

A render frame is emitted when data changed *or* an animation is still
running, never otherwise; everything that already looks the same costs one
tick of `step()` bookkeeping.

## Thread-safety scheme

* One mutex in the world of note: each pane's **VT state** (`State.mu`).
  The PTY reader goroutine holds it while parsing chunk; the render goroutine
  takes it briefly per pane (`drawPane`, `viewOf`, `paneMode`).
* Everything else is owned by the loop goroutine. The server hands state
  changes in through `post`, never by writing fields.
* The session server (client connections) blocks in `readFrame` and sends
  under a per-conn `wmu` (protecting the writer between attach/despawn).

## The wire protocol (`internal/session/proto.go`)

Every frame is `1 byte type + uint32 BE length + payload` (payload limit
16 MiB):

| Type | Dir | Meaning |
|---|---|---|
| `H` | client->server | hello: JSON `{Cols,Rows}` |
| `I` | client->server | raw terminal bytes |
| `R` | client->server | resize: JSON size |
| `K` | client->server | kill session |
| `P` / `p` | both | info request / response (used by `ananta ls`) |
| `S` | server->client | setup: `setup + NUL + teardown` escape sequences |
| `O` | server->client | bytes to paint |
| `X` | server->client | "you were detached" (payload = reason) |
| `Q` | server->client | session ended |

Attach sequence: `H` -> `S` -> (I/O loop) -> `X` or `Q`. The server allows a
single attached client; a second `H` displaces the first.

## Terminal emulator (`internal/vt`)

A vendored fork of vt10x (MIT, © 2013 James Gray; a Go port of suckless st's
state table): glyph grid line-by-line, attr bit flags, cursor management,
DEC private modes, OSC/D scanners, full line-graphics handling. On top of
that:

* `pushHistory` recording scroll-up of the primary screen,
* `SetScrollbackLimit`/`ScrollbackLen`/`Row`/`ClearScrollback` - the ring
  used by the mux above it,
* logical-row addressing (`scrollback.go` in the mux): history first, then
  the live screen, unifying screen + scrollback for selections and copy mode.

## Renderer (`internal/screen`)

A double-buffered cell grid: each frame is composed into `cur`, diffed
against `prev`, and changed cells are written with absolute cursor
positioning + attribute runs. Frames are bounded by **synchronized update
markers** (`CSI ? 2026 h/l`) and blinking cursor suppression, so no tears
appear mid-frame. The bar / panes draw via lib-less `Set`/`Text`/`Fill`
calls; there is no layout engine beyond a handful of numbers.

## Extending

* **New action**: add to `ActionInfo` (config/actions.go) + a case in
  `mux/actions.go:run`, then bind it in `default.toml`; the help overlay
  picks it up automatically.
* **New bar widget**: a `case` in `bar.go:widget` + name it anywhere in
  left/center/right; or just point a custom widget at a shell command.
* **New key events**: `internal/input/keys.go` is where raw terminal bytes
  become named chords.

## Testing

```
make test     # go test ./...
make vet      # go vet ./...
```

Test coverage is thin at the moment and focused on: key parsing (`input`),
config normalisation, the session protocol round-trip and the detach
keeps-panes-alive behaviour, and the vt scrollback ring.
