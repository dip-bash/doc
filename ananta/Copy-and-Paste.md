# Copy and Paste

ananta keeps one clipboard per session (the `clip` buffer in the mux).

## Getting text out

Any copy path (mouse drag, copy mode `y`, `Enter`) ends in `copyText`:

1. **OSC 52** - the escape sequence `ESC ] 52;c;<base64> BEL` is written to
   *your* terminal, so the clipboard lands wherever you actually sit - great
   over SSH, works out of the box with kitty, foot, wezterm, ghostty,
   alacritty, ...
2. **Native tool fallback**, in parallel: `wl-copy`, `xclip -selection
   clipboard`, `xsel`, `termux-clipboard-set`, `pbcopy` - the first one
   found in `$PATH` is used.
3. The text is kept in the session for `paste`, and the bar flashes
   `copied N chars`.

## Getting text in

`alt+p` (`paste` action):

1. If the session has a recorded `clip` it is written to the focused pane.
2. Otherwise a native tool is read back (`wl-paste`, `xclip -out`,
   `xsel --output`, `termux-clipboard-get`, `pbpaste`; empty output falls
   back with a `nothing to paste` nudge).

Line endings are converted to carriage returns (`\r`) so a multi-line paste
runs naturally through shells and editors, tmux-style.

```
mouse: drag inside a pane select+copy (release copies)
alt+y  -> paste mode 'y' ... copy
alt+p  -> paste into the focused pane
```

## Because the client terminal is involved

* OSC 52 is a **client-side** feature: it requires your terminal to accept
  clipboard requests. In constrained terminals it is often gated behind an
  option (`kitty`: `allow_remote_control`; `foot`: `osc52-*`).
* Native tools copy on the *server host* - where the panes run. Over SSH the
  combination matters: the OSC 52 path is the only one that reaches your
  physical machine; the `wl-copy` path copies on the remote.
* Bracketed paste mode is not emulated into panes yet - shells that expect
  it still work, they just don't highlight the paste.
