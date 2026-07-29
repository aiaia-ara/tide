# tIDE (tmux IDE)

A single shell script that opens a tmux session with your editor in one pane and
Claude Code in the other.

```
┌────────────────────────┬──────────────────┐
│                        │                  │
│         nvim           │      claude      │
│                        │                  │
└────────────────────────┴──────────────────┘
```

## Requirements

**tmux 3.7b or newer.** This is not optional and it is not a stylistic
preference. tmux 3.7a has a bug where content drawn inside a synchronized
update — which is how Neovim draws its frames — is never redrawn to the
terminal. The symptom is an editor pane that stays blank until something else
forces a repaint, with no error to explain it. Fixed upstream in 3.7b ("Fix so
that the end of a synchronized update again triggers a redraw").

```sh
tmux -V   # must report 3.7b or higher
```

Nothing in the script enforces this yet ([issue
#2](https://github.com/aiaia-ara/tide/issues/2)), so check it yourself if the
editor pane comes up blank. Also required: whichever editor and secondary command
you point it at — `nvim` and `claude` by default.

## Install

Put `bin/tide` on your `PATH`:

```sh
git clone https://github.com/aiaia-ara/tide.git
ln -s "$PWD/tide/bin/tide" ~/.local/bin/tide
```

## Usage

```sh
tide                      # nvim | claude, side by side, 50/50, in the current directory
tide -d ~/projects/foo    # start somewhere else
tide -V -l 30%            # stacked instead, secondary pane gets 30%
tide -e 'nvim .' -c lazygit
```

If a session with the given name already exists, `tide` attaches to it instead of
creating a second one.

| Flag | Meaning | Default |
| --- | --- | --- |
| `-s SESSION` | Session name | `tide` |
| `-w WINDOW` | Window name | `main` |
| `-d DIR` | Start directory | `.` |
| `-H` | Horizontal split, side by side | (default) |
| `-V` | Vertical split, stacked | |
| `-l SIZE` | Split size: `1%`–`99%`, or an absolute number of rows/columns | `50%` |
| `-e EDITOR` | Command for the main pane | `nvim` |
| `-c COMMAND` | Command for the split pane | `claude` |
| `-h` | Show help | |

`-e` and `-c` take a shell command, not just a program name, which is why
`-e 'nvim .'` works. Quotes inside the value have to balance.

## Behaviour worth knowing

**Panes stay open when their program exits.** Quitting Neovim drops that pane
into your shell rather than closing it, the same as if you had opened the pane
and run the program yourself. Exit the shell (`Ctrl+D` or `exit`) to actually
close the pane.

**It cannot be run from inside tmux.** Launch it from a plain terminal. Running
it from within an existing tmux session fails with `sessions should be nested
with care, unset $TMUX to force`. See
[issue #1](https://github.com/aiaia-ara/tide/issues/1) for the fix under
consideration.

## Why it is built the way it is

The script does everything in one tmux invocation, and deliberately builds the
layout *before* launching either program: `new-session`, then `split-window`,
then `respawn-pane` into each finished pane.

That ordering is load-bearing. Neovim 0.12 sizes its UI once at startup and can
miss resize events that arrive while it is still initialising, since `vim.pack`
runs synchronously during startup. Launching it into a pane that already has its
final size means it is never resized afterwards, so it cannot end up permanently
rendered at the wrong dimensions.
