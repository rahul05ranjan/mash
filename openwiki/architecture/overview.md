---
type: System Architecture
title: mash Shell Architecture
description: How the mash shell starts, picks input mode, resolves configuration, and processes each command line through precmd templating, cmd.Cmd dispatch, plugin fallback, foreach loops, and fork-based background jobs.
tags: [architecture, shell, cmd-loop, templating, jinja2, scripting]
resource: src/mash/app.py
---

# Shell Architecture

mash is a single Python package (`src/mash/`) built around Python's stdlib `cmd.Cmd`. Nearly all logic lives in one class, `MprovShell` in [src/mash/app.py](../../src/mash/app.py) (~650 lines).

## Startup path

1. The installed script `src/mash/mash` (bash) checks Python >= 3.8 and runs `python -m mash.main`.
2. [src/mash/main.py](../../src/mash/main.py) installs no-op SIGINT/SIGTERM handlers, constructs `MprovShell`, and chooses an input mode:
   - **stdin is a FIFO** (`S_ISFIFO(os.fstat(0).st_mode)`) → script piped in via `shell.setFile(sys.stdin)`.
   - **argv[1] is an existing path** → script file mode via `open(sys.argv[1])`.
   - otherwise → interactive REPL.
3. `shell.cmdloop()` runs. In file mode (`cmdloop` override), `self.quiet = True` suppresses the per-command "OK" noise and each line is fed through `onecmd(precmd(line))`; comments (`#`) are stripped here and again in `precmd`.

## The per-line pipeline (`precmd` → dispatch → `default`)

`precmd` is the heart of the shell ([app.py](../../src/mash/app.py), lines ~607-623):

- Lines starting with `#` become empty (comments).
- If inside a `foreach` loop, the line is **accumulated** into `forLoopCmds` instead of executed (only `endforeach` passes through).
- Otherwise the line is rendered through **Jinja2** (`renderString`) with all shell variables as the template context — this is how `{{var}}` substitution works on every command line, not just `let`.

Standard `cmd.Cmd` then dispatches `do_<name>` methods (`connect`, `create`, `let`, `foreach`, `seq`, ...). Unknown commands fall into `MprovShell.default()`, which attempts **plugin dispatch** — see [Plugin System](../plugins/development.md).

`self.print()` (not builtin `print`) writes to `self.stdout` so internal capture works: `execInternal()` swaps `self.stdout` for a `StringIO`, runs `onecmd`, and returns the captured text. This powers backtick assignment in `let x=\`some command\``.

## Variables and templating

Shell state lives on the instance: `variables` dict, `session` (a `requests.Session`), `mprovURL`, `models`, `processes`, loop state. `do_let` supports:

- literal assignment (`let foo=bar`)
- Jinja2 references to prior variables (`let x=hello {{foo}}` — rendered by `precmd` before `do_let` even sees it)
- `` `cmd `` → captured output of another shell command via `execInternal`
- `$var` → direct copy of another variable's value (important in loops, where precmd templating may already have substituted)

The result of the last successful CRUD call is stored in `MPROV_RESULT` by `_sendHttpRequest` (see [mProv Control Center Integration](../integrations/mprov-control-center.md)), making it available to subsequent `{{MPROV_RESULT.field}}` templates.

## Control flow: foreach and seq

`foreach item in list|varname` sets loop state and changes the prompt; subsequent lines are buffered by `precmd`. `endforeach` replays the buffered commands once per list item, setting the loop variable and running each buffered command through `renderString` before `onecmd`. **If any command in the loop returns False, the whole loop aborts.** `seq var start end width` builds a zero-padded list variable designed to feed `foreach`.

Note the loop model buffers commands, then executes them after `endforeach` — mid-loop edits at the prompt are not possible, and nested loops are not supported (single flat loop-state set on the instance).

## Background jobs (fork)

A command line ending in `&` inside `_sendHttpRequest` calls `os.fork()`: the child performs the HTTP request and `sys.exit(0)`; the parent records the pid in `self.processes` and returns immediately. `do_disconnect` waits for all background children (`os.wait()` loop) before closing the session. The `bmc` plugin has its own `&` handling in its `precmd` ([plugins/bmc.py](../../src/mash/plugins/bmc.py)).

## Config resolution

`load_config()` picks the first readable of `~/.mprov-mash.yaml` → `/etc/mprov/mash.yaml` → `/etc/mprov/jobserver.yaml` (the jobserver fallback was added so mash works out-of-the-box on mProv controller nodes; see issue #4 in git history). The YAML top-level entries are flattened into one dict; only `global.apikey` and `global.mprovURL` are consumed today. A `!include` YAML constructor is registered but is an unused no-op stub.

## Utilities

[src/mash/utils.py](../../src/mash/utils.py) provides:

- `rangeToList(spec)` — expands slurm-style node ranges (`compute00[01-20,25]`) into explicit node lists, enforcing zero-pad width consistency. Used by the `bmc` plugin.
- `getDottedStrValue(dstring, obj)` — walks `a.b.c` paths into nested dicts (currently not referenced by app.py; available for plugin use).

## Change guidance

- Adding a built-in command: add `do_<name>` to `MprovShell`; remember `precmd` templating runs first, and use `self.print` for capturable output.
- Anything touching `precmd` affects templating, comments, and foreach buffering simultaneously.
- No test suite exists; exercise changes with scripted sessions (`printf '...' | mash`).
