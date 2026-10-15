---
type: Playbook
title: mash Plugin System
description: How mash discovers and dispatches plugin commands through MprovShell.default, the PluginCMD class contract, the built-in bmc power plugin, and how to add a new plugin module under src/mash/plugins.
tags: [plugins, extensibility, cmd, bmc, playbook]
resource: src/mash/plugins/bmc.py
---

# Plugin System

mash's subsystem commands (currently just `bmc`) are plain modules dropped into the `mash.plugins` package. There is no registry, entry-point metadata, or install step: **the first word of any unrecognized shell command is treated as a plugin module name**.

## Dispatch mechanism

`MprovShell.default()` ([app.py](../../src/mash/app.py), lines ~80-114) is invoked by `cmd.Cmd` for unknown commands. It:

1. Splits the line into `pluginName` (first word) and the remaining args.
2. `importlib.util.find_spec(f"mash.plugins.{pluginName}")` — if no module matches, prints `Unrecognized command`.
3. Imports the module and looks for a class named exactly **`PluginCMD`**.
4. Instantiates it as `PluginCMD(self)` — the plugin always receives the parent `MprovShell` instance — and forwards the remaining args via `pluginInstance.onecmd(args)`.

Exceptions inside a plugin are caught at the dispatch boundary and reported as `Error: Exception in plugin <name>`, so a broken plugin cannot crash the shell.

`do_help("")` scans the `mash.plugins` package directory and lists every non-dunder module, so plugins appear in top-level help automatically. `help <plugin> [subcmd]` is proxied into the plugin's own `onecmd("help ...")`.

## The PluginCMD contract

A plugin module must define `PluginCMD`, which in practice subclasses `cmd.Cmd` and takes the mash shell in `__init__` ([plugins/bmc.py](../../src/mash/plugins/bmc.py)):

```python
class PluginCMD(cmd.Cmd):
  def __init__(self, mashCmd=None):
    self.mashCmd = mashCmd   # the MprovShell instance
    super().__init__()
```

Through `self.mashCmd` a plugin gets everything the shell has: the authenticated `session` and `mprovURL` for calling the mPCC, `err()`/`print()` for output, `variables` for scripting integration, and the `processes` list for background-job tracking. A plugin's own `do_<sub>` methods become subcommands (`bmc power ...`), and `default()` can route bad input to help.

Plugins are instantiated **per line**, so plugin instance state does not persist between commands; keep state on `self.mashCmd` if needed.

## The bmc plugin (reference implementation)

`bmc` issues node power operations through the mPCC — see [mProv Control Center Integration](../integrations/mprov-control-center.md#node-power-the-bmc-path) for the endpoint details.

- `bmc power <on|off|cycle|reset> <node_spec>` — expands slurm-style ranges with `rangeToList` ([utils.py](../../src/mash/utils.py)) and iterates `session.get(f"{mprovURL}power/{action}/?hostname={node}")` per node.
- Its `precmd` strips a trailing `&` into a `background` flag, mirroring the shell's backgrounding convention — though note the flag is currently only recorded, not acted on, so `bmc ... &` behaves synchronously.
- It guards on `mprovURL` being set and prints a "probably aren't connected" warning otherwise, but does not surface per-node HTTP response codes.
- A connection check was added deliberately (git history) so `bmc` fails loudly before hammering the API.

## Adding a new plugin

1. Create `src/mash/plugins/<name>.py` with a `PluginCMD(cmd.Cmd)` class that accepts the shell instance.
2. Implement `do_<subcommand>` methods; use `self.mashCmd.session` for authenticated mPCC calls rather than building a new `requests` client.
3. Add a docstring to each `do_` method — plugin help is surfaced through standard `cmd.Cmd` help and the top-level plugin listing.
4. No registration is needed; the module filename becomes the command word. Avoid names that collide with built-in `do_*` commands, since built-ins win (plugin dispatch only runs for unrecognized commands).

Caveats: plugin args arrive **after** the shell's Jinja2 templating pass in `precmd`, so plugins see `{{var}}` values already substituted; and because dispatch is module-name based, any importable module under `mash.plugins` is executable as a command.
