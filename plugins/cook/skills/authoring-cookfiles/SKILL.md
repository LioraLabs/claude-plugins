---
name: authoring-cookfiles
description: Use when writing or editing a Cookfile for the cook build system — adding recipes, chores, cook steps, tests, probes, seals, or placeholders — or when a Cookfile fails to parse, a recipe name is ignored, an edit to a source file rebuilds nothing or rebuilds everything, a test runs serially that could fan out, a chore silently does nothing, or work needs to be gated behind passing tests.
---

# Authoring Cookfiles

## Overview

Cook's own docs teach the language: **`document.md`** in the cook repo (The Cook Manual — first recipe, placeholders, connecting recipes, tests, chores, caching) and `standard/src/content/docs/*.mdx` for normative detail. Read those for *how it works*.

This skill is the part the docs don't teach: how to **shape** the graph, and the traps that fail **silently or with a misleading diagnostic**.

A Cookfile that parses and builds can still be badly shaped. The shaping decisions below are what separate a Cookfile that caches and parallelises from one that merely works.

## Shaping the graph

Four decisions, in the order they matter.

### 1. Fan out by default

If the work is per-item, make it per-item. A `test` body that references `$<in>` becomes **one cached unit per item**; a body that doesn't is one unit over the whole set.

```cook
recipe conformance: parser-lib
    ingredients "//standard/conformance/positive/*/Cookfile"
        "//standard/conformance/negative/*/Cookfile"
    test { node scripts/check.mjs --case $<in> }
```

284 fixtures, measured: one batched unit took 9.1s and re-ran entirely on any corpus edit. Fanned out it is 0.85s cold across cores, and editing one fixture re-runs **one** unit while 283 stay cached.

Batch only when the tool genuinely needs the whole set in one process (a linter that reports cross-file duplicates). "The harness already loops" is not a reason — give it a single-item entry point. Keep a `chore` for the whole-set view when a human wants one answer.

Fan-out demands a **shared** setup artifact. If each unit would build the same thing, hoist it into its own recipe with a declared output and depend on it — otherwise you pay for it N times.

### 2. Seal what determines, declare what drives

`ingredients` is the **driver**: what the command iterates or consumes. Everything else that changes the answer is a **determinant**, and belongs in a probe you `seal`. §22 says it outright — `files` is "the idiomatic way to name a set of files as a sealable determinant when they are not the consuming recipe's iteration driver."

```cook
probe claim:sites
    files { "VERSION" "src/lib.rs" "grammar.js" }

recipe claim-in-sync
    ingredients "scripts/check-claim.sh"   # drives
    seal claim:sites                        # determines
    test { scripts/check-claim.sh }
```

Putting determinants in `ingredients` fan-outs them (wrong granularity) or makes the command line lie about what it consumes. Putting drivers in a seal loses the fan-out entirely.

### 3. Cacheability is derived from the source — a seal never mints a key

