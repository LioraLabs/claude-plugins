---
name: authoring-cookfiles
description: Use when writing or editing a Cookfile for the cook build system — adding recipes, chores, gather lines, cook steps, tests, probes, seals, or placeholders — or when a Cookfile fails to parse, a recipe name is ignored, an edit to a source file rebuilds nothing or rebuilds everything, a test runs serially that could fan out, a chore silently does nothing, or work needs to be gated behind passing tests.
---

# Authoring Cookfiles

## Overview

Cook's own docs teach the language: **`document.md`** in the cook repo (The Cook Manual: first recipe, placeholders, connecting recipes, tests, chores, caching) and `standard/src/content/docs/*.mdx` for normative detail. Read those for *how it works*.

This skill is the part the docs don't teach: how to **shape** the graph, and the traps that fail **silently or with a misleading diagnostic**.

A Cookfile that parses and builds can still be badly shaped. The shaping decisions below are what separate a Cookfile that caches and parallelises from one that merely works.

> This skill targets the **Language v2** surface (`gather` / `seal`, Cook Standard v0.18). If you find `ingredients`, `unseal`, a trailing `seal`, a probe-body `files { }` / `tools { }` / `envs { }`, or `$<file:glob>` in an existing Cookfile, that file predates the cut and no longer parses. Each removed spelling has a migration diagnostic naming its replacement.

## Shaping the graph

Four decisions, in the order they matter.

### 1. Fan out by default

If the work is per-item, make it per-item. A `test` body that references `$<in>` becomes **one cached unit per item**; a body that doesn't is one unit over the whole set.

```cook
recipe conformance: parser-lib
    gather "//standard/conformance/positive/*/Cookfile"
           "//standard/conformance/negative/*/Cookfile"
    test { node scripts/check.mjs --case $<in> }
```

284 fixtures, measured: one batched unit took 9.1s and re-ran entirely on any corpus edit. Fanned out it is 0.85s cold across cores, and editing one fixture re-runs **one** unit while 283 stay cached.

Batch only when the tool genuinely needs the whole set in one process (a linter that reports cross-file duplicates). "The harness already loops" is not a reason: give it a single-item entry point. Keep a `chore` for the whole-set view when a human wants one answer.

Fan-out demands a **shared** setup artifact. If each unit would build the same thing, hoist it into its own recipe with a declared output and depend on it, otherwise you pay for it N times.

### 2. `gather` names what the command consumes; `seal` names what determines it

`gather` is the **driver**: the set the command iterates or reads as its work item. Everything else that changes the answer is a **determinant**, and belongs in a `seal`.

Two static hard errors enforce that vocabulary, and they fire before anything runs:

```
recipe 'build': the command never names the gathered files; seal them instead
recipe 'build': nothing gathers what this command references
```

The first says you gathered something no `cook` or `test` body names through `$<in>`. The second says a body references `$<in>` with no `gather` and no dependency-output driver. Neither consults the filesystem, so neither is avoidable by building in a different order.

```cook
files claim:sites
    "VERSION" "src/lib.rs"
    "grammar.js"

# gather drives, and the body names it; seal determines
recipe claim-in-sync
    gather "scripts/check-claim.sh"
    seal claim:sites
    test { sh $<in> }
```

`seal` also takes quoted globs inline, which desugar to an anonymous file determinant. Use a named `files` declaration for a set worth naming and reusing; use the inline form for a one-off:

```cook
recipe gate
    seal "config/*.toml" !"config/local.toml"
    test { scripts/validate.sh }
```

Putting determinants in `gather` either fans out over them (wrong granularity) or trips the first hard error above. Putting drivers in a `seal` loses the fan-out entirely, and a quoted seal operand never binds `$<in>`.

Said bluntly: **`gather` is the iteration surface, not a cache lever.** "I want this file to invalidate the unit" is never by itself a reason to gather it. Ask "does the command iterate or read this as its work item?" If yes, gather it and name it. If it merely *determines the answer*, seal it. Cache participation falls out of either choice; it is not the choice.

A recipe whose command only reads another recipe's output has **no** `gather` at all. Reference the producer by name (`$<producer>`) and let the edge carry the input.

### 3. A seal mints a cache key

**A non-empty seal set gives a unit a cache key even when it declares no input, no output, and no data member** (CS-0223). This inverted in Language v2, and it is now the most common wrong mental model in the other direction.

Consequences, both real and both verified:

