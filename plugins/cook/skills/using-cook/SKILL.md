---
name: using-cook
description: Use when operating the cook build system from the command line — running or discovering targets, investigating why something rebuilt or stayed cached, inspecting the cache or build logs, or when a bare `cook` invocation fails or a recipe name seems to be ignored.
---

# Using cook

## Overview

`cook --help` lists the subcommands and `document.md` in the cook repo is the manual. This skill is the operational knowledge that neither surfaces: what to reach for, and the places the CLI is quietly surprising.

## Orientation

```sh
cook menu            # every recipe and chore (alias: cook list)
cook <recipe>        # run one
cook +<recipe>       # ...when the name collides with a subcommand
```

**Bare `cook` runs a recipe named `build`.** Most projects don't have one, so plain `cook` just fails — always name a target.

`cook menu` hides nothing. Recipes minted by a module carry a `(from <module>)` annotation, which is the only signal that a target is machinery rather than something you'd invoke. A `__` prefix suppresses a recipe from progress output and shell completion but **not** from `menu`.

## "Why did this rebuild?" — `cook why`

```sh
cook why <recipe>          # per unit: cache key, every input + hash, HIT/MISS
cook why <recipe> --json   # {recipe, units[]}, status in {local_hit, shared_miss, ...}
```

Read-only; runs nothing. The build's own output already names causes —
`rebuild (input changed: src/util.h)` — so read that before escalating.

**What a cache key is made of.** Five things, and only one of them is your source:

| Component | Is |
|---|---|
| `command_hash` | the rendered command line |
| `env_contribution` | environment fingerprint — also the `@2d0680…` suffix on every key |
| `seal_contribution` | sealed probe values: compiler identity, pkg-config results |
| `inputs` | each declared input path + content hash |
| outputs | the declared output paths |

`seal_contribution` is the answer to the most common real question. A toolchain upgrade
with zero source changes invalidates everything, because the compiler's identity is a
sealed determinant — that is working as designed, not a cache bug.

**`shared_present: true` alongside `status: shared_miss` is normal.** It means the artifact
is in the store under a *different* key, not that the store is broken. And `manifest_diff`
is frequently `null` — do not expect a diff on every shared miss.

## Recipes take no parameters; chores do

Recipes are artifact-producing and cacheable. Ask one for a parameter and cook tells you to use a chore or a config preset (`cook <recipe> @preset`).

Chore parameters are **positional**, declared as bare names:

```cook
chore greet name
    echo hello $<name>
```

```sh
cook greet World
```

Multiple parameters are space-separated bare names, bound positionally in declaration
order (`chore greet name greeting` → `cook greet World Hello`).

**Discoverability trap:** `chore greet(name)`, `chore greet $<name>`, and
`chore greet <name>` all parse *silently* as a zero-parameter chore — the declaration
never errors, you just get an arity complaint at the call. Bare names only. (A correctly
declared chore missing an argument gives a good error, naming the parameter and its
declaration line.)

## Where state lives

| Path | What |
|---|---|
| `<project>/.cook/cache/` | per-recipe ledger: step keys, input hashes |
| `<project>/.cook/logs/<id>/` | `events.jsonl`, `manifest.toml`, per-node output |
| `<project>/.cook/probes/` | resolved probe values |
| `~/.cache/cook/cloud/` | global content-addressed artifact store — the "shared" in `HIT (shared)` |

The global store is shared across every project and gets large (17G is normal on a working machine). **There is no `cook cache gc`, no `cook cache path`, no size reporting** — `cook cache` only offers `verify`. Pruning means deleting the directory by hand.

Consequence: a `clean` chore that does `rm -rf build` does not give you a cold build. Outputs re-materialize from the store instantly. That is correct behaviour, and it surprises people.

## Machine-readable output

```sh
cook --output json <recipe>       # NDJSON build events on stdout — LIVE builds only
cook logs                         # browse past builds (-n 2, --last-failed)
cook test --report-json <path>    # test results
cook affected --since=<ref> --json
```

**`cook logs` ignores `--output json`** — it prints the human tree regardless. To analyse a
build you did not run, read the persisted files directly:

```
.cook/logs/<id>/events.jsonl   line 1 = the DAG (recipes[].deps, expected_nodes),
                               then node-* / recipe-completed / build-completed events
.cook/logs/<id>/manifest.toml  build_id, started_at, ended_at, exit_code
.cook/logs/<id>/nodes/         per-node captured output
```

`cook menu` and most commands write lint warnings to **stderr**, so anything piping their
output wants `2>/dev/null`.

## Module/binary version skew

Modules are installed per project under `./cook_modules/` (Lua **5.4 only** — a rock under another version silently does nothing). A module can call an API the installed `cook` binary doesn't have yet, and the failure is not obviously a version problem:

```
cook_cc/units/cc.lua:97: attempt to call a nil value (field 'dep_order')
```

(That specific example is historical — `dep_order` landed in cook 0.8.0 / Standard v0.15.
The shape recurs whenever a module outruns the binary.)

`cook menu` keeps working when this happens, because listing uses a body-free pass — so
**"menu works, builds don't" is the signature**, and any `attempt to call/index a nil value`
whose source path is under `cook_modules/` is version skew rather than a Cookfile bug.

Note the two version numbers are not directly comparable: the binary reports a *Standard*
version (`cook 0.8.0 (Cook Standard v0.15)`) while the rock has its own (`cook_cc 0.17.0-1`,
pinned in `cook.lock`, installed under `cook_modules/lib/luarocks/rocks-5.4/`). You are asking
whether the module needs a newer Standard than the binary implements.

Fix by moving whichever side is behind: `cook modules update <name>` for the module,
or upgrade the binary.
