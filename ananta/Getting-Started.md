# Getting Started

## 1. Build and install

```
go mod tidy
make build                 # produces ./ananta
make install               # or: go install ./cmd/ananta  ->  $(go env GOPATH)/bin/ananta
```

There are no runtime dependencies. Scrollbar widgets (cpu/mem/net/battery)
read `/proc` and `/sys` on Linux and simply hide on platforms where those
files don't exist.

## 2. Write your config

`./ananta -init` copies the embedded default config to
`~/.config/ananta/config.toml` and exits. `./ananta -print-config` prints the
same defaults to stdout.

You don't need a config file at all - the embedded defaults are always merged
under your file, so your `config.toml` may be 3 lines.

## 3. Start using it

```
ananta          # attach to session "main", create the server if needed
```

Within ananta:

| Chord (leader = `alt`) | What happens |
|---|---|
| `alt+enter` | open a new column to the right |
| `alt+s` | stack a new pane under the focused one |
| `alt+h l j k` (or arrows) | focus left / right / down / up |
| `alt+1`..`alt+9` | switch workspace |
| `alt+w` | close the pane |
| `alt+d` | **detach** - the session keeps running |
| `alt+/` | help overlay (shows *your* bindings) |
| `alt+o` | overview (bird's-eye view) |
| `alt+a` | vim-style copy mode |
| `alt+Q` | end the session for good |

## 4. Detach and reattach

`alt+d` detaches your client; the server keeps running and your panes keep
producing output. Close the terminal, drop SSH, whatever - come back with:

```
ananta            # attaches to "main"
ananta attach work
```

Reattaching re-lays everything out for the new terminal size, and the pane
content that scrolled by while you were away is all still in scrollback.

## 5. Sessions CLI

```
ananta ls                 # list live sessions (panes, attached/detached, pid)
ananta -s work            # like above but session "work"
ananta new build          # create + attach (errors if "build" exists)
ananta attach build       # attach to an existing session
ananta kill build         # end the session and every program in it
ananta local              # run in THIS terminal, no background server
```

The `local` mode is what tmux calls "without a server": when the terminal
goes, everything in it goes too. It is useful for quick throwaway work.

Next up: [Sessions](Sessions) explains the server model in more detail,
[Workspaces and Layout](Workspaces-and-Layout) shows the tiling model, and
[Key Bindings](Key-Bindings) is the complete cheat sheet.