- A sourceless `test` sealed on anything **is cached**. It runs once, records a pass, and replays that pass on every later invocation until a sealed value moves. If you sealed the wrong thing, the gate is green forever.
- The supported way to write an always-runs gate is **no source and no seal**. `recipe check` / `test { cargo test }` has nothing whose movement could invalidate it, so it runs every time, by rule.

So the question "do I want this cached?" is answered by giving it a source **or** a seal, and the question "do I want it to always run?" is answered by giving it neither.

A recipe carrying a `seal` but declaring no cacheable unit at all is rejected: `seal on recipe gate: no cook or test units to apply to`.

### 4. Tools are determinants

If a tool's version changes its output *and you commit that output*, the tool is part of your source:

```cook
tools generate:tool
    tree-sitter

recipe generate
    gather "grammar.js"
    seal generate:tool
    cook "src/parser.c" { tree-sitter generate $<in> }
```

Without this, a machine with a different CLI regenerates tracked files in place and **no key moves to say why**; the diff shows up later as an unexplained dirty tree. A `tools` declaration PATH-resolves each name and folds the resolved binary's content hash. The resolved *path* is deliberately not part of the value, so identical toolchains at different locations still share artifacts.

There is no inline tool or environment determinant form. An ambient environment variable that no unit consumes is folded by declaring an ordinary shell probe over it and sealing that probe by name:

```cook
probe host:flags
    lines { echo "$CFLAGS"; echo "$LDFLAGS" }
```

Environment a unit *does* consume auto-folds already; don't seal it twice.

## Gates are dependency edges, not Lua

A failed `test` unit blocks every unit that depends on it, so a dependency list **is** a gate:

```cook
recipe claim-in-sync
    gather "scripts/check-claim.sh"
    test { sh $<in> }

chore release version: cook-bin claim-in-sync
    git diff-index --quiet HEAD -- || { echo "dirty tree" >&2; exit 1; }
    > release.cut(version)
```

Red gate gives `skipped (upstream-failed)`, the body never runs, and the exit status is non-zero. Do not re-implement this as `if` statements in a module: Lua gates can't be run individually, aren't cached, don't appear in `cook test`, and capture their output instead of streaming it.

## Recipe or chore?

**Any recipe containing a `test {}` step is automatically a `cook test` root. There is no opt-out.** That single fact drives the choice:

| Put it in a **recipe** (`test {}`) | Put it in a **chore** |
|---|---|
| Asserts a property of the tracked tree | Asserts working-state (clean tree, credentials present) |
| Safe and cheap to run on every `cook test` | Re-enters `cook test`, or duplicates the suite |
| Should gate other work | Mutates things (commit, tag, push, install) |

Two consequences learned the hard way: a working-state assertion written as a test recipe turns `cook test` red during ordinary editing; and a test recipe that shells out to `cook test` recurses forever, or, scoped, makes a bare `cook test` run the suite **twice concurrently**, where the two runs fight over shared build directories.

## Chore bodies

- **Each line is its own process.** Shell variables do not persist between steps. Multi-step shell belongs in a script file the chore invokes.
- **A chore never caches and always runs.** It is outside the DAG's cache entirely, so none of the seal reasoning above applies to it.
- Shell steps **stream** output and abort the chore on non-zero. Lua steps calling `cook.sh` **capture** it, and a large failure can be truncated. Prefer shell steps for anything whose failure you need to read.
- Chores are excluded from `cook test` fan-out. That exclusion is the lever for "must run, must not be a test root."
- Keep the Cookfile declaring units and edges; push imperative shell into scripts. A pipeline assembled with `table.concat` in Lua is a smell.
- **A chore step must never require a human at a terminal.** A step that opens `$EDITOR`, prompts, or waits on stdin hangs forever when run headless (agents, CI), and an aborted multi-step chore leaves its `.cook/` state files behind for the retry to trip on. Interactive input needs a parallel non-interactive path: a chore parameter, a message file, or stdin. (Canonical casualty: the `cook rockspec` changelog editor in cook-modules.)
- A chore that mutates more than one repo should pre-flight ALL of them clean before touching any, and clean up its own droppings on abort. A half-run publishing flow is the most expensive kind of stale state.

## Recipe bodies are not Lua scripts

A recipe body admits only these, indented:

- `gather "glob" !"exclude"` (at most one per recipe), or `gather <name>` for a `files` declaration or an array probe, optionally followed by quoted globs that become every member's inputs
- `seal <probe-or-declaration-name> "glob" !"glob"`
- `cook "out" { shell }` or `cook "out" >{ lua }`, with an optional single trailing `local` / `pinned` / `nondet`
- `test { … }` or `test >{ … }`, with no tail at all
- `<module>.<fn>(...)`, a bare module call
- `#` comments **on their own line** (indented or at column 0; both are fine). A trailing comment on a step line is a parse error

