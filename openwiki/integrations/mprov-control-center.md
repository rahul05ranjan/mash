---
type: Integration
title: mProv Control Center (mPCC) Integration
description: How mash authenticates to the mProv Control Center, discovers data models from /datamodel/, maps CRUD commands to HTTP requests, validates fields client-side, stores results in MPROV_RESULT, and drives node power endpoints via the bmc plugin.
tags: [mprov, mpcc, http, api, auth, datamodel, bmc]
resource: src/mash/app.py
---

# mProv Control Center (mPCC) Integration

mash is a client of the mProv Control Center REST API. All communication goes through a single `requests.Session` on `MprovShell`, configured with a connection pool (100 connections, 20 retries, `pool_block`) mounted for both http and https.

## Connect and authentication

`do_connect` ([app.py](../../src/mash/app.py), lines ~164-204) supports three forms, each producing an `Authorization` header:

| Form | Header |
|---|---|
| `connect <url> apikey <key>` | `Api-Key <key>` |
| `connect <url> user <u> password <p>` | `Basic base64(u:p)` |
| `connect` (no args) | `Api-Key` from config file `global.apikey` |

Config resolution order and file shapes are covered in [Shell Architecture](../architecture/overview.md#config-resolution). Then `_connectToMPCC` performs a `GET <mprovURL>` as a liveness check; on HTTP 200 it calls `_getMPCCModels`.

Note: the Basic-auth branch builds `user:password` from `args[2]`/`args[4]` (the literal words `user`/`password` occupy arg positions), so argument order is strict and there is no quoting/escaping of the split — a known rough edge.

## Dynamic data model discovery

mash does not hard-code any mPCC resources. On connect, `_getMPCCModels` fetches `GET /datamodel/` for the model name list, then `GET /datamodel/?model=<name>` per model, storing each schema in `self.models`. Each model entry supplies at least:

- `endpoint` — the URL path segment under `mprovURL`
- `fields` — field names with a `required` flag

`models` and `model <name>` shell commands let users inspect what was discovered. This is why mash works against evolving mPCC versions without releases: the CRUD surface is defined server-side.

## CRUD → HTTP mapping

`create|retrieve|update|delete` (aliases `post|get|patch|delete`) all call `_sendHttpRequest(method, arg)` ([app.py](../../src/mash/app.py), lines ~479-589), which parses `arg` as `model [key=value ...]`:

- **Unknown model** → error; the model must exist in the discovered set.
- **POST/PATCH**: args are parsed with `shlex`, validated against `model.fields` (unknown field → error), sent as a JSON body. `create` additionally enforces required fields (`checkargs=True`); `update` does not. `_parseArgType` converts `[a,b,c]` literals into JSON lists; `{...}` dict literals are rejected ("not supported yet").
- **GET/DELETE**: args become a query string; an `id=` or `pk=` arg is also lifted into the URL path segment (`<endpoint><id>/?...`).
- A trailing `&` backgrounds the request via `os.fork()` — see [Shell Architecture](../architecture/overview.md#background-jobs-fork).

On 2xx, the parsed JSON body is stored in the shell variable `MPROV_RESULT` (suppressed in forked/background children, and on non-2xx an error + body text is printed instead). Scripts chain calls by templating `{{MPROV_RESULT...}}` into subsequent commands.

## Node power: the bmc path

The `bmc` plugin calls `GET {mprovURL}power/<action>/?hostname=<node>` for each expanded node ([plugins/bmc.py](../../src/mash/plugins/bmc.py)). Node ranges like `compute00[01-20,25]` are expanded client-side by `rangeToList` in [utils.py](../../src/mash/utils.py) — slurm-style, zero-padding preserved, mismatched pad widths rejected. This endpoint is outside the model-driven CRUD surface; it is a hardcoded mPCC route. See [Plugin System](../plugins/development.md).

## Gotchas

- `mprovURL` is concatenated directly with endpoints; a missing trailing slash on the configured URL yields malformed paths (model CRUD does `f"{mprovURL}{endpoint}"`, while liveness/discovery use `/datamodel/` with a leading slash).
- Requests use `timeout=None` throughout: a dead mPCC hangs the shell (except the liveness check, which catches and prints a generic error).
- `MPROV_RESULT` is set to `None` on validation errors, letting scripts test for failure.
- There is no OAuth/token refresh; long-lived API keys are the expected operator workflow.
