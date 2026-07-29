---
name: authoring-cook-modules
description: Use when writing or modifying a Lua module for the cook build system — a maker that registers work units, a probe, or anything called from inside a recipe body — or when module-registered work runs serially, rebuilds nothing, or fails at execute phase with a missing-file error.
---

# Authoring cook modules

## Overview

The reference is **`cook-modules/docs/authoring-guide.md`** (module anatomy, `cook.add_unit` fields, probes, seals, naming) plus Standard chapters 12, 22, 24, and 28. Read those for the API.

This skill is what still bites after reading them — all of it observed, not theorised.

## Check the runtime before you write a line

**The Standard documents APIs the installed binary may not have.** Writing to the spec will send you to functions that don't exist, and you'll find out at register time with `attempt to call a nil value`.

Enumerate the actual surface first:

```cook
register
    local names = {}
    for k in pairs(cook) do names[#names+1] = k end
    table.sort(names)
    print("cook.* = " .. table.concat(names, " "))
```

Run it with the binary you're targeting. `cook --version` reports a Standard version, but that is a claim about the language, not a guarantee that every section of §22 is implemented in that build. Names beginning `__` are private.

The check cuts both ways: the binary also REMOVES APIs (CS-0185 took `cook.add_test` and
`suite`), and a module calling a removed one dies at register time on every current-engine
workspace — masked, cruelly, by cached register replay until an input edit re-mints
(an open engine bug as of 2026-07-29). Two habits keep you honest:

- **A spec stub must be as strict as the engine, not more forgiving.** When the engine
  removes an API, make the stub RAISE on it (with the CS number), and make the stub enforce
  the replacement's register-time rules (e.g. `step_kind = "test"` forbids `suite` and
  outputs). A stub that still captures removed calls keeps the whole suite green on code no
  engine will run — busted 63/63 proves the module agrees with your stub, nothing more.
- **Smoke-test against a live engine before publishing.** Copy the changed files over a real
  consumer workspace's vendored `cook_modules/` tree and run its `cook check`. Register-time
  rejections that no stub models (field validation, payload shapes) only surface there.

## Adjacent `add_unit` calls run SERIALLY

The most expensive default in the system, and it fails silently — the build is correct either way, just N times slower.

```lua
-- WRONG: four units in a depth-4 chain. Measured 4.06s for 4x `sleep 1`.
for _, src in ipairs(sources) do cook.add_unit({...}) end

-- RIGHT: one step group, units run in parallel. Same work, 1.03s.
cook.step_group(function()
    for _, src in ipairs(sources) do cook.add_unit({...}) end
end)
```

Bare `add_unit` calls are sequential per §15.1. That same rule is what orders a fan-out's *aggregate* step (archive, concat, link) after the group — register it bare, outside the group, and the barrier is free.

Nested `step_group` is **unspecified**. A maker that always opens one cannot safely be called from inside another's group; the only protection is convention.

## `inputs` is a cache fold, not an ordering edge

Listing another recipe's output path in `inputs` creates **no** edge (§10.6 — cook never infers a dependency from path-string equality). It folds that path into the cache key and nothing more.

Ordering comes only from **naming** the other recipe. Get this backwards and the build fails at execute phase with a shell error (`cannot stat`, `No such file`), with no register-time diagnostic pointing at the real cause.

Which naming API depends on what you want, and on your runtime:

| Want | Call |
|---|---|
| Read the referent's export | `cook.import(name)` — forces the referent's body (Standard v0.15+) |
| Ordering + whole-recipe barrier | `cook.require_recipe(name)` |
| Ordering on one unit, no barrier | `cook.dep_order(name)` — Standard v0.15+ (cook 0.8.0). Absent in 0.7.x |

Do not use "did `cook.import` return non-nil?" as a proxy for "is this dependency wired
up?" Reading an export is not an edge — on v0.15+ the import even forces the body, so the
read reliably succeeds while nothing orders your unit against the referent, and the build
still breaks at execute time. Ordering is always a separate declaration.

A recipe nothing names never enters the DAG at all — it is not built, not queued, not mentioned. So a missing edge shows up as `No such file or directory` from your command, not as a race.

## Consuming an export needs a second maker

A recipe body is **not** a Lua scope. It admits maker calls and `cook "..." { }` steps, nothing else:

```
parse error: loose shell commands are not allowed in a recipe body (CS-0134):
`local info = cook.import("atlas")`
```

So a consuming recipe cannot unpack your export inline. Ship the consumer side as its own maker (`mymod.report{ from = "atlas" }`) that does the `import` internally. Plan the module's surface around this — it is not optional.

## Two names, two roles

Inside a maker:

- `cook.recipe_name()` → **qualified** name. This is the export identity; pass it to `cook.export`.
- `recipe.name` → **bare** declared name. This drives human-facing paths (`build/obj/<name>/`).

`recipe` is a global the engine binds into the caller's dynamic scope — no signature declares it.

Guard against being called outside a recipe body, which is what every shipped maker does:

```lua
local ok, id = pcall(cook.recipe_name)
if not ok then error("[mod.maker] must be called inside a recipe block", 2) end
```

## Probes are the determinant surface — for tools AND files

Anything that changes your output but never appears in `inputs` is a determinant, and a
sealed probe is how you declare it:

- **Tools.** A `tools` probe over the binaries you invoke. Skip it and a toolchain upgrade
  invalidates nothing — silent staleness, the worst failure class here.
- **Files the command never names.** A `files` probe over the glob. §22 calls this "the
  file-set twin of `tools`, and the idiomatic way to name a set of files as a sealable
  determinant when they are not the consuming recipe's iteration driver" — headers found
  via `-I`, templates, config read at runtime.

Both reach the key the same way: `cook.add_unit{ probes = {...}, seal = {...} }`. A change
then reports `rebuild (seal changed)`.

Do not reach for path-expanding placeholders to declare a non-argument input — that puts
the paths on the command line, which is a different job.

## Verify, don't infer

A passing build does not tell you *why* it passed. Two mechanisms that happen to agree look identical from the outside. To separate them:

**Bust keys by changing content, never by deleting directories.** Artifacts live in a
user-global content-addressed store (`~/.cache/cook/cloud/`) that survives `rm -rf .cook build`
and restores by key. Delete a directory and your "cold" run comes back `0.00s, all cached`,
which reads as a broken experiment. Rewrite an input's bytes instead, and pass `--no-publish`
so throwaway timings don't land in the user's shared store.

- **Parallelism:** `sleep 1` in each fan-out unit, and nonce the inputs so both runs are
  genuinely cold. Serial N units take N seconds; grouped they take one.
- **Ordering:** deleting a `require_recipe` only proves something if the dependent unit
  actually *reads* the referent's output at execute time. If the dependency is only a cache
  fold, removing the edge breaks nothing and you will wrongly conclude it was never needed.
  Construct the read deliberately.
- **Invalidation:** touch each input class and confirm the expected units rebuild.
