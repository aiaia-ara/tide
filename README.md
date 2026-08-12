# tIDE (tmux IDE)

A single shell script that opens a tmux session with your editor in one pane,
Claude Code in another, and a shell across the bottom.

```
┌────────────────────────┬──────────────────┐
│         nvim           │      claude      │
│                        │                  │
├────────────────────────┴──────────────────┤
│                  console                  │
└───────────────────────────────────────────┘
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
tide                      # nvim | claude side by side, console below, in the current directory
tide -d ~/projects/foo    # start somewhere else
tide -V -l 30%            # stacked instead, secondary pane gets 30%
tide -b 30%               # roomier console
tide -e 'nvim .' -c lazygit
tide -p .venv             # every pane inside the project's virtualenv
tide -p ~/.venvs/proj     # or one that lives somewhere else
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
| `-l SIZE` | Size of the split pane: `1%`–`99%`, or an absolute number of rows/columns | `50%` |
| `-b SIZE` | Height of the bottom console: `1%`–`99%`, or an absolute number of rows | `15%` |
| `-e EDITOR` | Command for the main pane | `nvim` |
| `-c COMMAND` | Command for the split pane | `claude` |
| `-p VENV` | Python virtualenv to launch every pane in | |
| `-h` | Show help | |

`-l` sizes the *split* pane, not the editor, so `-l 70%` gives 70% to `claude`
and the remaining 30% to `nvim`. `-b` is measured against the whole window, and
the console spans its full width whether you use `-H` or `-V`.

`-e` and `-c` take a shell command, not just a program name, which is why
`-e 'nvim .'` works. Quotes inside the value have to balance.

`-p` takes the virtualenv directory, not the `activate` script inside it. A
relative path is resolved against `-d`, so `tide -d ~/proj -p .venv` means
`~/proj/.venv` — but the virtualenv does not have to live next to your code, and
`-p ~/.venvs/thing` works just as well. tide checks for `VENV/bin/activate` and
refuses to start if it is not there, so a typo fails immediately rather than
three panes later.

## Behaviour worth knowing

**Panes stay open when their program exits.** Quitting Neovim drops that pane
into your shell rather than closing it, the same as if you had opened the pane
and run the program yourself. Exit the shell (`Ctrl+D` or `exit`) to actually
close the pane. The console pane is already just your shell, so one `Ctrl+D`
closes it.

**Panes are plain shells, not login shells.** Left alone, tmux starts a pane that
has no command of its own as a *login* shell, which on macOS re-runs
`/etc/zprofile` and therefore `path_helper`, rebuilding `PATH` with the system
directories first. tide sets tmux's `default-command`, so the console continues the
shell you launched tide from instead of repeating login setup: your `~/.zprofile`
runs once, in your terminal, rather than once in every pane. It also makes the
console consistent with the other two panes, whose shells were already plain. This
applies whether or not you use `-p`.

**`-p` has no `deactivate`.** `deactivate` is a shell function, and every tide pane
ends by replacing its shell — which functions do not survive. Leave the virtualenv
by leaving the session. The virtualenv marker in your prompt still works, since
prompts read `$VIRTUAL_ENV`, which is set.

**`-p` on an already-running session asks first.** Attaching to an existing session
re-runs none of the setup, so `-p` cannot affect its panes. Rather than appear to do
nothing, tide says so and asks whether to attach anyway.

**Using uv?** `uv run` prefers the project's own `.venv` over whatever `$VIRTUAL_ENV`
says, unless you pass `--active`. Point `-p` at the project's `.venv` and the two
never disagree.

**It cannot be run from inside tmux.** Launch it from a plain terminal. Running
it from within an existing tmux session fails with `sessions should be nested
with care, unset $TMUX to force`. See
[issue #1](https://github.com/aiaia-ara/tide/issues/1) for the fix under
consideration.

## Why it is built the way it is

The script does everything in one tmux invocation, and deliberately builds the
layout *before* launching either program: `new-session`, then both
`split-window`s, then `respawn-pane` into each finished pane.

That ordering is load-bearing. Neovim 0.12 sizes its UI once at startup and can
miss resize events that arrive while it is still initialising, since `vim.pack`
runs synchronously during startup. Launching it into a pane that already has its
final size means it is never resized afterwards, so it cannot end up permanently
rendered at the wrong dimensions. The console split has to come before the
respawns for the same reason — being full width, it shortens both of the panes
above it.

The console itself needs no `respawn-pane`: `split-window` already starts an
interactive shell in the start directory, which is all a console is.

Pane targets are `{top-left}` for the editor and a direction relative to it
(`{right-of}` or `{down-of}`) for the split pane. The absolute tokens the script
used when there were only two panes no longer work — with three panes open,
`{top}` matches whichever of the two upper panes tmux reaches first, which is not
necessarily the editor.
