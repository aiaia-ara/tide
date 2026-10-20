# How tide is built, and why

This is for anyone changing `bin/tide`. The README covers using tide; this covers
the decisions behind its launch, and the tmux and macOS behaviour they rest on.
Several of those behaviours are undocumented or contradict the tmux manual, so
each one says which versions it was last checked on.

Keep this file current: change it in the same commit as anything it describes.

## What tide builds

One tmux call builds this and attaches your terminal to it:

```
┌──────────────────────┬──────────────────────┐
│ editor (nvim)        │ side (claude)        │
├──────────────────────┴──────────────────────┤
│ console (your shell)                        │
└─────────────────────────────────────────────┘
```

Everything before that call is ordinary shell: reading options, checking sizes,
the virtualenv and the tmux version, and attaching if the session already exists.
The constraints are all in the launch.

## The launch, step by step

The commands are joined with `\;`, so tmux runs them in order as one chain.

| # | Command | What it does |
|---|---|---|
| 1 | `new-session` (attached) | Creates the session at your terminal's size. Its window is a throwaway. |
| 2 | `set-option default-command` | Makes a pane started without a command run a plain shell, plus the `-p` prefix. |
| 3 | `new-window -b` | Creates the real window in front of the throwaway, so it gets the usual index. Its pane is the console. |
| 4 | `split-window -b -v -E` | Adds an empty pane above the console for the editor: no program, no pty. |
| 5 | `resize-pane -t {down-of} -y` | Sets the console's height. |
| 6 | `split-window -d` | Splits the editor's pane and starts the side command in the new pane. The layout is now final. |
| 7 | `send-keys -R -t {top-left}` | Resets the empty pane's terminal state. |
| 8 | `respawn-pane -t {top-left}` | Starts the editor in the empty pane. |
| 9 | `kill-window -a` | Deletes every window except the current one, which is the throwaway. |

Every pane that runs a program needs a pseudo-terminal (pty) from the kernel.
Steps 1, 3, 6 and 8 each open one. Step 9 closes one, the throwaway's, and nothing
opens after it.

## Rules the launch keeps

**1. Real terminal size.** `new-session` runs first and attached, so tmux takes the
session's size directly from your terminal.

**2. Plain shells, not login shells.** With `default-command` empty, tmux starts a
pane's shell as a login shell. On macOS that runs `/etc/zprofile`, whose
`path_helper` rebuilds PATH with the system directories first, demoting anything a
parent put in front, a virtualenv included. tide sets `default-command` to
`exec "$SHELL"`, so panes run `~/.zshrc` but not the login files, and keep the PATH
they were given. It has to be set before step 3, since the console reads it.

**3. No program starts before its pane has its final size.** Neovim 0.12 sizes its
UI once at startup and can miss a resize that arrives while it is still loading
(`vim.pack` runs synchronously during startup), leaving it drawn at the wrong size.
So the console, which shortens both panes above it, is sized in step 5, and step 6
is the last change to the layout. The side command starts in step 6 and the editor
in step 8. The rule exists for Neovim, but the side command is held to it too.

**4. Nothing is killed to make room for a program.** No step replaces a running
program, because that is what set off the pty race (below). It is why the editor's
pane starts empty instead of holding a placeholder shell.

**5. Every pane starts with the same PATH.** tmux gives the pane `new-session`
creates a different PATH from the panes created after it (see the PATH rule
below). So none of the real panes is the `new-session` pane: they are all built in
the second window, and the first is deleted.

**6. Unambiguous pane targets.** With three panes open, `{top}` matches whichever
upper pane tmux reaches first, which is not necessarily the editor, and tmux gives
no error. The editor is addressed as `{top-left}`. `{down-of}` in step 5 can only be
the console, since just two panes exist at that point.

**7. The cursor starts in the editor.** Step 4 makes the empty pane the active one,
and step 6 uses `-d` so the new side pane does not take focus.

**8. `-p` travels in the command strings.** The virtualenv is written into each
pane's command and into `default-command`, which also covers panes you open later.
`$PATH` must reach the pane unexpanded, so it resolves against the pane's own PATH;
the format string is single-quoted so that cannot be lost. The paths go in with
`%q`, because the pane's shell parses the string again as code.

**9. The editor's pane is reset before the editor starts.** See "Empty panes"
below.

## tmux and macOS behaviour tide depends on

### The PATH rule (tmux)

The manual says a pane's environment comes from the session environment. For PATH
there is an exception, in `spawn.c`:

```c
if (c != NULL && c->session == NULL) { /* only unattached clients */
        ee = environ_find(c->environ, "PATH");
        if (ee != NULL)
                environ_set(child, "PATH", 0, "%s", ee->value);
}
```

A pane created by a client that is not attached yet gets that client's PATH,
overriding the session environment and even `-e PATH=`. Once the client has
attached, panes get the session environment's PATH, which is the one the tmux
server started with, and there `-e PATH=` and `set-environment PATH` do apply.

tide's `new-session` runs before its client attaches, so that pane gets your
terminal's PATH. Every later command runs after the attach, so its panes get the
server's. The two are the same when tide starts the server itself, and differ when
a server was already running and your PATH has changed since it started.

Checked on tmux 3.8, macOS 27.0.1 (October 2026), against a server that was
already running: `new-session`'s pane got the terminal's PATH; later panes got
the server's, or the `-e` or `set-environment` value when one was given; in a
detached chain every pane got the terminal's PATH, `-e` included. The `spawn.c`
code is the same in 3.7b. Notes from August 2026 said `-e PATH=` and
`set-environment` were ignored there; that was most likely measured in a detached
chain, but has not been re-checked on 3.7b.