Everything else is rejected. Both of these are real messages:

```
execute-phase `>` Lua is not allowed in a recipe body (CS-0134)
loose shell commands are not allowed in a recipe body (CS-0134): `local x = 1`
```

No `local`, no bare Lua statements, no loose shell. Put logic in a `register` block, a chore, or a module. **Chores** are the permissive form: they take bare shell lines and `>` Lua lines.

`files NAME` and `tools NAME` are **top-level** declarations, siblings of `recipe` and `probe`, not steps and not probe bodies. Their indented bodies take multiple entries across multiple lines.

## Traps

| Symptom | Cause | Fix |
|---|---|---|
| A parse error naming `#` on a line that looks fine: `cook: unexpected modifier '#'`, `gather: expected '"' or '!"', found: #`, `seal: malformed probe key '#'`, `unexpected text after test body: '#'` | **A trailing `#` comment is rejected on every declarative line.** No step keyword, declaration body, or recipe header admits one; what follows is parsed as more of the declaration | Put the comment on its own line, indented or at column 0. Comments *inside* a `{ … }` body, braces and all, are fine |
| `cook <name>` runs something else and your recipe never builds | The name collides with a CLI subcommand. The full set is `init`, `menu`, `list`, `modules`, `test`, `logs`, `cache`, `serve`, `emit-lua`, `affected`, `why`, `help` (check `cook --help`) | Rename, or invoke as `cook +<name>`. Cook prints a one-line notice that is easy to miss |
| Editing a header rebuilds nothing | Only **declared** determinants count. The core DSL does no `#include` scanning | A `files` declaration plus `seal`, or an inline `seal "src/*.h"`. See below |
| A gate never runs again, or a test replays a pass forever | A non-empty seal set **mints** the key (CS-0223), so a sourceless sealed test caches | To always run, remove the seal as well as the source. To gate on the right thing, seal the thing that actually moves |
| A recipe reading another recipe's output builds with an empty `$<in>`, and the only clue is `warning: input "build/gen.a" matched 0 files` | A `gather` pattern is a filesystem glob resolved at register time. A not-yet-built path matches zero files and contributes nothing. This case is **deliberately not diagnosed** (Standard Note 16.1.2.2); the `read-after-write with no ordering edge` diagnostic covers only `cook.add_unit` `inputs[]` | Never gather a produced path. Reference the producer: `$<producer>` in the body, or `: producer` in the dep list |
| Two recipes never order against each other despite one writing what the other reads | A literal path equal to another recipe's output creates **no** edge | Name the recipe: `$<other>` in a step, or a `: other` dep-list entry |
| `clean` chore runs, next build is still instant, even after `rm -rf .cook` | Outputs re-materialize from the content-addressed store at `~/.cache/cook`, which neither `rm -rf build` nor `rm -rf .cook` touches | This is correct. For a genuinely cold build you must invalidate keys, not delete outputs |
| `$<in> in a data fan-out shell body must be enclosed in single quotes` | Any **bare** `gather <name>` source puts the recipe in member-fan-out mode, and the whole-member reference must be single-quoted. This applies to a `files` declaration whose members are plain paths, not only to record probes | Write `'$<in>'`. Accessors (`$<in.stem>`, `$<in.FIELD>`) need no quoting |
| A step containing `$<file:…>` dies with `/bin/sh: syntax error near unexpected token` | `$<file:PATH>` was removed (CS-0187). It is not recognised as a placeholder at all, so it reaches the shell verbatim | For the key fold, a `files` declaration plus `seal`. To pass paths to a command, gather them and use `$<in>` |
| `placeholder $<ver.tag>: no config block declares 'ver.tag'` when `ver` is plainly a probe | A `$<…>` sigil is dispatched to the probe path only if its key contains a `:`. A single-segment probe key is unreachable from a command | Give the probe a namespaced key (`probe demo:ver`) and reference `$<demo:ver.tag>` |
| A member's recipe can't glob a sibling member's tree: `input pattern "../other/*.txt" escapes member root` | `../` escapes the member root and is rejected | Anchor at the workspace root: `gather "//other-member/**/*.txt"` |

## Declaring inputs that never appear on the command line

