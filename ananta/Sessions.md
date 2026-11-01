# Sessions

ananta separates the **server** (which owns every pane, PTY and emulator) from
the **client** (the thin program attached to your terminal). This is what
makes sessions persistent: killing the client terminal never touches your
programs.

## The lifecycle

```
$ ananta
  │
  ├─ no live session "main"?  fork itself with "server main" (Setsid, survives SIGHUP)
  │                           server writes socket XDG_RUNTIME_DIR/ananta-<uid>/main.sock
  ├─ client dials the socket, sends terminal size, gets setup sequences
  └─ loop: keystrokes -> server, drawn frames <- server

$ ananta         (from another terminal)
  │ takes over the session: the first client is told "another client attached"
```

* Exactly **one attached client per session at a time**. A new attach
  displaces the old one (which exits with a notice).
* While detached, panes keep running and their scrollback keeps filling;
  nothing is drawn.
* The server reads the config **when it starts**. After changing the config,
  restart the session (`ananta kill main && ananta`) to apply it.
* Panes opened later inherit the environment the *server* was launched with -
  as in tmux. If `$SSH_AUTH_SOCK` changed since, re-export it in the shell.

## CLI

```
ananta             attach to "main", creating it if needed
ananta -s NAME     attach to NAME
ananta new NAME    create NAME and attach (error if it already exists)
ananta attach NAME attach to an existing session
ananta ls          list live sessions
ananta kill NAME   end the session (and every program in it)
ananta local       run everything in this terminal; nothing survives it
ananta server NAME internal: used by `ananta` to spawn the daemon
```

`ananta ls` also pings every socket it finds and deletes the stale ones (from
a crashed server, when the socket no longer accepts a connection).

## Control a running session

Once a session is running, any shell or script can drive it with
`ananta <command> ...`. The program to run always goes in **double quotes,
as one argument**:

```
ananta open "htop" -n 3 -w 0.33 -s test   # htop, workspace 3, 33% wide, session "test"
ananta open "tail -f /var/log/syslog" --below
ananta open "make watch" -k                # -k / --keep-focus: don't move keyboard focus to it
ananta open "htop" --focus -n 3            # --focus: also switch the view to that workspace
ananta open                                # no program: a new shell
ananta workspace 3                         # switch the view to workspace 3
ananta focus left|right|up|down            # move focus
ananta width 50%                           # set the focused column's width
ananta send -e "make test"                 # type into the focused pane (-e presses Enter)
ananta close                               # close the focused pane
ananta list                                # list panes (id, workspace, column, width, program)
ananta action overview                     # run any action by name, as if its key was pressed
ananta action goto_workspace 2
```

Flags may go before or after the program. `-s NAME` picks the session;
otherwise it defaults to the session you are inside (every pane has
`ANANTA_SESSION` and `ANANTA_PANE` set), else `main`. Inside `ananta local`
there is no session, so name one with `-s`.

`-p ID` targets a specific pane (get ids from `ananta open` or `ananta
list`). It works on `close`, `send`, and `width`.

`ananta help` prints this usage.

## Files on disk

Everything lives under `$XDG_RUNTIME_DIR/ananta-<uid>/` (or the temp dir),
mode 0700:

| File | Purpose |
|---|---|
| `NAME.sock` | the unix socket clients dial (created 0600) |
| `NAME.log` | the server's stdout/stderr, for crash reports |

Session names may contain letters, digits, `.`, `_`, `-` (max 48 chars, see
`ValidName`).

## Ending things

* `leader+Q` inside the session - ends it; attached clients get "session ended".
* `ananta kill NAME` - from any terminal.
* `leader+d` / `alt+d` **only detaches**; the session keeps running and can
  be re-attached later.
