# mash (mProv Admin Shell)

This is the command line utility for talking to the mProv Control Center.  

## Requirements
The `mash` requires python 3.8 and above.  It also requires that the packages in the `requirements.txt` be installed.  As always, you can accomplish this with the following pip command: `pip install -r requirements.txt`

## Installation
Install the package from this repository (`pip install .`), which places the
`mash` command on your `PATH`.  You can also run it directly from a checkout
with `python -m mash.main`.

## Usage
`mash` is an interactive shell.  Run `mash` with no arguments for a prompt,
pass a script file as the first argument, or pipe a script on stdin.
Type `help` at the prompt for the full command list.

Before doing anything, connect to a Control Center:

```
connect <mprov_url> apikey <api_key>
connect <mprov_url> user <user> password <password>
connect        # with no arguments, read credentials from a config file
```

Config files are read in this order: `~/.mprov-mash.yaml`, then
`/etc/mprov/mash.yaml`, falling back to `/etc/mprov/jobserver.yaml`
(a jobserver host can run mash with its existing jobserver config).

Core commands (all implemented in `src/mash/app.py`):

- `create` / `retrieve` / `update` / `delete` — CRUD against mPCC data models
  (aliases: `post`, `get`, `patch`).  Append `&` to run a request in the
  background.
- `models` / `model <name>` — list supported data models / show a model's fields.
- `let <var>=<value>` — set a shell variable (values support Jinja2 templates
  such as `{{var}}`; a leading backtick captures the output of another command).
- `pvar <var>` (alias `p`) / `print <text>` — display variables or templated text.
- `seq <var> <start> <end> [width]` — build a zero-padded number list, then
  iterate it with `foreach <item> in <var>` ... `endforeach`.
- `disconnect`, `exit` / `quit`.

### Plugins
Extra verbs are provided by plugins under `src/mash/plugins/`:

- `bmc power on|off|cycle|reset <node_spec>` — power control through the mPCC.
  `<node_spec>` is a hostname or a Slurm-style node range
  (e.g. `compute00[01-20]`).  See `bmc help` in the shell.

## Documentation
See [`openwiki/quickstart.md`](openwiki/quickstart.md) for architecture and
detailed behavior notes.
