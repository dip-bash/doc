# FAQ

**Is ananta a tmux replacement?**
Partly. Sessions (detach/reattach, `ls`, `kill`, server/client split) mirror
tmux. Layouts don't: there is one endless scrolling strip per workspace, not
a fixed split grid, and there are no "windows" you tile by ratios - you stack
panes into columns and the strip scrolls.

**Why "columns of panes" instead of arbitrary splits?**
With scrolling-first design (like niri/PaperWM) every column is a fixed
fraction of the screen width; that makes resize behavior and animation
predictable. Vertical stacking inside a column gives you arbitrary counts.

**Can I use it over SSH?**
Yes - that's what the session model is for. `ananta ls` shows all your
sessions on the remote box, detach with `alt+d`, come back later. The OSC 52
clipboard even crosses the SSH hop (see
[Copy and Paste](Copy-and-Paste)).

**How many clients can be attached?**
One at a time. A second attach "steals" the session and the first client
exits with a notice - handy for moving between machines.

**Why is my pane dim/cursor-less when the terminal disconnects badly?**
The client terminal is exactly at the dimension it reported; a broken-resize
(SSH window shrinking races) makes the *server* resize all panes. Check
[Troubleshooting](Troubleshooting).

**Does the mouse thing respect my app?**
Yes: apps that enable mouse reporting get the raw events; ananta's own
drag-selection/scrolling only happen for apps that did not ask for the mouse.
Hold `shift` to force ananta's selection any time.

**What does the leader key get typed through?**
A chord that matches nothing in `[keys]` (with no `prefix` set) is forwarded
verbatim. Under `prefix`, an unmatched key after the prefix is also typed
through; pressing the prefix chord while pending sends the literal key.

**Why does scrolling the wheel send arrows to my pager?**
Full-screen apps that disabled mouse mode get arrow keys for the wheel,
like a regular terminal - the alternates' own history does not exist for
them, see [Scrollback](Scrollback).

**Where is the state? Is there any?**
There is no directory of persisted sessions to inspect. A session is only
the daemon process plus its unix socket; everything (panes, emulators) is
in memory. Killing the machine loses it, like tmux without plugins.

**Something in /proc and /sys concerns?**
Stat widget sampling happens server-side with its own goroutines; errors
(disks full etc.) are non-fatal.

**What's a good minimal config?**

```toml
leader = "alt"
[bar]
right = ["time"]
```

returns everything to defaults.

**Where does the name come from?**
"Ananta" (Sanskrit) means "infinite/endless", like the horizontal strip of
columns.
