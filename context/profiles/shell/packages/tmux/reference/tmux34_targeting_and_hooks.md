# tmux 3.4 platform facts

tmux 3.4 platform facts, each pinned empirically against tmux 3.4 (Ubuntu
noble), mostly via private-socket and execve probes: index targets silently
hit the wrong pane on a non-zero base-index (target %-pane-ids), pane hooks
live in global window scope (`show-hooks -wg`; bare `-g` under-displays),
remain-on-exit is a window option (`set -wg`), hook format variables expand
at execution time (`#{hook_session_name}` for destroy events), the C locale
drops the tab separator from `-F` format output, the first duplicate HOME
wins, and `show-options -gv` reports unset with exit 1 + empty stdout. None
of these is derivable from the tmux man page, and each one produced a real
misdiagnosis first.

## A non-zero base-index breaks index targets — never target pane indexes

A server configured with `pane-base-index 1` and `base-index 1` (check with
`tmux show-options -gv`) breaks any target of the form
`<session>.<pane-index>` that assumes 0:

- `tmux display-message -p -t <s>.0 …` **silently answers the ACTIVE
  pane** — no error, wrong pane. A guard reading
  `#{pane_current_command}` through an index target validates an
  unrelated pane.
- `tmux send-keys -t <s>.0 …` fails with `can't find pane: 0` (exit 1).

Correct pane addressing: resolve concrete %-pane-ids
(`tmux list-panes -t <s> -F '#{pane_pid} #{pane_id} #{pane_index}'`)
and target `%<id>` — base-index independent.

Two further traps, verified empirically:

- `list-panes -t <s>` WITHOUT `-s`/`-a` enumerates only the CURRENT
  window (pid-preference resolution degrades to that window for
  multi-window sessions).
- A dead remain-on-exit pane keeps its `#{pane_pid}`, reports `bash`
  to `#{pane_current_command}`, and swallows `send-keys` as an rc-0
  no-op.

Bite record: a resume executor hardcoded pane 0 — against a base-index-1
server it delivered ZERO resumes (send always failed) while its occupancy
guard validated the active pane; its test fake ignored the pane argument, so
tests were blind. Any tmux-targeting code — and any test fake for it — must
be pane-id based and pane-sensitive.

## Pane hooks live in global window scope

The `pane-died` and `pane-exited` hooks live in the **global window scope**.
`set-hook -g` arms them, but tmux scopes hooks by target object, and pane
hooks are window-level.

- Verify with `tmux show-hooks -wg`, or the targeted single-hook query
  `tmux show-hooks -g pane-died` (authoritative; resolves across
  scopes).
- A plain `tmux show-hooks -g` listing shows only
  `session-closed`/`session-created` and silently omits the pane hooks
  even when armed — reading that as "hooks disarmed" is a false alarm
  (hit twice in one investigation; the hooks were armed both times).

## remain-on-exit is a WINDOW option

`set -g remain-on-exit on` (session scope / table fallthrough) leaves
pre-existing windows at the default — panes die outright, and pane-died
(whose precondition is a retained dead pane) never fires. The working
form is `set -wg remain-on-exit on` (global window scope, consulted
LIVE by existing windows). This was the actual root cause of an inert
pane-death hook set whose hooks themselves were fine.

Verify with `tmux show-options -wgv remain-on-exit`.

## Hook format variables expand at execution time

tmux 3.4 expands format variables in `set-hook` `run-shell` commands
when the queued run-shell EXECUTES, not when the hook fires. For events
that destroy their subject object, the expansion therefore resolves
against the post-destruction context:

- `tmux kill-session`: pane-died and pane-exited NEVER fire;
  session-closed fires exactly once, but `#{session_name}` expands to
  the fallback current session (e.g. the remaining keeper session), NOT
  the killed session. `#{hook_session_name}` carries the correct name.
- Pane process death under a local `remain-on-exit off` override:
  pane-died never fires; pane-exited fires but BOTH spellings come back
  garbage (`#{session_name}` → wrong session, `#{pane_id}` → wrong
  pane; the hook-scoped spellings are EMPTY — the pane object is already
  destroyed). session-closed fires hook-scoped-correct.

Rule: when arming hooks for events that destroy their subject
(kill-*, teardown), use `#{hook_session_name}` / `#{hook_pane_id}`, or
expand at queue time. Never trust `#{session_name}` on session-closed or
pane-exited. Only pane-died — whose subject pane is RETAINED and still
exists when the hook runs — expands correctly with the plain spelling.
A hook set armed with `#{session_name}` targets the WRONG session for
kill-session and unretained-death events.

## C locale drops the tab separator from format output

tmux 3.4 emits `#{...}\t#{...}` format output WITHOUT the tab separator
when the process runs under the C/POSIX locale (`LANG=C` or `LC_ALL=C`,
or no LANG at all, e.g. `env -i`). Under any UTF-8 locale the tab
survives. Verified deterministic (3 repeats per variant).

Consequence: Go/shell code that splits tmux `-F` output on `\t` silently
sees 1 field instead of 2 under a sanitized environment and reports
`sessions = map[]` style empty results — no error. Tests pass from an
interactive shell (`LANG=en_US.UTF-8`) while any `env -i` clean-env repro
sees spurious tmux test failures. Always preserve LANG when reproducing
gate runs.

## First duplicate HOME wins (libc getenv)

With DUPLICATE `HOME` entries in the execve envp (e.g. Go's
`append(os.Environ(), "HOME="+x)`), tmux sources the FIRST occurrence's
conf — libc `getenv` returns the first match, so the temp HOME is
silently IGNORED. To redirect a scratch server's conf via HOME, filter
every `HOME=` entry out of `os.Environ()` and append exactly one. With
a single filtered HOME, tmux sources `$HOME/.tmux.conf` at server start
(env HOME wins; no getpwuid fallback needed when HOME is present once).

Server-start semantics that pair with this:

- The server sources the conf synchronously before the triggering
  `new-session -d` returns — no sleep is needed before querying a
  marker set by the conf.
- `-f /dev/null` on a query against an already-running server is inert
  (the conf only matters at server start); `-f` naming a nonexistent
  file is tolerated at server start.
- Ubuntu noble ships no `/etc/tmux.conf`; `-f` covers the user-conf
  vector.

## show-options -gv unset semantics

`tmux show-options -gv @user-opt`: exit 0 printing the value when set;
exit 1 with EMPTY stdout and `invalid option: @user-opt` on stderr when
unset. Assert on stdout only, never on the error. The alternative
`tmux display-message -p '#{@user-opt}'` exits 0 with an empty line
when unset.

## Verifying a hook-based tmux setup

Hooks survive in server memory after a conf re-source that no longer
contains them, but user options set by the same conf block revert
immediately — so a live user option set alongside the hooks (for example a
`@resurrect-dir` value) is a canary for whether the block is still present
in `~/.tmux.conf`. A regression battery touching such hooks must check
`show-hooks -wg`, `show-options -wgv remain-on-exit`, and the live canary
option — not just `show-hooks -g`. A configuration-management run that
re-renders `.tmux.conf` wipes such a block; re-arm after it.

Also confirmed working on 3.4: `#{>=:#{version},X}` format comparisons and
`%if`/`%endif` config conditionals.

Canonicalize tmux-reported paths (symlink chains) before comparing them in
tests.