The commonest case of *seal what determines* (above): headers found via `-I`, config read
at runtime, templates. Inputs the command never names, so nothing can infer them. The
core DSL does no `#include` scanning.

```cook
files headers
    "src/*.h"

recipe lib
    gather "src/*.c"
    seal headers
    cook "build/obj/$<in.stem>.o" { cc -Isrc -c $<in> -o $<out> }
    cook "build/lib/libx.a" { ar rcs $<out> $<in> }
```

Editing any matched header gives `rebuild (seal changed)` on every unit in the recipe. Adding or
removing a matched file counts too: the declaration's value is a map of path to content hash.

There is exactly **one** seal tier. A recipe-level `seal` is additive, order-independent, and
applies to every cacheable unit in the recipe, `cook` and `test` alike, regardless of where the
line sits. Per-step scoping was removed with the trailing `seal` and `unseal` forms; if you need
different determinants for different units, split the recipe.

Seals do not cross recipe boundaries. A downstream recipe folds an upstream's determinants in
transitively through the upstream's **output content**, so re-sealing downstream double-counts.

## Ordering comes from names, never paths

A **name reference** creates the cross-recipe edge and puts the referent in the build closure, with no dep-list entry required. `$<other>` in a step body is enough. Writing `"build/other/thing.a"` as a literal is not, however exactly it matches.

## Verify by running

A Cookfile that parses can still be wrong in the way that matters: under-declared determinants build clean and rebuild nothing. After writing one:

1. `cook <target>` cold, then again. The second run must be fully cached.
2. Touch each kind of input (source, header, data file) and confirm the *expected* units rebuild.
3. For a fan-out, edit **one** item and confirm exactly one unit re-runs. `N-1 cached` is the assertion.
4. For a gate, break it on purpose and confirm the dependent reports `skipped (upstream-failed)` and never runs its body.
5. For anything you sealed, confirm it is cached when you expected caching **and** that it re-runs when a sealed file moves. A seal mints a key now, so "it still runs every time" and "it never runs again" are both live failure modes.

Steps 2 to 5 are the ones people skip, and they are the ones that catch the silent failure. "It ran and passed" proves nothing about keys.

`cook why <recipe> --level unit` prints a unit's key, its folded inputs, its sealed probe values, and `last ran because:`. Reach for it the moment caching surprises you.

Tests are content-keyed like everything else: a cached test prints nothing and reports
`cached`. That looks like a broken test. To watch one actually run, change something that
changes the key.

A recipe with only a `test` step, no `gather` and no `cook`, is legal and useful.

## Quick reference

| Placeholder | Means |
|---|---|
| `$<in>` | this unit's input. **Re-binds down the step chain**: in a later step it is the previous step's collected outputs, which is what makes compile then archive work. Under a bare `gather <name>` source it must be single-quoted |
| `$<out>` | this unit's output. Parent directories are created for you; no `mkdir -p` |
| `$<in.stem>` | basename without extension; drives fan-out |
| `$<in.FIELD>` | a record member's field, when the recipe gathers an array probe |
| `$<out_1>`, `$<out_2>` | multi-output steps (body must be a block) |
| `$<recipe>` | another recipe's terminal outputs, space-joined; creates the edge |
| `$<NAME>` | a variable declared by a `config` block (`$<var.NAME>` disambiguates from a recipe of the same name). The retired `$<env.NAME>` form has its own diagnostic |
| `$<key:seg.field>` | a probe value. The key must contain a `:` to be recognised as one |
| `//glob` | in `gather` and `files`, anchors at the **workspace root**: how a member reaches a sibling member |

Chore parameters are **positional**: `chore fmt dir="src"` is invoked `cook fmt src`. Inside the body they bind three ways: as a Lua local, as `$<dir>`, and as the exported env var `$dir`.

## Decide fast

| Question | Answer |
|---|---|
| Per-item work? | Fan out. Give the tool a single-item entry point |
| Does the command iterate or read these files as its work item? | Yes → `gather`, and name them with `$<in>`. No → `seal` |
| Want it cached? | Give it a source **or** a seal. Either mints the key |
| Want it to always run? | Give it neither a source nor a seal |
| Does a tool's version change committed output? | `tools NAME` declaration + `seal` |
| One-off determinant not worth naming? | Quoted inline operand: `seal "config/*.toml"` |
| Should `cook test` run it every time? | Yes → recipe with `test {}`. No → chore |
| Needs to block something? | Make that something depend on it |
| Reading another recipe's output? | Reference `$<producer>`; never gather its path |
