---
type: Reference
title: mash Quickstart
description: Entry point for mash (mProv Admin Shell), a Python CLI that talks to the mProv Control Center (mPCC). Covers installation, shell usage, configuration, command surface, and links to architecture, integration, and plugin documentation.
tags: [mash, mprov, cli, quickstart, shell]
resource: src/mash/main.py
---

# mash (mProv Admin Shell)

`mash` is the command-line admin shell for the **mProv Control Center (mPCC)**, the control-plane API of the mProv NG bare-metal provisioning system. It is an interactive and scriptable `cmd`-based shell that authenticates to an mPCC, dynamically discovers the mPCC's data models, and exposes generic CRUD commands (`create`/`retrieve`/`update`/`delete`) against them, plus shell features like variables, Jinja2 templating, `foreach` loops, background jobs, and plugins such as `bmc` for node power control.

Package metadata lives in [setup.cfg](../setup.cfg) (distribution name `mprov_mash`), version in `VERSION` (currently `0.0.9`), and dependencies in [requirements.txt](../requirements.txt): `pyyaml`, `jinja2`, `requests`. Python >= 3.8 is required.

## Install and run

```bash
pip install -r requirements.txt   # dev deps
python -m build && pip install dist/*.whl   # or install the published mprov_mash package
mash                               # interactive shell
mash script.mash                   # run a script file
cat script.mash | mash             # pipe a script on stdin
```

The installed `mash` entrypoint is the bash wrapper [src/mash/mash](../src/mash/mash), which verifies Python >= 3.8 and then runs `python -m mash.main`. See [Architecture Overview](architecture/overview.md) for how input mode (interactive / file / FIFO) is detected and dispatched.

## First session

```text
<mProv> # connect https://mprov.example.com/ apikey <API_KEY>
<mProv> # models                      # list models discovered from the mPCC
<mProv> # model node                   # show one model's fields/schema
<mProv> # retrieve node name=compute0001
<mProv> # pvar MPROV_RESULT            # last response body is stored in MPROV_RESULT
<mProv> # bmc power on compute00[01-20]
<mProv> # exit
```

## Command surface

| Command | Purpose |
|---|---|
| `connect` / `disconnect` | Authenticate to an mPCC (API key, user/password, or config file) and fetch data models. See [mProv Control Center Integration](integrations/mprov-control-center.md). |
| `create` `retrieve` `update` `delete` (aliases `post` `get` `patch`) | Generic CRUD against any discovered mPCC model. |
| `let` / `pvar` (`p`) / `print` | Shell variables with Jinja2 `{{var}}` templating and backtick command capture. |
| `foreach` … `endforeach` | Loop over a list variable or whitespace string; commands are collected then replayed with templating. |
| `seq` | Build a zero-padded numeric sequence list variable (pairs with `foreach`). |
| `models` / `model <name>` | Inspect models discovered from the mPCC. |
| `bmc power <action> <node_range>` | Plugin command: node power on/off/cycle/reset via mPCC, accepting slurm-style node ranges. |
| `help` | Lists built-in commands **and** all discovered plugins in `mash.plugins`. |

Any unrecognized first word is looked up as a plugin module — see [Plugin System](plugins/development.md).

## Configuration file resolution

`connect` with no arguments (and only then) loads YAML config, first usable file wins ([src/mash/app.py](../src/mash/app.py), `load_config`):

1. `~/.mprov-mash.yaml` (user)
2. `/etc/mprov/mash.yaml`
3. `/etc/mprov/jobserver.yaml` — fallback so mash works on mProv jobserver hosts (added for issue #4)

Expected shape: a top-level `global:` section with `mprovURL` and `apikey`. The loader flattens the top-level YAML entries into one dict, and registers a no-op `!include` constructor (placeholder, currently unused).

## Build, versioning, release

- Version comes from the `VERSION` file, or the `MASH_VERSION` env var at build time ([setup.py](../setup.py)).
- [ci.yml](../.github/workflows/ci.yml) triggers on GitHub **release published**: writes the release tag into `VERSION`, builds with `python -m build`, and publishes to PyPI as `mprov_mash`.
- `gen_changelog.sh` prints commit summaries between tags (caveat: its commit links point at the `mprov_jobserver` repo — likely a copy-paste bug).

## Change guidance

- New shell behavior → [Architecture Overview](architecture/overview.md) (precmd/cmdloop/templating pipeline is the load-bearing machinery).
- New mPCC interaction or auth method → [mProv Control Center Integration](integrations/mprov-control-center.md).
- New subsystem command → add a module under `src/mash/plugins/` with a `PluginCMD` class ([Plugin System](plugins/development.md)).
- There are **no tests in this repo**; validate changes by running the shell against a live/mock mPCC or piping scripted sessions via `echo '...' | mash`.

## Backlog

- **skills/** directory: empty, no documentation value.
- **OpenAPI/model schema detail**: model shapes are discovered at runtime from the mPCC `/datamodel/` endpoint; not statically known in this repo.