**A seal narrows reuse of a key that already exists; it never creates one.** A sourceless test sealed on ten probes still runs on every invocation. If you want it cached, give it a source (`ingredients`, or a dependency's outputs). If you want it to always run, give it no source — that is the supported way to write a gate.

This is the single most common wrong mental model: `seal` is not "declare my inputs."

### 4. Tools are determinants

If a tool's version changes its output *and you commit that output*, the tool is part of your source:

```cook
probe generate:tool
    tools { tree-sitter }

recipe generate
    ingredients "grammar.js"
    seal generate:tool
    cook "src/parser.c" { tree-sitter generate }
```

Without this, a machine with a different CLI regenerates tracked files in place and **no key moves to say why** — the diff shows up later as an unexplained dirty tree. `tools { }` folds the resolved binary's content hash.

## Gates are dependency edges, not Lua

A failed `test` unit blocks every unit that depends on it, so a dependency list **is** a gate:

```cook
recipe claim-in-sync
    ingredients "scripts/check-claim.sh"
    test { scripts/check-claim.sh }

chore release version: cook-bin claim-in-sync
    git diff-index --quiet HEAD -- || { echo "dirty tree" >&2; exit 1; }
    > release.cut(version)
```

Red gate → `skipped (upstream-failed)`, body never runs, non-zero exit. Do not re-implement this as `if` statements in a module: Lua gates can't be run individually, aren't cached, don't appear in `cook test`, and capture their output instead of streaming it.

## Recipe or chore?

**Any recipe containing a `test {}` step is automatically a `cook test` root. There is no opt-out.** That single fact drives the choice:

| Put it in a **recipe** (`test {}`) | Put it in a **chore** |
|---|---|
| Asserts a property of the tracked tree | Asserts working-state (clean tree, credentials present) |
| Safe and cheap to run on every `cook test` | Re-enters `cook test`, or duplicates the suite |
| Should gate other work | Mutates things (commit, tag, push, install) |

Two consequences learned the hard way: a working-state assertion as a test recipe turns `cook test` red during ordinary editing; and a test recipe that shells out to `cook test` recurses forever — or, scoped, makes a bare `cook test` run the suite **twice concurrently**, where the two runs fight over shared build directories.

## Chore bodies

- **Each line is its own process.** Shell variables do not persist between steps. Multi-step shell belongs in a script file the chore invokes.
- Shell steps **stream** output and abort the chore on non-zero. Lua steps calling `cook.sh` **capture** it — and a large failure can be truncated. Prefer shell steps for anything whose failure you need to read.
- Chores are excluded from `cook test` fan-out. That exclusion is the lever for "must run, must not be a test root."
- Keep the Cookfile declaring units and edges; push imperative shell into scripts. A pipeline assembled with `table.concat` in Lua is a smell.

## Recipe bodies are not Lua scripts

A recipe body admits only these, indented:

- `ingredients "glob"` / `ingredients probe_key`
- `cook "out" { shell }` or `cook "out" >{ lua }`, with optional `seal`/`unseal`/`local|pinned|nondet` tail
- `test { … }`
- `<module>.<fn>(...)` — a bare module call
- `#` comments at **column 0**

Everything else is rejected. Both of these are real messages:

```
execute-phase `>` Lua is not allowed in a recipe body (CS-0134)
loose shell commands are not allowed in a recipe body (CS-0134): `local x = 1`
```

No `local`, no bare Lua statements, no loose shell. Put logic in a `register` block, a chore, or a module. **Chores** are the permissive form — they take bare shell lines and `>` Lua lines.

## Traps

| Symptom | Cause | Fix |
|---|---|---|
| `unclosed shell block (missing closing '}')` — but the `}` is right there | A `#` comment on the same line hides the `}` from the brace scanner | Put `}` on its own line, or drop the comment |
| `cook <name>` runs something else and your recipe never builds | The name collides with a CLI subcommand — the full set is `init`, `menu`, `list`, `modules`, `test`, `dag`, `logs`, `cache`, `serve`, `emit-lua`, `affected`, `why`, `help` (check `cook --help`) | Rename, or invoke as `cook +<name>`. Cook prints a one-line notice that is easy to miss |
| Editing a header rebuilds nothing | Only **declared** inputs count. The core DSL does no `#include` scanning | Declare them with a `files` probe + `seal` — see below |
| Two recipes never order against each other despite one writing what the other reads | A literal path equal to another recipe's output creates **no** edge (§10.6) | Name the recipe: `$<other>` in a step, or a `: other` dep-list entry |
| `clean` chore runs, next build is still instant | Outputs re-materialize from the content-addressed store (`~/.cache/cook`), which `rm -rf build` does not touch | This is correct. For a genuinely cold build you must invalidate keys, not delete outputs |
| `read-after-write with no ordering edge: 'B' … does not require 'A'` — but `recipe B: A` is right there | B declares one of A's **outputs** as its own `ingredients`. The "does not require" clause is **false** and sends you hunting the wrong thing | Drop the produced path from `ingredients`; reference the producer instead (`: $<A>` in the body). Keep only genuine sources as ingredients |
| A chore in an imported member does nothing — `0 nodes`, exit 0, body never runs | It declares a parameter. Any parameterised chore below an `import` is silently dropped (shell **and** Lua bodies alike) | Put parameterised chores in the **root** Cookfile. Verified bug as of 2026-07-26 |
| `files: expected a brace glob list … on one line` | Brace lists (`files { }`, `tools { }`) must be one physical line. `ingredients` and `cook` output lists *do* continue across indented lines — the asymmetry is easy to trip on | Collapse to one line, or use fewer, broader globs |
| A member's recipe can't glob a sibling member's tree | `../` escapes the member root and is rejected | Anchor at the workspace root: `ingredients "//other-member/**/*.txt"` |
| `ingredients <probe>` on a `files` probe errors with `unexpected symbol near '@'` | The `@files-manifest` sentinel leaks to the Lua VM on the source path (it is intercepted on the seal path) | Use the probe as a `seal`, not as an `ingredients` source. Verified bug as of 2026-07-26 |

## Declaring inputs that never appear on the command line

The commonest case of *seal what determines* (above): headers found via `-I`, config read
at runtime, templates. Inputs the command never names, so nothing can infer them — the
core DSL does no `#include` scanning.

```cook
probe headers
    files { "src/*.h" }

recipe lib
    ingredients "src/*.c"
    seal headers
    cook "build/obj/$<in.stem>.o" { cc -Isrc -c $<in> -o $<out> }
    cook "build/lib/libx.a" { ar rcs $<out> $<in> }
```

Editing any matched header gives `rebuild (seal changed)` on every sealed unit. Adding or
removing a matched file counts too — the probe's value is a map of path to content hash.

`seal` as its own recipe-level step applies to every step in the recipe; as a trailing
modifier (`cook "..." { … } seal headers`) it applies to that step alone. The command line
stays clean either way, which is the whole point.

## `$<file:>` is for when you *do* want the paths

`$<file:glob>` expands to the space-joined match list **and** folds the contents into the
key. Both, by design. It is for passing files to a command — not for silent input
declaration. Appending it to a compile line to "declare" headers produces:

```
cc: fatal error: cannot specify '-o' with '-c' ... with multiple files
```

which looks like a compiler problem and is not. If you need the fold without the argument,
use the sealed `files` probe above.

## Ordering comes from names, never paths

A **name reference** creates the cross-recipe edge and puts the referent in the build closure — no dep-list entry required. `$<other>` in a step body is enough. Writing `"build/other/thing.a"` as a literal is not, however exactly it matches.

## Verify by running

A Cookfile that parses can still be wrong in the way that matters — under-declared inputs build clean and rebuild nothing. After writing one:

1. `cook <target>` cold, then again — second run must be fully cached.
2. Touch each kind of input (source, header, data file) and confirm the *expected* units rebuild.
3. For a fan-out, edit **one** item and confirm exactly one unit re-runs. `N-1 cached` is the assertion.
4. For a gate, break it on purpose and confirm the dependent reports `skipped (upstream-failed)` and never runs its body.

Steps 2–4 are the ones people skip, and they are the ones that catch the silent failure. "It ran and passed" proves nothing about keys.

`cook why <recipe> --level unit` prints a unit's key, its folded inputs, its sealed probe values, and `last ran because:` — reach for it the moment caching surprises you.

Tests are content-keyed like everything else: a cached test prints nothing and reports
`cached`. That looks like a broken test. To watch one actually run, change something that
changes the artifact's bytes.

A recipe with only a `test` step — no `ingredients`, no `cook` — is legal and useful.

## Quick reference

| Placeholder | Means |
|---|---|
| `$<in>` | this unit's input. **Re-binds down the step chain** — in a later step it is the previous step's collected outputs, which is what makes compile→archive work |
| `$<out>` | this unit's output. Parent directories are created for you; no `mkdir -p` |
| `$<in.stem>` | basename without extension — drives fan-out |
| `$<out_1>`, `$<out_2>` | multi-output steps (body must be a block) |
| `$<recipe>` | another recipe's terminal outputs, space-joined; creates the edge |
| `$<file:glob>` | expands to the matched paths **and** folds them into the key — for passing files, not for silent declaration |
| `$<env.NAME>` | env var, explicitly namespaced |
| `//glob` | in `ingredients` / `files`, anchors at the **workspace root** — how a member reaches a sibling member |

Chore parameters are **positional**: `chore fmt dir="src"` is invoked `cook fmt src`. Inside the body they bind three ways: as a Lua local, as `$<dir>`, and as the exported env var `$dir`.

## Decide fast

| Question | Answer |
|---|---|
| Per-item work? | Fan out. Give the tool a single-item entry point |
| Does the step iterate these files? | Yes → `ingredients`. No → `files` probe + `seal` |
| Want it cached? | Give it a source. A seal alone will not do it |
| Want it to always run? | Give it no source |
| Does a tool's version change committed output? | `tools { }` probe + `seal` |
| Should `cook test` run it every time? | Yes → recipe with `test {}`. No → chore |
| Needs to block something? | Make that something depend on it |
| Reading another recipe's output? | Reference `$<producer>`; never list its path in `ingredients` |