To re-check: start a server whose PATH contains a marker; then, from a terminal
whose PATH contains a different marker, launch tide against that server and read
each pane's starting PATH (see "Testing" below).

### A failed command drops the rest of the chain (tmux)

When a command in a `\;` chain fails, tmux skips every command after it
(`cmdq_remove_group` in `cmd-queue.c`). A failure partway through the launch
leaves later panes without their programs, and the throwaway window still open.

Checked on tmux 3.8 (October 2026). To re-check:

```sh
tmux -L check -f /dev/null new-session -d \; select-window -t :9 \; display-message -p 'third command ran'
tmux -L check kill-server
```

It prints `can't find window: 9` and never `third command ran`.

### The pty race (macOS)

macOS keeps ptys in a table that grows 16 slots at a time and never shrinks until
reboot (`bsd/kern/tty_ptmx.c`). When every slot is taken, a new pty is given the
number one past the end, and the table is only grown later in the open, if no slot
has been freed by then. If another pty is released in between, the table is not
grown, the number points past the end, and the open fails with "Device not
configured".

`respawn-pane -k` releases a pane's pty and opens a new one microseconds apart, so
it supplies that release itself. tide used to launch that way. Whenever the table
happened to be exactly full (16, 32, 48… ptys in use across the whole machine),
the editor's respawn failed: its pane stayed blank, `tmux show-messages` logged
`Respawn pane failed: fork failed: Device not configured`, and the side command
never started, because the rest of the chain was dropped. Opening a pty while none
is being released works at any fill level; the table just grows.

Checked: source read from xnu-12377 (macOS 26), reproduced on macOS 27.0.1
(October 2026). Closing a pty and opening another within about 100µs at an exactly
full table failed 58 times in 60, and 0 times in 90 with one slot free. The old
launch failed 5 times in 5 with the table full at the editor's respawn. The
current launch started normally with each of its four pty opens placed on a full
table.

### Empty panes (tmux)

`split-window -E` creates a pane with no program and no pty; `new-session` cannot.
tmux puts an empty pane into a mode where a bare line feed also returns the cursor
to column 0 (`MODE_CRLF`, set in `spawn.c`), and `respawn-pane` keeps that mode
(`screen_reinit` in `screen.c`), so a program started there would inherit it.
`send-keys -R` resets the pane's terminal state, which clears it.

Checked on tmux 3.8 (October 2026): in an empty pane that had been respawned, a
line feed returned to column 0 without the reset, and kept its column with it.
Neovim 0.12.5 drew the same screen either way; the reset is there because `-e`
accepts any editor.

### The 3.7b floor (tmux)

tmux 3.7a never redraws content drawn inside a synchronized update, which is how
Neovim draws its frames, so the editor pane stays blank. 3.7b fixed it ("Fix so
that the end of a synchronized update again triggers a redraw"), and tide refuses
anything older. Everything the launch uses exists in 3.7b: `split-window -E` and
`resize-pane -y` with a percentage (checked in the 3.7b source), and
`send-keys -R` (since 1.6), `new-window -b` (since 3.2) and `kill-window -a`
(since 1.7). The launch has not been run on 3.7b.

## Testing

- **Use a throwaway server.** Give every test command its own socket
  (`tmux -L <name>`), so a mistake cannot touch the server you work in. To run tide
  itself against it, put a script named `tmux` first on PATH that runs
  `tmux -L <name> "$@"`.
- **Test against a server that already exists.** A server the test starts inherits
  the test's environment, which hides PATH bugs.
- **Reproduce tide's shape.** The PATH rule depends on an attached `new-session`
  followed by chained commands. A detached session, or separate tmux calls, behave
  differently.
- **Read a pane's starting PATH with `ps -E -ww -o command= -p <pane pid>`.** It
  prints the command before the environment, so take the last `PATH=` word, not a
  `PATH=` inside the command (as `-p` writes). It cannot read protected system
  programs such as `/bin/sleep`: run `sleep 600; true`, so a shell is the pane's
  process.

## Rejected alternatives

- **Placeholder shells replaced with `respawn-pane -k`.** How tide used to launch.
  It sets off the pty race.
- **The console as `new-session`'s pane.** No throwaway window needed, but the
  console then gets your terminal's PATH while the editor and side get the server's.
- **Writing the terminal's PATH into every pane's command.** Every pane agrees, but
  on a different PATH from today's. Launching from a terminal with a virtualenv
  active, without `-p`, would put its `bin` on PATH with no `VIRTUAL_ENV` set.
- **Typing the commands into running shells with `send-keys`.** The keystrokes race
  the shell's startup, land in its history, can be changed by an alias, and tmux
  cannot report a failure.
- **Passing the virtualenv with `-e` or `set-environment`.** `new-session -e PATH=`
  is overridden. On the later commands both work, but only because of the
  undocumented PATH rule; `-e` misses panes you open later, and
  `set-environment -g` changes every session on the server.
- **Exporting the virtualenv before `exec tmux`.** Reaches the panes only when tide
  happens to start the tmux server.
- **Login shells in panes.** `path_helper` reorders PATH, and `~/.zprofile` runs
  again in every pane.
- **Starting the programs before the layout is final.** Neovim can stay drawn at
  the wrong size (rule 3).
