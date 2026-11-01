# Copy Mode

`alt+a` (`copy_mode`) turns the focused pane into a vim-like text browser
with a cursor, working over the **entire scrollback plus screen** - the same
"logical rows" address space the mouse selection uses. Press the same key
again (or `Esc` / `q`) to leave with nothing copied.

## Motions

| Keys | Motion |
|---|---|
| `h j k l`, arrows | move by cells (with `count`) |
| `0` | start of line (`^` = first non-blank, `$`/`end` = last non-blank) |
| `w b e`, `W B E` | word motions, treating each row as text ending with newline |
| `f F t T` | find character (repeatable with `;` and `,`) |
| `g` `G` | top of history / last non-blank row |
| `{` `}` | previous / next paragraph |
| `H M L` | top / middle / last of the visible screen |
| `ctrl+u ctrl+d` | half page (with count) |
| `ctrl+b ctrl+f`, pgup/pgdn | page |

Counts work: `3j`, `12w`, `5f,`. `end`-style `Home` works too.

## Panning

The pane auto-scrolls to keep the copy cursor visible
(`cmEnsureVisible`); anchoring works exactly like manual scrolling and is
cleared when you leave the mode into the live screen.

## Selections

| Keys | Selection |
|---|---|
| `v` | character-wise (`VISUAL`) |
| `V` | line-wise (`V-LINE`) |
| `ctrl+v` | block selection (`V-BLOCK`) |
| `o` / `O` | swap cursor / anchor ends |

With an active selection the [bar](Status-Bar) shows the kind; pressing the
selection key again changes/turns it off.

## Copying

* `y` - copies the selection (or the cursor row as line-wise if none).
* `Enter` - same, and leaves copy mode.
* `Y` - force line-wise copy of the selection.

Copied text goes through the normal pipeline: OSC 52 to the client terminal +
native tools (`wl-copy`/`xclip`/`xsel`/`termux-clipboard-set`/`pbcopy`) and
fills the in-session `paste` buffer (`alt+p`). See
[Copy and Paste](Copy-and-Paste).

## Notes and gotchas

* The mouse is keyboard-only here: mouse events are ignored in copy mode.
* Rows are reconstructed from emulator glyphs; some spacing oddities can
  appear where the app filled the cell with double-width characters (not
  supported yet).
* `alt+a` was chosen because tmux' classic `alt+[` collides with the escape
  reporting of emulated apps.
